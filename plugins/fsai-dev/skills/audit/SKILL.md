---
name: audit
description: Adversarially verify a capability claim against actual code. Use for "audit this branch for X", "does this branch/code support X", "verify the claim that X", or as the audit phase of an fsai-dev pipeline run. Review mode only, reports a verdict, never fixes. Runs as a subagent given only the claim and a branch or diff.
version: 0.2.0
---

# Audit Phase

Adversarially verify one capability claim ("this branch supports full-plan brands on the funnel") against the code as it actually is. The claim is guilty until proven innocent.

## Phase Contract

- **Phase id**: `audit`
- **Inputs**: a capability claim stated in one sentence, and a branch or diff to audit. Stop and report if the claim is too vague to decompose.
- **Artifacts**: `audit.md` in the run directory; a verdict block in `pipeline.md` when part of a run. Standalone use returns the verdict directly.
- **Exit criteria**: every requirement of the claim classified as works / does-not-work / unverifiable, each with evidence.
- **Default gate**: `notify`.

## Method

1. **Decompose the claim** into the concrete requirements it implies. "Summit runs full+funnel" implies: the surface can be set, the public site serves it, publishing works, emails send, the builder admits it, setup state resolves. List them before reading any code.
2. **Trace each requirement through the real code path.** Follow the call chain from entry point to effect. Docs, comments, and ticket text are inadmissible; only code counts. Cite `file:line` or the function name for every conclusion.
3. **Classify each requirement**:
   - **Works**: the path exists and has no guard that excludes the claimed case. Cite the evidence that convinced you.
   - **Does not work**: name the root cause, not the symptom. If five features fail because one lookup throws, that is one finding with five consequences, not five findings.
   - **Unverifiable**: say why (needs a running environment, data-dependent, external service) and what would verify it.
4. **Converge on root causes.** Group does-not-work findings by shared cause. The valuable output is "one root cause, N downstream casualties", stated as a chain.
5. **Recommend a fix direction per root cause**, with the alternatives you considered and why they lose. Do not implement anything; this phase only reports.

If an `adversarial-review` skill is available in the session, apply its refutation discipline to your own findings before reporting: try to refute each does-not-work with a path you may have missed.

## Output shape

`audit.md` has three sections, in this order:

- **Already works**: requirement, evidence. Lead with what holds, so the reader knows what NOT to rebuild.
- **Does not work**: root cause first, then the casualty list, then the recommended fix direction with rejected alternatives and one-line reasons.
- **Unverifiable**: what, why, how to verify.

Close with a one-line verdict: claim holds / holds except <root causes> / does not hold.

## Rules

- Subagent-safe: assume no conversational context beyond the claim and the ref.
- Never soften a does-not-work into "may need attention". Binary classification with evidence, or unverifiable with a reason.
- A guard you expected but did not find is a finding too (e.g. an invalid state left unguarded). Report absences that the claim's safety depends on.
- Scope is the claim. Adjacent bugs you trip over get one line in a "Noticed, out of scope" footer, not investigation.

## Writing style

Write all prose in ASD-STE100 Simplified Technical English: short sentences (20 words for an
instruction, 25 for a description), active voice, one instruction per sentence, simple tenses, one
meaning per word, no noun clusters over three words, keep the articles, no em dashes, no idiom or
filler, vertical lists for steps and findings. Keep code identifiers, file paths, commands, and
error strings exact. Code and code comments follow the target repo's conventions, not this section.
Full rules: `${CLAUDE_PLUGIN_ROOT}/docs/phase-contract.md` section "Writing style".
