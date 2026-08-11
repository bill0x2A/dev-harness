---
name: ingest
description: Route any free text into the fsai-brain vault as correctly structured notes. Use when the user pastes information for the brain, says "put this in the brain", or invokes /fsai-brain:ingest with text such as a thought, a Slack thread, a meeting note, or "we decided X".
version: 0.1.0
---

# Ingest

Take whatever the user gives you and file it into the brain so the structure survives. The input can be anything: a sentence, a pasted Slack thread, meeting notes, a customer email, a half-formed idea.

## Locate the brain

The vault is at `$FSAI_BRAIN_PATH`, or `~/code/fsai-brain` if unset. Read its `CLAUDE.md` first and follow its conventions; that file wins over this one on any conflict. Run `git -C <vault> pull --ff-only` before writing.

## Route the content

Split the input into facts, then route each fact to a note type:

- A choice that was made, with a why → `Decisions/`
- A standing rule to apply again → `Precedents/`
- An unresolved choice someone must make → `Queue/` (fill `assigned`, a recommended `default`, and a `deadline`; if the input names none, ask)
- Progress or new information about a feature → update the note in `Features/` (state changes to `sellable` need explicit confirmation)
- A new or corrected definition → `Glossary/`
- Partner or customer information → `Ops/`
- Anything that fits no type → a free-form note in the most sensible folder, linked from related notes; never force it into a template

One input often yields several notes. A pasted thread may contain a decision, two open questions, and a feature update; file each separately and link them.

## Rules

- **Update before create.** Search the vault for an existing note on the same subject first. Extend it rather than creating a near-duplicate.
- **Preserve, never overwrite.** If new information contradicts an existing note, record the new state and what changed ("previously X, now Y, changed because Z"); do not silently erase the old claim. If the contradiction looks like an error rather than an update, ask.
- **Attribute.** When ingesting someone else's words (a thread, an email), name the source and keep load-bearing phrasing as a quote.
- **Link.** Wire `[[wikilinks]]` between everything filed from one input, and to the people involved.
- **The fog rule.** File only what the input actually says. Gaps stay gaps; note them as open questions if they matter.

## Confirm, then commit

1. Show the user the filing plan: each note to create or update, one line each.
2. On confirmation, apply the edits, set `updated` fields, and commit as `ingest: <short subject>`.
3. Push. If push fails on a non-fast-forward, pull and retry once; report a conflict instead of forcing.

## Style

Notes are read by non-developers. Plain short active sentences. No em dashes. Preserve the source's meaning exactly; simplify only the wording.
