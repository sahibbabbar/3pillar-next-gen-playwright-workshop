# From Zero to Agent-Driven Playwright Tests — Step-by-Step Guide

This guide walks you through everything from opening a fresh VSCode window to running fully agent-driven Playwright tests against the TravelApp demo, including visual screenshot comparison. Follow the steps in order — each section assumes the previous one is done.

> All commands assume macOS / Linux. On Windows use the PowerShell or WSL equivalents.

---

## Prerequisites

Install these once on your machine:

- **Node.js 18 or newer** — verify: `node --version`
- **VSCode** — https://code.visualstudio.com
- **Git** — verify: `git --version`
- **Claude Code CLI** — https://claude.com/claude-code (needed for the agent steps later, not for the plain Playwright steps)
- **The TravelApp demo running on `http://localhost:8080`** — start it however the workshop instructions specify and verify with:
  ```bash
  curl -sS -o /dev/null -w "%{http_code}\n" http://localhost:8080/
  # expect: 200
  ```

---

## Step 1 — Open VSCode and create the project

1.1. Launch VSCode.

1.2. Open the workshop project folder:
- **File → Open Folder…** → pick `ts-automation/`
- Or from a terminal:
  ```bash
  code /path/to/3pillar-playwright-agent-workshop/ts-automation
  ```

1.3. Open the integrated terminal: **`` Ctrl+` ``** (macOS: **`` Cmd+` ``**).

1.4. Recommended VSCode extensions:
- **Playwright Test for VSCode** (Microsoft) — green run/debug icons inline next to each test
- **ESLint** — optional, for linting
- **GitLens** — optional

Install via the Extensions panel (**`Ctrl+Shift+X`**) and search the names above.

---

## Step 2 — Install and configure Playwright (no AI)

If the project already has a `package.json` (this one does), skip 2.1 and go to 2.2.

2.1. **Initialize a brand-new project** (skip if `package.json` exists):
```bash
npm init -y
npm init playwright@latest -- --quiet --browser=chromium --lang=ts
```

2.2. **Install dependencies** for the existing project:
```bash
npm install
npx playwright install chromium
```

2.3. **Verify the install:**
```bash
npx playwright --version
# expect: Version 1.4x.x
```

2.4. **Inspect the configuration** in [playwright.config.ts](../playwright.config.ts) — key fields:
- `testDir: './tests'` — Playwright picks up every `*.spec.ts` here
- `baseURL: 'http://localhost:8080'` — `page.goto('/')` resolves to the TravelApp
- `fullyParallel: true` — tests run in parallel
- `reporter: [['html', ...], ['list']]` — HTML report + console list

2.5. **Run the existing test suite** to confirm the toolchain works:
```bash
npm test
```

---

## Step 3 — Write your first test without AI

Two ways: write it by hand, or record it with `codegen`.

### 3a. By hand

3a.1. Create a file `tests/manual-first.spec.ts`:
```ts
import { test, expect } from '@playwright/test';

test('home page redirects to sign-in', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveTitle(/Sign In/);
  await expect(page.getByRole('textbox', { name: 'Email address' })).toBeVisible();
});
```

3a.2. Run only that test:
```bash
npx playwright test tests/manual-first.spec.ts
```

3a.3. **Locator basics** to remember:
- `page.getByRole('button', { name: 'Sign In' })` — preferred; semantic
- `page.getByLabel('Email address')` — for form inputs with associated labels
- `page.locator('#some-id')` — fallback when no semantic anchor exists
- Avoid `page.locator('div > .foo > span:nth-child(3)')` — brittle

### 3b. Record with `codegen`

3b.1. Open the recorder against the live app:
```bash
npx playwright codegen http://localhost:8080
```

3b.2. A browser window opens alongside an Inspector window. Click around — every action is transcribed into Playwright code in real time.

3b.3. Copy the generated code, paste into a `*.spec.ts` file, clean it up, and run with `npx playwright test`.

---

## Step 4 — Set up the Playwright MCP (natural-language browser control)

MCP (Model Context Protocol) lets Claude Code drive a real browser through tool calls. Once configured, you can ask Claude in plain English to navigate, click, type, etc., and it executes those actions in a Playwright-controlled browser.

4.1. **Verify the MCP config** at the project root: [.mcp.json](../.mcp.json) — already present in this project:
```json
{
  "mcpServers": {
    "playwright-test": {
      "command": "npx",
      "args": ["playwright", "run-test-mcp-server"]
    }
  }
}
```

4.2. **What this server provides:**
- `browser_*` tools — `navigate`, `click`, `type`, `snapshot`, `take_screenshot`, etc.
- `planner_*`, `generator_*` tools — used by the subagents in Step 6
- `test_run`, `test_debug` — execute Playwright tests inside the MCP session

4.3. **Approve the MCP server** the first time Claude Code launches in this folder:
```bash
cd /path/to/ts-automation
claude
```
Claude Code reads `.mcp.json` and prompts you to approve the `playwright-test` server. Click **Approve**.

4.4. **Approve specific tools as needed.** The first time Claude calls `mcp__playwright-test__browser_navigate`, you'll see a permission prompt. Approve it (or check **Allow always** if you trust the action).

4.5. **Verify the tools are available** by asking Claude:
> Can you take a snapshot of http://localhost:8080?

If MCP is wired correctly, Claude calls `browser_navigate` followed by `browser_snapshot` and shows you the page structure.

---

## Step 5 — Drive the browser through natural language

With MCP set up, you can describe actions in English. Claude translates them into MCP tool calls.

5.1. **Examples of natural-language prompts** that work once MCP is approved:

> Navigate to http://localhost:8080, sign in with `demo@travelapp.com` / `demo`, and tell me what fields the booking form has.

> On http://localhost:8080, fill the Email field with the demo account, click Sign In, then snapshot the page.

> Click the One Way radio on the booking page and report whether the Return Date field disappears.

5.2. **Behind the scenes** Claude:
- Reads your prompt
- Calls `browser_navigate` → `browser_snapshot` → `browser_type` → `browser_click` → ...
- Reports back what it saw

5.3. **Use this for exploration**, not for production tests. Natural-language drives are great for:
- Mapping out a feature you've never seen
- Discovering selector IDs and labels
- Reproducing a bug interactively

For real test code, use the agent workflow in Steps 6–9 below.

---

## Step 6 — Set up the Playwright subagents

Subagents are Claude Code's specialised assistants defined as Markdown files. This project ships three under [.claude/agents/](../.claude/agents/):

| Agent | File | Purpose |
|---|---|---|
| Planner | [playwright-test-planner.md](../.claude/agents/playwright-test-planner.md) | Explores the app, writes a test plan |
| Generator | [playwright-test-generator.md](../.claude/agents/playwright-test-generator.md) | Drives the browser, writes runnable spec code |
| Healer | [playwright-test-healer.md](../.claude/agents/playwright-test-healer.md) | Runs failing specs, fixes them until green |

6.1. **They're already wired up** because they live under `.claude/agents/` in the project root. No install step needed.

6.2. **Verify Claude can see them** — ask:
> List the available subagents.

You should see all three in the response.

6.3. **Invoke a subagent** by mentioning it in a prompt, two ways:
- **`@`-mention the file:**
  > `@.claude/agents/playwright-test-planner.md` create a plan for ...
- **Reference by name in plain English:**
  > Use the playwright-test-planner subagent to create a plan for ...

Both work. The first form is more explicit.

---

## Step 7 — Create a test plan with the planner agent

7.1. **Make sure the TravelApp server is running** on `localhost:8080`. The planner needs to explore it interactively.

7.2. **Invoke the planner.** Be specific about scope (what to cover, what to exclude) and the output path:
> `@.claude/agents/playwright-test-planner.md` create a test plan for the **flight search** flow — happy paths only, no validation cases. Save as `travel-app-flight-search-plan.md` in the specs folder.

7.3. **What the planner does:**
- Calls `planner_setup_page` to spin up a Playwright session
- Signs in with demo credentials
- Explores the relevant pages, capturing real selectors and behaviour
- Calls `planner_save_plan` to write a Markdown file

7.4. **Output format** the planner produces (consumed by the generator in Step 8):
```markdown
### 1. Flight Search — Happy Paths

**Seed:** `tests/seed.spec.ts`

#### 1.1. Round-trip JFK → LHR

**File:** `tests/flight-search/round-trip-jfk-lhr.spec.ts`

**Steps:**
  1. Navigate to /booking.html ...
    - expect: heading "Where do you want to fly?" is visible
  2. ...

**Expected:**
  - The booking confirmation modal appears with the correct route ...
```

7.5. **Tips for planning prompts:**
- Scope tightly — "search only", "round-trip happy paths", "1–6 passengers". Vague briefs produce bloated plans.
- Tell it what to **exclude** — "no validation", "no checkout".
- Ask for verification of UI behaviour you're unsure about — "verify whether the app supports per-leg cabin class".

---

## Step 8 — Generate test scripts with the generator agent

8.1. **Invoke the generator** with a reference to the plan and the scenario index:
> `@.claude/agents/playwright-test-generator.md` generate the test for scenario 1.1 in `specs/travel-app-flight-search-plan.md`.

8.2. **What the generator does:**
- Reads the plan file
- Calls `generator_setup_page` with the seed file
- Drives the browser through every step using `browser_*` tools
- Reads the recorded log via `generator_read_log`
- Calls `generator_write_test` to emit the spec file

8.3. **Patterns the generator (and you, when reviewing) should follow:**

8.3.1. **Use the auth hooks** — never inline sign-in in the test:
```ts
import { signInAsDemo, closeBrowserContext } from '../../fixtures/auth.hooks';

test.describe('...', () => {
  test.beforeEach(async ({ page }) => { await signInAsDemo(page); });
  test.afterEach(async ({ page }) => { await closeBrowserContext(page); });
  // ...
});
```

8.3.2. **Bootstrap btn-check radios are visually hidden** — click the label, not the input:
```ts
await page.locator('label[for="trip-oneway"]').click();
await expect(page.locator('#trip-oneway')).toBeChecked();
```

8.3.3. **Icon glyphs leave leading spaces in accessible names** — use case-insensitive regex:
```ts
await page.getByRole('button', { name: /search/i }).click();
```

8.3.4. **Don't hardcode dynamic values** (prices, flight numbers, totals) — assert structurally:
```ts
await expect(page.locator('#summary-class')).toHaveText('Business');  // stable
await expect(page.locator('#summary-total')).toBeVisible();           // dynamic — visibility only
```

8.4. **Run the generated test:**
```bash
npx playwright test tests/flight-search/round-trip-jfk-lhr.spec.ts
```

8.5. **If green** — done. **If red** — go to Step 9.

---

## Step 9 — Heal failing tests with the healer agent

9.1. **Invoke the healer** with the failing spec path:
> `@.claude/agents/playwright-test-healer.md` run `tests/flight-search/round-trip-jfk-lhr.spec.ts` and see what's the issue.

9.2. **What the healer does:**
- Calls `test_run` to capture the failure
- Calls `test_debug` to step through and snapshot the page state at the failure point
- Diagnoses the cause (locator drift, hardcoded value, hidden input, missing wait, etc.)
- Edits the spec
- Re-runs
- Repeats until green
- Marks `test.fixme()` only if it's confident the test is correct but the app is broken

9.3. **What you should do:**
- Read the healer's final summary — it lists every change with a before/after table
- Sanity-check the changes — sometimes the healer over-relaxes assertions; you can tighten them back up
- Commit the spec

9.4. **Common heals you'll see:**
- `getByText('Economy $750 per person')` → `locator('#class-economy')` (price decoupled)
- `' Search'` → `/search/i` (icon-glyph leading space removed)
- `getByText('1', { exact: true })` → `locator('#detail-passengers')` (ambiguous text removed)

---

## Step 10 — Visual comparison of a login page screenshot

Playwright has built-in screenshot regression — capture a baseline image, compare on every run, fail if pixels diverge beyond a threshold.

10.1. **Create a visual test** at `tests/visual/login-page.spec.ts`:
```ts
import { test, expect } from '@playwright/test';

test.describe('Visual regression', () => {
  test('login page matches baseline', async ({ page }) => {
    await page.goto('/');
    await expect(page).toHaveScreenshot('login-page.png', {
      fullPage: true,
      maxDiffPixelRatio: 0.01,  // tolerate 1 % pixel drift
    });
  });
});
```

10.2. **Generate the baseline** the first time:
```bash
npx playwright test tests/visual/login-page.spec.ts --update-snapshots
```
This creates `tests/visual/login-page.spec.ts-snapshots/login-page-chromium-darwin.png` (filename includes browser + OS).

10.3. **Subsequent runs** compare against the baseline:
```bash
npx playwright test tests/visual/login-page.spec.ts
```
Failure output includes three images in the HTML report: **expected**, **actual**, **diff** (red highlight on changed pixels).

10.4. **Refresh the baseline** when an intentional UI change lands:
```bash
npx playwright test tests/visual/login-page.spec.ts --update-snapshots
```
Then commit the updated PNG.

10.5. **Tune sensitivity** in `playwright.config.ts`:
```ts
expect: {
  toHaveScreenshot: {
    maxDiffPixelRatio: 0.02,
    threshold: 0.2,           // per-pixel colour tolerance, 0–1
  },
},
```

10.6. **Tips:**
- Baselines are **OS- and browser-specific**. CI must use the same OS family that generated them, or use Docker.
- Mask volatile regions (timestamps, animations) with `mask: [page.locator('.live-clock')]`.
- For a login page specifically, mask any rotating hero images or carousels.

---

## Recap — the end-to-end agent loop

```
plan ──► generate ──► run ──► (red?) ──► heal ──► run ──► green
 ▲                                                          │
 └──────── refine plan ◄─────────────────────────────────────┘
```

| Phase | Driver | Output |
|---|---|---|
| Plan | `playwright-test-planner` agent | `specs/<feature>-plan.md` |
| Generate | `playwright-test-generator` agent | `tests/<feature>/<scenario>.spec.ts` |
| Run | `npx playwright test` (or `npm test`) | Pass/fail report |
| Heal | `playwright-test-healer` agent | Edited spec that passes |
| Visual | Playwright's `toHaveScreenshot` | Baseline PNG + diff on regression |

---

## Reference

- Project root [README.md](../README.md) — concise per-day operations cheat sheet
- Playwright docs — https://playwright.dev/docs/intro
- MCP docs — https://modelcontextprotocol.io
- Claude Code subagents — https://docs.anthropic.com/en/docs/claude-code/sub-agents
