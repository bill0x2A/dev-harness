---
name: implement
description: Execute an approved ExecPlan wave by wave, delegating code changes to subagents and verifying each wave against its exit criteria plus architecture and design-system checkers. Use as the implement phase of an fsai-dev pipeline run.
version: 0.1.0
---

# Implement Phase

Execute the approved plan wave by wave. The orchestrator plans, verifies, and integrates; subagents write the code.

## Phase Contract

- **Phase id**: `implement`
- **Inputs**: approved `plan.md` with waves (approval recorded in the pipeline Decision Log). Stop and report if the plan is missing or unapproved.
- **Artifacts**: code on the branch; per-wave status updates in `pipeline.md`; checker verdicts in the manifest Verdicts section.
- **Exit criteria**: all waves done; each wave's exit-criteria commands ran green; `yarn tsc:all` and `yarn lint:all` green; checker findings resolved or waived in the Decision Log.
- **Default gate**: `autonomous` overall; each wave completion is a `notify`.

## Method

For each wave, in plan order:

1. Mark the wave in-progress in `pipeline.md`.
2. Delegate the code changes to subagents. Do not write feature code inline: split the wave into parcels, brief each subagent with the plan context it needs, run independent parcels in parallel. Brief subagents to reuse existing code (search for prior art first) rather than duplicating logic inline.
3. Review and integrate the returned changes. You own coherence across parcels; subagents do not see each other's work.
4. Run the wave's exit-criteria commands from the plan. In fsai, `yarn tsc:all` and `yarn lint:all` are part of every wave's exit criteria regardless of what the plan says.
5. Run applicable checkers as parallel subagents: `arch-check` for waves touching backend code, `design-system-check` for waves touching frontend code. Both, if the wave touches both.
6. Resolve checker findings, or waive them with a reason in the Decision Log. A wave does not close with unresolved, unwaived findings.
7. Close the wave: update `pipeline.md`, tick the plan's progress checklist, post the notify summary (what shipped, what the checks said), continue to the next wave.

## Rules

- Do not verify UI by driving the user's running browser. Verification is tsc, lint, and tests; the user checks visuals.
- Exit-criteria failures are fixed, not narrated. Fixing red tests and type errors is in-scope autonomous work.
- Plan deviations (a wave needs splitting, an approach does not survive contact with the code) go in the plan's Decision Log and Surprises sections as they happen.
- Cannot meet a wave's exit criteria after honest attempts: mark the phase `blocked(<reason>)` in `pipeline.md` and surface it. Never close a wave with caveats buried in prose.
