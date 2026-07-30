---
name: plan
description: Author an ExecPlan from a sharpened spec or task statement, structured as independently verifiable waves. Use as the plan phase of an fsai-dev pipeline run, or standalone when asked to write an ExecPlan.
version: 0.1.0
---

# Plan Phase

Author an ExecPlan from the spec, broken into waves the implement phase can verify one at a time.

## Phase Contract

- **Phase id**: `plan`
- **Inputs**: `spec.md` from the grill phase, or the task statement from `pipeline.md` if grill was skipped. Stop and report if neither exists.
- **Artifacts**: `plan.md` in the run directory.
- **Exit criteria**: plan meets every requirement of the target repo's `.agent/PLANS.md` (or the fallback skeleton below); every milestone is a wave with runnable exit-criteria commands and expected results.
- **Default gate**: `approve` (plan sign-off).

## Method

1. Read the target repo's `.agent/PLANS.md`. If it exists, follow it to the letter: self-contained, novice-executable, plain language, Progress / Decision Log / Surprises & Discoveries / Outcomes & Retrospective sections, observable outcomes.
2. If the repo has no `PLANS.md`, use this fallback skeleton: purpose (what the user can do after, and how to see it working), milestones as waves, progress checklist, decision log.
3. Read the spec and the code it touches. Name files by full repo-relative path. Resolve ambiguities in the plan; do not outsource decisions to the reader.
4. Structure milestones as waves.

## Waves

The one addition this phase makes on top of PLANS.md: every milestone is an independently verifiable wave.

Each wave states:

- Scope: what exists after this wave that did not before.
- Exit criteria: exact commands to run and the expected results (test commands, `yarn tsc:all`, an HTTP transcript, a behavior a human can observe). "Compiles" alone is not an exit criterion.
- Dependencies on earlier waves, if any.

A wave must be closeable on its own: the implement phase runs its exit-criteria commands and either closes the wave or blocks. If a milestone cannot be verified independently, split or reorder it until it can.

## On completion

Write `plan.md`, update `pipeline.md` (phase row, Decision Log for any choices made while planning), then stop at the gate: summarize the waves and the key decisions, and ask for sign-off. Do not commit `plan.md` to git; run-dir artifacts stay local.
