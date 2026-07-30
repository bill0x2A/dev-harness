# The Pipeline Manifest

One manifest per task, created by the conductor before any phase runs. It is the single source of truth for a run: any session (or any teammate's session) must be able to resume the run from the manifest alone.

## Location

`.agent/pipelines/<yyyy-mm-dd>-<task-slug>/` in the **target repo** (the repo being worked on, not this plugin). The manifest is `pipeline.md`; phase artifacts live beside it (`research.md`, `spec.md`, `plan.md`, ...).

The conductor must ensure `.agent/pipelines/` is in the target repo's `.gitignore` before writing anything. Pipeline runs are local working documents, not committed history (they can be shared by copying the run directory). If the team later decides to commit them, flipping that is a one-line change here — phases never assume either way.

## Format

`pipeline.md` follows this template:

```md
# Pipeline: <task title>

Task: <one-paragraph statement of the task, in the user's words plus any clarifications>
Started: <date>  Branch: <branch name>
Status: proposed | running | blocked(<phase>) | done | abandoned

## Phases

| # | Phase | Gate | Status | Artifact |
|---|-------|------|--------|----------|
| 1 | research | notify | done | research.md |
| 2 | grill | approve | done | spec.md |
| 3 | plan | approve | in-progress | plan.md |
| 4 | implement | autonomous | pending | — |
| 5 | arch-check | autonomous | pending | verdict below |
| 6 | backend-testing | autonomous | pending | — |
| 7 | pr | approve | pending | PR URL |
| — | design-sync | — | skipped: no UI in this task | — |
| — | e2e | — | skipped: v2, no suite exists yet | — |

## Decision Log

- <date> <decision and why — every gate response, every mid-run course change>

## Verdicts

- arch-check: <pass | N findings (resolved/waived, listed)>

## Surprises & Discoveries

- <anything that changed the plan, with evidence>
```

## Rules

1. **Statuses**: `pending`, `in-progress`, `built`, `done`, `skipped: <reason>`, `blocked(<reason>)`. `built` means code exists but is unverified — the normal state of delegated subagent output. Verification is a distinct act: `built` moves to `done` only when the phase's exit criteria are actually checked and hold, never on the builder's claim alone. A phase is `done` only when its exit criteria (see its contract) hold.
2. **Every phase in the catalog appears** — either in the pipeline or in the skipped rows. Silent omission is the failure mode this document exists to prevent.
3. **Update at every phase boundary**, and mid-phase at any stopping point. If the session dies, the manifest is the handoff.
4. **Gate responses go in the Decision Log** verbatim-ish ("Bill approved plan with change: drop the Slack notify wave").
5. **Resume protocol**: on `/fsai-dev:feature` invocation, if a run directory for this task already exists, read `pipeline.md`, reconcile against reality (does the branch exist? do artifacts exist? is tsc green?), and continue from the first phase that is not `done`/`skipped` — do not re-propose the pipeline unless the task statement changed. Reconciling means downgrading statuses too: work recorded as `done` whose exit criteria no longer hold goes back to `built` or `in-progress`.

## Program trackers

A pipeline run is task-sized. Multi-workstream efforts (launch sprints, epics) get a **program tracker**: a separate living document (conventionally `.agent/plans/<date>-<program>.md` in the target repo) that owns the workstream list, cross-run ordering and critical path, program-level decisions, and open questions. Runs and trackers link both ways:

- Each workstream item that becomes real work spawns its own pipeline run; the run manifest carries a `Program: <path to tracker>` line under the task statement.
- The tracker records the run directory next to the item it spawned, and rolls up each run's status using the same status vocabulary.

The conductor executes one run at a time. Ordering *between* runs ("A4 must land before B starts") lives in the tracker, and the conductor checks the tracker's ordering constraints as an input precondition when a manifest declares a `Program:` line.
