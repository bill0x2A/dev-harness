---
name: e2e
description: E2E phase of the fsai-dev pipeline. Write, run, and fix Playwright suites for the fsai brand-dashboard, portals, and funnel against a locally running stack. Use after an implement wave that changes a user journey, when the user asks for browser coverage of a flow, or when invoked by the /fsai-dev:feature conductor.
version: 0.1.0
---

# E2E Phase

Browser coverage for user journeys. Unit tests prove the logic; this phase proves the journey. Repo-specific: the facts below describe the **fsai** repo. Verify before applying elsewhere.

## Phase Contract

- **Phase id**: `e2e`
- **Inputs**: an implemented branch, the journeys the change touches (from `plan.md` or `spec.md`, or from the user when standalone), and a local stack that serves that branch. Stop and report if the stack cannot be started.
- **Artifacts**: spec files in the suite that owns the surface, plus a verdict block in `pipeline.md`: suite, specs run, pass and fail counts, known-broken annotations, and coverage gaps.
- **Exit criteria**: every spec for the touched journeys is green locally with zero retries, and the full suite of that surface is green or every failure is explained as pre-existing with evidence.
- **Default gate**: `notify`.

Agent kind: `fresh`. Model tier: execution.

## The two suites (fsai)

| Surface | Suite | Specs | Run |
|---|---|---|---|
| brand-dashboard (3001), applicant and franchisee portals (3000), backend (4000) | `e2e/` (own `package.json`, not a workspace) | `e2e/specs/*.spec.ts` | `cd e2e && yarn test` |
| funnel (3002) | `apps/funnel/e2e/` | `apps/funnel/e2e/*.spec.ts` | `yarn workspace @fsai/funnel test:e2e` |

Read `e2e/README.md` before you write a spec. It holds the traps that cost days to find. When you learn a new trap, append it there; the README is the suite's memory, and the plugin is not.

No CI workflow runs either suite. Every verdict is local-only; say so in the verdict.

## Method

1. **Pick the suite** from the surface the change touches. A change that spans dashboard and funnel needs specs in both.
2. **Stand up the stack on the standard ports.** Ports are load-bearing: the backend redirects invites to `localhost:3001`, the portal resolves its API to `localhost:4000`, and CORS allows `3002`. Alternate ports fail in ways that look like network errors.
   - Dashboard and portal suite: the stack must already run (`yarn o` or `yarn dev`, plus Supabase on 54322). The suite starts nothing and tests whatever serves the ports. Confirm the served code is this branch before you trust a result.
   - Funnel suite: Playwright starts the backend and a funnel preview build itself, with rate-limit caps raised. Stop any running `yarn dev` for the funnel first, or cold-compile page loads time out. Do not reuse a running backend: real rate caps make specs 429 each other. The Inngest dev server must run or every FDD submission returns 500.
   - In a worktree, the stack must still serve the standard ports. Stop the main checkout's stack first, or run the suite from the main checkout against the branch.
3. **Map the change to journeys.** List the routes and actions the change touches. Grep the suite for those routes and labels; the matching specs are the regression set. Run them before you write anything new.
4. **Write the new specs** with the conventions below. One spec per journey. Name entities with a `Date.now()` suffix so seed data is never mutated.
5. **Run scoped, then wide.** `npx playwright test specs/<name>.spec.ts` for the new specs, then the full suite of that surface once.
6. **Fix loop.** On failure, open the trace (`npx playwright show-trace test-results/<dir>/trace.zip`) and classify before you touch anything:
   - **Product bug**: the journey is broken. Do not bend the spec. In a pipeline, return the finding to the implement phase. Standalone, call the Skill tool with `fsai-dev:diagnose`.
   - **Timing**: replace the wait with `expect.poll`, `waitForURL`, or a role assertion. Never add a sleep and never add retries; a retry hides flake.
   - **Environment**: stale build, wrong stack, rate limit, tour overlay. Fix the environment and rerun. Record it in Surprises if it cost time.
7. **Write the verdict** in `pipeline.md`:

       e2e: funnel suite, 20 specs, 20 pass, 0 fail (local only). New: application-currency.spec.ts.
       Known-broken carried: none. Gaps: DocuSeal signing not driven.

## Spec conventions (fsai)

- Import `test` and `expect` from the suite's `fixtures/test.ts`, never from `@playwright/test`. The wrapper adds the teardown ledgers.
- Selectors: `getByRole` and `getByText` first, `getByLabel` next. Add a `data-testid` only when no role or text can identify the element; the app has about a dozen, all deliberate.
- Actors: give every actor its own `browser.newContext()` and close it at the end. Log in through the real UI with the fixture helpers (`loginAs`, `asSuperadmin`, `loginToPortal`).
- Seed accounts share the password in `fixtures/db.ts`. Dashboard emails are stored with a `brand_` prefix and typed bare.
- Nothing sends email in dev. Use the DB bypasses: `verifyEmail` before sign-in, invite codes from `organisation_invites` through the backend redirect endpoint, DB-minted portal verification tokens.
- Product tours auto-start and their overlay eats clicks. Call `markTourSeen` before you drive a toured page. The user id is the auth user, not the agent.
- Closed shared-ui menus stay in the DOM. A repeated menu item needs `.last()`.
- Stripe: target the card iframe by its title, never by index. Type with `pressSequentially`.
- Entities you create through the UI go to the ledger by hand (`recordLearningCourse` and friends), or they leak into seeded brands.
- Use `page.route` only for behavior the suite deliberately does not drive (the rate limiter). Reproduce the real header shape when you stub.
- A spec that fails for a known product reason gets a `test.info().annotations` entry and a header comment. Never delete it and never skip it silently.
- Long journeys (guides, notifications, chat) may raise `test.setTimeout` per spec. Do not raise the config timeout.

## Rules

- **Never drive the user's own browser.** Headless Playwright only. The user checks visuals themselves.
- **Never run `db:reset`.** The suites never reset state; they suffix names and use ledgers.
- **Never mutate seed data** (Sunny Scoops, The CodFather, Glacier Grill, the seed accounts). Other specs depend on it.
- **Never change the port configuration** in either `playwright.config.ts`.
- **Fix test inputs, not prod code.** A fixture that trips its own edge case gets a better fixture, not a production guard.

## Writing style

Write all prose in ASD-STE100 Simplified Technical English: short sentences (20 words for an
instruction, 25 for a description), active voice, one instruction per sentence, simple tenses, one
meaning per word, no noun clusters over three words, keep the articles, no em dashes, no idiom or
filler, vertical lists for steps and findings. Keep code identifiers, file paths, commands, and
error strings exact. Code and code comments follow the target repo's conventions, not this section.
Full rules: `${CLAUDE_PLUGIN_ROOT}/docs/phase-contract.md` section "Writing style".
