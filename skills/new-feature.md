---
name: new-feature
description: >
  Master orchestrator for the complete feature development pipeline:
  Discovery, Specs, Architecture, Implement, QA, Security, Cleanup & Push.
  Use when user says "new feature", "build a feature", "implement end-to-end",
  or invokes /new-feature. Coordinates all sub-skills in sequence with
  mandatory user validation checkpoints between phases.
  Do NOT use for single bug fixes or small tweaks — use /implement directly.
---

# New Feature — Master Orchestrator

Pipeline complet : Discovery → Specs → Architecture → Implement → Tests → Security → Cleanup → Push.

---

## Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                     /new-feature Pipeline                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  /feature-discovery  → Understand the request                   │
│       ⚡ CHECKPOINT: User validates understanding                │
│  /specs              → Break into user stories                  │
│       ⚡ CHECKPOINT: User validates stories                      │
│  /architecture       → Design the technical solution            │
│       ⚡ CHECKPOINT: User validates design                       │
│  /implement          → Code each story (1 commit per story)     │
│  /qa-tests           → Full test pyramid                        │
│  /security-audit     → OWASP audit (score >= 80)                │
│  /cleanup-push       → Lint + build + push + documentation      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## CRITICAL RULES

### Rule 1: ONE feature per session

```
❌ FORBIDDEN: "Build models + routes + services + frontend in one go"
✅ CORRECT:  Feature A → commit → new session → Feature B
```

### Rule 2: User validation MANDATORY

After each major phase, STOP and ASK for validation before continuing.

```
1. Discovery done → "Here's what I understood. Continue?"
2. Specs done     → "Here are the X stories. Validate?"
3. Architecture   → "Here's the design. Implement?"
```

DO NOT chain steps automatically.

### Rule 3: Commit per story/module

```
❌ FORBIDDEN: Implement everything then 1 big commit
✅ CORRECT:  Models → commit → Schemas → commit → Routes → commit
```

### Rule 4: No placeholders

```
❌ FORBIDDEN: TODO, FIXME, pass, "implement later", ...
✅ CORRECT:  Complete, functional, tested code
```

---

## Your Mission (Claude)

### Step 0: Load context and baseline (MANDATORY)

Read project knowledge files:
- CLAUDE.md (workspace root + repos)
- Existing specs, architecture docs, feature registry

Then run the preflight → see `references/preflight-checklist.md`.
Create a feature branch from main: `git checkout -b feat/[slug]`

### Steps 1-7: Execute sub-commands in order

Each sub-command has its own detailed instructions:
1. `/feature-discovery` → understand the request → **CHECKPOINT**
2. `/specs` → break into user stories → **CHECKPOINT**
3. `/architecture` → design the solution → **CHECKPOINT**
4. `/implement` → code each story with incremental commits
5. `/qa-tests` → validate with full test pyramid
6. `/security-audit` → audit security (score >= 80, grade B minimum)
7. `/cleanup-push` → clean up, push, document

### Step 8: Final report

```markdown
## Feature Complete: [Title]

### Pipeline executed

| Step | Status | Details |
|------|--------|---------|
| Discovery | ✓ | Scope validated |
| Specs | ✓ | X user stories |
| Architecture | ✓ | X files planned |
| Implementation | ✓ | X commits |
| Tests | ✓ | X tests |
| Security | ✓ | Score XX/100 (Grade X) |
| Push | ✓ | PR: [url] |

### Tests
- Baseline: X tests → Final: Y tests → No regression: ✓

### How to test
1. [Step 1]
2. [Step 2]
```

---

## Usage

```bash
/new-feature "Description of the feature"
/new-feature --continue              # Resume interrupted pipeline
/new-feature "Description" --backend-only
/new-feature "Description" --frontend-only
```
