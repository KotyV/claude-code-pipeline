# Recovery Protocol -- Resuming after interruption or failure

How to detect, diagnose, and resume when something breaks mid-pipeline.
Referenced by `/new-feature`, `/implement`.

---

## 1. Detecting current state

When resuming an interrupted pipeline (`/new-feature --continue`):

```text
READ IMMEDIATELY:
1. .product/pipeline-state.json -> Where were we?
2. git log --oneline -10        -> Which commits were made?
3. git status                   -> Is there uncommitted work?
4. [your test runner] -q        -> Do tests still pass?
```

### Decision tree

```text
pipeline-state.json exists?
+-- YES -> Read current_step
|   +-- discovery/specs/architecture -> Resume that step (no code written)
|   +-- implement -> Check stories_completed vs stories_total
|   |   +-- All completed -> Move to qa-tests
|   |   +-- Partially -> Resume at story N+1
|   +-- qa_tests -> Re-run the tests
|   +-- security -> Re-run the audit
|   +-- push -> Resume the cleanup
+-- NO -> No pipeline in progress, start from scratch
```

---

## 2. Resuming by phase

### Resuming /implement (most common case)

```text
Situation: implement crashed at story 3 out of 6

1. Check committed stories:
   git log --oneline | grep "feat("
   -> Should list commits for US-1, US-2

2. Verify committed stories hold:
   [your test runner] -q
   -> If pass: resume at US-3
   -> If fail: see "Regression after interruption" below

3. Check state of uncommitted code:
   git diff --stat
   -> If US-3 code is in progress: evaluate if salvageable
   -> If too broken: git checkout -- . and restart US-3

4. Resume:
   Update pipeline-state.json with current stories_completed
   Continue at US-3 normally
```

### Regression after interruption

```text
Situation: tests that were passing before no longer pass

1. Identify the guilty commit:
   git log --oneline -5
   git stash  # save work in progress
   [your test runner] -q
   -> If pass: the problem is in the uncommitted code
   -> If fail: bisect to find the commit

2. If uncommitted code is the culprit:
   git stash pop
   Analyze the diff: git diff
   Fix the code, not the test
   Re-run tests

3. If a commit is the culprit:
   Option A: Fix in a new commit (preferred)
     -> fix(app): repair regression from US-N
   Option B: Revert the commit if the story is too broken
     -> git revert [commit-hash]
     -> Redo the story cleanly
   NEVER Option C: modify the test so it passes
```

### Resuming /qa-tests

```text
Situation: QA tests reveal a bug in the code

1. Identify the responsible story:
   Which endpoint/service is at fault?
   Which commit introduced it?

2. Fix in the service (not in the test):
   fix(app): US-N -- handle [edge case]

3. Re-run the full QA suite
   -> Previous tests must still pass
   -> The new test must pass too

4. NEVER delete a failing QA test
   -> The test is right, the code is wrong
```

### Resuming /security-audit

```text
Situation: a security fix breaks tests

1. Is the fix correct?
   -> If yes: adapt the tests to the new security constraints
   -> If no: revise the fix

2. Example: you add strict input validation
   -> Tests that sent invalid payloads now fail
   -> This is NORMAL: adapt the tests, not the fix
```

---

## 3. Retry limits

```text
RULE: 3 attempts max per story/step

Attempt 1: Try the initial approach
Attempt 2: Alternative approach (different pattern, different structure)
Attempt 3: Simplified approach (reduce the story scope)

If 3 failures:
  -> STOP
  -> Document the blocker in pipeline-state.json:
    "stories": {
      "US-3": {
        "status": "blocked",
        "attempts": 3,
        "blockers": ["Description of the problem"],
        "workaround": "Possible if we split into 2 stories"
      }
    }
  -> Ask the user for help
  -> Continue with subsequent stories if they are independent
```

---

## 4. Saving partial progress

```text
BEFORE attempting a risky fix:
1. git stash -m "WIP: US-N before risky fix"
2. Attempt the fix
3. If OK: git stash drop
4. If KO: git stash pop (restore previous state)

RULE: never lose code that was working.
Prefer a temporary WIP commit to lost code.
```

---

## 5. Escalating to the user

When to escalate (do not waste time):

```text
ESCALATE IMMEDIATELY if:
- Infrastructure error (Docker, DB, cache service down)
- Non-trivial merge conflict (> 3 files)
- Missing undocumented dependency
- Unexpected behavior from a third-party library
- Business question ("is this use case valid?")

DO NOT ESCALATE if:
- Failing test (analyze first)
- Type error (fix or ignore with a reason)
- File too large (refactor)
- Circular import (reorganize)
```
