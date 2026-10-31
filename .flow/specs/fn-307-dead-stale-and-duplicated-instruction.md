# Dead, stale and duplicated instruction text (audit 2026-10 wave 1)

## Goal & Context
<!-- Goal & Context: 20% [paraphrase], 80% [inferred] -->

The monthly maintenance pass asked to "go through every single skill and path to see what should be trimmed without regressing, efficiency gains". Seven read-only auditors covered every shipped skill, agent, reference, command shim, rule and template at 9.4.0 and measured what each path reads. The default fast paths (single-task work, impl-review through a CLI backend, make-pr, land) are already lean. The waste sits in text no agent can act on: pointers to sections that moved, rules contradicted by the code or by another file on the same path, history and outside-source names that shipped skills must not carry, files nothing reads, and rules restated two to four times across files that always load together.

This wave removes that text and fixes the small defects found with it. It changes no behaviour: every rule an agent follows today is still stated once, on the same path, after the change. Moves of text to files a run might not read, call reductions, and anything that changes what a model sees by default are out of this wave (wave 2 and fn-260).

## Architecture & Data Models

No new components. Every change edits an existing skill, reference, agent, command shim, rule, template or the Codex mirror generator. Canonical files are edited and the Codex mirror is regenerated, never hand-edited. Exact locations are re-derived by grep at build time, since line numbers drift.

## Edge Cases & Constraints

- A duplicate is removed only when the kept copy is read on every path that read the removed one; when one copy sits in a gated reference and the other on the common path, the common-path copy stays. [inferred]
- No text moves to a file the agent might not read on that path; "moved behind a link" is a behaviour change and belongs to fn-260. [inferred]
- Lines that block a known failure (a "has broken this" line, an explicit prohibition) stay unless the failure is impossible in code, shown by the code itself. [inferred]
- Tests assert behaviour or a minimal token (G2). Fences and tokens that tests execute or pin stay intact: work's tracker probe fences, the work-branch and plan-sync-inputs fences, plan-sync's strategy-drift sentences, the triage-skip and TRIAGE_DISABLED fences, host-review reservation and record fences, prime's run_bounded definition, setup's single Step 0 fence and HITS fence, the features verdict grammar, capture's mode-detection block and gated-reference links, refine's preflight bundle, the measured-numbers linker set, flow.md naming flow-next-flow, and the byte-identical FLOWCTL resolution comment the mirror generator checks. [inferred]
- Removed or corrected prose leaves no dangling reference elsewhere (grep before delete). [inferred]

## Acceptance Criteria

- **R1:** Flow's references describe 9.4.0 goal mode correctly: the PR-boundary reference no longer sends a goal run to a "measure step" that no longer exists, and auto.md and the backlog reference stop citing phases, passes, setup sections and PR probes that no longer exist (including the gh allowlist naming probes the host no longer runs, and backlog claims about who sets `noPlan` and who owns its verdict grammar). The QA-stage reference loses the shell probe it tells the reader never to run. Errors: no error surface; the flow routing, merge-destination, unattended-fence and backlog mirror-safety tests keep passing. [inferred]
- **R2:** The Codex mirror's prime scout-dispatch sentence is rewritten for Codex again: the generator's pattern matches the canonical sentence as it now reads, and the regenerated mirror shows the Codex wording. Errors: no error surface beyond the existing mirror check. [inferred]
- **R3:** Argument loops in resolve-pr and QA word-split under zsh as well as bash, using the repository's established portable form. Errors: an empty argument list still yields the default mode. [inferred]
- **R4:** Contradictions between files on one path are resolved to the behaviour the code implements: the comment resolver's cluster-mode output matches what resolve-pr parses; plan-review's host workflow has no empty dispatch heading and defines its receipt path before first use; completion review states one receipt rule (every verdict writes one), names the repair command its sentence promises, lists every backend in its re-review step, and leaves backend resolution to flowctl instead of an incomplete priority list; audit applies Update and Delete through `memory apply` only and drops its "if Phase 5 pushed" branch; the multi-task reference no longer calls rolling dispatch blocking; work's judge-tier condition, the worker's test rule, repo-scout's code-block budget, sync's list command, prospect's statuses and section references, and features' flowctl-reuse note each say one thing. Errors: no error surface; each corrected line matches the code path it describes. [inferred]
- **R5:** Every internal pointer resolves: the shared tracker gating predicate is cited where it lives, prime's map-suggestion and summary pointers, audit's harden-flow, scan and report pointers, memory-migrate's consent-gate name (and its broken opening sentence), impl-review's backend-specs claim about what is already loaded, tracker status-sync's step pointer, the spec-template-discovery reference's consumer list, map's version-range source and current version, the memory README template's optional fields and migrate command, the Cursor rule's reinstall advice, and the command shims' argument hints (claude and host reviewers, `--no-plan`, `--update`). The spec-template doc stops saying refine tags the person's words. Errors: no error surface; a link check over the changed files finds no target missing. [inferred]
- **R6:** Shipped text carries no flow-next history and no outside-source names: version archaeology, maintainer notes, eval and incident stories, issue and task numbers, "moved from" notes, token-saving claims, references to the development repository's own instruction files, and the named outside sources in prospect and strategy are removed, keeping each rule they accompany. Errors: no error surface. [inferred]
- **R7:** Files nothing reads are deleted (the two unused memory entry templates), and impl-review's workflow-common file, whose every rule is stated elsewhere on the same paths, is folded away: its spec-form examples join the backend-specs reference and the reads that pointed to it are dropped. Errors: no error surface; impl-review's host and flag paths still reach every rule they reached before. [inferred]
- **R8:** Rules stated more than once on one path are stated once, keeping the copy on the path where it acts, across flow and auto, work and the worker and plan-sync and sync, the review skills and quality auditor, resolve-pr and its resolver, QA and drive, capture, plan, refine, chart, prospect and strategy and the planning scouts, prime and its scouts, setup and features, and audit, memory-migrate, tracker-sync, deps, visual, the base skill and the command shims. Errors: no error surface; each removed copy has a kept copy on every path that read it. [inferred]

## Boundaries

- No behaviour change: no change to routing, gates, verdicts, consent, review scope, what gets written, posted or asked, or which stages run. [inferred]
- No text moves to a file read on fewer paths, no call reductions, and no description or prompt-template changes; those are wave 2 and fn-260. [inferred]
- The work skill's reviewer-overlap text stays with fn-260 R3, which already owns its removal. [inferred]
- Decisions this audit raised for the maintainer (the `FLOW_AUTONOMOUS` wording conflict, the clawpatch version range, retiring the removed-backend notices, dead `flow --auto` flags, stale tool versions prime writes into user repositories) are not settled here. [inferred]

## Decision Context

Last month's audit shipped as waves that each landed on review alone, with eval-gated items parked in fn-260; this wave follows that split. Text that is dead, contradicted or duplicated on one path can be judged from the code and the files, so review is enough. Moving text behind a link is not: earlier proximity and layout studies showed a reachable rule is not proof the agent still applies it, so those moves wait for an eval. The audit measured about 1-11 KB off each affected path from this wave, more on impl-review's host and flag paths (about 17 KB); the larger load wins (goal path, tracker push, prime gating) are fn-260 items. [inferred]

## Strategy Alignment

"Agent first" says machinery a model no longer needs is deleted, and "Remember the bitter lesson" says compensating scaffolding rots into cost; restated rules and dead pointers are that cost in prose form. [inferred]
