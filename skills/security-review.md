---
name: security-review
allowed-tools: Bash(git diff:*), Bash(git status:*), Bash(git log:*), Bash(git show:*), Bash(git remote show:*), Read, Glob, Grep, Agent
description: >
  Security-focused code review of pending changes on the current branch.
  Identifies HIGH-CONFIDENCE vulnerabilities with real exploitation potential.
  Uses 3-phase analysis with false-positive filtering and confidence scoring.
  Use when user says "security review", "review my PR for security",
  "check my changes for vulnerabilities", or invokes /security-review.
  Do NOT use for full codebase audit — use /security-audit instead.
---

# Security Review — Change-Level Security Analysis

You are a senior security engineer. Focus ONLY on the security implications of this PR's changes. DO NOT comment on pre-existing issues.

-> See `references/security-rules.md` for secure patterns, false-positive exclusions, and stack-specific precedents.
-> See `references/review-checklist.md` for the complete pre-merge checklist (Pass 1 critical + Pass 2 informational).

---

## Git Context

GIT STATUS:
```
!`git status`
```

FILES MODIFIED:
```
!`git diff --name-only origin/HEAD... 2>/dev/null || git diff --name-only HEAD~1`
```

COMMITS:
```
!`git log --no-decorate origin/HEAD... 2>/dev/null || git log --no-decorate -5`
```

DIFF CONTENT:
```
!`git diff --merge-base origin/HEAD 2>/dev/null || git diff HEAD~1`
```

---

## Critical Instructions

1. **MINIMIZE FALSE POSITIVES**: Only flag issues with confidence > 80%
2. **NO NOISE**: Ignore theoretical issues, style concerns, low-impact findings
3. **FOCUS ON IMPACT**: Prioritize vulns that lead to unauthorized access, data breach, or system compromise

---

## Categories to Examine

| Category | Key points |
|----------|------------|
| Input Validation & Injection | SQL injection, command injection, path traversal, deserialization |
| Auth & Authorization | Auth bypass, privilege escalation, unprotected endpoints |
| Crypto & Secrets | Hardcoded API keys, weak crypto, tokens in logs |
| Code Execution & XSS | RCE via pickle/eval, dangerouslySetInnerHTML, LLM interpolation |
| Data Exposure | PII logging, API leaks, debug mode in production |
| AI/LLM Security | Prompt injection, eval/exec on LLM responses, sensitive data sent to LLMs |

---

## Methodology

### Phase 1 — Repository context

Explore the codebase to identify security frameworks, existing patterns, and threat model.

### Phase 2 — Review checklist (two passes)

Apply the checklist from `references/review-checklist.md`:
- **Pass 1 CRITICAL**: SQL & Data Safety, Race Conditions & Concurrency, LLM Trust Boundary -> blocks merge
- **Pass 2 INFORMATIONAL**: Conditional Side Effects, Magic Numbers, Dead Code, LLM Prompts, Test Gaps, Crypto, Time Windows, Type Coercion, View/Frontend -> included in report

### Phase 3 — Vulnerability assessment

Examine each modified file, trace data flow, look for privilege boundaries crossed unsafely. Cross-reference with secure patterns from `references/security-rules.md`.

---

## False Positive Filtering

-> Apply false-positive filtering from references/security-rules.md (section 8).
-> Apply automatic exclusions and stack precedents from `references/security-rules.md`.

---

## Severity and Confidence

| Severity | Criteria |
|----------|----------|
| HIGH | Directly exploitable -> RCE, data breach, auth bypass |
| MEDIUM | Specific conditions required but significant impact |
| LOW | Defense-in-depth, limited impact |

Only report findings with confidence >= 0.8.

---

## Output Format

For each confirmed vulnerability:

```markdown
# Vuln N: [Category]: `[file:line]`

* Severity: [HIGH/MEDIUM/LOW]
* Confidence: [X.X/1.0]
* Description: [Precise description]
* Exploitation scenario: [How to concretely exploit]
* Recommendation: [Specific fix with code if possible]
```

If no confirmed vulnerabilities: "No security vulnerabilities identified in this PR's changes."

---

## Execution

1. **Identification sub-task**: Explore the codebase, analyze the changes
2. **Filtering sub-tasks**: For each vuln, filter false positives in parallel
3. **Final filtering**: Exclude any vulnerability with confidence < 0.8

Focus on HIGH and MEDIUM findings only.
