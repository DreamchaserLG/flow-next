# Goal mode: flow --goal builds toward a goal spec's bar

## Goal & Context
<!-- scope: business -->

`flow --auto` drives one ready spec to a pull request. It has no answer for "reach this outcome", where the outcome is known and checkable but the set of specs that gets there is not and only shows up by measuring, building and measuring again. Examples: a platform port (an app's macOS build ported to Linux), a migration off a legacy system driven by an AI-native target spec, a vision document or PRD worked down into shipped changes, a release bar of budgets and owner sign-offs. [paraphrase]

Today users run that work under a hand-written loop prompt around `flow --auto` ("mint specs for clear gaps, only stop when stuck"). It works for a while and then drifts: on long runs the host stops using flow, workers get hand-written briefs instead of work, minted specs skip review, a change merges without implementation review. The prompt carries the whole contract, so nothing holds the session to the stages, and the next tick cannot read what the last one measured or decided (#559: a release goal on a Rust/TypeScript generator; the same pattern drove a desktop app's macOS-to-Linux port). [paraphrase]

Goal mode makes that a flow-next primitive with minimal machinery: a goal is an ordinary spec whose acceptance criteria are the bar, and `flow --goal <anything>` captures the goal when it is not yet a spec, measures against the bar, mints the next spec for the largest gap, and builds it through the ordinary `flow --auto` pipeline, repeating until the bar is met or a person is needed. The goal spec is the storage that keeps a two-day run on track: every run re-reads it, and the run appends what it measured, minted and decided to it. [paraphrase]

Target user: anyone using flow-next in factory mode toward an outcome larger than one spec (a port, a migration, their own PRD or vision document, a release bar), who today drives `flow --auto` by hand-written loop prompts. [inferred]

## Architecture & Data Models
<!-- scope: technical -->

- **A goal is an ordinary spec.** No new spec kind, field or flowctl verb. Its Goal & Context states the outcome or links whatever describes it; its acceptance criteria are the bar; its Boundaries hold what no child may change. A person writes it, captures or charts it as today, or hands `--goal` the source and lets the run capture it. [paraphrase]
- **`--goal` takes whatever flow takes.** A spec id is the goal. Anything else (a document path, a URL, a tracker issue, pasted text) is captured first as the goal spec, unattended, through capture `from:goal` with what was given as its source, as attended `flow <idea>` routes intent to capture. The first goal-log entry records the source, and a later run given the same source resumes that goal instead of capturing it again. A source with no checkable outcome stops `NEEDS_HUMAN` asking for the bar, with nothing written. [paraphrase]
- **The goal spec is not marked ready.** The `--goal` invocation is the person's consent for the whole goal. A ready goal would be selected by a plain `flow --auto` and built as one direct spec; a goal run that finds its goal ready unreadies it and records that in the goal log. [inferred]
- **`--goal` implies `--auto`.** It is parsed with the other flow tokens; `--until=merge`, `--review=<backend>` and `--tick` pass through with their existing meaning; the routing block, review backend, QA gate, plan-vs-no-plan rule, strikes and dirty guard apply unchanged because each child runs through the ordinary `flow --auto` hop loop. The goal-only instructions live in a gated flow reference loaded only when `--goal` is present, as backlog mode's are; a run without `--goal` loads nothing new. [paraphrase]
- **The goal log.** The goal run keeps a `## Goal log` section at the end of the goal spec, written after acting and never before: dated entries for each measurement (criterion, observed result, the commit it ran on), each child minted (id, the criterion it serves, why it was the next gap), each child outcome (PR, merge, strike, retirement), each default the run chose, and each open question for the person (options, the measured cost of each, a recommendation). The person answers an open question by editing the goal spec; the next run reads the answer and records it as settled. The run never edits any other section of the goal spec. [inferred]
- **One goal tick.** Read the goal spec and its log. If a child the goal minted is still open, drive it with the ordinary hop loop. Otherwise measure every criterion on the default branch, append the result, and either end `GOAL_MET`, mint the next child and drive it, or stop for the person. One child is open at a time, so each measurement sees the previous child's merged effect. [inferred]
- **Measuring.** A criterion counts as met only on an observed result recorded with its commit: a command's output, a test, a benchmark, a merged child's receipts. A criterion only a person can judge (an owner sign-off, a visual verdict) becomes an open question, never a self-graded pass. [inferred]
- **Minting a child.** The run picks the gap it judges most valuable next, settles an uncertain premise first by a throwaway probe under the existing prototype-before-ask rules (scratch location, nothing staged), then mints the child through capture with the exact token `from:goal <goal-id>`. The child is sized for one pull request, so the shared plan-vs-no-plan rule usually records the direct (no-plan) route; a child that needs a plan gets one. The run adds the child as a dependency of the goal (`spec add-dep <goal> <child>`), marks the child ready, and drives it through normal routing: plan or no-plan, plan review, QA and every other gate follow the same rules as for a spec a person wrote. [paraphrase]
- **Capture under `from:goal`.** For the goal itself, its source is what `--goal` was given; for a child, the goal spec on disk and the run's recorded gap evidence. Either way the source is on disk or in the invocation, not conversation turns: it quotes the source where the user's turns would be quoted, carries the goal's Boundaries and settled decisions into the child, never asks (autofix semantics with the write authorized), and applies plan-vs-no-plan as `from:flow` does. The compaction refusal does not apply, because the source is on disk. A child that would need a question, would cross a goal Boundary, or duplicates an open spec is not minted: the run records an open question (or, for a ready duplicate, drives that spec as the child) instead. [inferred]
- **Closing a child the goal no longer needs.** The run may retire a child it minted that never merged when a probe falsified its premise or another child covers it (`spec close --retire superseded|moot --by <id>`), recording the reason. It never retires or edits a spec it did not mint. [paraphrase]
- **Verdict.** Every goal run ends with the ordinary `PILOT_VERDICT` line. `spec=` names the child the run worked on, or the goal when no child was dispatched; the reason starts with `goal <goal-id>; `. One verdict value is added: `GOAL_MET`, for a run that measured every criterion met. A driver keeps re-invoking on `ADVANCED` and stops on `GOAL_MET`, `NEEDS_HUMAN` and `BLOCKED`, as it does today. [inferred]
- **Driving it for days.** A host loop (`/goal` where the host has it, `/loop`, a stop hook, CI) re-invokes `flow --goal <id> --until=merge` until `GOAL_MET` or a stop. Nothing is resumed from a transcript: a crashed or compacted run resumes from the goal spec, the child specs and their receipts on disk. [paraphrase]
- **Parity.** The change is skill and reference prose plus capture's token, so it reaches every supported host through the existing mirrors; `scripts/sync-codex.sh` regenerates the Codex copy. No flowctl change. [inferred]

## Examples
<!-- scope: both -->

A goal spec (abridged):

```markdown
# Linux port reaches parity with the macOS app

## Goal & Context
Port the app to Linux. Parity reference: the shipped macOS build. Target design: docs/linux-port.md.

## Acceptance Criteria
- **R1:** `make test-linux` passes on Ubuntu 24.04 and Fedora 41 in CI.
- **R2:** Cold start under 800 ms on the reference laptop (`scripts/bench-start --json`, field `cold_ms`).
- **R3:** The owner approves the tray menu, notifications and global shortcuts on GNOME and KDE.

## Boundaries
- The macOS build and its release pipeline stay unchanged.
- At most 12 minted specs; stop and ask before more.
```

Invocations:

```text
/flow-next:flow --goal docs/linux-port.md     # captures the goal spec (fn-40) first, then as below
/flow-next:flow --goal fn-40                  # unattended; ends at each child's PR, a person merges
/flow-next:flow --goal fn-40 --until=merge    # factory mode; lands each child, re-measures, mints the next
# host loop (wording per host): re-run /flow-next:flow --goal fn-40 --until=merge until GOAL_MET or a stop
```

A goal log after a few runs (shape illustrative; the run writes it in prose it chooses):

```markdown
## Goal log
- 2026-10-20 measured on a1b2c3d: R1 fails (12 Linux test failures, all in tray/), R2 not run (no Linux build), R3 open.
- 2026-10-20 minted fn-41 for R1 (tray backend on libappindicator); probe: a scratch build links on Ubuntu 24.04. fn-41 merged as #212.
- 2026-10-21 measured on d4e5f6a: R1 met; R2 1,140 ms > 800 ms.
- 2026-10-21 open question for the owner: Wayland global shortcuts need the portal (works on GNOME 46+ and KDE 6) or an X11-only fallback. Recommendation: portal. Blocks R3.
```

Terminal lines:

```text
PILOT_VERDICT=ADVANCED spec=fn-41 stage=work+make-pr+land reason="goal fn-40; merged #212 for R1"
PILOT_VERDICT=NEEDS_HUMAN spec=fn-40 stage=- reason="goal fn-40; R3 needs the owner: Wayland shortcut route (see goal log)"
PILOT_VERDICT=GOAL_MET spec=fn-40 stage=- reason="goal fn-40; R1-R3 met on 9f8e7d6 (log has each result)"
```

[inferred]

## Edge Cases & Constraints
<!-- scope: technical -->

- **Without `--until=merge`** the run ends when the child's PR opens, as `flow --auto` does; the next run finds the open PR and defers to land (`DEFERRED_TO_LAND`) until it merges, then measures. Children never stack on unmerged children. [inferred]
- **Runaway minting.** The goal spec may state a budget or a stop date in prose and the run honours it. Independently, two consecutive merged children that moved no criterion stop the run `NEEDS_HUMAN` with the log, so a run that is not converging cannot mint indefinitely. A child that advances nothing still strikes out exactly as today. [inferred]
- **Depth one.** A minted child is never itself run as a goal; only the goal run mints, and only for its own goal. [paraphrase]
- **Open questions do not stop other work.** A question that blocks one criterion parks that criterion; the run continues on gaps that do not depend on the answer and stops `NEEDS_HUMAN` only when nothing else is workable. [paraphrase]
- **Settled stays settled.** An answer the person wrote, and Decision Context in the goal spec, are context for every minted child and its reviews; a child that reopens one is a review finding, not a choice. [paraphrase]
- **Goal-log edits are `.flow` bookkeeping.** They travel the way other `.flow` bookkeeping does, onto the next child's branch with its spec files; the dirty guard ignores them. A final entry with no next child is reported in the run's output. [inferred]
- **Probes stay out of the tree.** Probes use scratch locations under the prototype-before-ask rules; a probe that leaves non-`.flow` changes trips the existing dirty guard. [inferred]
- **One run per goal.** Two concurrent goal runs on one goal can mint for the same gap; the docs say so, and nothing locks it. [inferred]
- **`--backlog` with `--goal`.** Goal mode selects only the goal's children; backlog config and the `--backlog` flag do not apply and the flag gets the existing unknown-combination notice. [inferred]
- **Invalid goal.** `--goal` with no argument, a spec id that does not resolve, a closed goal, or a source that names no checkable outcome stops `NEEDS_HUMAN` before any write. [inferred]

## Acceptance Criteria
<!-- scope: both -->

- **R1:** `/flow-next:flow --goal <spec id or any source>` runs unattended as `--auto` does and passes `--until=merge`, `--review=<backend>` and `--tick` through with their existing meaning; the goal-only instructions load from a gated reference only when `--goal` is present, and a run without it behaves exactly as today. Errors: no argument, an unresolvable spec id, a closed goal → `NEEDS_HUMAN` before any write; `--backlog` with `--goal` → notice, backlog not applied.
- **R1a:** Given a source that is not a spec id, the run captures the goal spec from it through capture `from:goal`, records the source as the first goal-log entry, and continues; a later run given the same source resumes that goal rather than capturing a second one. Errors: a source with no checkable outcome → `NEEDS_HUMAN` asking for the bar, nothing written.
- **R2:** Each goal run reads the goal spec and its `## Goal log`, drives an open child it minted through the ordinary `flow --auto` hop loop (every stage, gate, strike and guard unchanged), or measures every criterion on the default branch and records each result with its commit in the goal log. A criterion only a person can judge is recorded as an open question, never as met. No error surface beyond the hop loop's own.
- **R3:** When a measured gap remains, the run mints at most one child at a time through capture `from:goal <goal-id>`, adds it as a dependency of the goal, marks it ready, routes it exactly as a person-written spec is routed, records the mint and its reason in the goal log, and drives it. A premise the run cannot settle by a scratch probe, a gap whose route is a product choice, or a child that would cross a goal Boundary becomes an open question instead of a mint. Errors: capture refuses or would ask → no mint, open question recorded.
- **R4:** Capture accepts the exact token `from:goal <goal-id or source>`: when capturing the goal, its source is what `--goal` was given; when minting a child, the goal spec on disk plus the run's gap evidence. It never asks, the compaction refusal does not apply, it carries the goal's Boundaries and settled decisions into a child, and it applies plan-vs-no-plan to a child as `from:flow` does. A user invocation of capture without the token behaves exactly as today. Errors: unknown goal id or unreadable source → exit without writing.
- **R5:** The run ends `GOAL_MET` only when every criterion has an observed result in the goal log showing it met; it stops `NEEDS_HUMAN` on an open question that blocks all remaining work, on a budget the goal spec states, and after two consecutive merged children that moved no criterion. The goal spec is left open for the person to accept and close. No error surface beyond these terminals.
- **R6:** Every goal run ends with one `PILOT_VERDICT` line whose reason starts with `goal <goal-id>; `, `spec=` naming the child worked on or the goal; `GOAL_MET` is added to the verdict values, and existing drivers keyed on `ADVANCED`, `NEEDS_HUMAN`, `BLOCKED`, `NO_WORK` and `DEFERRED_TO_LAND` keep working. No error surface.
- **R7:** The run may retire, with a recorded reason, only a child it minted that never merged; it never edits a goal section other than `## Goal log`, never edits a spec it did not mint, and never changes the goal's criteria, Boundaries or budget once the goal spec exists, including a goal spec it captured. No error surface beyond the existing retire command's.
- **R8:** Attended flow recognises a spec or document that states an outcome reached over several specs whose path is unknown (for example a port, a migration, a PRD) through one routing row, and recommends `flow --goal` with the same input. No error surface.
- **R9:** Docs describe goal mode where users look for it: a new flow-next.dev page under Autonomy (what a goal spec holds, the log, the run, the verdicts, driving it for days, the use cases: platform port, migration from a target spec, PRD or vision doc to shipped changes, release bars of budgets and sign-offs), with the navigation entry and updates to the going-autonomous, driving-a-loop, flow --auto and flow skill pages and the cookbook; the repo's skills reference, the per-host reach pages, GLOSSARY.md (goal, goal log), and STRATEGY.md's autonomy track (the consent boundary for goal mode is the `--goal` invocation; the run authors child specs only inside a named goal). The remaining downstream properties are walked at release. No error surface.
- **R10:** A test run on a small throwaway repository, started from a short goal document (not a spec id) with two or three criteria (at least one measured by a command and one that only a person can judge), captures the goal and reaches `GOAL_MET` through at least two minted children under `--until=merge`, every child taking the route normal routing gives it and showing work, implementation review and make-pr in its receipts; the run's transcript and goal log are summarised in the PR. Errors: a minted child routed differently from how the same spec written by a person would be, a skipped stage, or `GOAL_MET` with an unanswered person-judged criterion fails this.

## Boundaries
<!-- scope: business -->

- No new flowctl verb, spec field, spec kind, store under `.flow/` or config key. [paraphrase]
- No change to the one-spec `flow --auto` pipeline, its gates or its verdicts other than the added `GOAL_MET` value. [paraphrase]
- No parallel children, no cross-repository goals, no scheduler, budget accounting or dashboard; those belong to the orchestration layer above flow-next if they are ever needed. [inferred]
- The run never edits the goal's bar, Boundaries or budget once the goal spec exists, captures a goal only from a source the person handed to `--goal`, and never mints outside that goal. Prospect and chart stay attended. [paraphrase]
- No change to backlog mode. [inferred]

## Decision Context
<!-- scope: both -->

### Motivation
<!-- scope: business -->

Users already run this pattern by hand with loop prompts, and it drifts on long runs because the prompt is the only thing holding the stages. Putting the loop in the flow skill makes every child pass through the same pipeline as a human-written spec, and putting its state in the goal spec lets a multi-day run resume from disk. The maintainer's direction: use the spec as the storage that keeps two-day runs on track, keep machinery minimal, mint specs and build them with `flow --auto` (`--until=merge`) under normal routing, let all normal config (routing block, review backend) apply for free, and let `--goal` take whatever attended flow takes. [paraphrase]

Credit: @DanielKillenberger proposed goal mode and described the real case in #559. [paraphrase]

### Implementation Tradeoffs
<!-- scope: technical -->

#559 proposed more structure; each piece was weighed against STRATEGY.md "Agent first" (G4) and kept only where it prevents a named unattended-trust failure. [inferred]

- **Kept as guidance or a record:** the goal as an ordinary spec, the `depends_on` link, a capture token (as `from:flow` already is), the goal log written after acting, open questions answered in the spec, probes under the existing prototype-before-ask rules, the no-progress stop (prevents unbounded minting, and catches a child that misread the goal, since its criterion does not move), one verdict value (drivers need a stop signal distinct from `NO_WORK`). [inferred]
- **Rejected:** a `spec set-goal` verb and a `minted_by` field (the `--goal` argument and the goal's dependencies already identify the goal and its children); a `.flow/goals/<id>.json` measurement store (the goal log in the spec is durable, reviewed in PRs and read by the next run); a `--budget-specs` flag and budget accounting (a budget the person writes in the goal spec is honoured as written); a measured-or-verdict schema for bar items (the run judges how to measure each criterion and records the evidence; anything only a person can judge becomes a question); a `GOAL_STUCK` verdict and tick counters (`NEEDS_HUMAN` with the log covers it); forcing `review.skipAllowed` off, or a design review before work, for minted children (normal routing and config apply; the next measurement and the no-progress stop catch a child that misread the goal); a scheduled cut-and-simplify review checkpoint (a goal that wants one states it as a criterion); a `spec-completion-review` of the goal (every criterion is met only on recorded evidence, and each child already passes its own completion review). [inferred]
- **Later, outside flow-next:** parallel children, goal dashboards, approval surfaces for person-judged criteria, cross-repository goals and spend budgets are orchestration concerns for a control plane such as MergeFoundry, which already projects `.flow/` state and can read the goal log and child specs without new flow-next contracts. [inferred]
- **One open child at a time** keeps each measurement attributable to one merged change and avoids stacked unmerged branches; rolling work already parallelises inside a child. [inferred]
- **`--goal` accepts any source** because attended flow already routes any intent to capture; restricting it to spec ids would make the person capture by hand what the run can capture from the same source. The bar stays the person's because it comes from the source they named and is never edited afterwards. [paraphrase]
- **Goal spec not ready, `--goal` as consent** is the smallest way to stop a plain `flow --auto` from building a whole vision document as one spec without adding a field. [inferred]

## Strategy Alignment

Serves the **autonomy** track: it extends the dial from "one ready spec" to "an outcome a person named", with the same pipeline and gates at every child. Follows **Agent first**: no flowctl verb, field, store or schema; the run decides the next gap and records its measurements, mints and decisions afterwards in the goal log; the person is asked only for the bar, product calls and person-judged criteria. Moves the consent boundary for this mode from each spec's ready flag to the named goal, and STRATEGY.md records that change (R9). [inferred]

## Strategy Conflicts

STRATEGY.md's autonomy track says the driver never captures and backlog mode never authors a spec. Goal mode captures the goal spec from a source the person handed it and authors child specs, but only inside that goal and under its bar and Boundaries; R9 updates the track to say so. [inferred]
