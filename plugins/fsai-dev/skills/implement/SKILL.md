---
name: implement
description: Execute an approved ExecPlan wave by wave, delegating code changes to subagents and verifying each wave against its exit criteria plus architecture and design-system checkers. Supports sequential waves and parallel waves across git worktrees with file ownership. Use as the implement phase of an fsai-dev pipeline run.
version: 0.3.0
---

# Implement Phase

Execute the approved plan wave by wave. The orchestrator plans, verifies, and integrates; subagents write the code. Waves run in the mode the plan declares: sequential (default) or parallel.

Delegated output is `built`, never `done`. A subagent's green claim, or even a green branch, is code-exists-unverified until the orchestrator runs the verification itself.

## Phase Contract

- **Phase id**: `implement`
- **Inputs**: approved `plan.md` with waves (approval recorded in the pipeline Decision Log). Stop and report if the plan is missing or unapproved.
- **Artifacts**: code on the branch; per-wave status updates in `pipeline.md`; checker verdicts in the manifest Verdicts section.
- **Exit criteria**: all waves done; each wave's exit-criteria commands ran green; `yarn tsc:all` and `yarn lint:all` green; checker findings resolved or waived in the Decision Log.
- **Default gate**: `autonomous` overall; each wave completion is a `notify`.

## Sequential waves (default)

For each wave, in plan order:

1. Mark the wave in-progress in `pipeline.md`.
2. Delegate the code changes to subagents. Do not write feature code inline: split the wave into parcels, brief each subagent with the plan context it needs, run independent parcels in parallel. Brief subagents to reuse existing code (search for prior art first) rather than duplicating logic inline.
3. Review and integrate the returned changes. Mark the wave `built`. You own coherence across parcels; subagents do not see each other's work.
4. Run the wave's exit-criteria commands from the plan. In fsai, `yarn tsc:all` and `yarn lint:all` are part of every wave's exit criteria regardless of what the plan says.
5. Run applicable checkers as parallel subagents: `arch-check` for waves touching backend code, `design-system-check` for waves touching frontend code. Both, if the wave touches both.
6. Resolve checker findings, or waive them with a reason in the Decision Log. A wave does not close with unresolved, unwaived findings.
7. Close the wave (`built` to `done` only now, with exit criteria verified): update `pipeline.md`, tick the plan's progress checklist, post the notify summary (what shipped, what the checks said), continue to the next wave.

## Parallel waves

For waves the plan declares parallel (independent tickets, disjoint files):

1. **Check file ownership before fanning out.** The plan assigns every contested file exactly one owner per wave. A ticket that needs a file owned by another ticket moves to a later sub-wave; it does not share the file. Sub-waves also sequence producer/consumer splits: the ticket that builds a shared component runs in an earlier sub-wave than the tickets that consume it.
2. **Fan out**: one subagent per ticket, each in its own git worktree on its own branch. Brief each with its ticket scope, its owned files, and the files it must NOT touch.
3. **Collect**: each returned branch is `built`. Run per-branch sanity (tsc on the touched packages) but do not certify branches individually.
4. **Integrate**: merge all wave branches into a dedicated integration branch. Merge conflicts mean the ownership map was wrong; record that in Surprises and fix the map before the next wave.
5. **Verify on the integration branch, in a dedicated worktree** with its own `node_modules` and built packages (never the user's checkout). Full battery: tsc, lint, unit suites, backend fast lane, and any e2e specs the plan names. Integration verification IS the wave's exit criterion; the wave is `done` only when the integration branch passes it.
6. Checkers and findings-resolution as in the sequential flow, run against the integration diff.

## pr-train delivery

When the plan declares `pr-train`, the pr phase runs at each wave close instead of once at
the end: after a wave's exit criteria pass and checker findings resolve, invoke `fsai-dev:pr`
scoped to that wave. Wave exit then also requires that wave's PR open and the declared
integration surface green after merge. The manifest's pr row accumulates one URL per wave.
Default remains `single-pr`: one terminal pr phase after all waves close.

## Rules

- Do not verify UI by driving the user's running browser. Verification is tsc, lint, and tests; the user checks visuals.
- Exit-criteria failures are fixed, not narrated. Fixing red tests and type errors is in-scope autonomous work.
- Plan deviations (a wave needs splitting, an approach does not survive contact with the code) go in the plan's Decision Log and Surprises sections as they happen.
- Cannot meet a wave's exit criteria after honest attempts: mark the phase `blocked(<reason>)` in `pipeline.md` and surface it. Never close a wave with caveats buried in prose.
