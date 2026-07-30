---
name: code-review
description: Code-review phase of the fsai-dev pipeline. General adversarial review of a wave or branch diff hunting logic bugs, data-clobbering, race conditions, permission gaps, and edge cases that named-rule checkers cannot see. Review mode only, reports findings with concrete failure scenarios, never fixes. Runs as a subagent given only a branch or diff, or inline with refutation discipline when the executor cannot spawn subagents. Use when asked to review a diff for bugs, or when invoked by the implement phase.
version: 0.2.0
---

# Code-Review Phase

Adversarial review of changed code. The rule checkers (arch-check, design-system-check) enforce named rules; this phase hunts the bugs no rule names: logic errors, data-clobbering, races, permission gaps, broken edge cases. It found nothing if it only found style.

## Phase Contract

- **Phase id**: `code-review`
- **Inputs**: a branch or diff (branch vs master by default). Optionally `spec.md` / `plan.md` from the run dir for intent; without them, infer intent from the diff and say so in the verdict.
- **Artifacts**: a verdict block in the pipeline manifest: `pass`, or numbered findings, each with file:line, a concrete failure scenario, and a suggested fix.
- **Exit criteria**: verdict written; every finding resolved or explicitly waived in the Decision Log.
- **Default gate**: autonomous. Findings block the implement phase's exit.

Subagent-safe: executable from only the inputs above, no conversational context. When the executing agent cannot spawn subagents (it is itself a fork or subagent), run the review inline instead, after implementation is complete. Inline mode has anchoring risk (reviewing code you wrote): re-derive intent from the diff alone, not from memory of writing it, and apply the refutation step to every finding without exception. State `reviewed inline (subagent unavailable)` in the verdict header so the manifest records the mode.

## Method

1. **Scope**: `git diff master...<branch>` (or the provided diff). Review changed code and the call sites it touches; do not audit the whole repo.
2. **Read as a hostile reviewer.** For each changed function or query, actively try to construct a failure: What input, state, or interleaving makes this do the wrong thing? Who else writes this data? What happens on the second call, the concurrent call, the empty list, the deleted row, the other tenant's id?
3. **Findings need failure scenarios.** A finding is a concrete story: "when X and Y, this produces Z". No failure scenario means no finding. Style opinions, naming preferences, and refactor ideas are out of scope; drop them.
4. **Verify before reporting.** Trace each suspected bug through the actual code paths (callers, guards upstream, DB constraints). If a guard elsewhere already prevents it, it is not a finding. If an adversarial-review skill is available in the session, apply its refutation discipline: attempt to refute each of your own findings before it goes in the verdict.
5. **Hunt list**, in priority order:
   - Writes that clobber fields other actors own (partial-update endpoints overwriting unrelated columns)
   - Missing tenancy/permission fences on new queries or endpoints
   - Races and ordering: concurrent mutation, fire-and-forget side effects, retry duplication
   - Edge inputs: empty, null, zero, duplicate, stale id, deleted parent
   - State machines: transitions that skip guards or leave orphaned state
   - Error paths: swallowed errors, partial failure leaving inconsistent data
6. **Verdict format**:

       code-review: 3 findings
       1. apps/backend/src/api/deals/dealsService.ts:142: CRM sync overwrites rep-owned
          fields. Scenario: rep edits stage while sync in flight; sync PUT clobbers it.
          Fix: field-level merge, only write CRM-owned columns.
       2. ...

   `pass` when the hunt produced nothing that survived verification; say what was hunted so pass is meaningful.

## Rules

- Review mode only: never edit code. Findings go to the implement phase (pipeline) or the user (standalone).
- Report at most the findings that matter; ten weak findings bury two real ones.
- If the diff is too large to review honestly in one pass, say so and review by file group; never silently sample.
