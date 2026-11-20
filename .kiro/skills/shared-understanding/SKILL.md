---
name: shared-understanding
description: Reach shared understanding with the user in at most two batched question rounds, then write the workspace steering files product.md and structure.md. Use before starting a Kiro spec, when the user describes a new product or feature, or when the request is ambiguous or admits multiple reasonable interpretations.
---

# Shared Understanding (batched, credit-aware)

Goal: resolve every decision that changes the spec, using as few agent turns as possible, then persist the result as steering so every later spec phase reads it without re-asking.

## Rules

1. **Codebase first.** If code, config, or git history answers a question, read it instead of asking. On an empty workspace, skip this step.
2. **Batch, never drip.** All questions of a round go in ONE message. Never ask one question and wait.
3. **Max 2 rounds, max 15 questions in total.** Round 1 covers everything you can foresee. Round 2 exists only for questions that depend on round-1 answers. No round 3.
4. **Recommend every answer.** Each question carries your recommended answer and a one-line reason, so the user can reply "ok" to most of them.
5. **Ask only what changes the spec.** Skip cosmetic choices, naming, and hypothetical future requirements.

## Round format

Group questions by area, in this priority order. Omit empty areas.

1. **Scope** — what is in, what is explicitly out.
2. **Users and goals** — who uses it, what success looks like (measurable).
3. **Contract** — inputs, outputs, public surface, integrations.
4. **Data** — shape, source of truth, persistence.
5. **Edge cases and errors** — empty, invalid, concurrent, unavailable dependency.
6. **Structure** — packages/modules, their responsibility, allowed dependencies between them.
7. **Constraints** — stack, performance, security, compliance, deadlines.

```
## Round 1/2

### Scope
Q1. [question]
    Recommended: [answer] — [reason]
Q2. ...

### Structure
Q7. ...

Reply with the numbers you want to change, e.g. "Q2: no, ...; Q7: ...". Everything else is accepted as recommended.
```

## After the answers

- If no answer opens new branches → go straight to writing the files. Do not run round 2 just because it is allowed.
- If the user says "your call" / "stop" → write the files, recording every unanswered question under **Assumptions**.

## Output: write both files, then stop

Write to the workspace, overwriting only after showing the user what changes if the files already exist.

### `.kiro/steering/product.md`

```markdown
---
inclusion: always
---
# Product

## Goal
[one paragraph]

## Users
- [user type]: [what they need]

## In scope
- ...

## Out of scope
- ...

## Success criteria
- [measurable criterion]

## Decided behaviors
- [edge case / error] → [agreed behavior]

## Constraints
- ...

## Assumptions
- [open branch] → [default chosen] (not confirmed by the user)
```

### `.kiro/steering/structure.md`

```markdown
---
inclusion: always
---
# Structure

## Packages
| Package | Responsibility | May depend on |
|---|---|---|
| `name` | [one line] | `other`, ... |

## Rules
- Every task touches at most 3 packages from the table above, named explicitly.
- Dependencies not listed in "May depend on" are forbidden.
- A new package requires updating this table first.

## Stack
- [language, framework, key libraries — only what is decided]
```

## Exit

Reply with one line per file written, the number of assumptions recorded, and this handoff:

> Steering ready. Open a new chat and start the spec (requirements → design → tasks).

Do not start the spec, the design, or any implementation in this session.