---
name: arch-check
description: Backend architecture checker for fsai. Reviews a diff against the 3-layer pattern (routes/controller/service/entity), SDK type rules, schema rules, and domain-module boundaries. Review mode only, reports findings, never refactors. Runs as a subagent given only a branch or diff.
version: 0.1.0
---

# arch-check

Review a backend diff against `docs/backend-architecture.md` in the fsai repo. Report findings; do not fix anything. For refactoring, the `domain-refactor` skill exists.

## Phase Contract

- **Phase id**: `arch-check`
- **Inputs**: a diff or branch (default: current branch vs `master`). If neither resolves, stop and report `blocked(no diff)`.
- **Artifacts**: a verdict block for the pipeline manifest. Either `pass`, or numbered findings, each with `file:line`, the rule violated, and a suggested fix. No file artifact.
- **Exit criteria**: verdict written to the manifest (or returned to the caller when run as a subagent).
- **Default gate**: `autonomous`. Findings block the `implement` phase's exit until resolved or explicitly waived.

## Procedure

1. `git diff master...HEAD -- 'apps/backend/**'` (or the diff you were given). If empty, verdict is `pass (no backend changes)`.
2. Read each changed backend file in full, not just hunks. Layer violations hide in unchanged context.
3. Check every rule below against changed code only. Pre-existing violations in untouched code are out of scope; note them in one line at most.
4. Before reporting a finding, verify it against the current file state. Do not report from the diff alone.
5. Emit the verdict block.

## Rules (fsai-specific)

Authoritative source: `docs/backend-architecture.md` in the fsai repo. Read it if any rule below needs more context. Dependency chain is strict: `Route -> Controller -> Service -> Entity -> Database`.

1. **Routes are declarative only.** `<domain>Routes.ts` registers paths and forwards to controllers. No middleware logic, validation, or business logic.
2. **Controllers are thin.** No business logic, no DB queries, no branching on domain state, no side effects. Input-shape checks and permission assertions only. Decorators `@ValidateAuth` and `@AsyncHandler` on every method.
3. **Controllers never call entities.** Services are the only layer that calls entities. A controller importing `*Entity` is a finding.
4. **Assertions run at the entry boundary.** Permission checks (`assert*`) are called from controllers, not buried inside services. Services calling `permissionsService` directly is legacy, not a pattern to copy; flag new instances.
5. **Services enforce, entities determine.** Business rules live in entity predicates (`can*`, `is*`) returning `{ allowed, reason }`; services check the predicate and throw (`ValidationError` etc.). A service re-implementing a rule inline, or an entity throwing business errors, is a finding.
6. **No independent resolution in services.** If a service needs an internal ID, it must come from an async predicate's returned context (`{ allowed: true, instanceId }`), not a separate resolve call before the action.
7. **Services own transactions; side effects after commit.** `database.transaction` belongs in services. Notifications/emails fire after the transaction succeeds, never inside it. Entities may emit `eventBridge` events after data writes; entities must not send notifications or emails.
8. **SDK is the single source of truth for request/response types.** Controllers use `TypedRequestBody` / `TypedRequestParams` / `ParamsToBodyRequest` / `ParamsToQueryRequest` parameterized with types imported from `@fsai/sdk` (`<Action><Entity>Params` / `<Action><Entity>Return` in `packages/sdk/src/types/`). Inline or controller-local request/response types are a finding.
9. **Schema comes from `@fsai/supabase`.** Table definitions imported from the package; no hand-written SQL strings in services or entities; migrations generated, never hand-edited.
10. **New tables need RLS enabled** in the generated migration.
11. **Domain-module boundaries.** Cross-domain imports go through the module's `index.ts`, `<domain>Events.ts`, or `<domain>Queries.ts`; deep imports of another domain's entity/assertions are findings. Tables belong to one module: cross-domain reads use the owning module's service methods or exported query fragments; cross-domain writes never. A new query fragment must encode the module's validity rules (soft-delete, tenancy, status), not be a bare table alias.
12. **Naming conventions.** Files `<domain>Routes|Controller|Service|Entity|Assertions|Utils.ts`. Method prefixes: `can*`/`is*` predicates, `resolve*` private, `assert*` throws PermissionError, `format*` in utils. General-purpose helpers live in `api/utils/`, not entity files.

## Verdict format

Return exactly one of:

    arch-check: pass (N backend files reviewed)

or:

    arch-check: 3 findings
    1. apps/backend/src/api/deals/dealsController.ts:88
       Rule 3 (controller calls entity). `dealEntity.canConvert` invoked from controller.
       Fix: move the predicate check into `dealsService.convertDeal` and call the service.
    2. <file>:<line>
       Rule N (<short name>). <one-line evidence>.
       Fix: <one-line concrete fix>.

Severity ordering: layer violations and SDK-type violations first, naming last. If a finding is uncertain, say so in the finding rather than omitting it.
