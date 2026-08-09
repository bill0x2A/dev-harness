---
name: backend-testing
description: Write and run backend tests for the fsai repo. Encodes test-lane selection, mock traps, DB gotchas, and the write-run-fix loop. Use for any backend test work in fsai, and before diagnosing "regressions" that appear only in test runs.
version: 1.2.0
---

# Backend Testing (fsai)

Phase skill for writing and running backend tests. The repo-specific knowledge below applies to the **fsai repo** (`apps/backend`, Vitest, two test lanes). For other repos, keep the process, rediscover the specifics.

## Phase Contract

- **Phase id**: `backend-testing`
- **Inputs**: implemented code on a branch (implement phase complete or a bugfix diff)
- **Artifacts**: test files co-located with source (`.test.ts`); a test-results verdict in the pipeline manifest (lane used, pass/fail vs baseline, any skips with reasons)
- **Exit criteria**: fast-lane suite green for touched domains (failure set no worse than merge-base baseline); no skipped tests without a recorded reason
- **Default gate**: autonomous

## 1. Schema-parity preflight

Before interpreting ANY red integration or e2e suite as a code failure, check migration parity: compare the migrations journal (`packages/supabase/migrations/meta/_journal.json` in fsai) against the applied migrations in the target DB (drizzle's applied-migrations table). Check BOTH the dev DB (:54322) and the fast-lane DB (:5434); they are different instances and drift independently.

Drift is the first hypothesis, not the last. Dev-DB drift has twice presented as something else entirely: "pretty much all e2e failing" with nothing wrong in the specs, and funnel analytics silently recording zero rows with zero errors (fire-and-forget ingest swallows schema errors). A red suite on a drifted DB tells you nothing about the code. Apply the missing migrations, re-run, then diagnose whatever is still red.

## 2. Lane selection

Two lanes, different configs, different capabilities:

| | Fast lane | Heavy lane |
|---|---|---|
| Command | `yarn workspace @fsai/backend test:fast <path>` | `yarn workspace @fsai/backend test` (`TEST_LANE=heavy`) |
| DB | Real disposable Postgres container (port 5434), BEGIN/ROLLBACK per test | fsai_test DB, `resetDatabase()` in beforeEach |
| Runs green locally | Yes (with known baseline) | **No: CI-only.** ~73 env failures locally (401 auth, `inngest.createFunction is not a function`, Failed query) on any branch, identically |
| Includes | Unit + `*.integration.test.ts` | Everything incl. db.js-mock unit tests |

Rules of thumb:

- Verify locally with the **fast lane**. Let heavy run in CI.
- Heavy stack, when genuinely needed locally: `yarn test:infra:up`, then exec vitest inside the backend container. The image bakes source: rebuild with `up -d --build backend` after every edit.
- **Baselines drift; never trust absolute counts.** Establish the baseline by running the same scope on the merge-base and compare failure SETS. The branch's failure set must be a subset of the baseline's.
- `tsc:all` can false-green on stale turbo cache. Gate backend changes with `npx tsc --noEmit` from `apps/backend`. After editing SDK contracts, rebuild first: `yarn workspace @fsai/sdk build` (backend resolves `@fsai/sdk` from built dist).

## 3. Mock traps

Both setup files (`src/test/setup.ts`, `src/test/setup-fast.ts`) eagerly call `createApp()`, resolving the app module graph BEFORE test-file `vi.mock` applies. Any module reached through the in-process app gets the REAL implementation in BOTH lanes. Symptoms: real permission assertion runs (`invalid input syntax for type uuid`), real service runs (`expected undefined to be 'brand'`).

Canonical workarounds, in preference order:

1. **Spy the singleton** (the proven recipe): `vi.spyOn(realImportedSingleton, 'method').mockResolvedValue(...)` on the same object the app dispatches through; `afterEach(vi.restoreAllMocks)`. Works for service singletons (`export const x = new X()`), `permissionsService`, `database.query.*`, and node builtins when prod code imports the module object and calls through it (`import dns from 'node:dns/promises'`; `dns.lookup(...)`; test spies `dns.lookup`).
2. **Registry override** where one exists: e.g. `registerAssertion('assertBrandPermissions', async () => {})` from `aperture/permissionGate.js` in beforeEach, instead of mocking the assertions module.
3. **`vi.resetModules()` + per-test dynamic `await import(...)`** for plain function modules (e.g. workflow actions): keeps the file's `vi.mock` factories, fresh registry applies them. Dynamic import WITHOUT resetModules still gets the setup-cached real instance. Reference: `workflow-actions.test.ts`.
4. **Seed real fixtures** (real uuids, real permission rows): heavier, fully faithful. What the aperture e2e tests do.

Known unmockables and traps:

- `vi.mock('db/db.js')` tests: fast lane defeats the mock (test hits the real fast DB), heavy hangs locally. These are **CI-only**. Verify locally via the sibling test that mocks the ENTITY layer instead; those pass in both lanes.
- `inngestClient` is globally mocked in setup-fast; per-test `vi.mock` of it never applies. Assert events via the harness's `getEmittedInngestEvents()`. You cannot make it throw: extract pure logic (chunking etc.) into an exported function and unit-test that.
- **CJS named-import boot crash**: `import { RestException } from 'twilio'` crashes the backend at boot under native ESM (cjs-module-lexer). Vitest interop and tsc both mask it; it only shows when the server boots. Use the default export (`twilio.RestException`). Applies to any CJS dep's non-lexer-visible named exports.
- Frontend aside (brand-dashboard): vitest is node-env, no jsdom, no @testing-library. Importing a hook module crashes collection via shared-ui to headlessui. Test pure logic through a co-located `*.utils.ts` seam with type-only imports.

## 4. Rules

- **NEVER run `db:reset`** (dev Supabase). Non-negotiable. The fast-lane container is separate and disposable; the safe fast-lane wipe is `docker compose -f apps/backend/docker-compose.fast.yml down -v`.
- Integration/curl tests target `http://localhost:4000`, never prod. Backend must be running (`yarn dev:backend`).
- When a contrived test input trips a prod edge case, **fix the test input**, not prod code. Only harden prod when a real caller could send that shape, or when asked.
- Don't run tests in worktrees without `node_modules` (they hang). Write code first; `yarn install` and test at the end.
- The dev DB (local Supabase, :54322) and the fast-lane DB (:5434) are different Postgres instances. **Migrate both** or `yarn dev` won't see new schema while tests do (silently absent UI, 500s).
- Run the fast suite SOLO: never concurrent with another `test:fast`, `tsc:all`, or `build`. The shared fast-lane PG exhausts connections ("too many clients") and produces phantom failures across unrelated domains. Before diagnosing a regression that only appears in agent worktrees: check for concurrent runs, re-run solo.

## 5. The write-run-fix loop

1. **Write** tests co-located with source. Prefer entity-layer mocks or singleton spies (Section 3) over module mocks. Follow the domain's existing test conventions.
2. **Run scoped**: `yarn workspace @fsai/backend test:fast <path>`.
3. **On fast-lane DB drift** ("relation already exists", drift-reapply from migration #1):
   - `docker compose -f apps/backend/docker-compose.fast.yml --project-directory apps/backend down -v`
   - then run `test:fast` DIRECTLY (its preflight starts the container and takes the clean fresh-database path). Do NOT run `yarn fast:up` in between: it restores a stale baseline dump that re-poisons the schema_sha.
   - If you must migrate manually: `up -d --wait` before `fast:migrate` (it does not wait for the DB itself).
   - Preflight bypass for PURE tests (no app-table queries), from `apps/backend`: `TEST_LANE=fast yarn vitest run --config vitest.fast.config.ts <path>`.
4. **Fix** red tests. Distinguish real regressions from harness artifacts using Sections 3 and 4 before touching prod code.
5. **Verify scope-wide**: touched domains under `test:fast`, failure set vs merge-base baseline, plus `npx tsc --noEmit` in `apps/backend`.
6. **Record** the verdict in the pipeline manifest: lane, scope, baseline comparison, skips with reasons.

## Writing style

Write all prose in ASD-STE100 Simplified Technical English: short sentences (20 words for an
instruction, 25 for a description), active voice, one instruction per sentence, simple tenses, one
meaning per word, no noun clusters over three words, keep the articles, no em dashes, no idiom or
filler, vertical lists for steps and findings. Keep code identifiers, file paths, commands, and
error strings exact. Code and code comments follow the target repo's conventions, not this section.
Full rules: `${CLAUDE_PLUGIN_ROOT}/docs/phase-contract.md` section "Writing style".
