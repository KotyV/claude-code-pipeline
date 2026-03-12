---
name: cleanup-push
description: >
  Final pipeline step: lint, format, type check, full test suite,
  build verification, git push, and spec documentation updates.
  Use when user says "cleanup", "finalize", "push", "ship it",
  or invokes /cleanup-push. Updates functional-spec.md, technical-spec.md,
  features-registry.md, architecture-overview.md, README.md, and CLAUDE.md.
  Do NOT use mid-implementation — finish /implement and /qa-tests first.
---

# Cleanup & Push — Finalization

Last step of the pipeline: lint, build, final tests, push, documentation.

---

## Your Mission

### Step 1: Quality gates

-> Run ALL gates — see `references/quality-gates.md` for exact commands.

```bash
# Backend
[your linter] [source directory] --fix
[your formatter] [source directory]
[your type checker] [source directory]

# Frontend (if applicable)
[your frontend linter] src/ --fix
[your frontend type checker]
```

### Step 2: Final tests

```bash
[your test runner] tests/ -v --tb=short
[your frontend test runner]  # if applicable
```

Compare with the baseline. If regressions: STOP, fix them.

### Step 3: Build check

```bash
# Backend
docker compose build
# Frontend
[your frontend build command]
```

### Step 4: Git push and PR

```bash
git status
git add [forgotten files]
git commit -m "chore: cleanup before push"

git push -u origin [branch]
gh pr create --title "[type]: [short description]" --body "$(cat <<'EOF'
## Summary
- [main changes]

## Test plan
- [ ] Unit tests pass
- [ ] Type check OK
- [ ] Build OK
EOF
)"
```

### Step 5: Update documentation

**MANDATORY** after each completed feature:

#### functional-spec.md
Feature description from the user's perspective, impact, endpoints, screens.

#### technical-spec.md
Data models, architecture, technical decisions, tests.

#### features-registry.md
```markdown
- [x] [slug] — [title] — [date] — [scope: backend/frontend/full-stack]
```

#### architecture-overview.md
Update if new modules/tables/routes were added.

#### README.md (of impacted repos)

Check each section:
- Tech Stack, Project Structure, API Endpoints, CI Pipeline, Docker, Environment Vars, Production

If NO section is impacted: do not touch. If AT LEAST ONE: update.

#### CLAUDE.md (of impacted repos)
If new conventions, patterns, or rules -> add in the appropriate section.

### Step 6: Testing instructions

```markdown
## How to test

### Prerequisites
- Docker running: `docker compose up -d`

### Steps
1. [Action 1] — Expected result: [xxx]
2. [Action 2] — Expected result: [xxx]
```

### Step 7: Automatic post-mortem

Add in `.product/pipeline-state.json`:

```json
{
  "post_mortem": {
    "blockers_encountered": ["..."],
    "retries": { "US-3": "Circular import — resolved with TYPE_CHECKING" },
    "patterns_discovered": ["..."],
    "improvements_for_next_time": ["..."]
  }
}
```

---

## Final Checklist

- [ ] Lint + type check pass
- [ ] Tests >= baseline (zero regression)
- [ ] Build passes
- [ ] Quality gates pass (see `references/quality-gates.md`)
- [ ] Git clean (no forgotten files)
- [ ] Git push done + PR created
- [ ] functional-spec.md, technical-spec.md, features-registry.md updated
- [ ] architecture-overview.md updated
- [ ] README.md updated (if sections impacted)
- [ ] CLAUDE.md updated (if new conventions)
- [ ] Testing instructions provided
- [ ] Post-mortem recorded in pipeline-state.json
- [ ] pipeline-state.json status: completed

---

## Usage

```bash
/cleanup-push
```
