---
name: groundwork-discovery
description: >
  Use BEFORE brainstorming — when the user wants to capture WHAT a feature should do as Gherkin scenarios.
  Trigger when the user says "I need to build X", "let's spec this out", "what should this feature do?",
  or wants to formally capture a new edge case or bug as a BDD scenario.
---

# groundwork-discovery

Elicits complete Gherkin scenarios through structured conversation — before any code is written.
The `.feature` files produced here are the source of truth for both Superpowers (implementation) and groundwork-verify (validation).

## Where this fits in the Superpowers flow

```
groundwork-discovery (WHAT should it do?) → brainstorming (HOW should we build it?) → writing-plans → implementation → groundwork-verify
```

This skill captures **observable behaviour** as Gherkin scenarios. It does NOT decide architecture, tech choices, or implementation approach — that's brainstorming's job. After discovery writes the `.feature` file, suggest the user continues with the brainstorming skill.

## When to use

- User wants to specify WHAT a feature should do — before deciding HOW to build it
- User says "I need to build X" — start here to capture behaviour, then hand off to brainstorming
- A bug or edge case was found and needs formal capture as a scenario
- User says "let's spec this out", "what should this feature do?", "what are the edge cases?"

## When NOT to use

- User already knows WHAT and wants to decide HOW (architecture, tech choices) → use brainstorming directly
- User already has `.feature` files and wants to generate tests → use groundwork-verify
- User wants to run or debug existing tests
- The task is pure implementation with no specification gap

## Quick reference

| Command | Purpose | Output |
|---|---|---|
| `/groundwork new <feature>` | Full discovery interview from scratch | `docs/specs/features/<domain>/<feature>.feature` |
| `/groundwork extend <feature>` | Append scenarios to existing feature | Same file, new scenarios appended |
| `/groundwork review [feature]` | Format features as Superpowers context | Formatted context block |

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

**After writing the file**, suggest continuing with the superpowers flow:
> "The feature spec is ready. To design HOW to build this, continue with the **brainstorming** skill."

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

**After appending**, suggest continuing with the superpowers flow:
> "New scenarios added. If this changes the implementation approach, continue with the **brainstorming** skill to revisit the design."

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

---

## Common mistakes

| Mistake | Fix |
|---|---|
| Writing UI-specific steps ("clicks button with id btn-submit") | Write behavioural steps ("attempts login with incorrect credentials") |
| Forgetting the interface tag (`@api`, `@cli`, `@web`) on Feature line | Always ask which interface the feature exposes — groundwork-verify depends on it |
| Asking multiple questions at once | One question per message — wait for the answer before continuing |
| Skipping phases because "it's a simple feature" | Run all phases — skip only when the user explicitly says "skip" or "move on" |
| Modifying existing scenarios in `/groundwork extend` | Append only — never change existing scenarios |
| Writing implementation details in scenarios | Scenarios describe observable behaviour, not how the system achieves it |
| Discussing architecture or tech choices during discovery | Discovery captures WHAT, not HOW — redirect architecture questions to brainstorming |
| Skipping the brainstorming handoff after writing the feature file | Always suggest continuing with brainstorming to design the implementation |
