# fsai-dev

Composable development pipeline for FSAI work.

## How it works

1. **Phase skills** are the units of process. Each declares a contract: inputs, artifacts, exit criteria, and a default gate (`autonomous` / `notify` / `approve`). See [docs/phase-contract.md](docs/phase-contract.md).
2. **The pipeline manifest** (`.agent/pipelines/<date>-<slug>/pipeline.md` in the target repo, gitignored) is the source of truth for a run: selected phases, gates, statuses (including `built` = exists-but-unverified), decision log. Any session can resume a run from the manifest alone. Multi-workstream efforts add a **program tracker** above the runs. See [docs/pipeline-manifest.md](docs/pipeline-manifest.md).
3. **The conductor** (`/fsai-dev:feature <task>`) classifies the task, proposes a pipeline (your first approval gate — edit the phase list to mix and match), then runs it, stopping only at `approve` gates.

Phases also work standalone: `/fsai-dev:research`, `/fsai-dev:grill`, `/fsai-dev:backend-testing`, etc.

## Phase catalog (v1)

| Phase | Purpose |
|---|---|
| `feature` | Conductor: propose, run, and resume pipelines (with a size floor: trivial fixes get a one-line minimal pipeline, no ceremony) |
| `diagnose` | Reproduce and root-cause a bug into `diagnosis.md`; entry phase for bugfix runs |
| `research` | API/approach research producing a decision doc with a recommendation |
| `grill` | Interrogate the feature against domain language and docs into a sharpened spec |
| `plan` | ExecPlan authoring (defers to target repo's PLANS.md), milestones as verifiable waves |
| `implement` | Wave execution via subagents (sequential, or parallel across worktrees with file ownership), per-wave verification and checker runs |
| `audit` | Adversarially verify a capability claim against the code (review-only, subagent-safe) |
| `code-review` | Adversarial code review of a diff — logic bugs the rule checkers can't name (review-only, subagent-safe) |
| `arch-check` | Backend diff vs the 3-layer architecture (review-only, subagent-safe) |
| `design-system-check` | Frontend diff vs FSAI design system rules (review-only, subagent-safe) |
| `backend-testing` | Write/run backend tests; encodes fsai test-lane knowledge and mock traps |
| `frontend-testing` | Write/run frontend unit tests via the `*.utils.ts` node-env seam (fsai brand-dashboard) |
| `pr` | PR creation per repo conventions; delivery-mode aware (`single-pr`, `pr-train` independent or stacked via `gh stack`) |

Planned (named in the catalog so runs can skip them explicitly, not silently): `design-sync` (MagicPath reconcile), `e2e` (Playwright), `prod-context` (Sentry/BetterStack), `staging-e2e` (CI, not a session phase).

## Writing style

Every phase writes its artifacts, verdicts, gate questions, and PR bodies in ASD-STE100 Simplified Technical English: short active sentences, one instruction per sentence, one meaning per word, no em dashes or idiom, vertical lists for steps. Code identifiers, file paths, and commands stay exact. Code and code comments follow the target repo's conventions. See [docs/phase-contract.md](docs/phase-contract.md).

## Trust dial

Gates are per-run data, not baked-in behavior. Overnight autonomous run: flip everything to `notify`. Risky change: add `approve` gates. Same skills either way.

## Model routing and executors

Phases are assigned a model by cognitive demand: `fable` for judgment (diagnose, code-review, audit, grill, plan, research), `opus` for execution (implement waves, testing, rule checkers, pr), `sonnet` for mechanical steps. Never `haiku`. When a tier's model is unavailable the conductor substitutes down `fable` > `opus` > `sonnet` and logs it, so a run never stalls on one dry quota; the manifest records which model actually ran each phase. Phases spawn as `fork` (needs conversation context: grill, plan, research) or `fresh` (needs only its declared Inputs: every subagent-safe phase, implement waves, testing, pr). The `implement` phase can execute a wave with an alternate engine (`codex`, pilot); foreign diffs face the same checker phases, which is what makes substitution safe. See [docs/phase-contract.md](docs/phase-contract.md).
