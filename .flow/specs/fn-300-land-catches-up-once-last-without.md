# Land catches up once, last, without restarting patience

## Goal & Context
<!-- scope: business -->
<!-- Goal & Context: 0% [user], 55% [paraphrase], 45% [inferred] -->

Land brings a pull request up to date with its base (a server-side branch update) as the first thing it does whenever the PR is behind, before it knows whether review, threads, CI and patience would let the PR merge. Two costs follow on a busy repository: [paraphrase]

- **Early catch-ups are wasted.** Land catches up, then waits on a review, a thread or the patience window; meanwhile the base moves again and the PR is behind again, so the next tick catches up again. Each catch-up is a new head and a full CI cycle, and only the last one before the merge ever mattered. [paraphrase]
- **Every catch-up restarts patience.** On an unattended merge, land waits a patience window after the last push so review bots can post. A catch-up is a push, so it restarts the window, although it adds no code the PR's author wrote and the bots already had their window on the actual change. [paraphrase]

Land also needs to know when the repository requires up-to-date branches. That requirement can live in a ruleset rather than classic branch protection; a repository whose only strict rule is in a ruleset reads as unprotected through the classic protection read. [paraphrase]

What gets merged does not change: a PR on a repository that requires up-to-date branches is still caught up and tested before merging, as today. [paraphrase]

Target user: maintainers whose PRs wait on land, and anyone running `flow --auto --until=merge` on an active repository. [inferred]

## Architecture & Data Models
<!-- scope: technical -->

- **Catch up last.** Land evaluates conflicts, threads, CI, review decision and patience first. It catches up only when being behind the base is the only thing between the PR and a merge it is otherwise allowed to make. A conflict, or a refused catch-up, still stops `BLOCKED` naming the branch that needs a rebase, as today. [paraphrase]
- **Patience counts from the last authored push.** A branch update that only merges the base into the PR (no change authored for the PR) does not restart the patience window; the window counts from the last push that changed the PR's own diff. The existing push-evidence fallbacks apply to that push. [paraphrase]
- **One retry, then hand back.** After a catch-up, land waits for the new CI run as today. If the base moved again during that run and the repository's rules require another catch-up, land does it once more; a third consecutive catch-up for the same PR is not attempted, and land stops `RESOLVING` with a reason naming the moving base, so the caller's cadence or a person decides. [paraphrase]
- **Read strictness from rulesets too.** Whether the repository requires up-to-date branches is read from both classic branch protection and the rulesets that apply to the base branch. When neither read succeeds, land behaves as if up-to-date branches are required. [paraphrase]
- **Behind base where the rules allow it:** land still catches up before merging, as today, only now last; it never merges behind the base. [paraphrase]
- **Visible.** The verdict reason says when land deferred a catch-up to the end and when it stopped on a moving base. [inferred]
- **Size.** This reorders land's existing steps and re-anchors one timer; it adds no step, reference file, flowctl verb or state, and keeps land's instruction text within its budget. [paraphrase]

## Examples
<!-- scope: both -->

**Busy base, strict rules.** Today: tick 1 catches up (CI 12 min), tick 2 waits 10 min of patience from that catch-up, the base moves twice meanwhile, tick 3 catches up again, and so on. With this change: tick 1 sees review pending and does not catch up; tick 2 sees review approved, CI green on the authored head and patience elapsed since the author's last push, catches up once, and waits for CI; tick 3 merges. One CI cycle for the base instead of one per tick. [inferred]

**Moving base.** After the catch-up, two more base merges land during CI; the repository requires up-to-date branches, so land catches up a second time; a third move during that run stops it:

```text
LAND_VERDICT=RESOLVING pr=812 reason="base moved during two consecutive catch-up CI runs; not catching up a third time"
```

[inferred]

**Strict rule in a ruleset.** A repository has no classic branch protection, and its "Required CI" ruleset enforces strict status checks. Land reads the ruleset, knows up-to-date branches are required, and catches up last instead of trying a merge that the forge would refuse. [inferred]

**Patience anchor.** The author's last push was 14:02; land's catch-up pushed at 14:20. At 14:21 the patience window (10 min) has elapsed, measured from 14:02, so the only wait is the CI run on the caught-up head. [inferred]

## Edge Cases & Constraints
<!-- scope: technical -->

- A merge-queue repository: the queue tests the merge result; land's existing queued handling is unchanged. [inferred]
- A stacked PR whose base is its parent's branch: catch-up behaviour toward the parent is unchanged; this spec concerns the trunk base. [inferred]
- A catch-up that brings a conflict: `BLOCKED` naming the branch, as today. [paraphrase]
- A push by the PR's author after a catch-up restarts patience as any authored push does. [inferred]
- A human's current merge authorization still waives patience, unchanged. [paraphrase]

## Acceptance Criteria
<!-- scope: both -->

- **R1:** Land catches up a behind PR only when every other gate it applies (conflicts, threads, CI, review decision, patience) would otherwise allow the merge. Errors: conflict or refused catch-up → `BLOCKED` naming the branch, unchanged. [paraphrase]
- **R2:** The patience window counts from the last push that changed the PR's own diff; a catch-up that only merges the base does not restart it. Errors: push evidence unavailable → the existing fallbacks, applied to the authored push. [paraphrase]
- **R3:** Land performs at most two consecutive catch-ups for one PR and then stops `RESOLVING` with a reason naming the moving base. No error surface beyond today's stops. [paraphrase]
- **R4:** Land reads the up-to-date requirement from both classic branch protection and applicable rulesets. Errors: neither read succeeds → treated as required. [paraphrase]
- **R5:** Land's instruction text gains no new step, reference file, flowctl verb or state, and stays within its line budget. No error surface. [paraphrase]
- **R6:** Observed land ticks on this project's own PRs (strict rules in a ruleset) and on a scratch repository with strict rules off show fewer catch-ups and fewer CI cycles per merged PR than before, and no change in what was merged. No error surface beyond the observation. [inferred]

## Boundaries
<!-- scope: business -->

- Land still never rebases, force-pushes or merges without current authorization; every other gate is unchanged. [paraphrase]
- No merging behind the base on the agent's judgment that base changes are unrelated; a repository that wants merge-result testing turns on strict checks or a merge queue, and land honours it. [paraphrase]
- Carrying review verdicts across rebases by patch identity, removed by fn-250, stays removed. [paraphrase]

## Decision Context
<!-- scope: both -->

### Motivation
<!-- scope: business -->

The first version of this spec let land merge behind the base when the agent judged the base changes unrelated. The maintainer rejected that on 2026-10-15: it adds an unattended judgment that can break the base branch (a base change that adds a new caller of code the PR changes is easy to miss), it helps only repositories that already allow behind merges, and it does nothing on a repository like this one, whose strict rule makes every catch-up mandatory. Ordering the catch-up last and anchoring patience on the author's push remove most of the waiting without changing what gets merged. [paraphrase]

Delivery order: 5 of 6 in this set; independent. Route: direct. [paraphrase]

## Strategy Alignment

Serves the **Autonomous mode** track: unattended landing spends less time on repeated catch-ups and restarted patience. Follows the **Agent first** design principle: no new mechanism, a reorder of existing steps.

## Strategy Conflicts

None found.
