# groundwork

Two Cursor skills that add a BDD/Gherkin layer to the [obra/superpowers](https://github.com/obra/superpowers) workflow.

```
groundwork-discovery  →  .feature files  →  superpowers  →  groundwork-verify
     (interview)         (source of truth)   (implement)     (step definitions)
                                                                     ↓
                                                           CI runs BDD tests
```

---

## Why

Superpowers is good at implementing things, but it needs a precise spec to work from. Without one, the AI fills in the gaps with assumptions that may not match your intent.

These skills make the specification step explicit:
- `groundwork-discovery` forces the conversation that surfaces requirements, edge cases, and explicit exclusions — before any code is written
- `groundwork-verify` turns those specs into executable Cucumber.js tests that verify the implementation actually does what was specified

The `.feature` files are not documentation. They are the contract between intent and implementation.

---

## Skills

### `groundwork-discovery`

Runs a Socratic interview to produce `.feature` files from a high-level description.

| Command | What it does |
|---|---|
| `/groundwork new <feature-name>` | Full discovery interview for a new feature |
| `/groundwork extend <feature-name>` | Adds scenarios to an existing feature (append only) |
| `/groundwork review [feature-name]` | Formats `.feature` files as context to paste into Superpowers |

The interview walks through: happy path → actors/permissions → input validation → system errors → idempotency → concurrency → rollback. You can say "I don't know yet" (`@pending`), "not relevant" (`@out-of-scope`), or "move on" at any point.

Output: `docs/specs/features/<domain>/<feature>.feature`

### `groundwork-verify`

Generates Cucumber.js step definitions from the `.feature` files. Tests are black-box — they only interact with the system through its external interface (`@api`, `@cli`, `@web`). No internal code is touched.

| Command | What it does |
|---|---|
| `/groundwork verify [feature-name]` | Generate or update step definitions |

Runs a gap analysis first (missing, broken, obsolete), then generates only what's needed. On the first run it sets up the full `docs/specs/` scaffolding with `package.json`, Cucumber config, hooks, and `.env.test.example`.

Output: `docs/specs/step-definitions/<domain>/<feature>.steps.js`

---

## Workflow

```
1. /groundwork new <feature>     → interview → .feature file
2. /groundwork review            → context block for Superpowers
3. Superpowers implements        → code in repo
4. /groundwork verify <feature>  → step definitions
5. CI runs: cd docs/specs && npm test
6. Found a bug or edge case?     → /groundwork extend first, then fix, then /groundwork verify
```

---

## File structure

```
docs/
  specs/
    CONVENTIONS.md              ← shared conventions, read by both skills
    features/
      <domain>/<feature>.feature
    step-definitions/
      <domain>/<feature>.steps.js
    support/
      hooks.js
      world.js
    package.json                ← cucumber.js (+ playwright if @web features exist)
    cucumber.js
    .env.test.example
```

---

## Installation

Install via the Cursor superpowers plugin. Both skills live in `skills/` and are picked up automatically.

Each skill reads `docs/specs/CONVENTIONS.md` on every invocation. If it doesn't exist yet, `groundwork-discovery` will offer to create it after the first `.feature` file is written.
