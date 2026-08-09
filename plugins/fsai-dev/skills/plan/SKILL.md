---
name: plan
description: Author an ExecPlan from a sharpened spec or task statement, structured as independently verifiable waves. Use as the plan phase of an fsai-dev pipeline run, or standalone when asked to write an ExecPlan.
version: 0.5.0
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
5. Declare the delivery mode with a one-line reason: `single-pr` (default, one terminal PR for the run) or `pr-train` (each wave ships as its own PR). A pr-train plan also picks the substrate:
   - `independent`: name the integration surface (master or an integration branch) and which waves ship as which PRs.
   - `stacked` (GitHub stacked PRs via `gh stack`): favor when waves form a dependency chain (shared file ownership, each building on the last); name the stack order explicitly. Disjoint waves ship as independent PRs either way; a mixed graph gets one stack for the chain plus independent PRs for the disjoint waves. Preview gate: do not assume stacked works until the repo has accepted its first stack server-side; the plan names `independent` as the fallback.
   Record mode and substrate on the manifest's Delivery line; changing the conductor's initial mode gets a Decision Log entry.

## Waves

The one addition this phase makes on top of PLANS.md: every milestone is an independently verifiable wave.

Each wave states:

- Scope: what exists after this wave that did not before.
- Exit criteria: exact commands to run and the expected results (test commands, `yarn tsc:all`, an HTTP transcript, a behavior a human can observe). "Compiles" alone is not an exit criterion.
- Dependencies on earlier waves, if any.

A wave must be closeable on its own: the implement phase runs its exit-criteria commands and either closes the wave or blocks. If a milestone cannot be verified independently, split or reorder it until it can.

### Parallel waves

A wave whose tickets are independent may declare `mode: parallel`. It must then also declare:

- **File ownership per ticket.** Name the contested files explicitly and give each exactly one owner (e.g. "3827 is the sole `Sidebar.tsx` owner; 3831 must not touch it"). A ticket needing another ticket's file moves to a later sub-wave.
- **Sub-wave ordering with the reason**, when one ticket builds something others consume ("2a builds the mismatch-notice component; 2b's three consumers run after").
- **The integration verification battery**: the commands run on the merged integration branch that close the wave. Individual branches passing is only `built`; the integration battery is the exit criterion.

## On completion

Write `plan.md`, update `pipeline.md` (phase row, Decision Log for any choices made while planning), then stop at the gate: summarize the waves and the key decisions, and ask for sign-off. Do not commit `plan.md` to git; run-dir artifacts stay local.

## Writing style

Write all prose in ASD-STE100 Simplified Technical English: short sentences (20 words for an
instruction, 25 for a description), active voice, one instruction per sentence, simple tenses, one
meaning per word, no noun clusters over three words, keep the articles, no em dashes, no idiom or
filler, vertical lists for steps and findings. Keep code identifiers, file paths, commands, and
error strings exact. Code and code comments follow the target repo's conventions, not this section.
Full rules: `${CLAUDE_PLUGIN_ROOT}/docs/phase-contract.md` section "Writing style".
