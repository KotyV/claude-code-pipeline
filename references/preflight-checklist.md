# Preflight Checklist -- Validation before starting

Mandatory verification BEFORE starting any pipeline phase.
Referenced by `/implement`, `/qa-tests`, `/new-feature`.

---

## Quick checklist (copy-paste)

```bash
# 1. Git clean?
git status
# Expected: "nothing to commit, working tree clean"
# If dirty: commit or stash before continuing

# 2. Branch up to date?
git pull origin main --dry-run
# If behind: git pull origin main

# 3. Dependencies installed?
# [your package manager] -- verify deps are in sync
# Examples:
#   pip list | head -5
#   npm ls --depth=0 2>&1 | tail -5
#   go mod verify
#   cargo check

# 4. Test baseline (unit tests mandatory)
[your test runner] [unit test directory] 2>&1 | tail -5
# Note: X passed, Y failed
# If Y > 0: STOP -- fix existing tests BEFORE starting new work
# Integration tests (optional, skip if no local DB):
# [your test runner] [integration test directory] 2>&1 | tail -5

# 5. Type check / lint baseline
[your linter] [source directory] 2>&1 | tail -3
[your type checker] [source directory] 2>&1 | tail -3
# Note the number of pre-existing errors (do not add new ones)
```

---

## Decisions based on results

### Git dirty

```text
Case 1: Files unrelated to the current feature
  -> git stash -m "WIP: unrelated changes"
  -> Continue the pipeline
  -> git stash pop after the feature

Case 2: Files from a previous feature not yet pushed
  -> STOP -- finish and push the previous feature
  -> Then start the new one

Case 3: Modified config files (.env, settings)
  -> Verify it is not a secret
  -> Commit if OK: git commit -m "chore: update local config"
```

### Test baseline failures

```text
Case 1: Flaky tests (pass 1 out of 2 times)
  -> Re-run 3 times: [your test runner] -x [specific test file]
  -> If fail < 3 times: note as flaky, continue with caution
  -> If fail 3/3: it is a real bug, fix BEFORE starting

Case 2: Tests broken since the last merge
  -> git log --oneline -5 to identify the guilty commit
  -> Fix and commit: fix(tests): repair broken test from [commit]
  -> Re-verify baseline

Case 3: Tests that depend on infrastructure (DB, Redis, etc.)
  -> Verify infrastructure is running (docker compose up, etc.)
  -> Ignore integration tests if no local DB
  -> Unit tests MUST pass without infrastructure
```

### Type check with pre-existing errors

```text
Rule: NEVER increase the number of type errors.
1. Count current errors: [your type checker] [source directory] 2>&1 | grep -c "error"
2. Note the number (e.g., "6 errors")
3. After implementation: same command
4. If errors > baseline: fix before committing
5. If errors == baseline: OK, they are pre-existing
```

---

## Baseline template for pipeline-state.json

```json
{
  "baseline": {
    "tests_passed": 76,
    "tests_failed": 0,
    "type_errors": 6,
    "lint_errors": 0,
    "git_status": "clean",
    "branch": "main",
    "last_commit": "abc1234"
  }
}
```
