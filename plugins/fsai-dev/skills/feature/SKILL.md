---
name: feature
description: Pipeline conductor. Use when the user says "/feature <task>", "run the pipeline", "start a pipeline for X", or "build this feature end to end". Composes fsai-dev phase skills into a gated pipeline; proposes phases, runs them in order, stops only at gates.
version: 0.6.1
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
- Full-stack feature: research, grill, plan, implement, arch-check, design-system-check, code-review, backend-testing, frontend-testing, pr
- Backend-only: drop design-system-check and frontend-testing
- Bugfix: diagnose, implement, then the checkers for the surface (arch-check + backend-testing
  for backend bugs; design-system-check + frontend-testing for frontend bugs), code-review, pr.
  grill/plan only if the fix is architectural.
- AI-engine: as backend-only; note in the proposal that ai-feature-loop runs inside implement

The proposal also carries an initial delivery mode: `single-pr` (default) or `pr-train`
(each wave ships its own PR onto a declared integration surface; see pipeline-manifest.md).
The plan phase may change it later with a Decision Log entry.

Size floor: when the expected change is trivially small (single root cause, roughly one
commit, no schema or contract changes), do not create a run directory or a full proposal
table. Offer a one-line minimal pipeline instead: diagnose, fix, test, pr. Escalate to a
full run only if diagnosis reveals something bigger; escalation creates the run directory
then, with the diagnosis as its first artifact.

Present the proposal as a short table (phase, gate, reason if skipped) and get explicit
approval via AskUserQuestion. Offer: approve as-is, edit phases, edit gates. The user editing
the list IS the mix-and-match mechanism; apply their edits verbatim and log them.

## 3. Initialize

After approval, in this order:
1. Ensure `.agent/pipelines/` is in the target repo's `.gitignore`; append it if missing.
2. Create `.agent/pipelines/<yyyy-mm-dd>-<task-slug>/pipeline.md` from the template in
   pipeline-manifest.md, with Status: running, the Delivery line (pr-train also names the
   integration surface), an empty `Models:` line, and the approved phase table.
3. Create the working branch per repo convention (fsai: `claude/<ticket-id>-<short-desc>`;
   no ticket, no convention: `claude/<task-slug>`). Never work on the default branch.
4. Log the approval (and any edits) in the Decision Log.

## 4. Execute

Run selected phases in order. In `pr-train` delivery, the pr phase runs inside implement at
each wave close instead of as a terminal phase; its manifest row accumulates one PR URL per
wave. For each phase:
1. Mark `in-progress` in the manifest.
2. Verify the phase's declared Inputs exist. Missing input: mark `blocked(<reason>)`, surface
   it, stop the pipeline.
3. Call the Skill tool with `fsai-dev:<phase-id>`. Phase skills may defer to
   repo-local or other-plugin skills; that is their business, not yours. Spawn it with the
   model its tier assigns and the agent kind it gets, both per the contract's Model routing
   and Agent kind sections. If that model is unavailable or its pool is exhausted, walk the
   substitution order to the next available model, log the substitution in the Decision Log,
   and continue. A run never stalls because one pool is dry.
4. Exception: `arch-check`, `design-system-check`, and `code-review` spawn as parallel
   `fresh` agents against the branch diff (they are subagent-safe by contract, so they get
   artifact paths, not artifact contents). Launch all that are selected in one message. Their
   findings must be resolved or explicitly waived (logged) before `implement` exits. When
   subagent spawning is unavailable (the pipeline itself runs inside a fork), subagent-safe
   phases run inline per their contracts; record the mode in the manifest.
5. Verify the phase's Exit criteria hold, then mark `done` and record its artifact. Never mark
   `done` on partial completion; use `blocked(<reason>)` and say so. Update the manifest's
   `Models:` line with the model that actually ran the phase.
6. Apply the gate:
   - `approve`: set phase and pipeline Status to `blocked(gate)`, end your turn with a concise
     summary of the artifact and the specific question needing an answer.
   - `notify`: print a 2-4 line outcome summary, continue. A phase contract may attach a
     question with a recommended default to its notify gate (for example diagnose's
     patch-or-proper question). State the question and the default, continue on the default,
     and record the user's answer in the Decision Log when it arrives.
   - `autonomous`: log to manifest, continue.
7. Log surprises in Surprises & Discoveries as they happen, not retrospectively.

On resuming from an approve gate: record the user's response in the Decision Log, apply any
requested changes, set Status back to `running`, continue.

## 5. Complete

When the last phase is `done`: set Status: done, then summarize the run from the manifest:
what shipped, artifacts, decisions made at gates, anything skipped and why, open follow-ups.
If the pipeline is abandoned mid-run, set Status: abandoned with a Decision Log entry rather
than leaving it `running`.

### Continuous improvement

At completion (Status: done or abandoned), close the skill-evolution loop. The skill-evolution
PostToolUse hook has already appended `pending` stubs to the target repo's
`.claude/skill-evolution/observations.jsonl` for each phase invocation; leave those lines alone.
Append one assessed observation per executed phase to the same file (append-only; create the
directory if missing), following skill-evolution's observation schema:
- `skillName` exactly as invoked (`fsai-dev:<phase-id>`), `pluginName` `"fsai-dev"`.
- `taskSummary`: the manifest's Task line, compressed to one sentence.
- `outcome`: `completed` if the phase reached `done` cleanly; `partial` if it needed waivers,
  manual fixes, or gate-driven rework; `failed` if it ended `blocked` on its own defect;
  `abandoned` if the run was abandoned while it was live.
- `userCorrections`: count of Decision Log entries where the user changed that phase's output.
- `errors`: waived findings and blocked reasons. `notes`: relevant Surprises entries plus any
  friction with the phase skill itself (missing guidance, wrong defaults, contract mismatch).
  When the phase ran on anything other than its tier-assigned model, say which model ran it.
  Without that, a weak result from a substitute model reads as a weak phase skill and the
  amendment loop draws the wrong conclusion.

Then scan the jsonl for fsai-dev phases with repeated `partial`/`failed` outcomes across runs.
If any, suggest `skill-evolution:skill-amend` for them in the run summary, and note that
amendments to fsai-dev skills land in the plugin repo (edit, bump the skill version, push),
never in the installed cache copy.

## Conduct rules

- The manifest is the source of truth. Update it at every phase boundary and every stopping
  point; if the session dies it is the handoff.
- You orchestrate; phases do the work. Do not inline a phase's job because it seems quick.
- Between gates, work autonomously. Do not ask permission mid-phase; blocked-on-input is what
  `blocked(<reason>)` and gates are for.
- Repo rules (CLAUDE.md, design system, migration rules) always win over process convenience.

## Writing style

Write all prose in ASD-STE100 Simplified Technical English: short sentences (20 words for an
instruction, 25 for a description), active voice, one instruction per sentence, simple tenses, one
meaning per word, no noun clusters over three words, keep the articles, no em dashes, no idiom or
filler, vertical lists for steps and findings. Keep code identifiers, file paths, commands, and
error strings exact. Code and code comments follow the target repo's conventions, not this section.
Full rules: `${CLAUDE_PLUGIN_ROOT}/docs/phase-contract.md` section "Writing style".
