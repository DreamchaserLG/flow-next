# Decided audit changes, each with its test (audit 2026-10 wave 3)

## Goal & Context
<!-- Goal & Context: 40% [paraphrase], 60% [inferred] -->

The October efficiency audit's line-by-line review flagged changes that could not ship as plain trims, because each changes what an agent reads, asks or accepts. The maintainer decided five of them on 2026-10-31 and asked that each spec "test what it needs to test so we're not flying blind". This spec holds those five, each with the check that must pass before it ships. The sixth flagged question, the comment resolver's cluster output contract, stays parked in fn-260. [paraphrase]

## Architecture & Data Models

No new components and no new flowctl verbs. Changes edit existing skill text (capture, work, plan-review, prime, resolve-pr); comparison runs and their records live in the maintainer's eval repository, never in this one. [inferred]

## Edge Cases & Constraints

- A comparison run holds model, effort and harness constant between before and after, and records its inputs, outputs and the measure it compares; a change whose comparison shows a regression does not ship. [inferred]
- Tests assert behaviour or a minimal token (G2); new tests drive a real fence or path. [inferred]
- The full unit suite and Ruff pass once at the end, and the Codex mirror is regenerated and its check passes. [inferred]

## Acceptance Criteria

- **R1:** Capture asks its must-ask cases the way the working rules say: related questions go in one prompt per round instead of one per turn, each still with its recommendation and confidence tier. Errors: a case whose answer changes which other cases apply is asked in a later round, not guessed. Verified by a small screen on fixture conversations that each trigger two or more must-ask cases: the grouped version asks fewer rounds, every case is still answered, and the saved specs match the before version's on title, criteria and tags. [paraphrase]
- **R2:** On work's default single-task inline path, the task and spec bodies are read once, through the anchor bundle, keeping the metadata reads the gates need; the no-plan fork, a direct continuation and the multi-task route keep their body reads. Errors: a failed anchor read stops as today. Verified by a comparison run of single-task work on fixture tasks (the worker-stage bank or equivalent) before and after: the same tests pass, the task is done with the same evidence, and the transcript shows the bodies read once instead of twice. [inferred]
- **R3:** Plan review keeps its checkpoint before every dispatch but reads the spec body only when a fix is needed, and the fix step reads the current, possibly person-edited spec before editing it. Errors: no error surface beyond the fix loop's. Verified by one SHIP round and one NEEDS_WORK round on fixture specs before and after: same verdicts and receipts, and on NEEDS_WORK a hand edit made to the spec before the fix survives the fix. [inferred]
- **R4:** Prime's environment cross-reference (DE1) and setup-stage (DE4) checks read the values `flowctl prime classify` already emits instead of recomputing them, keeping prime's judgment on top (the undeclared-variable threshold and the documented-config downgrade). Errors: an absent emitter field falls back to today's host check. Verified before the switch by comparing the emitter's values with the host's current results on at least four representative repositories (a monorepo, a single-package app, a library, a repository with no environment template): the switch ships only where they agree, and each disagreement is either fixed in the emitter or keeps that check on the host. [inferred]
- **R5:** resolve-pr refuses a pull request or comment URL whose repository is not the current checkout's, with a message naming both repositories and telling the person to run it from that repository's checkout; it fetches and changes nothing. Errors: a URL it cannot parse fails with a message naming it, as today. Verified by a test that runs the argument fence with a foreign-repository URL against a stub `gh` and asserts the refusal message and that no fetch ran, and with a same-repository URL that it proceeds. [inferred]

## Boundaries

- The comment resolver's cluster output contract stays in fn-260 until decided. [inferred]
- No new configuration keys or flags; resolve-pr gains no option to act on another repository. [inferred]

## Decision Context

The maintainer settled the five on 2026-10-31: capture follows the working rules' grouping ("the intent was grouping multiple questions per round", with a test), and work's double read, plan review's spec read, prime's emitter switch and resolve-pr's repository handling were each approved. For resolve-pr, refusing a foreign repository was chosen over carrying the repository through every call: land and people run resolve-pr from the pull request's own checkout, and acting on another repository's pull request from this checkout would edit the wrong files. [paraphrase]
