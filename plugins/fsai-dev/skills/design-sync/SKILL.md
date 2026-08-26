---
name: design-sync
description: Design-sync phase of the fsai-dev pipeline. Propose MagicPath designs from a spec, reconcile existing designs against the spec, or pull a selected revision as a fidelity handoff for implement. Use when a task changes UI and the user wants designs to iterate on, when designs exist and must be checked against the spec, or when invoked by the /fsai-dev:feature conductor.
version: 0.1.0
---

# Design-Sync Phase

Keep the spec and the MagicPath canvas in agreement. This phase wraps the target repo's `magicpath` skill and adds what that skill does not know: the FSAI tokens, the brief, the reconciliation table, and the publishing boundary. Repo-specific facts below describe the **fsai** repo; verify before applying elsewhere.

## Phase Contract

- **Phase id**: `design-sync`
- **Inputs**: `spec.md` (or the grill output, or the user's statement when standalone), the list of surfaces to design, and a MagicPath project to work in. The target repo's `magicpath` skill must be present (`.claude/skills/magicpath` in fsai). Stop and report if `npx -y magicpath-ai whoami -o json` is not authenticated or the skill is missing.
- **Artifacts**: `design.md` in the run directory: the project canvas URL, one row per component (surface, `generatedName`, `componentId`, selected `revisionId`, share URL), the brief given to each authoring agent, and the reconciliation table. Side effects: components on the MagicPath canvas.
- **Exit criteria**: every surface in scope has a component on the canvas with FSAI tokens applied and a verified preview image; the reconciliation table has no unresolved row; the user approved the designs.
- **Default gate**: `approve` (design sign-off).

Agent kind: `fork` for the brief and the reconciliation (they need the spec's reasoning). Canvas authoring runs in `fresh` subagents from the written brief. Model tier: judgment for this phase; execution for the authoring subagents.

Position in a run: after `grill`, before `plan`. The plan phase reads `design.md` and cites design decisions by row.

## Modes

Pick one per invocation and name it in the manifest row.

- **propose**: no designs exist. Write briefs, author components, reconcile, present for sign-off.
- **reconcile**: designs exist. Export their source, compare to the spec, present the gap table for decisions.
- **pull**: designs are approved. Export the selected revisions and write the fidelity handoff for `implement`.

## Method

1. **Load the tools.** Call the Skill tool with `magicpath`. It owns the CLI commands and the canvas rules; this phase supplies the FSAI knowledge. Fetch `https://designsystem.franchisesystems.ai/llms.txt`, then the one to three pages the surfaces need.
2. **Choose the project.** List team projects (`list-projects --team "Franchise Systems Ai" -o json`) and reuse the project for this workstream when one exists. Create one only after the user names the workspace; in a pipeline, name it in the proposal so the gate covers it.
3. **Write one brief per surface** and save it in `design.md` before any agent runs. A brief holds: what the surface is for, the spec decisions it must show (cite them), the states it must cover (empty, loading, error, Lite variant when the surface has one), the data it displays with illustrative values, the shared-ui primitives it should mirror, the canvas size, and the token port instruction below. State the reasoning behind each decision so the agent can contradict it, and ask the agent to flag anything in the brief that turns out wrong once it reads the code.
4. **Author on the canvas** (propose mode), one `fresh` subagent per surface:
   - One screen per component. One `code start` session per screen, each with its own `--dir`. Fully interactive, responsive, centered, no device mockups.
   - Port the FSAI tokens: copy `packages/config-tailwind/theme.css` into the session's `src/index.css`. Keep the scaffold's `@import 'tailwindcss'`, `@plugin "tailwindcss-animate"`, and body reset. Append the FSAI `@theme` palette, `@theme inline`, and `@layer theme :root`. Drop `@plugin "@headlessui/tailwindcss"`; it is not installed there. Because `--color-*: initial` wipes the Tailwind defaults, the component must use FSAI tokens only (`text-strong`, `bg-blue-bg-accent`, `border-default`, `gray-16-light`), never `slate-*` or `blue-600`.
   - Do not use MagicPath themes (`get-theme`). The team's two `Franchise Systems` themes return `NOT_FOUND`; the `theme.css` port is the only working path.
   - `code submit --wait`, then verify: download `previewImageUrl` from `list-components <projectId> -o json` and look at it. A build success alone does not prove that the token classes resolved.
5. **Record the manifest.** For each component: `generatedName`, `componentId`, the selected `revisionId` (from `selection -o json`), and the share URL (`share <generatedName> -o json`). Label the block "illustrative numbers only, public URLs".
6. **Reconcile.** Build the table in `design.md`: one row per spec decision, with the design's treatment and a status of `matches`, `deviates: <reason>`, or `missing`. In reconcile mode, get the facts from source, not from screenshots: `code context <componentId> --revision <revisionId> --dir <empty staging dir>`, one staging dir per component, never an app root, never `code start` for an export. Every `deviates` and `missing` row is a decision for the gate, not a silent fix.
7. **Gate.** Under `approve`: present the canvas URL, the per-component share URLs, and the reconciliation table with its open rows. Stop for sign-off. On approval, record it in the Decision Log with any requested changes, mark the phase `done`, and hand `design.md` to `plan`.
8. **Pull** (pull mode): export each approved revision to a staging dir and write the fidelity handoff in `design.md` per the magicpath skill's local-code reference. Precedence when sources disagree: the user's explicit changes, then local runtime behavior and accessibility, then MagicPath presentation. Never `add` MagicPath components into an fsai app. Designs are the parity target; `implement` rebuilds them with `@fsai/shared-ui` primitives, and a primitive is a match only when its computed output is identical.

## Publishing boundary

A MagicPath share URL is public. Apply this rule before you author, not after:

- Never put credentials, customer data, or security controls on the canvas.
- Internal business logic, such as scoring formulas and their constants, is acceptable when it serves fidelity. The user set this bar on 2026-08-20.
- For a borderline class of data, ask the user. Do not scrub unilaterally. A scrubbed revision does not remove the original from the component's team-visible revision history, so a scrub after the fact is incomplete by construction.

## Rules

- This phase reconciles designs against the spec. `design-system-check` reviews code against design-system rules. Do not duplicate it.
- Never run `view` in parallel. Keep an embedded browser on the project canvas, not on component previews, unless asked.
- One staging dir per exported component. Never point `code context` at an app root; it overwrites `src/App.tsx` and `src/index.css`.
- Real data or no surface: designs show illustrative values, and the manifest says so. Never fabricate a metric the product cannot produce.
- Respect the repo's UI rules in the brief: shared-ui primitives, semantic tokens, "Members" not "Owners", no em dashes in copy, no raw IDs, primary actions never disabled.
- When the user adds this phase mid-run, move it from `skipped` to selected with a Decision Log entry, and run it before the next implement wave that touches the surface.

## Writing style

Write all prose in ASD-STE100 Simplified Technical English: short sentences (20 words for an
instruction, 25 for a description), active voice, one instruction per sentence, simple tenses, one
meaning per word, no noun clusters over three words, keep the articles, no em dashes, no idiom or
filler, vertical lists for steps and findings. Keep code identifiers, file paths, commands, and
error strings exact. Code and code comments follow the target repo's conventions, not this section.
Full rules: `${CLAUDE_PLUGIN_ROOT}/docs/phase-contract.md` section "Writing style".
