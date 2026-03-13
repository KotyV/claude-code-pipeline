---
name: feature-discovery
description: >
  Analyzes a raw feature request into a structured discovery document
  with outcomes, outputs, scope, constraints and risks.
  Use when user says "analyze this feature", "discovery phase",
  "what should we build", or invokes /feature-discovery.
  First step of the /new-feature pipeline. Requires user validation
  before proceeding to /specs.
  Do NOT use for technical design — use /architecture instead.
---

# Feature Discovery — In-Depth Request Analysis

First step of the `/new-feature` pipeline. Transform a raw request into a structured discovery document.

-> See references/domain-examples.md for concrete domain examples.

---

## Prerequisites

Read MANDATORY:

- CLAUDE.md (workspace root + repos) — Business context, domain model, conventions
- .product/ (features-registry, functional-spec, technical-spec) — Do not re-implement an existing feature

---

## Your Mission

### Step 1: Understand the request

- **WHAT**: What does the user concretely want?
- **WHY**: What problem does it solve? What business impact?
- **FOR WHOM**: Who uses this feature? (end user, admin, data team, etc.)
- **WHERE**: Which repo? API, Frontend, or both?

### Step 2: Ask clarification questions

If the request is ambiguous:
- Backend only or full-stack?
- Which API endpoints are needed?
- What data in DB? New model or extension?
- Required integrations? (external APIs, async jobs, third-party services)
- Which part of the UI is impacted?

### Step 3: Define Outcomes and Outputs

**Outcomes** (business results): Which KPI improves? How will we know it is a success?

**Outputs** (concrete deliverables): API endpoints, frontend pages/components, data models, async jobs

### Step 4: Scope IN / OUT

```
IN SCOPE (what we build now):
- ...

OUT OF SCOPE (for later):
- ...
```

### Step 5: Constraints and Risks

Technical dependencies, known limitations, integration risks, performance concerns.

### Step 6: Definition of Done

List of acceptance criteria that make the feature "done".

---

## Output

Create the file `.product/features/[slug]-discovery.md`:

```markdown
# Feature Discovery: [Title]

## Original request
[What the user asked for]

## Analysis
### What / Why / For whom / Scope
[...]

## Outcomes
1. [Outcome 1]

## Outputs
1. [Endpoint/Page/Component 1]

## Scope
### IN
- ...
### OUT
- ...

## Constraints
- ...

## Risks
- ...

## Definition of Done
- [ ] [Criterion 1]
```

---

## MANDATORY CHECKPOINT

```
"Here is what I understood about feature [X].
- Scope: [backend/frontend/full-stack]
- X outcomes, Y outputs
- Constraints: [list]

Is this correct? Shall we proceed to specs?"
```

**DO NOT continue without explicit user validation.**

---

## Usage

```bash
/feature-discovery "Description of the feature"
```
