---
name: frontend-testing
description: Frontend-testing phase of the fsai-dev pipeline. Write and run frontend unit tests for the fsai brand-dashboard via the co-located *.utils.ts node-env seam. Use for any brand-dashboard unit-test work, when frontend logic needs test coverage, or when invoked by the implement phase of an fsai-dev pipeline run.
version: 0.2.0
---

# Frontend-Testing Phase

Unit testing for fsai's brand-dashboard. Repo-specific: the constraints below describe the **fsai** repo's `apps/brand-dashboard`; verify before applying elsewhere.

## Phase Contract

- **Phase id**: `frontend-testing`
- **Inputs**: frontend code on a branch (typically after an implement wave touching brand-dashboard).
- **Artifacts**: co-located `*.utils.ts` modules with `*.utils.test.ts` suites, plus a test-results verdict in the pipeline manifest.
- **Exit criteria**: suites green; the extracted pure logic of the change is covered; `tsc --noEmit` and lint green on the touched app.
- **Default gate**: autonomous.

## The environment constraint (fsai)

brand-dashboard vitest runs in **node env**: no jsdom, no @testing-library, no renderHook. A test that imports a hook or component module fails at COLLECTION, because the import chain reaches `@fsai/shared-ui` then @headlessui and crashes with `ReferenceError: Element is not defined`. A fresh worktree additionally needs workspace dists built (`npx turbo run build --filter='@fsai/brand-dashboard^...'`) or collection fails on `@fsai/tiptap` and friends.

Consequence: you cannot unit-test hooks or components directly. Test pure logic through a seam.

## The *.utils.ts seam

1. **Extract** the pure logic of the change (selection resolution, sorting, gating predicates, data shaping) into a co-located `<name>.utils.ts` with zero runtime imports (`import type` only). Established pattern: `useOrganisationSelection.utils.ts` (FSAI-3785).
2. **Test** that file in `<name>.utils.test.ts` beside it. Plain vitest, table-driven where inputs enumerate.
3. **Keep the hook thin**: the hook wires React state to the utils functions; untested glue stays trivial enough to review by eye.

When the interesting logic cannot be extracted (it is genuinely React lifecycle behavior), say so in the verdict rather than writing a shallow test that imports nothing real. That gap is e2e territory.

## Run commands (fsai)

    cd apps/brand-dashboard
    npx cross-env ENV=TEST vitest run <path>        # one suite
    yarn workspace @fsai/brand-dashboard test        # full app suite

## Rules

- **Never verify UI by driving the user's running browser.** Verification is tsc + lint + unit suites plus a clear description of the visual change; the user checks the rendered result themselves.
- **Never fix correctness bugs with `useCallback`/`useMemo`.** Memoization is perf-only; code must stay correct with every memo stripped. A render loop means a structural cause (usually an effect writing derived state every render); fix that, and unit-test the extracted logic.
- **Fix test inputs, not prod code.** When a test trips an edge case in its own fixture, correct the fixture; do not add production guards for test-only inputs.
- **Don't run tests in worktrees without node_modules.** Write code first; install and test at the end, or build the workspace dists as above.

## Writing style

Write all prose in ASD-STE100 Simplified Technical English: short sentences (20 words for an
instruction, 25 for a description), active voice, one instruction per sentence, simple tenses, one
meaning per word, no noun clusters over three words, keep the articles, no em dashes, no idiom or
filler, vertical lists for steps and findings. Keep code identifiers, file paths, commands, and
error strings exact. Code and code comments follow the target repo's conventions, not this section.
Full rules: `${CLAUDE_PLUGIN_ROOT}/docs/phase-contract.md` section "Writing style".
