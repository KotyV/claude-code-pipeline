---
name: qa-tests
description: >
  Validates code quality through the full test pyramid: 70% unit,
  20% integration, 10% E2E. Includes vicious edge cases, boundary
  values, concurrency, corrupted data, and malicious inputs.
  Use when user says "run tests", "write tests", "QA", "validate quality",
  or invokes /qa-tests.
  Do NOT use for security-specific audits — use /security-audit instead.
---

# QA Tests — Quality Validation

Complete test pyramid: unit -> integration -> edge cases.

---

## Prerequisites

- `.product/specs/[slug]-specs.md` — Gherkin acceptance criteria
- CLAUDE.md (repos) — Testing conventions

## Coverage and quality targets

-> See `references/quality-gates.md` for the coverage table by file type and test quality criteria.

---

## Test Pyramid

```
        +---------+
        |   E2E   |  10% — Critical smoke tests
        +---------+
        |  Integ  |  20% — API endpoints, complete workflows
        +---------+
        |  Unit   |  70% — Pure business logic, services, utils
        +---------+
```

---

## Your Mission

### Step 1: Inventory test cases from specs

For each User Story:
- Transform Gherkin scenarios into test cases
- Add edge cases not covered by specs
- Identify security cases (auth, validation)

### Step 2: Unit tests (70%)

```python
class TestCompositeScore:
    def test_all_perfect_scores(self):
        values = {"metric_a": 1.0, "metric_b": 1.0}
        assert compute_composite_score(values) == 100.0

    def test_empty_metrics(self):
        assert compute_composite_score({}) == 0.0
```

### Step 3: Integration tests (20%)

```python
async def test_create_resource(client, sample_data):
    response = await client.post("/api/v1/resources", json=sample_data)
    assert response.status_code == 201
    assert "id" in response.json()
```

### Step 4: Vicious edge cases — "Stress the happy path"

**MANDATORY**: For each endpoint/service, cover ALL these categories.

#### 4a. Boundary values (boundary testing)

```python
class TestScoreThresholds:
    def test_exactly_on_upper_threshold(self):
        assert get_score_label(75.0) == "green"

    def test_just_below_upper_threshold(self):
        assert get_score_label(74.9999) == "yellow"

    def test_exactly_on_lower_threshold(self):
        assert get_score_label(50.0) == "yellow"

    def test_zero_score(self):
        assert get_score_label(0.0) == "red"

    def test_negative_score(self):
        assert get_score_label(-1.0) == "red"

    def test_score_above_max(self):
        assert get_score_label(105.0) == "green"
```

#### 4b. Toxic values (NaN, Infinity, None)

```python
class TestToxicValues:
    def test_nan_input(self):
        assert compute_score({"value": float("nan")}) == 0.0

    def test_infinity_input(self):
        assert compute_score({"value": float("inf")}) == 0.0

    def test_all_none_values(self):
        assert compute_score({"a": None, "b": None}) == 0.0

    def test_mix_valid_and_none(self):
        result = compute_score({"a": 0.8, "b": None})
        assert result > 0
```

#### 4c. Empty and minimal collections

```python
class TestEmptyStates:
    async def test_list_endpoint_no_data(self, client):
        response = await client.get("/api/v1/resources")
        assert response.status_code == 200
        assert response.json()["items"] == []

    async def test_history_empty(self, client, resource_id):
        response = await client.get(f"/api/v1/resources/{resource_id}/history")
        assert response.status_code == 200
        assert response.json()["trend"] is None

    async def test_history_single_point(self, client, resource_id):
        response = await client.get(f"/api/v1/resources/{resource_id}/history")
        assert response.json()["trend"] is None
```

#### 4d. Malicious inputs (injection, overflow)

```python
class TestMaliciousInputs:
    async def test_sql_injection_in_path(self, client):
        response = await client.get("/api/v1/resources/'; DROP TABLE resources;--")
        assert response.status_code == 422

    async def test_uuid_invalid_format(self, client):
        response = await client.get("/api/v1/resources/not-a-uuid")
        assert response.status_code == 422

    async def test_uuid_valid_but_nonexistent(self, client):
        fake_uuid = "00000000-0000-0000-0000-000000000000"
        response = await client.get(f"/api/v1/resources/{fake_uuid}")
        assert response.status_code == 404

    async def test_extremely_long_string(self, client):
        response = await client.post("/api/v1/resources", json={"name": "a" * 100_000})
        assert response.status_code == 422

    async def test_empty_body(self, client):
        response = await client.post("/api/v1/resources", json={})
        assert response.status_code == 422
```

#### 4e. Pagination and filter abuse

```python
class TestPaginationAbuse:
    async def test_page_negative(self, client):
        response = await client.get("/api/v1/resources?page=-1")
        assert response.status_code == 422

    async def test_page_absurdly_high(self, client):
        response = await client.get("/api/v1/resources?page=999999")
        assert response.status_code == 200
        assert response.json()["items"] == []

    async def test_limit_too_large(self, client):
        response = await client.get("/api/v1/resources?limit=1000000")
        assert response.status_code == 422
```

#### 4f. Concurrency and double submission

```python
class TestConcurrency:
    async def test_double_creation(self, client, sample_data):
        r1 = await client.post("/api/v1/resources", json=sample_data)
        r2 = await client.post("/api/v1/resources", json=sample_data)
        assert r1.status_code == 201
        assert r2.status_code in (201, 409)  # Never 500
```

#### 4g. Corrupted data in DB

```python
class TestCorruptedData:
    async def test_record_with_null_fields(self, db, record_with_nulls):
        result = await compute_aggregate(db, record_with_nulls.id)
        assert result == 0.0

    async def test_orphan_record(self, db, orphan_record):
        results = await list_all_records(db)
        assert orphan_record.id not in [r.id for r in results]
```

#### 4h. Auth and security

```python
class TestAuthEdgeCases:
    async def test_expired_token(self, client_with_expired_token):
        response = await client_with_expired_token.get("/api/v1/resources")
        assert response.status_code in (401, 403)

    async def test_malformed_token(self, client):
        response = await client.get("/api/v1/resources",
                                     headers={"Authorization": "Bearer not.a.jwt"})
        assert response.status_code in (401, 403)

    async def test_no_auth_on_protected_endpoint(self, unauthenticated_client):
        response = await unauthenticated_client.get("/api/v1/resources")
        assert response.status_code in (401, 403)

    async def test_health_needs_no_auth(self, unauthenticated_client):
        response = await unauthenticated_client.get("/health")
        assert response.status_code == 200
```

#### 4i. External dependency failures

```python
class TestExternalDependencyFailures:
    async def test_external_api_returns_malformed_json(self, mock_api_malformed):
        result = await process_external_response(mock_api_malformed)
        assert result.status == "error"
        assert result.data is None

    async def test_external_api_timeout(self, mock_api_timeout):
        result = await process_external_response(mock_api_timeout)
        assert result.status in ("error", "timeout")

    async def test_external_api_rate_limited(self, mock_api_429):
        with pytest.raises(RateLimitError):
            await call_external_api(params={"query": "test"})
```

#### 4j. Property-based testing

Use Hypothesis (Python) or fast-check (TS/JS) to generate random inputs and find unexpected edge cases.

```python
from hypothesis import given, strategies as st

@given(score=st.floats(allow_nan=True, allow_infinity=True))
def test_score_label_never_crashes(score):
    result = get_score_label(score)
    assert result in ("green", "yellow", "red")
```

#### 4k. Snapshot testing

Verify API response schema stability with snapshots. Detect unintended schema changes by comparing against stored snapshots.

```python
def test_resource_response_schema(client, snapshot):
    response = client.get("/api/v1/resources/valid-id")
    assert response.status_code == 200
    snapshot.assert_match(response.json(), "resource_response.json")
```

#### 4l. Contract testing

Verify frontend/backend compatibility using shared schemas or generated types. Ensure API responses match the types the frontend expects.

```python
def test_api_response_matches_shared_schema(client):
    response = client.get("/api/v1/resources")
    assert response.status_code == 200
    # Validate against the shared schema used by both frontend and backend
    SharedResourceSchema.model_validate(response.json())
```

### Step 5: Run and verify

```bash
# Backend
[your test runner] tests/unit/ -v --tb=short --cov=app --cov-report=term-missing
[your test runner] tests/integration/ -v --tb=short

# Frontend
[your frontend test runner] --coverage
```

### Step 6: Report

```markdown
## Test Results

### Summary
- Total: XX tests — Passed: XX — Failed: XX

### Critical cases covered
- [ ] Happy path all endpoints
- [ ] Validation errors (422)
- [ ] Resource not found (404)
- [ ] Auth required (401)
- [ ] Business edge cases
```

---

## Troubleshooting

### Flaky tests (pass 1 time out of 2)

1. Execution order -> each test creates its own data (isolated fixtures)
2. Timestamps -> freeze time or mock the clock
3. Async race condition -> verify all DB calls are awaited
4. DB state leaking -> transaction rollback in fixture

### Fixtures not working

1. Scope mismatch: "session" fixture used in a "function" test
2. Missing import in conftest.py
3. Async fixture without proper async fixture decorator

### Tests too slow (> 30 seconds)

1. Profile to identify slowest tests
2. Unit tests MUST NOT hit the DB -> mock
3. Parallelize if supported by your test runner

---

## Usage

```bash
/qa-tests                          # Full tests
/qa-tests --feature [slug]         # Tests for a feature
/qa-tests --unit-only              # Unit tests only
/qa-tests --coverage               # With coverage report
```

## Next step

-> `/security-audit --feature [slug]` to audit for vulnerabilities.
