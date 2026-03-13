# Review Checklist — Pre-Merge Analysis

Shared reference for `/security-review` and `/cleanup-push`.
Consult this file for every PR before merge.

---

## Instructions

Review the `git diff origin/main` output for the issues listed below. Be specific — cite `file:line` and suggest fixes. Skip anything that's fine. Only flag real problems.

**Two-pass review:**
- **Pass 1 (CRITICAL):** SQL & Data Safety, Race Conditions, LLM Trust Boundary. These block the merge.
- **Pass 2 (INFORMATIONAL):** All remaining categories. Included in the PR body but non-blocking.

**Output format:**

```
Pre-Landing Review: N issues (X critical, Y informational)

**CRITICAL** (blocking merge):
- [file:line] Problem description
  Fix: suggested fix

**Issues** (non-blocking):
- [file:line] Problem description
  Fix: suggested fix
```

If nothing found: `Pre-Landing Review: No issues found.`

Be terse. For each issue: one line describing the problem, one line with the fix. No preamble, no summaries.

---

## Pass 1 — CRITICAL (blocks merge)

### SQL & Data Safety

- String interpolation in SQL (even if values are `.to_i`/`int()` — use bind params or ORM)
- TOCTOU races: check-then-set patterns that should be an atomic `WHERE` + `UPDATE`
- `update_column`/`update()` bypassing validations on fields that have or should have constraints
- N+1 queries: eager loading missing for associations used in loops (`.options(selectinload(...))`, `.includes()`, `DataLoader`, etc.)

### Race Conditions & Concurrency

- Read-check-write without uniqueness constraint (e.g., `where(hash=x).first()` then `save()` without handling concurrent insert)
- `get_or_create` / `find_or_create_by` on columns without unique DB index — concurrent calls can create duplicates
- Status transitions without atomic `WHERE old_status = ? UPDATE SET new_status` — concurrent updates can skip or double-apply transitions
- `html_safe` / `dangerouslySetInnerHTML` / `v-html` on user-controlled data (XSS)

### LLM Output Trust Boundary

- LLM-generated values (emails, URLs, names) written to DB or passed to mailers without format validation. Add lightweight guards (`EMAIL_REGEXP`, `urlparse`, `.strip()`) before persisting.
- Structured tool output (arrays, dicts) accepted without type/shape checks before database writes.
- LLM responses interpolated into SQL, HTML, or shell commands without sanitization.

---

## Pass 2 — INFORMATIONAL (non-blocking)

### Conditional Side Effects

- Code paths that branch on a condition but forget a side effect on one branch. Example: item promoted to verified but URL only attached when a secondary condition is true — the other branch promotes without the URL.
- Log messages that claim an action happened but the action was conditionally skipped.

### Magic Numbers & String Coupling

- Bare numeric literals used in multiple files — should be named constants
- Error message strings used as query filters elsewhere (grep for the string — is anything matching on it?)

### Dead Code & Consistency

- Variables assigned but never read
- Version mismatch between PR title and VERSION/CHANGELOG files
- CHANGELOG entries that describe changes inaccurately
- Comments/docstrings that describe old behavior after the code changed

### LLM Prompt Issues

- 0-indexed lists in prompts (LLMs reliably return 1-indexed)
- Prompt text listing available tools/capabilities that don't match what's actually wired up
- Word/token limits stated in multiple places that could drift

### Test Gaps

- Negative-path tests that assert type/status but not the side effects (URL attached? field populated? callback fired?)
- Assertions on string content without checking format (e.g., asserting title present but not URL format)
- Missing `assert_not_called()` / `.expects(:something).never` when a code path should explicitly NOT call an external service
- Security enforcement features (blocking, rate limiting, auth) without integration tests verifying the enforcement path end-to-end

### Crypto & Entropy

- Truncation of data instead of hashing (last N chars instead of SHA-256) — less entropy, easier collisions
- `random.random()` / `Math.random()` for security-sensitive values — use `secrets` / `crypto.randomUUID()`
- Non-constant-time comparisons (`==`) on secrets or tokens — vulnerable to timing attacks

### Time Window Safety

- Date-key lookups that assume "today" covers 24h — report at 8am only sees midnight→8am under today's key
- Mismatched time windows between related features — one uses hourly buckets, another uses daily keys for the same data

### Type Coercion at Boundaries

- Values crossing language/serialization boundaries where type could change (numeric vs string) — hash/digest inputs must normalize types
- Hash/digest inputs that don't call `str()` / `.toString()` before serialization — `{ cores: 8 }` vs `{ cores: "8" }` produce different hashes

### View / Frontend

- Inline `<style>` blocks in components rendered in loops (re-parsed every render)
- O(n*m) lookups in views (`Array.find()` in a loop instead of a `Map`/index)
- Client-side filtering (`.filter()`) on DB results that could be a `WHERE` clause server-side (unless intentionally avoiding it)

---

## Gate Classification

```
CRITICAL (blocks merge):            INFORMATIONAL (in PR body):
├─ SQL & Data Safety                ├─ Conditional Side Effects
├─ Race Conditions & Concurrency    ├─ Magic Numbers & String Coupling
└─ LLM Output Trust Boundary        ├─ Dead Code & Consistency
                                     ├─ LLM Prompt Issues
                                     ├─ Test Gaps
                                     ├─ Crypto & Entropy
                                     ├─ Time Window Safety
                                     ├─ Type Coercion at Boundaries
                                     └─ View / Frontend
```

---

## Suppressions — DO NOT flag these

- "X is redundant with Y" when the redundancy is harmless and aids readability
- "Add a comment explaining why this threshold/constant was chosen" — thresholds change during tuning, comments rot
- "This assertion could be tighter" when the assertion already covers the behavior
- Consistency-only changes (wrapping a value in a conditional to match how another constant is guarded)
- "Regex doesn't handle edge case X" when the input is constrained and X never occurs in practice
- "Test exercises multiple guards simultaneously" — that's fine, tests don't need to isolate every guard
- Threshold changes that are tuned empirically and change constantly
- Harmless no-ops (e.g., `.filter()` on an element that's never in the array)
- **ANYTHING already addressed in the diff you're reviewing** — read the FULL diff before commenting
