---
name: bug-fix
description: >
  TDD-driven bug fix pipeline: Diagnose, Reproduce (failing test), Fix, Verify (test green),
  Cleanup & Push. Use when user says "fix bug", "debug this", "there's an error",
  or invokes /bug-fix. Always starts with a failing test that reproduces the bug,
  then fixes the code until the test passes. Includes regression prevention.
  Do NOT use for new features — use /new-feature instead.
---

# Bug Fix — TDD-Driven Bug Resolution

Pipeline: Diagnose → Reproduce (RED) → Fix (GREEN) → Verify → Push.

---

## CRITICAL RULES

### Rule 1: Test FIRST — always

```
❌ FORBIDDEN: Fix the code then write a test after
✅ CORRECT:  Write failing test → fix code → test passes
```

The reproduction test is the MOST IMPORTANT deliverable. It prevents regression forever.

### Rule 2: Minimum changes

```
❌ FORBIDDEN: Refactor surrounding code "while we're at it"
✅ CORRECT:  Change ONLY what's necessary for the fix
```

### Rule 3: NEVER modify the test to make it pass

The test is right, the code is wrong.

### Rule 4: Two commits minimum

```
Commit 1: test: reproduce [bug description]     ← RED
Commit 2: fix: [bug description]                 ← GREEN
```

---

## Your Mission (Claude)

### Phase 0: Load context (MANDATORY)

Read project knowledge files (CLAUDE.md, etc.).
Run preflight → see `references/preflight-checklist.md`.
Create a fix branch: `git checkout -b fix/[slug]`

---

### Phase 1: DIAGNOSE — Understand the bug

1. **Collect symptoms**: exact error message, reproduction steps, environment, since when
2. **Locate the code**: which file/function, which data flow, which inputs trigger it
3. **Formulate root cause** in ONE sentence:
   "The bug occurs because [component X] does [action Y] instead of [action Z] when [condition C]."

#### CHECKPOINT

```
"## Diagnosis
**Symptom**: [what happens]
**Root cause**: [why]
**Files involved**: [list]
**Planned fix**: [approach]

Continue with reproducing the bug via a test?"
```

**DO NOT continue without validation.**

---

### Phase 2: REPRODUCE — Failing test (RED)

#### Choose the right test level

```
├── Bug in a service/pure logic → UNIT test
├── Bug in an API endpoint → INTEGRATION test
├── Bug in an external client → UNIT test with mock
├── Bug in a UI component → COMPONENT test
└── Bug across layers → Test at the lowest possible level
```

#### Write the test

The test MUST:
1. Reproduce EXACTLY the bug conditions
2. Have a descriptive name: `test_[action]_when_[condition]_[expected]`
3. Assert the EXPECTED behavior (not the broken behavior)
4. Be as small and isolated as possible

#### Verify it FAILS then commit

```bash
# Run the specific test — it MUST fail
[your test runner] [test path]::[test_name]

git add tests/
git commit -m "test: reproduce [bug description]"
```

#### CHECKPOINT

```
"RED test confirmed:
- Test: [name]
- Current: FAIL ✗ [error]
- Expected: PASS ✓ [correct behavior]

Proceed with fix?"
```

---

### Phase 3: FIX — Make the test pass (GREEN)

→ Consult `references/clean-code-guidelines.md` if the fix touches sensitive areas.

1. Change the MINIMUM code necessary
2. Follow existing conventions
3. Verify the test passes
4. Commit:

```bash
git add [modified files — NOT the tests]
git commit -m "fix: [description]

Root cause: [1-line explanation]
Reproduced by: [test name]"
```

---

### Phase 4: VERIFY — No regression

→ Run quality gates — see `references/quality-gates.md`.

```bash
# Full test suite — must be >= baseline
[your test runner] [all tests]

# Type check + lint
[your linter]
[your type checker]
```

---

### Phase 5: PUSH

```bash
git push -u origin fix/[slug]
# Create PR with bug/fix/test details
```

---

## Troubleshooting

### Reproduction test already passes
1. Verify conditions are EXACTLY those of the bug
2. Check that mocks simulate the failing behavior correctly
3. Check git log for a recent fix

### Fix breaks other tests
1. If tests check INCORRECT behavior (pre-bug) → update tests
2. If tests check CORRECT behavior → reduce fix scope
3. NEVER delete a test without understanding why it fails

### Bug impossible to reproduce in unit test
1. Go up a level: integration test
2. If external dependency: finer mock
3. Last resort: endpoint-level regression test

### Fix too complex (> 50 lines)
STOP → ask user. Probably: wrong diagnosis, deeper bug, or multiple entangled bugs.

→ See `references/recovery-protocol.md` for resuming after interruption.

---

## Usage

```bash
/bug-fix "Description of the bug"
/bug-fix --continue
/bug-fix "Description" --diagnose-only
/bug-fix "Description" --no-push
```
