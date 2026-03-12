---
name: specs
description: >
  Transforms validated discovery into implementable specifications:
  Epics, User Stories with Gherkin acceptance criteria, API contracts,
  data models, and MoSCoW prioritization.
  Use when user says "write specs", "create user stories",
  "define acceptance criteria", or invokes /specs.
  Requires a completed /feature-discovery document as input.
  Do NOT use for architecture or implementation — use /architecture or /implement.
---

# Specs — Technical Specifications and User Stories

Transform the validated discovery into implementable specifications.

---

## Prerequisites

Read MANDATORY:

- `.product/features/[slug]-discovery.md` — Validated discovery
- CLAUDE.md (workspace root + repos) — Data model, endpoints, conventions
- .product/ (functional-spec, technical-spec) — Current spec

---

## Your Mission

### Step 1: Define the Epics (max 3)

Group the discovery outputs into logical epics.

### Step 2: Create User Stories

For each epic:

```
**US-[N]: [Title]**
As a [persona],
I want [action],
So that [benefit].

Priority: P0 (Must) / P1 (Should) / P2 (Could)
Effort: XS (<2h) / S (2-4h) / M (4-8h) / L (1-2d) / XL (2-5d)
```

### Step 3: Acceptance Criteria (Gherkin)

For each user story, at minimum 3 scenarios:

```gherkin
Scenario: Nominal case
  Given [context]
  When [action]
  Then [expected result]

Scenario: Error case
  Given [context]
  When [invalid action]
  Then [expected error]

Scenario: Edge case
  Given [edge case context]
  When [action]
  Then [expected behavior]
```

### Step 4: API Contracts (if applicable)

```yaml
POST /api/v1/resources
  Request:
    field_1: string (required)
    field_2: string (required)
  Response 201:
    id: uuid
    status: "pending"
  Response 422:
    detail: "Validation failed"
```

### Step 5: Data Models (if applicable)

```
Table: [name]
  id: UUID (PK)
  field_1: string (required)
  created_at: timestamp with timezone
  Relations: belongs_to [X], has_many [Y]
```

### Step 6: MoSCoW Prioritization

| Story | Priority | Effort | Dependencies |
|-------|----------|--------|--------------|
| US-1  | Must     | S      | -            |
| US-2  | Must     | M      | US-1         |

---

## Output

Create the file `.product/specs/[slug]-specs.md` with the complete structure.

---

## MANDATORY CHECKPOINT

```
"I generated X user stories across Y epics.
- P0 (Must): X stories
- P1 (Should): Y stories
- Estimated total effort: [XS-XXL]

Validate these specs before architecture?"
```

**DO NOT continue without explicit validation.**

---

## Usage

```bash
/specs --feature [slug]
```
