# TravelApp Playwright Test Suite

End-to-end tests for the TravelApp demo, generated and maintained with the help of three Claude Code subagents:

| Agent | Role |
|---|---|
| `playwright-test-planner` | Explores the live app, produces a Markdown test plan |
| `playwright-test-generator` | Reads a plan scenario, drives the browser, writes a runnable spec |
| `playwright-test-healer` | Runs failing specs, diagnoses, fixes, retries until green |

---

## 1. Prerequisites

- **Node.js 18+** (required by `@playwright/test`)
- **The TravelApp dev server running on `http://localhost:8080`**
  Start it however the workshop instructions specify (Docker, static server, etc.). Tests will fail with `ERR_CONNECTION_REFUSED` if it's not up. Verify with:
  ```bash
  curl -sS -o /dev/null -w "%{http_code}\n" http://localhost:8080/
  # expect: 200
  ```

## 2. Install

```bash
npm install
npx playwright install chromium     # one-time browser download
```

## 3. Run tests

| Command | What it does |
|---|---|
| `npm test` | Run all tests headless |
| `npm run test:headed` | Run with the browser visible |
| `npm run test:ui` | Playwright's interactive UI mode |
| `npm run test:debug` | Run with the Playwright Inspector |
| `npm run report` | Open the HTML report from the last run |

Run a single spec:

```bash
npx playwright test tests/one-way-booking/one-way-1pax-jfk-lhr.spec.ts
```

## 4. Project structure

```
.
├── .claude/agents/         # Subagent definitions (planner, generator, healer)
├── fixtures/
│   └── auth.hooks.ts       # Shared sign-in / teardown helpers
├── pages/                  # (reserved for Page Objects)
├── specs/                  # Markdown test plans (one per feature)
├── tests/
│   ├── seed.spec.ts        # Recording-time seed for the generator agent
│   ├── one-way-booking/    # Generated specs (one-way scenarios)
│   └── round-trip-booking/ # Generated specs (round-trip scenarios)
├── playwright.config.ts
└── tsconfig.json
```

## 5. Adding a new test — the agent workflow

The repeatable loop is **plan → generate → run → heal**. Each step is a separate subagent invocation.

### Step 1 — Plan the scenarios

Ask Claude Code:

> `@.claude/agents/playwright-test-planner.md` create a test plan for `<feature>` and save as `<file>` in the specs folder.

The planner signs into the live app, explores the relevant pages, and writes a structured Markdown plan to `specs/<file>.md` containing:

- Application overview (what was actually observed)
- One or more numbered scenarios with `**Steps:**` and `**Expected:**` blocks
- The seed file reference (`**Seed:** tests/seed.spec.ts`)

**Tip:** be specific in the request — scope (e.g. "search only", "happy paths only"), variations to cover (passenger counts, routes), and exclusions ("no validation", "no round-trip"). Vague briefs lead to bloated plans.

### Step 2 — Generate the spec

Ask Claude Code:

> `@.claude/agents/playwright-test-generator.md` generate test scripts for `<plan-file>`.

The generator picks one scenario from the plan, drives the browser through every step in real time, and writes the recorded actions to the path specified in the scenario's `**File:**` field.

**Best practices to bake into the brief** (lessons from this codebase):

- Use the auth hooks from `fixtures/auth.hooks.ts` — don't inline sign-in:
  ```ts
  test.beforeEach(async ({ page }) => { await signInAsDemo(page); });
  test.afterEach(async ({ page }) => { await closeBrowserContext(page); });
  ```
- **Bootstrap `btn-check` radios are visually hidden** — click the visible `<label>`, not the input:
  ```ts
  page.locator('label[for="trip-oneway"]').click();
  expect(page.locator('#trip-oneway')).toBeChecked();
  ```
- **Icon glyphs leave leading spaces in accessible names** — always use case-insensitive regex:
  ```ts
  page.getByRole('button', { name: /search/i });   // not ' Search'
  ```
- **Cabin class is card-based, not radios** — use IDs: `#class-economy`, `#class-business`, `#class-first`.
- **Don't hardcode dynamic values** (prices, flight numbers, totals). Mock data shifts between sessions. Assert structurally:
  ```ts
  expect(page.locator('#summary-class')).toHaveText('Business');  // stable
  expect(page.locator('#summary-total')).toBeVisible();           // dynamic — visibility only
  ```

### Step 3 — Run the spec

```bash
npx playwright test path/to/new-spec.spec.ts
```

If green, you're done. If red, go to step 4.

### Step 4 — Heal failures

Ask Claude Code:

> `@.claude/agents/playwright-test-healer.md` run `<spec>` and see what's the issue.

The healer runs the test, captures the failure, fixes the code, and re-runs until it passes (or marks `test.fixme()` if it's confident the test is correct but the app is broken). Common fixes the healer handles automatically:

- Brittle `getByText` swapped for `getByRole` / IDs
- Hardcoded values replaced with structural assertions
- Hidden-input clicks redirected to labels
- Missing waits replaced with `expect(...).toBeVisible()`

## 6. Authentication pattern

The demo app redirects unauthenticated users to a sign-in page. Every spec needs a signed-in browser before its first assertion.

The runtime mechanism is **NOT** [tests/seed.spec.ts](tests/seed.spec.ts) — that file only exists to seed the **generator agent** during recording. At runtime, specs use the helpers from [fixtures/auth.hooks.ts](fixtures/auth.hooks.ts):

```ts
import { signInAsDemo, closeBrowserContext } from '../../fixtures/auth.hooks';

test.beforeEach(async ({ page }) => { await signInAsDemo(page); });
test.afterEach(async ({ page }) => { await closeBrowserContext(page); });
```

Demo credentials: `demo@travelapp.com` / `demo` (hard-coded inside `signInAsDemo`).

## 7. Configuration

Key bits in [playwright.config.ts](playwright.config.ts):

- `baseURL: 'http://localhost:8080'` — `page.goto('/')` resolves to the TravelApp.
- `fullyParallel: true` — specs run in parallel (the auth hook gives each test its own clean context).
- `retries: 0` locally, `2` on CI.
- `trace: 'on-first-retry'`, `screenshot: 'only-on-failure'`, `video: 'retain-on-failure'` — debugging artifacts captured automatically.

## 8. Troubleshooting

| Symptom | Likely cause |
|---|---|
| `ERR_CONNECTION_REFUSED` at `localhost:8080` | TravelApp dev server isn't running |
| First assertion fails because heading `Where do you want to fly?` is missing | Spec skipped sign-in — wire up `signInAsDemo` in `beforeEach` |
| `getByText('Economy $750 per person')` fails after a while | Mock prices shifted — replace with structural assertion (`#class-economy` click + `#summary-class` text) |
| Bootstrap radio click does nothing | Input is hidden — click `label[for="..."]` instead |
| `' Search'` button not found | Icon glyph in accessible name — use `/search/i` regex |
| Test passes via MCP `test_run` but fails via `playwright test` | The MCP runner skips project `dependencies` — keep auth in `beforeEach`, not in a separate `setup` project |

## 9. Useful element IDs in the TravelApp

Discovered during agent exploration — prefer these over text-based locators when assertions need to be stable:

- **Search form:** `#trip-roundtrip`, `#trip-oneway`
- **Search results:** `#results-date`, `#select-flight-btn-outbound-<n>`, `#select-flight-btn-return-<n>`
- **Flight details:** `#class-economy`, `#class-business`, `#class-first`, `#summary-class`, `#summary-total`, `#detail-passengers`, `#detail-flight-number`, `#detail-dep-date`
- **Nav:** `#nav-user-name`
