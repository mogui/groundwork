---
name: groundwork-discovery
description: >
  Conducts a structured Socratic interview to elicit Gherkin BDD scenarios from a high-level feature description.
  Use this skill whenever the user wants to define, specify, or design a new feature before implementation —
  even if they just say "I need to build X" or "let's design this feature" or "help me think through this functionality".
  Also use when the user discovers a new edge case or bug during use and wants to formally capture it.
  Outputs versioned .feature files that serve as the source of truth for implementation and testing.
  Commands: /groundwork new, /groundwork extend, /groundwork review
---

# groundwork-discovery

Elicits complete Gherkin scenarios through structured conversation — before any code is written.
The `.feature` files produced here are the source of truth for both Superpowers (implementation) and groundwork-verify (validation).

## First action

Read `docs/specs/CONVENTIONS.md` if it exists. If not, use the defaults in this skill and offer to create it at the end.

---

## Commands

### `/groundwork new <feature-name>`

Discovery from scratch for a new feature.

**Flow:**

1. Ask for a high-level description of the feature in one or two sentences
2. Enter interview mode — one question at a time, in this order:

| Phase | Question focus |
|---|---|
| Happy path | Walk me through the ideal use of this feature step by step |
| Actors | Who can use this feature? Are there different roles with different permissions? |
| Input validation | What input does this feature receive? What can be malformed, empty, or out of range? |
| System errors | What can go wrong outside the user's control? (network, database, third-party services) |
| Idempotency | What happens if the operation is repeated? Is it safe to call twice? |
| Concurrency | What if two users trigger this simultaneously? |
| Rollback | Can this operation be undone? What is the expected behaviour if it fails halfway? |

3. For each answer, build the Gherkin scenario internally
4. When all phases are covered (or explicitly skipped), write the `.feature` file

**Interview rules:**

- Ask one question at a time. Wait for the answer before continuing.
- Keep questions concrete and provocative, not abstract. Instead of "are there error cases?" ask "what happens if the user submits the form while already logged in on another device?"
- Stop a phase when: the user gave a concrete behaviour, explicitly excluded the case, or the next question would be redundant with a previous answer

**Escape hatches — recognise these responses at any point:**

| User says | Behaviour |
|---|---|
| "I don't know yet" / "not sure" | Tag scenario `@pending`, continue to next phase |
| "doesn't matter" / "not relevant" | Tag case `@out-of-scope` with a brief note, continue |
| "move on" / "next" / "skip" | Close current phase, move to next |
| "that's enough" / "done" | End interview, write file |

**Output:** `docs/specs/features/<domain>/<feature-name>.feature`

---

### `/groundwork extend <feature-name>`

Add new scenarios to an existing feature — typically edge cases discovered during use.

**Flow:**

1. Read the existing `.feature` file
2. Summarise the scenarios already covered (2-3 lines max)
3. Ask what new case the user discovered or wants to add
4. Enter focused interview mode — only ask about gaps not already covered
5. Append new scenarios to the file

**Critical rule:** Never modify existing scenarios. Append only.

---

### `/groundwork review [feature-name]`

Prepare `.feature` files as context for a Superpowers session.

**Flow:**

1. List all available `.feature` files under `docs/specs/features/`
2. If no argument given, ask which features are relevant to the current session
3. Output a formatted context block ready to paste into Superpowers

---

## Gherkin writing rules

### Granularity

Scenarios must describe **observable behaviour**, not implementation details or UI specifics.

```gherkin
# Wrong — fragile, breaks on every UI change
When the user clicks the button with id "btn-submit"
Then they see "Password incorrect" in red below the password field

# Right — behavioural, survives refactoring
When the user attempts login with incorrect credentials
Then the operation fails with an authentication error
```

### Interface type tag

Every `.feature` file must have a tag on the `Feature:` line declaring the interface type.
This is consumed by groundwork-verify to select the right toolchain.

```gherkin
@api        # REST or GraphQL
@cli        # command-line interface
@web        # browser-based frontend
```

### Scenario tags

| Tag | Meaning |
|---|---|
| `@pending` | Identified but behaviour not yet defined |
| `@out-of-scope` | Explicitly excluded — decision documented |
| `@wip` | In progress, not yet stable |
| `@critical` | Blocking — if this fails, the feature is not shippable |

---

## Output format

```gherkin
@api
Feature: <feature name>
  <one-line description>

  Background:
    Given <shared precondition if any>

  @critical
  Scenario: <happy path name>
    Given <initial state>
    When <action>
    Then <observable outcome>
    And <additional outcome if needed>

  Scenario: <edge case name>
    ...

  @pending
  Scenario: <case not yet defined>
    # TODO: define expected behaviour

  @out-of-scope
  Scenario: <excluded case>
    # Decision: <brief reason>
```

---

## File structure

```
docs/
  specs/
    CONVENTIONS.md
    features/
      <domain>/
      <feature-name>.feature
```

If the `docs/specs/` directory doesn't exist, create it. If `CONVENTIONS.md` doesn't exist, offer to create it after the first `.feature` file is written.
