---
name: flow-test-agent
description: Use when setting up a flow test agent for a new project, or when asking how to build, extend, or use one. Covers the full lifecycle: build at project start, architecture decisions, writing good journeys, and using the agent day-to-day.
---

# Flow Test Agent

A flow test agent runs scripted Playwright journeys through your app and uses Claude AI to assert that each step looks correct. It seeds test data, tears it down, produces Markdown reports, and files GitHub issues for new findings.

**Build this at project start** — before you have flows to test, the same way you'd set up CI or a linter. It grows alongside the product. Retrofitting it later means catching up on debt while moving fast; building it first means it's always current.

## Part 1: Build It

### When to build

Every project that has a user interface and ships frequently. You don't need to wait for signals like "we keep breaking things" — if you're shipping, you need this.

### Architecture

```
flow-test/
  cli.ts                  # Entry point — Commander CLI
  config.ts               # Env var loading, EnvironmentConfig
  types.ts                # Shared types: Journey, Scenario, Finding, RunReport, etc.
  orchestrator.ts         # Drives a single journey: seed → mock → auth → replay → report
  replayer.ts             # Replays a saved action log with fresh AI assertions
  ai-reviewer.ts          # Wraps Anthropic SDK, reviewStep(), token budget enforcement
  auth-client.ts          # createTestSession(), injectSession() — local OTP vs remote HMAC
  seeder.ts               # Seeds and tears down test data via exec_sql RPC
  dom-snapshot.ts         # Captures simplified DOM for AI context
  mock-interceptor.ts     # Intercepts and optionally chaos-fails mocked requests
  reporter.ts             # Generates Markdown reports, baseline diffing, GitHub issues
  journeys/
    index.ts              # Registry: Record<string, Journey>
    onboarding.ts         # One file per journey
  scenarios/
    index.ts              # Registry: Record<string, ScenarioConfig>
    fresh_user.ts         # One file per scenario
  fixtures/
    seeds/
      fresh_user.sql      # One SQL seed file per scenario
      teardown.sql        # Cleans up all test data by marker (e.g. email domain)
  baselines/
    fresh_user-onboarding.json   # Known findings — committed, used for deduplication
  logs/                   # Action logs from runs (for replay)
  reports/                # Markdown reports (gitignored)
  setup/
    test-auth-bypass/
      index.ts            # Supabase Edge Function: HMAC-authenticated session minting
```

### Scenarios vs journeys

**Scenarios** are database states: a specific user with specific data pre-seeded. Examples: `fresh_user` (no store, 0 credits), `user_with_credits` (1 store, 10 credits), `multi_store` (2 stores).

**Journeys** are scripts: a sequence of actions + AI assertions that a user of a given scenario would take. One journey = one user story. Examples: `onboarding`, `catalog-browsing`, `buy-credits`.

Each journey declares which scenario it runs against. Multiple journeys can share a scenario.

### Authentication pattern

**Local (Supabase local dev):** Use OTP + Mailpit. Send OTP via `signInWithOtp`, poll Mailpit API for the 6-digit code, verify it, inject the resulting session into localStorage. Avoids GoTrue admin API which requires ES256 JWT in newer Supabase versions.

**Staging/remote:** Deploy a Supabase Edge Function (`test-auth-bypass`) that:
1. Guards with `FLOW_TEST_ENABLED=true` at startup — throws if missing
2. Verifies HMAC-SHA256 signature + timestamp (60s window) on every request
3. Finds or creates the test user, account, and connections
4. Mints a session via `generateLink` + `verifyOtp`
5. Returns `{ access_token, refresh_token, user_id, account_id }`

The client signs requests with the shared `TEST_AUTH_SECRET` before calling the edge function.

### Session injection

Don't navigate through the login UI — inject tokens directly into localStorage in the format Supabase expects, then reload. The app's `onAuthStateChange` picks it up as a real session.

```ts
const storageKey = `sb-${supabaseHostname}-auth-token`;
localStorage.setItem(storageKey, JSON.stringify({ access_token, refresh_token, token_type: "bearer", expires_in: 3600, expires_at: ... }));
location.reload();
```

### Seeder

Use an `exec_sql` RPC function for seeding — it lets you run arbitrary SQL against the local Supabase instance without needing the GoTrue admin API. Create it once:

```sql
CREATE OR REPLACE FUNCTION exec_sql(sql_text text) RETURNS void AS $$
BEGIN EXECUTE sql_text; END;
$$ LANGUAGE plpgsql;
```

Seed files are plain SQL. Teardown deletes by a test-specific marker (e.g. email domain `@yourproject-test.local`, shop domain `*.yourproject-test.invalid`).

### AI reviewer

Wrap the Anthropic SDK. For each `assert-ai` step:
1. Take a full-page screenshot
2. Capture a simplified DOM snapshot (strip scripts, styles, data attributes — keep structure and text)
3. Call Claude with: the assertion prompt, the URL, the DOM snapshot, a base64 screenshot, and any console errors / network failures observed so far
4. Parse the response for `pass: true/false`, `finding`, `reasoning`, `severity`
5. Enforce a token budget before each call — skip AI assertions if budget exhausted and log a warning

### Finding IDs and baselines

Finding IDs must be **stable** — based on journey name and step number, not timestamps. Example: `onboarding-step4`. This allows baseline deduplication: known findings are suppressed, new ones surface, fixed ones are noted.

Baselines are committed JSON files. After a local run, update the baseline to acknowledge findings you've triaged. On CI/staging, compare against the committed baseline.

### Chaos mode

A mock interceptor wraps `fetch` and optionally fails requests at a configured rate. Use a seeded RNG for deterministic replay. This tests error handling paths without needing a broken backend.

### Environment guard

The agent must never run against production. Enforce this two ways:
1. `config.ts` throws if `--env production` is passed
2. The `test-auth-bypass` edge function throws at startup if `FLOW_TEST_ENABLED !== "true"`

---

## Part 2: Set It Up for a New Project

### Step 1: scaffold the directory

Create `flow-test/` at the project root with `tsconfig.json` pointing to `../dist-flow-test` as outDir.

### Step 2: install dependencies

```bash
npm install --save-dev playwright @anthropic-ai/sdk commander tsx
npx playwright install chromium
```

### Step 3: create exec_sql

Run once against your local Supabase:

```sql
CREATE OR REPLACE FUNCTION exec_sql(sql_text text) RETURNS void AS $$
BEGIN EXECUTE sql_text; END;
$$ LANGUAGE plpgsql;
```

### Step 4: write your first skeleton journey

One journey, one AI assertion, before you have anything else. This proves the infrastructure works:

```ts
export const onboardingJourney: Journey = {
  name: "onboarding",
  scenario: "fresh_user",
  steps: [
    { action: "navigate", url: "/" },
    { action: "assert-ai", prompt: "The page has loaded and shows a welcome or dashboard state. No error messages are visible." },
  ],
};
```

### Step 5: add the agent definition

Create `.claude/agents/flow-test-agent.md` (see the agent definition template below).

### Step 6: grow it

Add a journey every time you add a user-facing feature. The implementation plan for any user-facing feature must include a task to add or extend the relevant journey.

---

## Part 3: Use It Day-to-Day

### Invocation

```bash
# Single journey
npx tsx flow-test/cli.ts --journey onboarding

# All journeys for a scenario
npx tsx flow-test/cli.ts --scenario fresh_user

# Everything
npx tsx flow-test/cli.ts --scenario all

# Chaos mode
npx tsx flow-test/cli.ts --scenario all --chaos --chaos-rate 0.3

# Replay a previous log (re-runs AI assertions only, faster)
npx tsx flow-test/cli.ts --replay flow-test/logs/<file>.json --scenario fresh_user
```

Via the Claude agent (if `.claude/agents/flow-test-agent.md` exists): just ask Claude to run a journey or scenario — it handles env setup automatically.

### Writing good AI assertions

Bad: `"The page looks correct."`
Bad: `"The user can see their products."`

Good: `"The catalog page is visible. There is a list of products with names and images. No error banners or empty states are shown."`
Good: `"A success toast or confirmation message is visible after clicking Publish. The button is no longer in a loading state."`

Rules:
- Describe what you'd see if it worked, not what you hope is true
- Include what should NOT be present (no errors, no loading spinners)
- One assertion per step — don't bundle multiple checks
- Reference specific UI elements by their visible label, not their DOM structure

### Interpreting findings

- **critical / high**: Fix before merging. These are broken user flows.
- **medium**: Triage — could be a real bug or an assertion that's too strict. Read the reasoning.
- **low**: Note for later. Usually cosmetic or timing-related.

### Updating baselines

After triaging findings from a local run, update the baseline:
- If a finding is a real bug you've fixed: re-run and the finding should disappear; the baseline updates automatically on the next local run.
- If a finding is a known issue you're deferring: accept it into the baseline so it stops surfacing as new.
- Never update the baseline on CI — only local runs produce committed baselines.

### Integrating into PR review

Run the relevant journeys as part of reviewing any PR that touches user-facing flows. The flow test agent is an input to code review, not a replacement for it.

---

## Agent Definition Template

Create `.claude/agents/flow-test-agent.md` in your project:

```markdown
---
name: flow-test-agent
description: Runs <project> E2E flow tests. Use when asked to test a journey, verify a flow, run the test agent, or check for regressions. Handles environment setup automatically and summarises findings when done.
tools: Bash, Read, Glob, Grep
---

You are the <project> flow test agent. [List journeys and scenarios here.]

[Standard pre-flight: start Supabase if needed, start dev server if needed, run tests, summarise findings.]
```
