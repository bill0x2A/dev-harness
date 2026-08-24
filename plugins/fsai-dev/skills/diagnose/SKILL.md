---
name: diagnose
description: Diagnose phase of the fsai-dev pipeline. Disciplined diagnosis loop for hard bugs and performance regressions. Reproduce, minimise, hypothesise, instrument, fix, regression-test. Use when the user says "diagnose this" or "debug this", reports a bug, says something is broken/throwing/failing, describes a performance regression, or when invoked by the /fsai-dev:feature conductor as the entry phase of a bugfix run.
version: 0.4.1
---

# Diagnose Phase

A discipline for hard bugs. Skip phases only when explicitly justified.

When exploring the codebase, use the project's domain glossary to get a clear mental model of the relevant modules, and check ADRs in the area you're touching.

## Phase Contract

- **Phase id**: `diagnose`
- **Inputs**: a bug report or observed failing behavior (error message, wrong output, perf regression). Optionally production context (Sentry/BetterStack pulls) when available.
- **Artifacts**: `diagnosis.md` in the run dir: repro steps, the feedback loop used, root cause with file:line evidence, fix direction, and a final post-mortem (why chain, root class, class elimination; see Phase 6). In the conductor's minimal bugfix mode there is no run dir; the diagnosis summary and the post-mortem answers go in the eventual PR description instead.
- **Exit criteria**: root cause demonstrated, not hypothesized: a repro exists and the correct hypothesis has been confirmed by instrumentation or bisection.
- **Default gate**: notify on root cause. When a quick patch and the proper fix diverge, the gate carries the patch-or-proper question with a recommended default (see Phase 5).

Standalone use (outside a pipeline) works as before: same loop, findings reported in conversation and the commit/PR message.

## Phase 1: Build a feedback loop

**This is the skill.** Everything else is mechanical. If you have a fast, deterministic, agent-runnable pass/fail signal for the bug, you will find the cause: bisection, hypothesis-testing, and instrumentation all just consume that signal. If you don't have one, no amount of staring at code will save you.

Spend disproportionate effort here. **Be aggressive. Be creative. Refuse to give up.**

### Ways to construct one, in roughly this order

1. **Failing test** at whatever seam reaches the bug: unit, integration, e2e.
2. **Curl / HTTP script** against a running dev server.
3. **CLI invocation** with a fixture input, diffing stdout against a known-good snapshot.
4. **Headless browser script** (Playwright / Puppeteer): drives the UI, asserts on DOM/console/network. Never the user's own running browser session.
5. **Replay a captured trace.** Save a real network request / payload / event log to disk; replay it through the code path in isolation.
6. **Throwaway harness.** Spin up a minimal subset of the system (one service, mocked deps) that exercises the bug code path with a single function call.
7. **Property / fuzz loop.** If the bug is "sometimes wrong output", run 1000 random inputs and look for the failure mode.
8. **Bisection harness.** If the bug appeared between two known states (commit, dataset, version), automate "boot at state X, check, repeat" so you can `git bisect run` it.
9. **Differential loop.** Run the same input through old-version vs new-version (or two configs) and diff outputs.
10. **HITL bash script.** Last resort. If a human must click, drive _them_ with `scripts/hitl-loop.template.sh` so the loop is still structured. Captured output feeds back to you.

Build the right feedback loop, and the bug is 90% fixed.

### Iterate on the loop itself

Treat the loop as a product. Once you have _a_ loop, ask:

- Can I make it faster? (Cache setup, skip unrelated init, narrow the test scope.)
- Can I make the signal sharper? (Assert on the specific symptom, not "didn't crash".)
- Can I make it more deterministic? (Pin time, seed RNG, isolate filesystem, freeze network.)

A 30-second flaky loop is barely better than no loop. A 2-second deterministic loop is a debugging superpower.

### Non-deterministic bugs

The goal is not a clean repro but a **higher reproduction rate**. Loop the trigger 100x, parallelise, add stress, narrow timing windows, inject sleeps. A 50%-flake bug is debuggable; 1% is not. Keep raising the rate until it's debuggable.

### When you genuinely cannot build a loop

Stop and say so explicitly (in a pipeline: mark the phase `blocked(no feedback loop)`). List what you tried. Ask the user for: (a) access to whatever environment reproduces it, (b) a captured artifact (HAR file, log dump, core dump, screen recording with timestamps), or (c) permission to add temporary production instrumentation. Do **not** proceed to hypothesise without a loop.

Do not proceed to Phase 2 until you have a loop you believe in.

## Phase 2: Reproduce

Run the loop. Watch the bug appear.

Confirm:

- [ ] The loop produces the failure mode the **user** described, not a different failure that happens to be nearby. Wrong bug = wrong fix.
- [ ] The failure is reproducible across multiple runs (or, for non-deterministic bugs, reproducible at a high enough rate to debug against).
- [ ] You have captured the exact symptom (error message, wrong output, slow timing) so later phases can verify the fix actually addresses it.

Do not proceed until you reproduce the bug.

## Phase 3: Hypothesise

Generate **3-5 ranked hypotheses** before testing any of them. Single-hypothesis generation anchors on the first plausible idea.

Each hypothesis must be **falsifiable**: state the prediction it makes.

> Format: "If <X> is the cause, then <changing Y> will make the bug disappear / <changing Z> will make it worse."

If you cannot state the prediction, the hypothesis is a vibe. Discard or sharpen it.

**Show the ranked list to the user before testing.** They often have domain knowledge that re-ranks instantly ("we just deployed a change to #3"), or know hypotheses they've already ruled out. Cheap checkpoint, big time saver. Don't block on it; proceed with your ranking if the user is AFK.

## Phase 4: Instrument

Each probe must map to a specific prediction from Phase 3. **Change one variable at a time.**

Tool preference:

1. **Debugger / REPL inspection** if the env supports it. One breakpoint beats ten logs.
2. **Targeted logs** at the boundaries that distinguish hypotheses.
3. Never "log everything and grep".

**Tag every debug log** with a unique prefix, e.g. `[DEBUG-a4f2]`. Cleanup at the end becomes a single grep. Untagged logs survive; tagged logs die.

**Perf branch.** For performance regressions, logs are usually wrong. Instead: establish a baseline measurement (timing harness, `performance.now()`, profiler, query plan), then bisect. Measure first, fix second.

## Phase 5: Fix + regression test

In a pipeline run, the confirmed root cause is the phase boundary: write `diagnosis.md`, fire the notify gate, and let the implement/testing phases own the fix (this phase's loop and minimised repro carry forward as their verification signal). Standalone, continue here.

**Patch or proper fix.** At the root-cause boundary, decide which fix to write. When a quick patch and the proper fix genuinely diverge, put the question in the gate summary:

- State both options, the cost of each, and the risk the patch carries.
- Give a recommendation and a default: "patching unless you say otherwise". Continue on the default; an AFK run must not stall here.
- If the patch is chosen or the default applies, file the proper fix as the class-elimination candidate in the same gate output. A patch must never silently become permanent.
- Record the choice in the Decision Log.

When the two options do not diverge, say so in one line and continue.

Write the regression test **before the fix**, but only if there is a **correct seam** for it.

A correct seam is one where the test exercises the **real bug pattern** as it occurs at the call site. If the only available seam is too shallow (single-caller test when the bug needs multiple callers, unit test that can't replicate the chain that triggered the bug), a regression test there gives false confidence.

**If no correct seam exists, that itself is the finding.** Note it. The codebase architecture is preventing the bug from being locked down. Flag this for the next phase.

If a correct seam exists:

1. Turn the minimised repro into a failing test at that seam.
2. Watch it fail.
3. Apply the fix.
4. Watch it pass.
5. Re-run the Phase 1 feedback loop against the original (un-minimised) scenario.

React-specific rule (fsai): never fix a correctness bug (render loop, React #185) with `useCallback`/`useMemo`. They are perf-only; code must stay correct with every memo stripped. Fix the structural cause, usually an effect that fires every render and unconditionally writes derived state.

## Phase 6: Cleanup + post-mortem

Required before declaring done:

- [ ] Original repro no longer reproduces (re-run the Phase 1 loop)
- [ ] Regression test passes (or absence of seam is documented)
- [ ] All `[DEBUG-...]` instrumentation removed (`grep` the prefix)
- [ ] Throwaway prototypes deleted (or moved to a clearly-marked debug location)
- [ ] The hypothesis that turned out correct is stated in the commit / PR message, so the next debugger learns

### Why chain

After the fix is verified, build a short why chain in `diagnosis.md`:

1. Why 1 is the immediate technical cause: the confirmed root cause.
2. Keep asking why until the answer stops being about this code: why was it written this way, why did nothing catch it.
3. Stop when one more why would not change what you do next. Two to four whys is typical.

Record the chain as a numbered list. The deepest why is the input to the next two steps.

### Classify the root

Classify the deepest why. More than one class can apply.

- **Code problem**: the structure allowed the bad state. Mechanism: a type that makes the bad state unrepresentable, a lint rule, a schema constraint, a narrower API, a test seam that does not exist yet.
- **Knowledge problem**: a person did not know a fact that exists. Mechanism: make the fact findable. Record it in the FSAI vault (`~/Documents/fsai-vault`) as a reference or teaching note in the area it belongs to. Name the missing fact, never the person.
- **Standards problem**: the team has no agreed rule, or the rule is not enforced. Mechanism: a note in the vault's `Practices/` folder, plus a candidate rule for `arch-check` or `design-system-check` when the rule is mechanical enough to check.

### Class elimination

Then answer this question in `diagnosis.md`:

> What deeper change could we make to eliminate this class of error entirely?

Rules for the answer:

1. The answer must address the deepest why in the chain and use the mechanism for its class. "Be more careful with X" is not an answer; write "none found" instead and say why no class mechanism applies.
2. Give a rough cost (hours, days, or "large refactor") and name the files, notes, or checker the change touches.
3. Answer after the fix is in, not before. You know more now than when you started.

**Ask always, act separately.** The answer never widens the current fix. Route it instead:

- In a pipeline: the conductor surfaces it at the gate as a "class-elimination candidate". On approval it becomes a Linear ticket, or a vault note when it is not yet ticket-ready. It is never extra scope for this run.
- Standalone: state it in the summary and offer to file it. Do not implement it in the same change without the user's explicit yes.
- If the change ships later and sets a standing rule, record it in the vault's `Practices/` folder so the class stays dead.

## Writing style

Write all prose in ASD-STE100 Simplified Technical English: short sentences (20 words for an
instruction, 25 for a description), active voice, one instruction per sentence, simple tenses, one
meaning per word, no noun clusters over three words, keep the articles, no em dashes, no idiom or
filler, vertical lists for steps and findings. Keep code identifiers, file paths, commands, and
error strings exact. Code and code comments follow the target repo's conventions, not this section.
Full rules: `${CLAUDE_PLUGIN_ROOT}/docs/phase-contract.md` section "Writing style".
