# Wasted calls on the hot paths (audit 2026-10 wave 2)

## Goal & Context
<!-- Goal & Context: 20% [paraphrase], 80% [inferred] -->

The same monthly audit that produced wave 1 (asked to find what "should be trimmed without regressing, efficiency gains") found tool calls, API calls and re-reads whose output nothing uses or that re-derive a value the run already holds. Each costs a round trip, context or wall time on every run that reaches it. The largest is the deps skill, which runs one `flowctl show` per spec in the repository when only open specs need it; on this repository that is about 290 calls instead of about 20 (minutes instead of seconds).

Every item was re-checked line by line by the host on Gordon's request for full confidence. Items whose saving rests on a behaviour or contract change were withdrawn to fn-260; what remains has identical outcomes: the same files written, the same verdicts, the same gates.

## Architecture & Data Models

No new components. Changes edit existing skill text, references and agent prompts. Every value a removed call produced is already in hand from a call the run keeps: the specs list, the run's preflight snapshot, or the same command's earlier output.

## Edge Cases & Constraints

- A removed call is removed only when its value is already in hand at that point on every path. [inferred]
- Fail-open probes keep failing open (a missing or unparsable snapshot value still opens the gate). [inferred]
- Tests assert behaviour or a minimal token (G2); fences tests execute keep passing. [inferred]
- Verification for the whole wave: the full unit suite and Ruff pass once at the end and the Codex mirror check passes; each criterion below states the before/after comparison that shows its outcome is unchanged, recorded in the PR with the call counts. [inferred]

## Acceptance Criteria

- **R1:** The deps skill reads full spec detail only for specs that are not done, taking id, title and status for every spec from the specs list; its ready and blocked lines, phases and output are identical. Errors: a spec whose detail read fails is reported as today. Verified by running the skill's gather and both jq steps on this repository before and after: the ready, blocked and phase output is byte-identical and the `flowctl show` count drops to the number of open specs. [inferred]
- **R2:** Review skills drop calls whose output nothing reads: completion review's per-backend spec read (Phase 0 already validated the spec), the repository-root lookups set and never read in impl-review, plan-review, completion review and sync (two of them are Bash calls that do nothing else), and impl-review's CLI workflow step that recomputes a diff base the setup resolved and prints a log nothing uses. Errors: no error surface beyond the kept calls'. Verified by one completion review on a CLI backend against a fixture spec before and after (same verdict, receipt carries the same fields) and by the existing review fence tests. [inferred]
- **R3:** impl-review's optional phases dispatch deep passes and validation through one backend-generic arm plus the host arm instead of four identical per-backend arms. Errors: an unknown backend still fails as today. Verified by a test that runs the collapsed dispatch arm with a stub flowctl for each of the four CLI backends and sees the same `<backend> deep-pass` and `<backend> validate` invocations the four arms produced. [inferred]
- **R4:** Capture's glossary scan takes the glossary from its preflight snapshot instead of listing it again, capture's strategy alignment takes strategy status from the snapshot (it still reads the strategy body), and refine's doc-aware pass takes the glossary from its preflight bundle. Errors: a missing or unparsable snapshot value opens the gate (fail open), as today. Verified by the planning-preflight and tracker caller-execution tests, plus one capture and one refine run on a fixture repository with a populated glossary and strategy: the same glossary proposals and strategy tags as before, with the removed calls absent from the transcript. [inferred]
- **R5:** (withdrawn 2026-10-31 after line-by-line review) The plan-sync single-read saved one call per wave at the cost of trusting a value across the run; not worth it. Work's double read of spec and task bodies moved to fn-260 as a measurement. [inferred]
- **R6:** resolve-pr's detached push path reads the head repository, owner and branch in one pull-request call instead of two. Errors: a failed read stops the push as today. Verified by a test that runs the push fence from a detached checkout against a stub `gh` (the land stub pattern) and asserts the push target URL and ref, and that `gh` was called once. [inferred]
- **R7:** Prime's workflow scout drops the two GitHub calls its output has no slot for (recent runs, PR bodies); prime's feedback-latency check keeps its own run listing. Errors: no error surface. Verified by one prime `--report-only` run on this repository before and after: the Pillar 8 workflow rows and the feedback-latency line are unchanged. [inferred]
- **R8:** (withdrawn 2026-10-31 after line-by-line review) Reading the spec only on a fix overrides plan review's written mandatory re-anchor; moved to fn-260 as a decision. [inferred]

## Boundaries

- Outcomes do not change: same files written, same verdicts, same gates, same questions. [inferred]
- No moves of instruction text between files and no description or prompt-template changes; those belong to fn-260. [inferred]
- Kept after review: capture's three gate probes stay at the steps they gate; refine's second specs read stays (it runs after the write-back and can differ); impl-review's pointer to the deep-pass templates stays. [inferred]

## Decision Context

These are call-level waste with a value already in hand, so equivalence can be checked from the code without an eval. The review withdrew four items whose saving needs a decision or a measurement instead: work's double body read on the default path (the largest default-path saving, about 7-10 KB, but it reorders anchoring), plan review's spec read, prime's switch to the emitter's environment cross-reference (the skill never mentions those fields, so the host's check would change), and resolve-pr's cross-repository URL handling (every gh call in the skill assumes the current repository; the fix is a contract choice). All four are in fn-260. [inferred]

## Strategy Alignment

flowctl already renders these values once; re-deriving them is the hand-assembly last month's "flowctl renders what agents hand-assemble" wave removed, and "flowctl grows only under burden of proof" holds: no new verb is needed. [inferred]
