---
name: security-audit
allowed-tools: Bash(git diff:*), Bash(git status:*), Bash(git log:*), Bash(git show:*), Bash(git commit:*), Bash(git add:*), Read, Glob, Grep, Edit, Agent
description: >
  Audits codebase for security vulnerabilities across 6 categories:
  Auth & Authorization (25%), Injection (20%), Input Validation (20%),
  Sensitive Data & Logging (15%), Configuration & Dependencies (10%),
  AI/LLM Security (10%). Produces a 0-100 score with grades A-F.
  Auto-fixes critical and high vulnerabilities.
  Includes false-positive filtering with confidence scoring (>80% only).
  Use when user says "security audit", "check for vulnerabilities",
  "OWASP check", or invokes /security-audit.
  Do NOT use for functional testing — use /qa-tests instead.
  Do NOT use for PR-level security review — use /security-review instead.
---

# Security Audit — Vulnerability Assessment

Analyze the codebase to detect vulnerabilities. Score 0-100 with grades A-F.

-> See `references/security-rules.md` for secure patterns, decision tree, false-positive exclusions, and stack-specific precedents.

---

## Audit Categories (weighting)

| # | Category | Weight | Key points |
|---|----------|--------|------------|
| 1 | Auth & Authorization | 25% | JWT verification on every route, no hardcoded secrets, CORS config |
| 2 | Injection | 20% | SQL (ORM only), Command, Path Traversal, Deserialization, SSRF (external API calls) |
| 3 | Input Validation | 20% | Strict schema validation, size limits, UUID format, capped pagination |
| 4 | Sensitive Data & Logging | 15% | No PII/tokens in logs, structured logging, generic error messages |
| 5 | Config & Dependencies | 10% | Conditional CORS, security headers, dependency audit, non-root Docker, rate limiting (endpoints without rate limiting), CSRF (mitigated by Bearer token auth — no cookies = no CSRF) |
| 6 | AI/LLM Security | 10% | Prompt isolation, untrusted responses, timeouts |

---

## Scoring

| Severity | Points deducted |
|----------|-----------------|
| Critical | -25 |
| High     | -15 |
| Medium   | -8  |
| Low      | -3  |

| Grade | Score | Meaning |
|-------|-------|---------|
| A | 90-100 | Production ready |
| B | 80-89 | Minor improvements needed (minimum for deploy) |
| C | 70-79 | Below minimum — improvements required |
| D | 60-69 | Fixes required before deploy |
| F | <60 | Not deployable |

## Confidence Scoring

| Score | Action |
|-------|--------|
| 0.9-1.0 | Report — certain exploit |
| 0.8-0.9 | Report — clear pattern |
| 0.7-0.8 | Report with caveat |
| < 0.7 | DO NOT report (too speculative) |

---

## Your Mission

### Step 1: Codebase context

Identify security frameworks in place (auth library, ORM, validation), spot existing validation patterns, list high-risk files.

### Step 2: Scan the codebase

For each category: list files, verify each point, identify vulnerabilities, classify by severity, assign a confidence score.

### Step 3: Filter false positives

-> Apply false-positive filtering from references/security-rules.md (section 8).

### Step 4: Calculate the score

Initial score 100, subtract points per confirmed vulnerability (confidence >= 0.7).

### Step 5: Report

```markdown
## Security Audit Report

**Date**: YYYY-MM-DD | **Score**: XX/100 (Grade X) | **Feature**: [slug]
**Findings reported**: X (out of Y detected, Z excluded as false positives)

### Summary by category

| Category | Score | Issues |
|----------|-------|--------|
| Auth & Authorization | XX/25 | ... |
| Injection | XX/20 | ... |
| Validation | XX/20 | ... |
| Data & Logging | XX/15 | ... |
| Config & Deps | XX/10 | ... |
| AI/LLM Security | XX/10 | ... |

### Confirmed vulnerabilities

#### CRITICAL (confidence >= 0.9)
- **[Category] [Description]** — File: [path:line] — Confidence: X.X
  Impact: [...] — Scenario: [...] — Fix: [...]

#### HIGH / MEDIUM / LOW
[same format]

### False positives excluded (transparency)
| # | Pattern detected | Exclusion reason |
|---|-----------------|------------------|

### Priority recommendations
1. [Action 1]
```

### Step 6: Fix if critical findings

If critical or high vulnerabilities found: fix, commit `fix(security): [description]`, re-audit, verify no regression.

---

## Common False Positives — Decision Tree

-> See `references/security-rules.md` for the complete decision tree.

Most frequent cases:
1. CORS wildcard in dev -> OK if conditional by environment
2. JWT lib "vulnerable" -> OK if algorithm is fixed (e.g., RS256)
3. Stack traces in dev -> OK if debug mode is conditional
4. Raw SQL without user input -> OK
5. LLM responses in logs -> warning only (not CRITICAL)
6. CSRF with Bearer-token-only auth -> not applicable (no cookies = no CSRF vector)

---

## Usage

```bash
/security-audit                    # Full audit
/security-audit --feature [slug]   # Audit a feature
/security-audit --quick            # Critical + high only
```

## Next step

-> `/cleanup-push` to finalize, update docs, and push.
