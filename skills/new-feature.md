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

Complete pipeline: Discovery → Specs → Architecture → Implement → Tests → Security → Cleanup → Push → Docs.

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
│  Refactor pass       → Dead code, SRP, split files > 300 lines │
│  /qa-tests           → Full test pyramid                        │
│  /security-audit     → OWASP audit (score >= 80)                │
│  /cleanup-push       → Lint + build + push + documentation      │
│  Auto-merge          → gh pr checks --watch + gh pr merge       │
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
Record the baseline in `.product/pipeline-state.json` (see State Management below).

### Steps 1-7: Execute sub-commands in order

Each sub-command has its own detailed instructions:
1. `/feature-discovery` → understand the request → **CHECKPOINT**
2. `/specs` → break into user stories → **CHECKPOINT**
3. `/architecture` → design the solution → **CHECKPOINT**
4. `/implement` → code each story with incremental commits
5. **Refactor pass** → clean dead code, enforce SRP, split files > 300 lines
6. `/qa-tests` → validate with full test pyramid
7. `/security-audit` → audit security (score >= 80, grade B minimum)
8. `/cleanup-push` → clean up, push, document

### Step 5b: Refactor pass (MANDATORY after implement)

After implementation, BEFORE QA tests, perform a cleanup pass:

1. **Dead code**: remove unused imports, orphaned functions/variables/classes replaced by your changes
2. **SRP**: if a modified file exceeds 300 lines → identify responsibilities and split into sub-modules
3. **Re-exports**: if code was moved, maintain re-exports for backward compatibility of existing imports
4. **Verify**: run tests after each refactor to guarantee zero regression

```bash
# Detect unused imports in modified files
[your linter] --select unused-imports [modified files]

# Check file sizes
wc -l [modified files]
```

If a modified file exceeds 300 lines, commit the refactor separately:

```bash
git commit -m "refactor: split [file] into [sub-modules]"
```

### Step 8: Auto-merge

```bash
gh pr checks [PR_NUMBER] --watch
gh pr merge [PR_NUMBER] --squash --delete-branch
```

If a check fails: diagnose, fix, re-push, re-verify.

### Step 8b: Update documentation (MANDATORY — DO NOT SKIP)

**This step is the #1 cause of stale docs. It MUST run BEFORE the final report.**

If your project maintains spec/knowledge files (examples below are common conventions — adapt to your project), update them now:

1. **Feature registry / CHANGELOG** — Add the feature to the "Done" list with date, scope, and PR(s)
2. **Functional spec** — If user-facing behavior changed (new endpoints, new pages, new metrics), update the impacted sections. Update the "Last updated" date.
3. **Technical spec / architecture doc** — If file structure, data models, technical patterns, or test layout changed, update accordingly.
4. **README.md** — If tech stack, project structure, env vars, or API endpoints changed.
5. **CLAUDE.md** — If new conventions, patterns, or rules were introduced.

**Rule**: if NONE of these files are impacted (pure cosmetic, typo), note "docs: no changes needed" in the final report. Otherwise, list the files updated.

→ See `/cleanup-push` for the full documentation update routine.

### Step 9: Final report

```markdown
## Feature Complete: [Title]

### Pipeline executed

| Step | Status | Details |
|------|--------|---------|
| Discovery | ✓ | Scope validated |
| Specs | ✓ | X user stories |
| Architecture | ✓ | X files planned |
| Implementation | ✓ | X commits |
| Tests | ✓ | X tests, Y% coverage |
| Security | ✓ | Score XX/100 (Grade X) |
| Push | ✓ | PR: [url] — merged |

### Files created/modified
[list]

### Tests
- Baseline: X tests → Final: Y tests → No regression: ✓

### User impact
[What changes for the end user]

### How to test
1. [Step 1]
2. [Step 2]

### Documentation updated
- [list of files]

### Next steps
[What remains]
```

---

## State Management

**File: `.product/pipeline-state.json`** — OPTIONAL but strongly recommended. Update at each step so `--continue` can resume cleanly.

```json
{
  "feature_slug": "user-dashboard",
  "started_at": "2026-01-15T14:00:00Z",
  "current_step": "implement",
  "scope": "full_stack",
  "baseline": { "tests_passed": 76, "tests_failed": 0 },
  "steps": {
    "discovery": { "status": "completed", "output": ".product/features/user-dashboard-discovery.md" },
    "specs": { "status": "completed", "stories_count": 6 },
    "architecture": { "status": "completed", "output": ".product/architecture/user-dashboard.md" },
    "implement": { "status": "in_progress", "stories_completed": 3, "stories_total": 6, "commits": [] },
    "qa_tests": { "status": "pending" },
    "security": { "status": "pending" },
    "push": { "status": "pending" }
  }
}
```

See `references/recovery-protocol.md` for resume logic.

---

## Execution Modes

```bash
/new-feature "Description"                 # Full pipeline
/new-feature --continue                    # Resume interrupted (see references/recovery-protocol.md)
/new-feature "Description" --backend-only  # API only
/new-feature "Description" --frontend-only # UI only
/new-feature "Description" --mvp-only      # Only P0 stories (skip P1/P2)
```

---

## Final Checklist — Before declaring "complete"

→ See `references/quality-gates.md` for exact gate commands.

- [ ] All checkpoints validated by the user
- [ ] Quality gates passed (11 gates)
- [ ] Tests >= baseline (no regression)
- [ ] Type errors <= baseline
- [ ] At least one commit per module/story
- [ ] DB migrations created if models changed
- [ ] Security audit score >= 80 (Grade B minimum)
- [ ] Documentation updated (see Step 8b)
- [ ] Git push + PR created + CI checks passed + PR merged
- [ ] Testing instructions provided
