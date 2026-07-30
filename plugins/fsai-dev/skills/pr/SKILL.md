---
name: pr
description: Create the pull request for a completed pipeline branch, delegating to the pr-description skill when installed. Use as the pr phase of an fsai-dev pipeline run.
version: 0.2.0
---

# PR Phase

Open the pull request for the implemented, tested branch.

## Phase Contract

- **Phase id**: `pr`
- **Inputs**: implemented and tested branch (implement and testing phases done in `pipeline.md`); `plan.md` for the summary and the not-included list. Stop and report if the branch has failing checks or unclosed waves.
- **Artifacts**: PR URL(s) recorded in the `pipeline.md` phases table (one per wave in `pr-train` delivery).
- **Exit criteria**: PR open; CI-relevant checks passing locally (`yarn tsc:all`, `yarn lint:all`, the test commands the plan names).
- **Default gate**: `approve` (pre-merge).

## Method

1. Verify the branch is clean: local checks green, no uncommitted work that belongs in the PR.
2. Audit what is staged for the PR before pushing:
   - Never commit plan or spec documents; run-dir artifacts (`.agent/pipelines/`) stay local.
   - In fsai, never commit migration `_journal.json` or snapshot files.
3. If a `pr-description` skill is available in the session, invoke it via the Skill tool and follow it. It owns description quality; this phase supplies the plan and branch context.
4. Otherwise follow the target repo's conventions. In fsai:
   - Title: `<type>(<pkg>): <description>` with types `feat`/`fix`/`refactor`/`tooling` and package abbreviations `bp` (brand-dashboard), `ap` (applicant-portal), `bd` (backend), `sdk`; omit the package for cross-cutting changes.
   - Body: use `.github/PULL_REQUEST_TEMPLATE.md` and the `docs/pr-descriptions.md` standard: summary says what, not how; screenshots for UI changes; testing checklist; a "not included" section for deferred work (pull this from the plan's decision log and unshipped waves).
5. Open the PR with `gh`, record the URL in `pipeline.md`, then stop at the gate: link the PR, state what was verified, list anything deferred, and ask for pre-merge approval.

## pr-train mode

When the run's Delivery line is `pr-train`, this phase is invoked once per wave by implement, not as the terminal phase:

- Scope the title and description to the wave (its scope statement and exit criteria), not the whole plan.
- Target the plan's declared integration surface as the base branch.
- Append the PR URL to the pr row in `pipeline.md`; the row holds one URL per wave.
- The input precondition relaxes to "this wave's exit criteria passed"; other waves may still be open.
- The pre-merge approve gate applies per PR unless the run's gate table says otherwise.
