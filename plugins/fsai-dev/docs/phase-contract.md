# The Phase Contract

Every phase skill in this plugin declares the same five-part contract. The contract is what makes phases composable: the conductor (`/fsai-dev:feature`) chains phases by matching one phase's artifacts to the next phase's inputs, and a human can audit any run by reading the manifest without knowing how any phase works internally.

## Contract fields

Every phase skill's `SKILL.md` must contain a `## Phase Contract` section with exactly these fields:

- **Phase id**: kebab-case name, stable across versions. Used in pipeline manifests.
- **Inputs**: what must exist before the phase runs. Either artifacts from earlier phases (named by their manifest path) or external things ("a Linear ticket", "a MagicPath share URL"). If an input is missing at runtime, the phase must stop and report — never improvise the input.
- **Artifacts**: what the phase produces and the exact path where it lands, relative to the run directory (see pipeline-manifest.md). A phase that produces no file artifact (e.g. a checker) produces a **verdict block** in the manifest instead.
- **Exit criteria**: how to tell the phase is done. Machine-checkable wherever possible ("`yarn tsc:all` green", "playwright suite passes", "all findings resolved or waived"). A phase must not be marked `done` in the manifest until its exit criteria hold.
- **Default gate**: `autonomous`, `notify`, or `approve` (see below). The conductor may override per run; the default is what the phase recommends for itself.

## Gate model

Gates are data in the pipeline manifest, not behavior baked into skills. The same phase can run at different trust levels on different runs.

- **`approve`** — hard stop. The phase completes its work, writes its artifact, updates the manifest to `blocked(gate)`, and the conductor ends its turn with a clear summary and the specific question needing an answer. Work does not proceed past this phase until the user responds.
- **`notify`** — post a concise summary of the phase outcome (in-terminal in v1; Slack in a later version), mark the phase `done`, and continue immediately.
- **`autonomous`** — log the outcome to the manifest and continue. No user-facing output beyond normal progress notes.

Default gate assignments across the catalog: pipeline proposal, plan sign-off, design sign-off, and pre-merge are `approve`. Research findings, wave completions, and test results are `notify`. Implementation, test-writing, and fixing red tests are `autonomous`.

## Rules for phase authors

1. **Never silently skip.** If a phase decides part of its work doesn't apply, it records `skipped: <reason>` in the manifest. Skipping is a reviewable decision.
2. **Artifacts are self-contained.** Someone reading only the run directory must be able to understand what was decided and why (ExecPlan spirit: see `.agent/PLANS.md` in the target repo).
3. **Respect repo rules.** Phases defer to the target repo's CLAUDE.md, docs, and design system. This plugin encodes *process*; the repo encodes *rules*. Where a phase embeds repo knowledge (e.g. backend-testing lane traps), it must say which repo it applies to.
4. **Checkers are subagent-safe.** Phases whose contract says "runs as subagent" must be executable from only their inputs (a diff, a spec path) with no conversational context.
5. **Fail loudly.** A phase that cannot meet its exit criteria marks itself `blocked(<reason>)` in the manifest and surfaces it — it never marks itself done with caveats buried in prose.

## Phase catalog

| Phase id | Purpose | Default gate | Status |
|---|---|---|---|
| `diagnose` | Reproduce and root-cause a bug into `diagnosis.md`; the entry phase for bugfix runs | notify (on root cause) | v1 |
| `research` | API/library/approach research → decision doc with a recommendation | notify | v1 |
| `grill` | Interrogate the feature idea against domain language and docs → sharpened spec | approve (spec sign-off) | v1 |
| `plan` | Author an ExecPlan from the spec, broken into waves | approve (plan sign-off) | v1 (thin; defers to target repo's PLANS.md) |
| `design-sync` | MagicPath pull/push + reconcile designs against spec | approve (design sign-off) | v2 (wraps existing `magicpath` skill) |
| `implement` | Execute plan waves; per-wave verification; delegates code to subagents | autonomous, notify per wave | v1 |
| `code-review` | General adversarial code review of a wave/branch diff — logic bugs the rule checkers can't see; runs as subagent | autonomous (findings block implement exit) | v1 |
| `arch-check` | Backend diff vs 3-layer architecture + domain rules; runs as subagent | autonomous (findings block implement exit) | v1 |
| `design-system-check` | Frontend diff vs FSAI design system rules; runs as subagent | autonomous (findings block implement exit) | v1 |
| `audit` | Adversarially verify a capability claim against the code ("does this branch support X?"); runs as subagent | notify | v1 |
| `backend-testing` | Write/run backend tests; encodes fsai lane knowledge and mock traps | autonomous | v1 |
| `frontend-testing` | Write/run frontend unit tests; encodes brand-dashboard vitest node-env and test-seam knowledge | autonomous | v1 |
| `e2e` | Playwright suites: write, run, fix | notify | v2 |
| `pr` | Create the PR (delegates to `pr-description` skill where installed) | approve (pre-merge) | v1 |
| `prod-context` | Pull Sentry/BetterStack context for bugfixing (folds into `diagnose`) | autonomous | v2 |
| `staging-e2e` | Post-merge staging verification | n/a (CI/scheduled agent, not a session phase) | v2 |

v2 phases are listed so the conductor can name them as explicitly skipped rather than pretending they don't exist.

## Continuous improvement

The framework feeds the `skill-evolution` plugin where installed. Its PostToolUse hook already records a `pending` observation stub in the target repo's `.claude/skill-evolution/observations.jsonl` for every phase-skill invocation; the conductor's completion step closes the loop by assessing each executed phase (outcome, gate edits as corrections, waivers and surprises as notes) so stubs don't rot at `pending`. When a phase accumulates bad outcomes, `skill-evolution:skill-amend` proposes evidence-based changes to that phase skill — amendments to skills in this plugin should land here (edit, bump the skill version, push), not in a local cache copy.
