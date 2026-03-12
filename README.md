# Claude Code Pipeline

A production-grade development pipeline for [Claude Code](https://claude.com/claude-code) — 10 slash commands + 6 reference guides that enforce quality at every step.

Built by a non-developer who needed to ship production software. Battle-tested on a real FastAPI + React codebase that scored A in a comparative audit against code written by senior engineers.

## What This Is

A set of custom slash commands (skills) for Claude Code that turn it into a disciplined engineering pipeline:

- **Structured workflow**: Discovery → Specs → Architecture → Implement → QA → Security → Push
- **Mandatory checkpoints**: Claude stops and asks for validation before proceeding
- **Quality gates**: Lint, type check, tests, security — enforced before every commit
- **TDD bug fixes**: Always reproduce the bug with a failing test first
- **No placeholders**: Zero TODO/FIXME — every commit is complete and tested

## Skills

| Skill | What it does |
|-------|-------------|
| `/new-feature` | Orchestrates the full pipeline: Discovery → Specs → Architecture → Implement → QA → Security → Push |
| `/bug-fix` | TDD pipeline: Diagnose → Reproduce (failing test) → Fix (test passes) → Verify → Push |
| `/implement` | Codes each user story with quality gates, incremental commits, and checkpoint validation |
| `/feature-discovery` | Analyzes a raw feature request into outcomes, outputs, scope, constraints, risks |
| `/specs` | Transforms discovery into Epics, User Stories with Gherkin acceptance criteria, API contracts |
| `/architecture` | Designs the technical blueprint: file inventory, patterns, data flow, ADRs, implementation order |
| `/qa-tests` | Full test pyramid (70% unit, 20% integration, 10% E2E) with vicious edge cases |
| `/security-audit` | OWASP-based audit across 6 categories, 0-100 scoring, false-positive filtering |
| `/security-review` | PR-focused security review of pending changes with confidence scoring |
| `/cleanup-push` | Final step: lint, build, tests, push, documentation updates |

## References (shared guidelines)

Skills reference these shared docs instead of duplicating content:

| File | Content |
|------|---------|
| `references/clean-code-guidelines.md` | SRP, design patterns (Repository, Strategy, Factory), error handling, logging, input validation |
| `references/quality-gates.md` | 11 automated gates that must pass before every commit |
| `references/security-rules.md` | Secure patterns by stack, decision tree ("is this a vulnerability?"), false-positive exclusions |
| `references/domain-examples.md` | Concrete code examples: API endpoint, scoring logic, React component, tests |
| `references/preflight-checklist.md` | Pre-implementation verification: git clean, tests baseline, type check baseline |
| `references/recovery-protocol.md` | How to detect state, resume after interruption, handle regressions |

## Quick Start

### 1. Copy into your project

```bash
mkdir -p .claude/commands
cp skills/*.md .claude/commands/
cp -r references .claude/commands/
```

### 2. Adapt to your stack

The skills are written for **FastAPI + React + TypeScript** but the patterns are universal. You'll want to:

- Update command names in `quality-gates.md` (e.g., replace `uv run pytest` with your test runner)
- Update `domain-examples.md` with examples from your codebase
- Update `security-rules.md` with your auth/infra stack
- Update `preflight-checklist.md` with your CI commands

### 3. Use in Claude Code

```bash
# Full feature pipeline
/new-feature "Add user dashboard with activity feed"

# Fix a bug with TDD
/bug-fix "API returns 500 when user has no activity"

# Just implement (skip discovery/specs/arch)
/implement --feature dashboard

# Security audit
/security-audit
```

## Design Principles

### Each piece of information lives in exactly one place

Skills reference shared docs (`→ see references/quality-gates.md`) instead of copy-pasting the same rules everywhere. This keeps skills short (~150 lines each) and references authoritative.

### Skills are guards, not suggestions

The skills don't just tell Claude what to do — they tell it what NOT to do:
- Don't chain steps without user validation
- Don't commit without passing quality gates
- Don't write a fix before writing a failing test
- Don't use TODO/FIXME/pass — ship complete code or don't commit

### Checkpoints prevent runaway AI

Every major phase ends with a mandatory checkpoint where Claude presents its work and asks for approval. This prevents the classic failure mode of AI coding 500 lines in the wrong direction.

## Architecture

```
.claude/commands/
├── new-feature.md          ← Master orchestrator (chains all skills)
├── feature-discovery.md    ← Step 1: Understand the request
├── specs.md                ← Step 2: User stories + acceptance criteria
├── architecture.md         ← Step 3: Technical design
├── implement.md            ← Step 4: Write code (per story)
├── qa-tests.md             ← Step 5: Test pyramid + edge cases
├── security-audit.md       ← Step 6: Vulnerability scan
├── security-review.md      ← Alternative: PR-focused security
├── cleanup-push.md         ← Step 7: Finalize + push + docs
├── bug-fix.md              ← Standalone: TDD bug resolution
└── references/
    ├── clean-code-guidelines.md
    ├── quality-gates.md
    ├── security-rules.md
    ├── domain-examples.md
    ├── preflight-checklist.md
    └── recovery-protocol.md
```

## Adapting to Your Stack

These skills were built for a specific stack but are designed to be forked and adapted:

| Current | Replace with your... |
|---------|---------------------|
| `uv run pytest` | Test runner (`npm test`, `go test`, `cargo test`) |
| `uv run ruff check` | Linter (`eslint`, `golangci-lint`, `clippy`) |
| `uv run mypy` | Type checker (`tsc`, `pyright`) |
| `FastAPI + Pydantic` | Your API framework + validation |
| `SQLAlchemy` | Your ORM/database layer |
| `Auth0 + JWT` | Your auth provider |
| `structlog` | Your logging library |

## Background

This pipeline was developed iteratively while building a production evaluation platform (FastAPI + React + PostgreSQL) in ~20 hours. The resulting code scored A in a comparative audit against 3 other codebases written by senior engineers, with:

- 97% test coverage
- Zero `except Exception` (9 custom exception classes instead)
- Zero hardcoded secrets
- Full CI/CD with K8s deployment
- Strict type checking (mypy + TypeScript strict mode)

The skills encode the discipline that made this possible.

## License

MIT — fork it, adapt it, make it yours.

Made with caffeine in Marseille. For Charlie.
