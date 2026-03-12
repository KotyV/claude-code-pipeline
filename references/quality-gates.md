# Quality Gates -- Automated verification before every commit

These gates are non-negotiable: code does not ship if these checks fail (except Gate 7 which is a warning).
Referenced by `/implement`, `/cleanup-push`.

---

## Gate 1: Lint + format (blocking)

```bash
# Backend -- must return 0 errors
[your linter] [source directory]
# If fail: [your linter] --fix [source directory], then re-check

# Frontend -- must return 0 errors
[your linter] [source directory] --max-warnings=0
```

**Verdict**: If lint fails, do NOT commit.

---

## Gate 2: Type check (blocking, threshold)

```bash
# Backend -- count errors
TYPE_ERRORS=$([your type checker] [source directory] 2>&1 | grep -c "error")
echo "Type errors: ${TYPE_ERRORS:-0}"

# Compare with baseline from pipeline-state.json
# Rule: TYPE_ERRORS <= baseline.type_errors
# New errors = BLOCKING

# Frontend -- must return 0
[your type checker]
```

**Verdict**: If new type errors appear, fix them before committing.

---

## Gate 3: Tests (blocking)

```bash
# Backend -- all must pass
[your test runner] [unit test directory]
# Expected: "X passed" with no "failed"

# Frontend
[your test runner]
```

**Verdict**: If a test fails, fix it. See `references/recovery-protocol.md`.

---

## Gate 4: File size (blocking)

```bash
# Check that no source file exceeds 300 lines (excluding tests)
find [source directory] -name "*.py" -not -path "*/test*" -exec wc -l {} + | sort -rn | head -10
# Adapt for your language: *.ts, *.go, *.rs, etc.
# Any file > 300 lines -> split BEFORE committing
```

**Verdict**: If file > 300 lines, refactor (see clean-code-guidelines.md section 2).

---

## Gate 5: Function size (blocking)

Manual but systematic verification:

```text
For each file MODIFIED in this commit:
1. Open the file
2. For each function/method:
   - Count lines of LOGIC (excluding docstrings, imports, blank lines)
   - If > 20 lines -> decompose into sub-functions
3. For each class:
   - Count public methods
   - If > 7 public methods -> extract into a sub-class or separate service
```

**Verdict**: If God Function detected, refactor BEFORE committing.

---

## Gate 6: No placeholders (blocking)

```bash
# Search for TODO, FIXME, placeholder implementations
grep -rn "TODO\|FIXME\|NotImplementedError\|implement later" [source directory] --include="*.py" --include="*.ts" --include="*.tsx" --include="*.go" --include="*.rs" || echo "Clean"

# Check for suspicious bare pass/return statements
# Python:
grep -rn "^\s*pass$" [source directory] --include="*.py" | grep -v "__init__\|except\|class.*:" || echo "Clean"
# TypeScript:
grep -rn "throw new Error.*not implemented" [source directory] --include="*.ts" --include="*.tsx" || echo "Clean"
```

**Verdict**: If placeholder found, implement or remove it.

---

## Gate 7: No magic strings for statuses (warning, non-blocking)

```bash
# Look for hardcoded status strings that should be enums
grep -rn '"pending"\|"running"\|"completed"\|"failed"' [source directory]/services/ [source directory]/api/ --include="*.py" --include="*.ts" || echo "Clean"
# If found -> replace with the corresponding enum (OrderStatus, TaskStatus, etc.)
```

---

## Gate 8: Basic security (blocking)

```bash
# No secrets in code
grep -rn "password\|secret\|api_key\|token" [source directory] --include="*.py" --include="*.ts" | grep -v "test\|\.env\|settings\|config\|verify_token\|_auth\|type\|interface" || echo "Clean"

# No print()/console.log() in production code
grep -rn "^\s*print(" [source directory] --include="*.py" || echo "Clean"
grep -rn "console\.log(" [source directory] --include="*.ts" --include="*.tsx" | grep -v "test\|spec" || echo "Clean"
```

**Verdict**: If hardcoded secret or debug statement found, fix it.

---

## Gate 9: Advanced secret detection (blocking)

Regex patterns to detect common secrets in code:

```bash
# Known API keys (OpenAI, Anthropic, AWS, GitHub, Slack)
grep -rn -E "sk-[a-zA-Z0-9]{20,}" [source directory] || echo "Clean"
grep -rn -E "sk-ant-[a-zA-Z0-9]{20,}" [source directory] || echo "Clean"
grep -rn -E "ghp_[a-zA-Z0-9]{36,}" [source directory] || echo "Clean"
grep -rn -E "AKIA[A-Z0-9]{16}" [source directory] || echo "Clean"
grep -rn -E "xox[bpsa]-[a-zA-Z0-9-]+" [source directory] || echo "Clean"

# Private keys
grep -rn "BEGIN.*PRIVATE KEY" [source directory] || echo "Clean"

# Connection strings with credentials
grep -rn -E "(postgresql|mysql|mongodb)://[^:]+:[^@]+@" [source directory] | grep -v "\.env\|settings\|example\|test" || echo "Clean"
```

**Verdict**: If secret detected, remove immediately. Store in `.env`, add to `.gitignore`.

---

## Gate 10: Sensitive files (blocking)

```bash
# Verify that .env is not tracked by git
git ls-files --cached | grep -E "\.env$|\.env\.local$|\.env\.production$" || echo "Clean"

# Verify that credential files are not tracked
git ls-files --cached | grep -E "\.pem$|\.key$|credentials\.json$|service-account" || echo "Clean"

# Verify that .gitignore contains sensitive patterns
grep -c "\.env" .gitignore > /dev/null 2>&1 && echo "OK: .env in gitignore" || echo "WARN: .env NOT in gitignore"
```

**Verdict**: If sensitive file tracked by git, run `git rm --cached`, add to `.gitignore`.

---

## Gate 11: Dangerous patterns (blocking)

```bash
# eval/exec with variables (Python)
grep -rn -E "eval\(|exec\(" [source directory] --include="*.py" | grep -v "test\|#.*eval" || echo "Clean"

# subprocess with shell=True (Python)
grep -rn "shell=True" [source directory] --include="*.py" || echo "Clean"

# dangerouslySetInnerHTML (React)
grep -rn "dangerouslySetInnerHTML" [source directory] --include="*.tsx" --include="*.ts" --include="*.jsx" || echo "Clean"

# Raw SQL with string interpolation (any language)
grep -rn -E 'text\(f"|text\(f'"'"'|`SELECT.*\$\{' [source directory] --include="*.py" --include="*.ts" || echo "Clean"

# pickle.load with untrusted input (Python)
grep -rn "pickle\.load\|pickle\.loads" [source directory] --include="*.py" | grep -v "test" || echo "Clean"

# yaml.load without SafeLoader (Python)
grep -rn "yaml\.load(" [source directory] --include="*.py" | grep -v "SafeLoader\|safe_load\|test" || echo "Clean"

# eval() in JavaScript/TypeScript
grep -rn -E "\beval\(" [source directory] --include="*.ts" --include="*.tsx" --include="*.js" | grep -v "test\|spec" || echo "Clean"
```

**Verdict**: If dangerous pattern found, refactor with the secure alternative.

---

## Full sequence before commit

```text
For each story before git commit:

1.  [your linter] [source directory]          -> 0 errors
2.  [your type checker] [source directory]    -> <= baseline errors
3.  [your test runner] [test directory]       -> 0 failed
4.  Modified files < 300 lines               -> verified
5.  Modified functions < 20 lines of logic   -> verified
6.  grep TODO/FIXME                          -> 0 results
7.  grep secrets/print                       -> 0 results
8.  Advanced secret detection (Gate 9)       -> 0 results
9.  Sensitive files not tracked (Gate 10)    -> 0 results
10. Dangerous patterns (Gate 11)             -> 0 results

If a gate fails: FIX it, then re-verify ALL gates.
Do not fix one gate and break another.
```

---

## Coverage by file type (for /qa-tests)

| File type | Minimum coverage | Target coverage |
|-----------|-----------------|----------------|
| Services (business logic) | 90% | 100% |
| Core algorithms / scoring | 100% | 100% |
| Routes / controllers (HTTP layer) | 75% | 85% |
| Schemas / DTOs (validation) | 60% | 75% |
| Auth (security) | 100% | 100% |
| Models (DB) | 50% | 70% |
| Utils / helpers | 80% | 90% |

### Test quality (not just quantity)

```text
Every test MUST have:
- At least 1 explicit assertion (not just "it doesn't crash")
- A name that describes the behavior tested (test_returns_404_when_item_not_found)
- Realistic test data (not "test", "abc", 123)

Minimum ratio per endpoint:
- 1 happy path
- 2 error cases (not found + validation)
- 2 edge cases (boundaries, degenerate inputs)
= Minimum 5 tests per endpoint
```
