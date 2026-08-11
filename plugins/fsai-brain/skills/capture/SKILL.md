---
name: capture
description: End-of-day sweep for developers. Captures everything done today (commits, PRs, decisions made in the session) into the fsai-brain vault as ledger updates, decisions, precedents, and queue items. Use when the user says "capture today", "end of day", or invokes /fsai-brain:capture.
version: 0.1.0
---

# Capture

Turn today's work into brain updates. Run at the end of a working day, usually inside the session where the work happened, so conversation context counts as a source.

## Locate the brain

The vault is at `$FSAI_BRAIN_PATH`, or `~/code/fsai-brain` if unset. Read its `CLAUDE.md` first and follow its conventions; that file wins over this one on any conflict. Run `git -C <vault> pull --ff-only` before writing.

## Gather today's work

Collect from every source that applies; skip sources that do not:

1. **The current session.** Decisions made in conversation, questions left open, terminology settled, anything the user was told "not yet" about. This is the richest source; mine it first.
2. **Git.** In the repo(s) worked in today: `git log --since=midnight --author=<user>` across local branches, plus anything committed to the vault itself.
3. **GitHub.** `gh pr list --author @me` for PRs opened or merged today in `franchiseai/fsai`.
4. **Pipeline manifests.** If the working repo has `.agent/pipelines/`, read today's manifests: phase outcomes, waived findings, and Decision Log entries are capture material.

## Map work to notes

For each piece of work, decide what it changes in the brain:

- **Feature progress** → update the feature note's `state` and `updated`, and refresh its "What ops can say" line. A merged PR alone justifies at most `built` or `shipped`; `sellable` requires the user to confirm ops can promise it. Create missing feature notes from `Templates/Feature.md`.
- **A decision made today** → a note in `Decisions/`, with `affects` links and `supersedes` when it replaces an older decision. Close any `Queue/` item it answers (set `state: answered`, link the decision).
- **A rule that will recur** → a note in `Precedents/`, only if the user confirms it is a standing rule and not a one-off choice.
- **An open question** → a note in `Queue/` from `Templates/Question.md`, with a recommended `default`, a `deadline`, and `assigned` set to the person who can answer it.
- **A new or sharpened term** → create or update the `Glossary/` note.

Do not invent. If a source is ambiguous (for example, a merged PR whose customer effect is unclear), ask the user in the summary step instead of guessing. The fog rule applies to writing as much as reading.

## Confirm, then commit

1. Show the user a short plan: each note to create or change, one line each, with the state transitions spelled out (`SMS v2: built -> shipped`).
2. On confirmation, apply the edits, set `updated` fields, and commit to the vault as `capture: <date> <user>` (one commit; this is one logical event).
3. Push. If push fails on a non-fast-forward, pull and retry once; report a conflict instead of forcing.
4. Reply with what was captured and anything deliberately left out (for example, work on an unmerged experiment the user wants kept quiet until it lands).

## Style

Notes are read by non-developers. Plain short active sentences. No em dashes. No commit-message jargon: "the funnel now sends follow-up emails automatically", not "feat(bd): wire lead.created to email workflow".
