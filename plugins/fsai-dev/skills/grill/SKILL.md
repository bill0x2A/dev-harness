---
name: grill
description: Grill phase of the fsai-dev pipeline. Interrogation session that challenges a plan against the existing domain model, sharpens terminology, updates documentation (CONTEXT.md, ADRs) inline as decisions crystallise, and produces a sharpened spec. Use when the user wants to stress-test a feature idea against their project's language and documented decisions, or when invoked by the /fsai-dev:feature conductor.
version: 0.1.0
---

# Grill Phase

## Phase Contract

- **Phase id**: `grill`
- **Inputs**: task statement (from the pipeline manifest, or directly from the user when run standalone). Optional: `research.md` from the research phase; if present, read it first and treat its recommendation as a challengeable assumption, not a settled fact.
- **Artifacts**: `spec.md` in the run directory: the sharpened spec capturing scope, resolved terminology, decisions made during the session, and explicit deferrals. Side effects: `CONTEXT.md` glossary updates and ADRs in the target repo, per the formats below.
- **Exit criteria**: every challenged assumption is resolved or explicitly deferred in `spec.md`; the user has approved the spec.
- **Default gate**: `approve` (spec sign-off)

When run standalone (outside a pipeline), everything below works as before; skip the `spec.md` artifact only if the user does not want one.

<what-to-do>

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

Ask the questions one at a time, waiting for feedback on each question before continuing.

If a question can be answered by exploring the codebase, explore the codebase instead.

</what-to-do>

<supporting-info>

## Domain awareness

During codebase exploration, also look for existing documentation:

### File structure

Most repos have a single context:

```
/
├── CONTEXT.md
├── docs/
│   └── adr/
│       ├── 0001-event-sourced-orders.md
│       └── 0002-postgres-for-write-model.md
└── src/
```

If a `CONTEXT-MAP.md` exists at the root, the repo has multiple contexts. The map points to where each one lives:

```
/
├── CONTEXT-MAP.md
├── docs/
│   └── adr/                          ← system-wide decisions
├── src/
│   ├── ordering/
│   │   ├── CONTEXT.md
│   │   └── docs/adr/                 ← context-specific decisions
│   └── billing/
│       ├── CONTEXT.md
│       └── docs/adr/
```

Create files lazily: only when you have something to write. If no `CONTEXT.md` exists, create one when the first term is resolved. If no `docs/adr/` exists, create it when the first ADR is needed.

## During the session

### Challenge against the glossary

When the user uses a term that conflicts with the existing language in `CONTEXT.md`, call it out immediately. "Your glossary defines 'cancellation' as X, but you seem to mean Y. Which is it?"

### Sharpen fuzzy language

When the user uses vague or overloaded terms, propose a precise canonical term. "You're saying 'account'. Do you mean the Customer or the User? Those are different things."

### Discuss concrete scenarios

When domain relationships are being discussed, stress-test them with specific scenarios. Invent scenarios that probe edge cases and force the user to be precise about the boundaries between concepts.

### Cross-reference with code

When the user states how something works, check whether the code agrees. If you find a contradiction, surface it: "Your code cancels entire Orders, but you just said partial cancellation is possible. Which is right?"

### Update CONTEXT.md inline

When a term is resolved, update `CONTEXT.md` right there. Don't batch these up: capture them as they happen. Use the format in [CONTEXT-FORMAT.md](./CONTEXT-FORMAT.md).

`CONTEXT.md` should be totally devoid of implementation details. Do not treat `CONTEXT.md` as a spec, a scratch pad, or a repository for implementation decisions. It is a glossary and nothing else.

### Offer ADRs sparingly

Only offer to create an ADR when all three are true:

1. **Hard to reverse**: the cost of changing your mind later is meaningful
2. **Surprising without context**: a future reader will wonder "why did they do it this way?"
3. **The result of a real trade-off**: there were genuine alternatives and you picked one for specific reasons

If any of the three is missing, skip the ADR. Use the format in [ADR-FORMAT.md](./ADR-FORMAT.md).

### Build spec.md as you go

Record each resolved decision in `spec.md` in the run directory as it lands, in the same inline spirit as the glossary updates. Structure:

- **Goal**: what the user can do after this ships, in one or two sentences
- **Scope**: what is in, what is explicitly out
- **Decisions**: each resolved question with its answer and one line of why
- **Deferred**: questions raised but consciously postponed, with who or what unblocks them
- **Terminology**: pointers to the CONTEXT.md terms touched this session

## Ending the session

The session ends when no challenged assumption remains unresolved and undeferred. Present `spec.md` for sign-off. In a pipeline under the `approve` gate: mark the phase `blocked(gate)` in the manifest, summarize the spec and any deferrals, and stop for approval. On approval, record the response in the Decision Log and mark the phase `done`.

</supporting-info>
