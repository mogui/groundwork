---
name: groundwork-verify
description: >
  Use when the user wants to validate that implemented code matches its specifications,
  generate integration tests from feature files, or check if code still satisfies existing scenarios.
  Trigger after implementation completes a feature. Also use when the user asks
  "does the code do what we specified?" or "generate tests from the feature files".
---

# groundwork-verify

Generates Cucumber.js step definitions that test the system as a black box through its external interface.
Step definitions live in `docs/specs` — a self-contained layer with its own `package.json`, independent of the project's stack.

## When to use

- After implementation completes a feature and you need to verify it matches the spec
- User asks to generate tests from `.feature` files
- User asks "does the code do what we specified?"
- New scenarios were added via `/groundwork extend` and step definitions need updating

## When NOT to use

- User wants to define or design a feature → use groundwork-discovery
- User wants unit tests or internal code tests — this skill is black-box only
- No `.feature` files exist yet

## Quick reference

| Command | Purpose |
|---|---|
| `/groundwork verify <feature>` | Generate/update step definitions for one feature |
| `/groundwork verify` | Generate/update step definitions for all features |

| Feature tag | Tooling | Assertions via |
|---|---|---|
| `@api` | `node-fetch` | HTTP status, JSON body |
| `@cli` | `child_process.exec` | stdout, stderr, exit code |
| `@web` | Playwright | Page elements, navigation |

## First action

Read `docs/specs/CONVENTIONS.md` if it exists.
Then read existing step definitions in `docs/specs/step-definitions/` to understand the style already in use before generating new ones.

---

## Architecture

### Prerequisite

The system under test exposes a standard external interface. Step definitions never touch internal code — they interact exclusively through:

- **`@api`** — REST / GraphQL endpoints
- **`@cli`** — command-line process
- **`@web`** — browser UI

### Fixed toolchain

Cucumber.js for all cases. Playwright added if any feature is tagged `@web`.

```
docs/
  specs/
    package.json              ← cucumber.js (+ playwright if needed)
    cucumber.js               ← cucumber config
    CONVENTIONS.md
    features/
      <domain>/<feature>.feature
    step-definitions/
      <domain>/<feature>.steps.js
    support/
      hooks.js                ← Before/After setup and teardown
      world.js                ← shared state across steps
    .env.test.example         ← required env vars, no secrets
```

### Interface → tooling map

| Feature tag | HTTP client | Notes |
|---|---|---|
| `@api` | `node-fetch` | JSON request/response assertions |
| `@cli` | `child_process.exec` | spawn process, assert stdout/exit code |
| `@web` | Playwright | headless browser, behavioural assertions |

A single feature can mix `@api` and `@web` steps if needed.

---

## Command

### `/groundwork verify [feature-name]`

Generate or update step definitions. Without argument, processes all `.feature` files.

---

### Phase 1 — Interface discovery

Read all target `.feature` files and identify the interface tag (`@api`, `@cli`, `@web`).

If a feature file has no interface tag, ask before proceeding:
> "This feature has no `@api`, `@cli` or `@web` tag. Which interface should the tests use?"

Never assume the interface type.

Also check: is there a `BASE_URL` or entry point already configured somewhere? (`.env`, `README`, existing step definitions)

---

### Phase 2 — Gap analysis

Compare `.feature` files against existing step definitions in `docs/specs/step-definitions/`:

| Status | Action |
|---|---|
| **Missing** — scenario with no step definition | Generate |
| **Broken** — step definition referencing a changed endpoint or selector | Regenerate |
| **Obsolete** — step definition with no matching scenario | Report to user, do not delete |

Only generate what is missing or broken. Never rewrite working step definitions.

Report the gap analysis before generating:
```
Found 3 scenarios without step definitions:
  - auth/login.feature: "login with expired token"
  - payments/checkout.feature: "checkout with empty cart" (new)
  - payments/checkout.feature: "duplicate order submission" (new)
Generating...
```

---

### Phase 3 — Generate step definitions

For each step definition:

**`@api` pattern:**
```javascript
// Scenario: <scenario name from .feature>
Given('a registered user exists with email {string}', async function(email) {
  // TODO: adjust to your seed/fixture mechanism
  this.user = await createTestUser({ email });
});

When('they attempt login with incorrect credentials', async function() {
  this.response = await fetch(`${process.env.BASE_URL}/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email: this.user.email, password: 'wrong' })
  });
});

Then('the operation fails with an authentication error', async function() {
  assert.strictEqual(this.response.status, 401);
});
```

**`@web` pattern:**
```javascript
// Scenario: <scenario name from .feature>
Given('a registered user exists', async function() {
  // TODO: adjust to your seed/fixture mechanism
  this.user = await createTestUser();
});

When('they attempt login with incorrect credentials', async function() {
  await this.page.goto(`${process.env.BASE_URL}/login`);
  await this.page.fill('[data-testid="email"]', this.user.email);
  await this.page.fill('[data-testid="password"]', 'wrong');
  await this.page.click('[data-testid="submit"]');
  // TODO: adjust selectors to match your actual markup
});

Then('the operation fails with an authentication error', async function() {
  await expect(this.page.locator('[data-testid="error-message"]')).toBeVisible();
});
```

**`@cli` pattern:**
```javascript
// Scenario: <scenario name from .feature>
When('they run the export command', async function() {
  this.cliOutput = await exec(`node cli.js export --user=${this.user.id}`);
});

Then('the export succeeds', async function() {
  assert.strictEqual(this.cliOutput.exitCode, 0);
});
```

**Rules for all patterns:**
- Add `// Scenario: <name>` comment at the top referencing the `.feature` origin
- Mark with `// TODO:` anything requiring manual configuration (seed mechanisms, selectors, URLs)
- For `@pending` scenarios, generate a stub with `pending()` — do not skip silently
- For `@out-of-scope` scenarios, generate nothing

---

### Phase 4 — Setup (first run only)

If `docs/specs/package.json` does not exist:

1. Generate `docs/specs/package.json`:
```json
{
  "name": "specs",
  "private": true,
  "scripts": {
    "test": "cucumber-js",
    "test:tag": "cucumber-js --tags"
  },
  "dependencies": {
    "@cucumber/cucumber": "^10.0.0",
    "node-fetch": "^3.0.0"
  }
}
```
Add `"@playwright/test": "^1.0.0"` and `"playwright": "^1.0.0"` if any `@web` feature exists.

2. Generate `docs/specs/cucumber.js`:
```javascript
module.exports = {
  default: {
    paths: ['features/**/*.feature'],
    require: ['step-definitions/**/*.steps.js', 'support/**/*.js'],
    format: ['progress-bar', 'html:reports/cucumber-report.html'],
    parallel: 2
  }
}
```

3. Generate `docs/specs/support/world.js` with shared state (page instance for Playwright, auth tokens, last response)

4. Generate `docs/specs/support/hooks.js` with `Before` / `After` lifecycle hooks for setup and teardown

5. Generate `docs/specs/.env.test.example`:
```
BASE_URL=http://localhost:3000
# Add other required env vars here
```

6. Add to the project root `package.json` (if it exists):
```json
"test:bdd": "cd docs/specs && npm test"
```

7. Add `docs/specs/.env.test` to `.gitignore`

---

## Output summary

After generation, always report:

```
Generated:
  docs/specs/step-definitions/auth/login.steps.js        (3 scenarios)
  docs/specs/step-definitions/payments/checkout.steps.js  (2 scenarios)

Stubs (pending):
  docs/specs/step-definitions/auth/login.steps.js        → "login with biometric" (@pending)

Skipped:
  payments/checkout.feature → "guest checkout" (@out-of-scope)

Obsolete (no matching scenario — not deleted):
  docs/specs/step-definitions/auth/register.steps.js     → "register with invite code"

Next steps:
  1. Copy docs/specs/.env.test.example → docs/specs/.env.test and fill in BASE_URL
  2. Review // TODO: comments in generated files
  3. cd docs/specs && npm install && npm test
```

---

## Common mistakes

| Mistake | Fix |
|---|---|
| Rewriting working step definitions | Always run gap analysis first — only generate what is missing or broken |
| Assuming the interface type without checking the tag | Ask the user if no `@api`, `@cli`, or `@web` tag is present |
| Importing or calling internal project code in step definitions | Step definitions are black-box — interact only through the external interface |
| Skipping `@pending` scenarios silently | Generate a stub with `pending()` so they show up in test reports |
| Deleting obsolete step definitions without asking | Report obsolete definitions to the user — never delete automatically |
| Forgetting `// TODO:` markers on config-dependent code | Mark seed mechanisms, selectors, and URLs that need manual adjustment |
