# groundwork

Three Claude Code skills that add a BDD/Gherkin layer to the [obra/superpowers](https://github.com/obra/superpowers) workflow.

```
groundwork-discovery  →  .feature files  →  superpowers  →  groundwork-scaffold-interface  →  groundwork-verify
     (interview)         (source of truth)   (implement)       (draft SPEC-INTERFACE.md)       (step definitions)
                                                                        ↓                             ↓
                                                                  dev reviews                  CI runs BDD tests
```

---

## Why

Superpowers is good at implementing things, but it needs a precise spec to work from. Without one, the AI fills in the gaps with assumptions that may not match your intent.

These skills make the specification step explicit:
- `groundwork-discovery` forces the conversation that surfaces requirements, edge cases, and explicit exclusions — before any code is written
- `groundwork-scaffold-interface` reads the codebase and generates a draft `SPEC-INTERFACE.md` — the external contract of your system (endpoints, pages, auth flow, UI elements)
- `groundwork-verify` turns specs into executable Cucumber.js tests that verify the implementation matches the contract

The `.feature` files are not documentation. They are the contract between intent and implementation.

### Black-box testing by design

`groundwork-verify` never reads application code. It generates step definitions exclusively from `.feature` files and `SPEC-INTERFACE.md`. This prevents the tests from learning the implementation's patterns (including bugs) and replicating them. Only `groundwork-scaffold-interface` reads code — and its output is a draft that the dev must review and approve.

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

### `groundwork-scaffold-interface`

Reads the codebase and generates a **draft** `SPEC-INTERFACE.md` for dev review.

| Command | What it does |
|---|---|
| `/groundwork scaffold-interface` | Scan codebase and generate draft `SPEC-INTERFACE.md` |

Detects interface type, base URLs, auth mechanism, REST endpoints, web pages, and UI elements. Marks uncertain items with `# TODO: verify`. The dev reviews, corrects, and approves the file before running verify.

Output: `docs/specs/SPEC-INTERFACE.md` (draft)

### `groundwork-verify`

Generates Cucumber.js step definitions from `.feature` files. Tests are black-box — they only interact with the system through its external interface (`@rest`, `@cli`, `@web`, `@rest+web`). No application code is read.

| Command | What it does |
|---|---|
| `/groundwork verify [feature-name]` | Generate or update step definitions |

Requires `SPEC-INTERFACE.md` to exist. Runs a gap analysis first (missing, broken, obsolete), then generates only what's needed. For web features, outputs a **selector prompt** listing all required `data-testid` attributes — ready to paste into your coding agent.

On the first run it sets up the full `docs/specs/` scaffolding with `package.json`, Cucumber config, hooks, and `.env.test.example`.

Output: `docs/specs/step-definitions/<domain>/<feature>.steps.js`

---

## Workflow

```
1. /groundwork new <feature>              → interview → .feature file
2. /groundwork review                     → context block for Superpowers
3. Superpowers implements                 → code in repo
4. /groundwork scaffold-interface         → draft SPEC-INTERFACE.md → dev reviews and approves
5. /groundwork verify <feature>           → step definitions (+ selector prompt for web features)
6. CI runs: cd docs/specs && npm test
7. Found a bug or edge case?              → /groundwork extend first, then fix, then /groundwork verify
```

If BDD tests fail with network errors or missing elements (not assertion failures), `SPEC-INTERFACE.md` may be stale — update it and re-run `/groundwork verify`.

---

## File structure

```
docs/
  specs/
    CONVENTIONS.md              ← shared conventions
    SPEC-INTERFACE.md           ← external interface contract (endpoints, pages, UI elements)
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
```bash
npx skills add mogui/groundwork
```
