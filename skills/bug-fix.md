---
name: bug-fix
description: >
  TDD-driven bug fix pipeline: Diagnose, Reproduce (failing test), Fix, Verify (test green),
  Security Review, Cleanup & Push. Use when user says "fix bug", "debug this", "there's an error",
  or invokes /bug-fix. Always starts with a failing test that reproduces the bug,
  then fixes the code until the test passes. Includes regression prevention and
  security review of the diff before push.
  Do NOT use for new features — use /new-feature instead.
---

# Bug Fix — TDD-Driven Bug Resolution

Pipeline: Diagnose → Reproduce (RED) → Fix (GREEN) → Verify + Security Review → Push.

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

**Goal**: identify root cause BEFORE writing code.

1. **Collect symptoms**: exact error message, reproduction steps, environment, since when
2. **Locate the code**: which file/function, which data flow, which inputs trigger it, why existing tests didn't catch the bug
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

**Goal**: write a test that proves the bug exists.

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
5. Use realistic data

#### Verify it FAILS then commit

```bash
# Run the specific test — it MUST fail
[your test runner] [test path]::[test_name]
```

**The test MUST fail.** If the test passes, the bug is not correctly reproduced.

```bash
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

**DO NOT continue without validation.**

---

### Phase 3: FIX — Make the test pass (GREEN)

**Goal**: minimum changes to turn the test GREEN.

→ Consult `references/clean-code-guidelines.md` and `references/security-rules.md` if the fix touches auth, inputs, or external APIs.

1. Change the MINIMUM code necessary
2. Follow existing conventions (CLAUDE.md)
3. Verify the test passes:

```bash
[your test runner] [test path]::[test_name]
```

4. Commit the fix:

```bash
git add [modified files — NOT the tests]
git commit -m "fix: [description]

Root cause: [1-line explanation]
Reproduced by: [test name]"
```

---

### Phase 3b: REFACTOR — Clean up dead code

After the fix, check if the change left dead code or made a file too large:

1. **Dead code**: unused imports, orphaned functions/variables from the fix
2. **Size**: if the fixed file exceeds 300 lines → consider splitting (separate commit)
3. **Duplication**: if the fix introduced duplicated code → extract into a shared function

```bash
# Detect unused imports in modified files
[your linter] --select unused-imports [modified files]
```

If refactoring was done, commit separately:

```bash
git commit -m "refactor: clean up dead code after [bug description]"
```

> Note: this step is lightweight for bug fixes. Do not refactor beyond the scope of the fix.
> If major refactoring is needed, note it and do it in a dedicated session.

---

### Phase 4: VERIFY — No regression

→ Run the relevant quality gates — see `references/quality-gates.md`.

Minimum for a bug fix:

```bash
# Full test suite — must be >= baseline
[your test runner] [all tests]

# Type check + lint
[your linter] [source directory]
[your type checker] [source directory]
```

If a gate fails: fix and re-verify.

#### Security review of the diff

Run `/security-review` to check that the fix does not introduce a vulnerability.
This runs a targeted review on the diff (not a full audit) — fast and focused.

If HIGH-severity findings are reported: fix before pushing.

---

### Phase 5: PUSH — Finalize

```bash
git push -u origin fix/[slug]
gh pr create --title "fix: [description]" --body "$(cat <<'EOF'
## Bug
**Symptom**: [what was happening]
**Root cause**: [why]
**Impact**: [who was affected]

## Fix
[Description of the correction in 2-3 sentences]

## Reproduction test
- `test_[name]` in `tests/[path]` — reproduces the exact bug

## Verification
- [x] RED test before fix
- [x] GREEN test after fix
- [x] Full suite: no regression
- [x] Type check + Lint: OK
EOF
)"

# Wait for CI and merge
gh pr checks [PR_NUMBER] --watch
gh pr merge [PR_NUMBER] --squash --delete-branch
```

If a check fails: diagnose, fix, re-push, re-verify.

---

### Phase 5b: Update the bug log (if your project tracks one)

If your project keeps a bug log (e.g., `docs/bug-log.md`, `CHANGELOG.md`, or similar), add an entry:

1. **Add an entry** with PR number, symptom, root cause, fix applied
2. **Flag recurring patterns** — if the bug matches an existing pattern (timeout, parsing, null cascade, race condition), note it so the pattern becomes visible over time
3. Update the "Last updated" date at the top of the file if present

This step is OPTIONAL — skip if your project does not maintain a bug log.

### Phase 6: Final report

```markdown
## Bug Fix Complete: [Title]

| Phase | Status | Details |
|-------|--------|---------|
| Diagnose | ✓ | Root cause identified |
| Reproduce | ✓ | RED test: [test name] |
| Fix | ✓ | GREEN test, [N] files modified |
| Refactor | ✓/⊘ | Dead code cleaned / Nothing to clean |
| Verify | ✓ | [N] tests, no regression |
| Security Review | ✓ | /security-review — no vulnerabilities |
| Push | ✓ | PR: [url] — merged |

### Commits
1. `test: reproduce [bug]`
2. `fix: [bug]`

### Tests
- Baseline: X tests → Final: X+1 tests → No regression: ✓
```

---

## Troubleshooting

### Reproduction test already passes
1. Verify conditions are EXACTLY those of the bug (same input, same state)
2. Check that mocks simulate the failing behavior correctly
3. If environment-specific: simulate the conditions (timeout, rate limit, missing data)
4. If the test truly passes: check git log for a recent fix

### Fix breaks other tests
1. If tests check INCORRECT behavior (pre-bug) → update the tests
2. If tests check CORRECT but incompatible behavior → reduce the fix scope
3. NEVER delete a test without understanding why it fails

### Bug impossible to reproduce in unit test
1. Go up a level: integration test
2. If external dependency: finer mock
3. If race condition: use concurrent calls in the test
4. Last resort: endpoint-level regression test

### Fix too complex (> 50 lines)
STOP → ask user. Likely: wrong diagnosis, deeper bug, or multiple entangled bugs.

→ See `references/recovery-protocol.md` for resuming after interruption.

---

## Usage

```bash
/bug-fix "Description of the bug"
/bug-fix --continue
/bug-fix "Description" --diagnose-only
/bug-fix "Description" --no-push
```
