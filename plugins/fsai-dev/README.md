# fsai-dev

Composable development pipeline for FSAI work.

## How it works

1. **Phase skills** are the units of process. Each declares a contract: inputs, artifacts, exit criteria, and a default gate (`autonomous` / `notify` / `approve`). See [docs/phase-contract.md](docs/phase-contract.md).
2. **The pipeline manifest** (`.agent/pipelines/<date>-<slug>/pipeline.md` in the target repo, gitignored) is the source of truth for a run: selected phases, gates, statuses, decision log. Any session can resume a run from the manifest alone. See [docs/pipeline-manifest.md](docs/pipeline-manifest.md).
3. **The conductor** (`/fsai-dev:feature <task>`) classifies the task, proposes a pipeline (your first approval gate — edit the phase list to mix and match), then runs it, stopping only at `approve` gates.

Phases also work standalone: `/fsai-dev:research`, `/fsai-dev:grill`, `/fsai-dev:backend-testing`, etc.

## Phase catalog (v1)

| Phase | Purpose |
|---|---|
| `feature` | Conductor: propose, run, and resume pipelines |
| `research` | API/approach research producing a decision doc with a recommendation |
| `grill` | Interrogate the feature against domain language and docs into a sharpened spec |
| `plan` | ExecPlan authoring (defers to target repo's PLANS.md), milestones as verifiable waves |
| `implement` | Wave execution via subagents, per-wave verification and checker runs |
| `arch-check` | Backend diff vs the 3-layer architecture (review-only, subagent-safe) |
| `design-system-check` | Frontend diff vs FSAI design system rules (review-only, subagent-safe) |
| `backend-testing` | Write/run backend tests; encodes fsai test-lane knowledge and mock traps |
| `pr` | PR creation per repo conventions (delegates to pr-description where installed) |

Planned (named in the catalog so runs can skip them explicitly, not silently): `design-sync` (MagicPath reconcile), `e2e` (Playwright), `prod-context` (Sentry/BetterStack), `staging-e2e` (CI, not a session phase).

## Trust dial

Gates are per-run data, not baked-in behavior. Overnight autonomous run: flip everything to `notify`. Risky change: add `approve` gates. Same skills either way.
