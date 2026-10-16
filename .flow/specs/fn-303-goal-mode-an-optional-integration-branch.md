# Goal mode: an optional integration branch

## Goal & Context
<!-- scope: business -->

Goal mode lands each child spec on the default branch, one reviewed change at a time, and measures the bar there. That is the right default. Some goals cannot show the default branch a half-finished state: a migration or port where half-moved code cannot ship, a version upgrade where the default branch has to keep building on the old version until the switch, a release bar that only means something as a whole. For those, the children should land on an integration branch and reach the default branch through one final pull request. Daniel Killenberger raised it while reviewing goal mode. [paraphrase]

Target user: anyone running `flow --goal` on a goal whose intermediate states must not reach the default branch. [inferred]

## Architecture & Data Models
<!-- scope: technical -->

- **Opt-in in the goal spec.** The person writes it into the goal spec, for example under Boundaries: "children land on the goal branch; the default branch changes only through the final pull request". The run reads that sentence like any other part of the goal. No flag, field or config key. [paraphrase]
- **The goal spec's `branch_name` is the integration branch.** Every spec already carries one. The run creates that branch from the default branch's remote head once, pushes it, and records it in the goal log. [paraphrase]
- **Children build on it.** Each child's work branches from the integration branch and its pull request targets it, so each child stays one reviewed change against the integration branch. Work gains a `--base=<ref>` argument that sets the base its branch fence already reads (`DEFAULT_BASE`); the goal run passes it, and make-pr gets the matching `--base`. Land merges into whatever base the pull request has, unchanged. [inferred]
- **Measure on it.** Goal mode's measure step uses the integration branch's remote head instead of the default branch's. [paraphrase]
- **Keep it current.** Before each measurement the run merges the default branch's remote head into the integration branch and pushes the merge (never a rebase or force-push). A conflict it cannot resolve within the goal's Boundaries is an open question for the person. [paraphrase]
- **The final pull request.** At `GOAL_MET` the run opens one pull request from the integration branch to the default branch and gives it a full implementation review of the whole diff. Its body is built from the goal log. It lands only with merge consent on the current invocation, as any other pull request does. Whether it opens through make-pr for the goal spec or directly is the implementer's call, decided by what make-pr already supports for a spec with no tasks. [inferred]
- **Unchecked children are not green.** When pull requests into the default branch report checks and a child's pull request into the integration branch reports none, the run stops `NEEDS_HUMAN`, naming the likely cause: the repository's CI triggers only on pull requests into the default branch. [paraphrase]
- **Docs.** The goal-mode page gains an "Integration branch" section: when to use it, the sentence to put in the goal spec, and that the repository's CI must run on pull requests into that branch (for GitHub Actions, the branch or a pattern such as `goal/**` under `pull_request: branches`, and in branch protection if the checks should block merges). [paraphrase]
- **Parity.** Skill and reference prose plus work's one argument, through the existing mirrors; no flowctl change. [inferred]

## Edge Cases & Constraints
<!-- scope: technical -->

- Without the sentence in the goal spec, nothing changes: children land on the default branch as today. [paraphrase]
- The integration branch already exists (a resumed goal): the run uses it and does not recreate it. It exists but was not created by this goal, per the goal log: `NEEDS_HUMAN`. [inferred]
- The default branch cannot be merged in cleanly and the conflict touches a Boundary or a product choice: open question; the run keeps working on gaps that do not depend on it only when the integration branch still builds. [inferred]
- The final pull request is closed without merge: `NEEDS_HUMAN`; the run never reopens or replaces it. [inferred]
- `--until=merge` lands children into the integration branch and, only with that same invocation's consent, the final pull request. [inferred]

## Acceptance Criteria
<!-- scope: both -->

- **R1:** A goal spec that states children land on the goal branch makes the run create the integration branch from its `branch_name` (once), build every child from it, open every child's pull request against it, and measure on its remote head. A goal spec without that statement behaves exactly as before. Errors: an existing branch the goal log does not record as the goal's → `NEEDS_HUMAN`.
- **R2:** Work accepts `--base=<ref>`, which sets the base its branch fence uses; without it the base resolves as today. Errors: an unknown or unfetchable ref → `BLOCKED` before any branch is created.
- **R3:** Before each measurement on an integration branch, the run merges the default branch's remote head into it and pushes the merge, never rebasing or force-pushing. Errors: an unresolvable conflict → open question in the goal log.
- **R4:** At `GOAL_MET` on an integration branch, the run opens one pull request from it to the default branch with a body built from the goal log, runs a full implementation review on it, and lands it only under the invocation's merge consent. Errors: the pull request is closed unmerged → `NEEDS_HUMAN`.
- **R5:** When pull requests into the default branch report checks and a child's pull request into the integration branch reports none, the run stops `NEEDS_HUMAN` naming the CI trigger as the likely cause. No error surface beyond that stop.
- **R6:** The goal-mode docs page has an "Integration branch" section covering when to use it, the goal-spec sentence, and the CI and branch-protection configuration it needs; the plugin's skills reference names work's `--base`. No error surface.

## Boundaries
<!-- scope: business -->

- No flowctl verb, spec field or config key. [paraphrase]
- No change to goal mode without the goal-spec statement. [paraphrase]
- No rebase, force-push or history rewrite of the integration branch or any child branch. [paraphrase]
- No CI configuration changes in the user's repository; the run reports, the person configures. [inferred]

## Decision Context
<!-- scope: both -->

Landing every child on the default branch keeps each change small and checked by the default branch's own CI, so it stays the default. An integration branch trades that for keeping intermediate states off the default branch, at the cost of drift (handled by merging the default branch in before each measurement) and one large final review. A question at the start of a goal run was rejected because goal runs are unattended; the person states the choice in the goal spec instead. The goal spec's `branch_name` serves as the integration branch so no new field is needed, and the final pull request answers #559's open question of whether a goal gets a pull request of its own: it does only in this mode. [paraphrase]

Credit: suggested by @DanielKillenberger while reviewing goal mode (#559). Delivery: before 9.0.0, so goal mode ships with it. [paraphrase]

## Strategy Alignment

Serves the autonomy track; follows Agent first (one sentence in the goal spec, no new field or flag beyond work's base argument, which names the unattended failure it prevents: children built from the wrong base while their pull requests target the integration branch) and Consent comes from a person (the person chooses the mode in the goal spec; the final merge still needs merge consent on the invocation). [inferred]

## Strategy Conflicts

None found. [inferred]
