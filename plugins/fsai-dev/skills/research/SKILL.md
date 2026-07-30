---
name: research
description: Research phase of the fsai-dev pipeline. Investigate candidate APIs, libraries, or approaches for a task and produce a decision doc with a single recommendation. Use when a task needs external research before planning, or when invoked by the /fsai-dev:feature conductor.
version: 0.1.0
---

# Research Phase

Produce a decision, not a survey. The artifact exists so the next phase (grill or plan) can proceed without re-opening the question.

## Phase Contract

- **Phase id**: `research`
- **Inputs**: task statement (from the pipeline manifest, or directly from the user when run standalone). Optional: links, tickets, prior notes the user supplies. If the task statement is missing or too vague to name what is being decided, stop and report.
- **Artifacts**: `research.md` in the run directory.
- **Exit criteria**: a single recommendation is stated with reasoning; every load-bearing claim (pricing, rate limits, API capabilities, licensing) is verified against a primary source and cited; rejected alternatives are listed with reasons; open questions are explicit.
- **Default gate**: `notify`

## Method

1. **Frame the decision first.** Write one sentence: "We are choosing X to achieve Y under constraints Z." Every subagent gets this sentence. If you cannot write it, the input is insufficient; stop and report.

2. **Fan out in parallel.** Launch subagents in one batch, each with a distinct angle:
   - One web-search agent per candidate API/library/vendor: capabilities, pricing, limits, auth model, deprecation signals.
   - One Explore agent on the target repo: existing patterns the candidate must fit (SDK layer, service structure, existing integrations that do something similar). A candidate that fights the repo's architecture loses points regardless of features.
   - One agent for prior art in the repo's own history: has this been evaluated before? Check docs, ADRs, and any prior research notes.

3. **Verify claims against primary sources.** Pricing pages, official docs, and changelogs only. Blog posts and LLM memory are leads, not sources. Pricing especially: quote the number, the plan name, the URL, and the date checked. Stale pricing has burned this team before.

4. **Check fit before recommending.** The recommendation must name where the candidate lands in the target repo (which package, which service layer, what the integration seam is). "Best tool" that does not fit the repo's patterns is not the recommendation.

5. **Write `research.md`** in the run directory:
   - **Decision frame**: the one sentence from step 1.
   - **Recommendation**: one candidate, with reasoning and the integration seam in this repo.
   - **Rejected alternatives**: name, one line each on why not. No feature matrices.
   - **Verified claims**: each load-bearing fact with source URL and date checked.
   - **Open questions**: anything that could overturn the recommendation, and who can answer it.

6. **Gate.** Under `notify`: summarize the recommendation in two or three sentences, mark the phase `done` in the manifest, continue. Standalone: deliver the summary and stop.

## Rules

- One recommendation. If two candidates are genuinely tied, say which you would pick anyway and put the tiebreaker question in open questions.
- Keep `research.md` under two pages. Selectivity is the value.
- Never let a subagent's summary stand in for a primary source on pricing or limits; re-verify anything surprising yourself.
