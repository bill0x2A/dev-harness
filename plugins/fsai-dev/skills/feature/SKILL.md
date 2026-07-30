---
name: feature
description: Pipeline conductor. Use when the user says "/feature <task>", "run the pipeline", "start a pipeline for X", or "build this feature end to end". Composes fsai-dev phase skills into a gated pipeline; proposes phases, runs them in order, stops only at gates.
version: 0.1.0
---

# /feature: pipeline conductor

You are conducting a development pipeline. The contract system is defined in
`${CLAUDE_PLUGIN_ROOT}/docs/phase-contract.md` and the manifest format in
`${CLAUDE_PLUGIN_ROOT}/docs/pipeline-manifest.md`. Read both now if not already in context.
Do not invent phases, gates, or statuses; the catalog and gate model in those docs are authoritative.

## 1. Resume check

Slugify the task. Look in the target repo for `.agent/pipelines/*<task-slug>*/pipeline.md`.
If one exists: follow the resume protocol in pipeline-manifest.md. Read the manifest, reconcile
against reality (branch exists? artifacts exist? typecheck green?), fix stale statuses, and
continue from the first phase that is not `done` or `skipped`. Do not re-propose the pipeline
unless the task statement changed.

## 2. Propose (approve gate, always)

Classify the task: backend-only, frontend-only, full-stack, bugfix, AI-engine, refactor.
Select phases from the catalog and assign each phase its gate (default gate unless the run
warrants stricter or looser). Every catalog phase appears in the proposal: selected with an
order, or skipped with a reason. v2 phases are always skipped-with-reason, never selected.

Typical shapes:
- Full-stack feature: research, grill, plan, implement, arch-check, design-system-check, backend-testing, pr
- Backend-only: drop design-system-check
- Bugfix: usually just implement, arch-check, backend-testing, pr; grill/plan only if the fix is architectural
- AI-engine: as backend-only; note in the proposal that ai-feature-loop runs inside implement

Present the proposal as a short table (phase, gate, reason if skipped) and get explicit
approval via AskUserQuestion. Offer: approve as-is, edit phases, edit gates. The user editing
the list IS the mix-and-match mechanism; apply their edits verbatim and log them.

## 3. Initialize

After approval, in this order:
1. Ensure `.agent/pipelines/` is in the target repo's `.gitignore`; append it if missing.
2. Create `.agent/pipelines/<yyyy-mm-dd>-<task-slug>/pipeline.md` from the template in
   pipeline-manifest.md, with Status: running and the approved phase table.
3. Create the working branch per repo convention (fsai: `claude/<ticket-id>-<short-desc>`;
   no ticket, no convention: `claude/<task-slug>`). Never work on the default branch.
4. Log the approval (and any edits) in the Decision Log.

## 4. Execute

Run selected phases in order. For each:
1. Mark `in-progress` in the manifest.
2. Verify the phase's declared Inputs exist. Missing input: mark `blocked(<reason>)`, surface
   it, stop the pipeline.
3. Invoke the phase skill via the Skill tool: `fsai-dev:<phase-id>`. Phase skills may defer to
   repo-local or other-plugin skills; that is their business, not yours.
4. Exception: `arch-check` and `design-system-check` run as parallel subagents against the
   branch diff (they are subagent-safe by contract). Launch both in one message when both are
   selected. Their findings must be resolved or explicitly waived (logged) before `implement`
   exits.
5. Verify the phase's Exit criteria hold, then mark `done` and record its artifact. Never mark
   `done` on partial completion; use `blocked(<reason>)` and say so.
6. Apply the gate:
   - `approve`: set phase and pipeline Status to `blocked(gate)`, end your turn with a concise
     summary of the artifact and the specific question needing an answer.
   - `notify`: print a 2-4 line outcome summary, continue.
   - `autonomous`: log to manifest, continue.
7. Log surprises in Surprises & Discoveries as they happen, not retrospectively.

On resuming from an approve gate: record the user's response in the Decision Log, apply any
requested changes, set Status back to `running`, continue.

## 5. Complete

When the last phase is `done`: set Status: done, then summarize the run from the manifest:
what shipped, artifacts, decisions made at gates, anything skipped and why, open follow-ups.
If the pipeline is abandoned mid-run, set Status: abandoned with a Decision Log entry rather
than leaving it `running`.

## Conduct rules

- The manifest is the source of truth. Update it at every phase boundary and every stopping
  point; if the session dies it is the handoff.
- You orchestrate; phases do the work. Do not inline a phase's job because it seems quick.
- Between gates, work autonomously. Do not ask permission mid-phase; blocked-on-input is what
  `blocked(<reason>)` and gates are for.
- Repo rules (CLAUDE.md, design system, migration rules) always win over process convenience.
