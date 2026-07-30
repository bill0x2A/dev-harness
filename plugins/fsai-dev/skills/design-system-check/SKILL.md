---
name: design-system-check
description: Frontend design-system checker for fsai. Reviews a diff against FSAI design system rules (shared-ui components, tokens, SDK usage, known component traps). Review mode only, reports findings, never fixes. Runs as a subagent given only a branch or diff.
version: 0.1.0
---

# design-system-check

Review a frontend diff (brand-dashboard, applicant-portal, frandev-lite, shared packages) against FSAI design system rules. Report findings; do not fix anything.

## Phase Contract

- **Phase id**: `design-system-check`
- **Inputs**: a diff or branch (default: current branch vs `master`). If neither resolves, stop and report `blocked(no diff)`.
- **Artifacts**: a verdict block for the pipeline manifest. Either `pass`, or numbered findings, each with `file:line`, the rule violated, and a suggested fix. No file artifact.
- **Exit criteria**: verdict written to the manifest (or returned to the caller when run as a subagent).
- **Default gate**: `autonomous`. Findings block the `implement` phase's exit until resolved or explicitly waived.

## Procedure

1. `git diff master...HEAD -- 'apps/brand-dashboard/**' 'apps/applicant-portal/**' 'apps/frandev-lite/**' 'packages/shared-ui/**' 'packages/shared-brand-dashboard/**'` (or the diff you were given). If empty, verdict is `pass (no frontend changes)`.
2. Read each changed component file in full.
3. Check the rules below against changed code only. Pre-existing violations in untouched code are out of scope; note them in one line at most.
4. When correct usage of a shared-ui component is in doubt, fetch 1-3 targeted pages from the live docs: `https://designsystem.franchisesystems.ai/docs/<path>.mdx` (e.g. `components/picker`, `patterns/forms`). Prefer live docs over cached knowledge.
5. Verify each suspected finding against the current codebase before reporting. Some traps below may have been fixed upstream (e.g. a hardened component); confirm the trap still exists before flagging it.
6. Emit the verdict block.

## Rules (fsai-specific)

1. **UI primitives come from `@fsai/shared-ui` or `@fsai/shared-brand-dashboard`.** Hand-rolled buttons, modals, inputs, selects, or badges that duplicate an existing shared component are findings. Recompose existing components before creating new ones.
2. **Icons come from `@fsai/icons`.** No inline SVGs or third-party icon imports where a library icon exists.
3. **No duplicated component code across apps.** A component copied between apps belongs in shared-ui (primitives) or the app that owns the domain.
4. **Data fetching goes through the SDK.** No direct `fetch`/`axios` calls in frontend apps; use SDK services and hooks.
5. **No `neutral-*` Tailwind classes.** Use semantic tokens or the `gray-XX-light` scale.
6. **No `orange-stronger` token in brand-dashboard.** `bg/ring/text-orange-stronger` silently paints transparent (token does not exist). Use a real `orange-bg-*` token or a literal value.
7. **Domain DataTable list rows live in apps, not shared-ui.** shared-ui holds primitives only; a domain-specific row component added to shared-ui is a finding.
8. **No hardcoded placeholder data.** Every displayed value is wired to real data, or the surface is dropped. Placeholder strings, fake counts, and lorem content are findings.
9. **No raw IDs shown to users in workflow-builder surfaces.** Use human-readable labels.
10. **`Picker.Trigger` child buttons inside forms need `type="button"`.** Picker.Trigger clones a typeless child button, which defaults to `type=submit` and auto-submits the form. Verify Picker.Trigger has not been hardened upstream before flagging (rule 5 of the procedure).

## Verdict format

Return exactly one of:

    design-system-check: pass (N frontend files reviewed)

or:

    design-system-check: 2 findings
    1. apps/brand-dashboard/src/pages/Leads.tsx:142
       Rule 5 (neutral-* class). `text-neutral-500` on the empty-state copy.
       Fix: use the semantic muted text token or `gray-60-light`.
    2. <file>:<line>
       Rule N (<short name>). <one-line evidence>.
       Fix: <one-line concrete fix>.

Severity ordering: broken rendering and data-integrity rules (6, 8, 10) first, styling tokens last. If a finding is uncertain, say so in the finding rather than omitting it.
