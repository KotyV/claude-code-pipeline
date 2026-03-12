---
name: implement
description: >
  Transforms validated specs and architecture into working, tested,
  committed code following clean code principles and design patterns.
  Use when user says "implement", "code this", "build the stories",
  or invokes /implement. Each user story gets its own commit.
  Requires validated specs and architecture as input.
  Consult references/clean-code-guidelines.md for code quality rules.
  Do NOT use for test-only work — use /qa-tests instead.
---

# Implement — Specs → Complete Tested Code

Transform validated specs and architecture into functional, tested, committed code.

---

## Prerequisites

Read MANDATORY:

- Specs document (user stories to implement)
- Architecture document (technical blueprint)
- CLAUDE.md (conventions)
- `references/clean-code-guidelines.md` — SRP, patterns, sizes, anti-patterns, edge cases, logging, validation
- `references/domain-examples.md` — Concrete implementation examples

Preflight → see `references/preflight-checklist.md`.

---

## IMPLEMENTATION RULES

### Zero placeholders

```
❌ FORBIDDEN: TODO, FIXME, pass, "implement later", NotImplementedError
✅ CORRECT:  Complete, functional, tested code
```

### Follow existing patterns

BEFORE writing code: look at how a similar module is done in the codebase, copy the pattern, adapt.

### Structured logging (never print)

→ See `references/clean-code-guidelines.md` section 8.

### Error handling

→ See `references/clean-code-guidelines.md` section 6.

---

## Your Mission

### For each User Story in implementation order:

#### 0. Plan BEFORE writing

```
1. What functions will I create? (names + signatures)
2. Does each function do ONE thing? (SRP)
3. Is there existing code I can reuse?
4. Which design pattern? (Repository, Strategy, Factory...)
5. What edge cases must I handle? (null, empty, NaN, limits)
```

#### 1. Write the code

- Follow conventions from CLAUDE.md
- Type hints everywhere
- Handle edge cases FIRST in each function
- Apply SRP: 1 function = 1 responsibility, max 20 lines of logic
- Explicit names, named constants (no magic numbers)

#### 2. Quality gates

→ Run gates BEFORE testing — see `references/quality-gates.md`.

#### 3. Test

```bash
# At minimum: happy path + error case per endpoint/service
[your test runner] [test path]
```

If tests fail: analyze, fix, re-run. If persists after 3 attempts → see `references/recovery-protocol.md`.

#### 4. CHECKPOINT BEFORE COMMIT

```
"Final check for US-N:
- [ ] Tests pass (no regression)
- [ ] Type check OK
- [ ] Quality gates passed
- [ ] Code complete — no TODO/FIXME

Ready to commit this story?"
```

**DO NOT commit without explicit validation.**

#### 5. Commit

```bash
git add [story files]
git commit -m "feat: US-N — [short description]"
```

---

## Troubleshooting

→ See `references/recovery-protocol.md` for resuming after interruption or regression.

### Circular import

```
1. Identify the loop: A imports B imports A
2. Extract common interface into a separate file
3. Use TYPE_CHECKING for type-only imports
4. Rule: models should never import services
```

### File too large (> 300 lines)

Split into sub-modules by logical blocks, re-export from `__init__` if needed.

---

## Usage

```bash
/implement --feature [slug]
/implement --feature [slug] --story US-3
/implement --feature [slug] --mvp-only
```

## Next step

→ `/qa-tests --feature [slug]` to validate with the full test pyramid.
