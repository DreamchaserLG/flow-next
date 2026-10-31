# Wasted calls on the hot paths (audit 2026-10 wave 2)

## Goal & Context
<!-- Goal & Context: 20% [paraphrase], 80% [inferred] -->

The same monthly audit that produced wave 1 (asked to find what "should be trimmed without regressing, efficiency gains") found tool calls, API calls and re-reads whose output nothing uses or that re-derive a value the run already holds. Each costs a round trip, context or wall time on every run that reaches it. The largest is the deps skill, which runs one `flowctl show` per spec in the repository when only open specs matter; on this repository that is 292 calls instead of about 20 (about two minutes instead of a few seconds).

This wave removes those calls with identical outcomes: the same files written, the same verdicts, the same gates. It also fixes resolve-pr fetching review threads from the current repository when given a pull request URL for another one.

## Architecture & Data Models

No new components. Changes edit existing skill text, references, agent prompts and one resolve-pr script. Every value a removed call produced is already available from a call the run keeps: the run's preflight snapshot, the anchor bundle, the route call, or the earlier read of the same value.

## Edge Cases & Constraints

- A removed call is removed only when its value is already in hand at that point on every path; where a snapshot can be stale by design (capture's fresh tracker check at close), the fresh call stays. [inferred]
- Capture's tracker reads stay as they are: the tracker caller-execution oracle pins them. [inferred]
- Fail-open probes keep failing open after merging (a missing or unparsable snapshot key still opens the gate). [inferred]
- Tests assert behaviour or a minimal token (G2); fences tests execute keep passing. [inferred]

## Acceptance Criteria

- **R1:** The deps skill reads full spec detail only for open specs, taking status for every spec from the specs list; its graph, phases and output are identical. Errors: a spec whose detail read fails is reported as today. [inferred]
- **R2:** Review skills drop calls whose output nothing reads: completion review's per-backend spec read, the repository-root lookups set in five places and read nowhere (including a Bash call that does only that), and impl-review's CLI workflow recomputing the diff base and branch the setup already resolved. Errors: no error surface beyond the kept calls'. [inferred]
- **R3:** impl-review's deep pass reads its pass templates only on the host backend, since flowctl loads them for CLI backends, and its optional phases use one backend-generic dispatch arm plus the host arm instead of four identical per-backend arms; the validate step names every backend that merges its result. Errors: an unknown backend still fails as today. [inferred]
- **R4:** Capture and refine take glossary, strategy and gate values from their preflight snapshot instead of re-running the same probes, and capture reads its three gate sentinels in one pass; refine lists specs once per run. Errors: a missing or unparsable snapshot value opens the gate (fail open), as today. [inferred]
- **R5:** Work reads the plan-sync settings once per wave as one subtree. On the default single-task inline path, work reads the task and spec bodies once, through the anchor bundle, keeping the metadata read the gates need; the no-plan fork, a direct continuation and the multi-task route keep their body reads. Errors: a failed anchor read stops as today. [inferred]
- **R6:** resolve-pr's detached push path reads the head repository, owner and branch in one pull-request call, and a pull request URL for another repository fetches that repository's threads. Errors: a malformed URL fails with a message naming it; a focused test covers the cross-repository URL. [inferred]
- **R7:** Prime's workflow scout drops the two GitHub calls its output has no slot for, and prime's environment-and-setup step consumes the emitter's environment cross-reference and setup stages instead of recomputing them, keeping its judgment step. Errors: an emitter field that is absent falls back as today. [inferred]
- **R8:** Plan review keeps its pre-dispatch checkpoint but reads the spec body only when a fix is needed. Errors: no error surface beyond the fix loop's. [inferred]

## Boundaries

- Outcomes do not change: same files written, same verdicts, same gates, same questions. [inferred]
- No moves of instruction text between files and no description or prompt-template changes; those belong to fn-260. [inferred]
- Capture's tracker reads and its close-time fresh tracker check stay. [inferred]

## Decision Context

These are call-level waste with a value already in hand, so equivalence can be checked from the code and a run, without an eval. Two items sit close to behaviour and get a direct check at build time: work's single read of task and spec bodies (one single-task inline run, compared with today's) and plan review reading the spec only on a fix (one SHIP and one NEEDS_WORK round). The flow driver's run-start brief and its plan-vs-no-plan read when the route call already decided were also found, but each changes what the model sees, so they are fn-260 items. [inferred]

## Strategy Alignment

flowctl already renders these values once; re-deriving them in prose is the hand-assembly the "flowctl renders what agents hand-assemble" wave removed last month, and "flowctl grows only under burden of proof" holds: no new verb is needed. [inferred]
