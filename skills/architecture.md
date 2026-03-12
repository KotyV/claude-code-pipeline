---
name: architecture
description: >
  Designs the complete technical blueprint before writing any code:
  file inventory, interfaces, data flow, design patterns, ADRs,
  and implementation order respecting dependencies.
  Use when user says "design the architecture", "technical design",
  "plan the implementation", or invokes /architecture.
  Requires validated /specs as input. Consult references/clean-code-guidelines.md
  for pattern selection.
  Do NOT use for writing code — use /implement after architecture is validated.
---

# Architecture — Technical Design

Complete technical blueprint BEFORE writing a single line of code.

---

## Prerequisites

Read MANDATORY:

- `.product/specs/[slug]-specs.md` and `.product/features/[slug]-discovery.md`
- CLAUDE.md (workspace root + repos) — Structure, conventions, patterns
- .product/ (architecture-overview, technical-spec)
- `references/clean-code-guidelines.md` — Pattern catalog (Repository, Strategy, Factory, DI)

Examine existing patterns in the codebase: services, models, routes, auth.

---

## Your Mission

### Step 1: Analyze the existing codebase

- Which existing modules can be reused?
- Which patterns are already established? (do not reinvent)
- Are there conflicts with existing code?

### Step 2: List files to create/modify

```
Files to CREATE:
- app/models/xxx.py              — Data model
- app/schemas/xxx.py             — Validation schemas
- app/services/xxx_service.py    — Business logic
- app/routes/xxx.py              — Routes
- tests/unit/test_xxx.py         — Unit tests
- migrations/xxx_*.py            — Migration

Files to MODIFY:
- app/main.py                    — Register router
```

### Step 3: Identify Design Patterns

-> See `references/clean-code-guidelines.md` section 3 for the full catalog.
-> See `references/domain-examples.md` for concrete implementations.

For each module:

1. **Which pattern?** Repository (ALWAYS for DB), Strategy (if variants), Factory (if complex objects), DI via dependency injection (ALWAYS)
2. **SRP decomposition?** 1 function = 1 job, max 20 lines. If "and" in the name -> 2 functions. If > 4 params -> dataclass/schema
3. **Public interface?** Max 5-7 public methods per service. Internal helpers prefixed `_`. No DB access in routes.

### Step 4: Data Flow

```
Input (HTTP request)
  -> Route handler (input validation)
    -> Service layer (business logic)
      -> ORM queries (database)
      <- Results
    <- Response model
  <- HTTP response (JSON)
```

### Step 5: Architectural Decision Records (ADR)

If a non-trivial choice is made:

```
ADR-XXX: [Title]
Context: [Why]
Decision: [What we chose]
Consequences: [Impact]
Alternatives considered: [What we did not choose and why]
```

### Step 6: Implementation Order

```
1. Models + Migration (foundation)
2. Schemas (depend on models)
3. Services (depend on models + schemas)
4. Routes (depend on services + schemas)
5. Tests (depend on everything)
```

### Step 7: Pre-implementation Checklist

- [ ] No conflict with existing code
- [ ] Patterns consistent with the codebase
- [ ] Migrations are reversible (up + down)
- [ ] No breaking change on existing endpoints
- [ ] Queries optimized (no N+1)
- [ ] Inputs validated, auth applied
- [ ] Design patterns identified for each module
- [ ] Each service max 5-7 public methods
- [ ] No method with "and" in its name (SRP)
- [ ] No file planned > 300 lines
- [ ] Degenerate cases listed for each service

---

## Output

Create the file `.product/architecture/[slug].md` with the complete blueprint.

---

## MANDATORY CHECKPOINT

```
"Here is the technical design:
- X files to create, Y files to modify
- Implementation order: [list]
- ADR: [key decisions]
- Identified risks: [list]

Proceed with implementation?"
```

**DO NOT continue without explicit validation.**

---

## Usage

```bash
/architecture --feature [slug]
```
