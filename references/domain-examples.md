# Domain Examples -- Concrete Code Patterns

Concrete implementation examples for common patterns.
Referenced by `/implement`, `/architecture`, and `/qa-tests`.

Adapt these examples to your domain and stack. The patterns are universal; only the names change.

---

## 1. Adding an API endpoint (complete pattern)

### Service (responsibility: business logic)

```python
# app/services/item_service.py

from app.db.models.item import Item, ItemHistory
from app.schemas.items import ItemDetailResponse, ItemHistoryPoint

# Named constants -- no magic numbers
HIGH_THRESHOLD = 75.0
MEDIUM_THRESHOLD = 50.0
MIN_TREND_POINTS = 2


async def get_item_detail(
    db, item_id: str
) -> ItemDetailResponse:
    """Fetch item with history and computed fields."""
    item = await _fetch_item(db, item_id)
    history = await _fetch_history(db, item_id)
    trend = _compute_trend(history)
    return ItemDetailResponse(
        item=item,
        history=history,
        trend=trend,
        status=_get_status_label(item.current_score),
    )


# --- Private helpers (prefixed with _) ---

async def _fetch_item(db, item_id: str) -> Item:
    """Fetch single item or raise ResourceNotFoundError."""
    result = await db.execute(select(Item).where(Item.id == item_id))
    item = result.scalar_one_or_none()
    if item is None:
        raise ResourceNotFoundError(f"Item {item_id} not found")
    return item


async def _fetch_history(
    db, item_id: str
) -> list[ItemHistoryPoint]:
    """Fetch item history sorted by date."""
    result = await db.execute(
        select(ItemHistory)
        .where(ItemHistory.item_id == item_id)
        .order_by(ItemHistory.recorded_at.asc())
    )
    return [ItemHistoryPoint.from_model(r) for r in result.scalars()]


def _compute_trend(history: list[ItemHistoryPoint]) -> float | None:
    """Compute trend from history. None if < 2 points."""
    if len(history) < MIN_TREND_POINTS:
        return None
    recent = history[-1].value
    previous = history[-2].value
    if previous == 0:
        return None
    return round((recent - previous) / previous * 100, 2)


def _get_status_label(score: float | None) -> str:
    """Map score to status label. Handles None/NaN/Infinity."""
    if score is None or not math.isfinite(score):
        return "unknown"
    if score >= HIGH_THRESHOLD:
        return "healthy"
    if score >= MEDIUM_THRESHOLD:
        return "warning"
    return "critical"
```

### Route (responsibility: HTTP layer)

```python
# app/api/routes/items.py

@router.get("/items/{item_id}", response_model=ItemDetailResponse)
async def get_item(
    item_id: Annotated[str, Path(description="UUID of the item")],
    db: Annotated[Session, Depends(get_db)],
    auth: Annotated[dict, Depends(verify_auth)],
) -> ItemDetailResponse:
    """Get item detail with history and trend."""
    return await item_service.get_item_detail(db, item_id)
```

### Schema (responsibility: validation)

```python
# app/schemas/items.py

class ItemDetailResponse(BaseModel):
    """Full item detail with history."""
    item: ItemSummary
    history: list[ItemHistoryPoint]
    trend: float | None = None
    status: str

class ItemHistoryPoint(BaseModel):
    value: float
    recorded_at: datetime
    run_id: str
```

```typescript
// TypeScript equivalent -- app/types/items.ts

interface ItemDetailResponse {
  item: ItemSummary;
  history: ItemHistoryPoint[];
  trend: number | null;
  status: string;
}

interface ItemHistoryPoint {
  value: number;
  recordedAt: string;  // ISO 8601
  runId: string;
}
```

---

## 2. Weighted Score Computation (pattern: edge cases first)

```python
# app/services/scoring_service.py

import math

WEIGHTS = {
    "accuracy": 2.0,
    "completeness": 1.5,
    "relevance": 1.5,
    "speed": 1.0,
}


def compute_weighted_score(metrics: dict[str, float | None]) -> float:
    """Weighted average of metrics on a 0-100 scale.

    Handles gracefully:
    - Empty dict -> 0.0
    - All None values -> 0.0
    - NaN/Infinity values -> filtered out
    - Mix of valid/invalid -> computed on valid only
    """
    # 1. Edge case: nothing
    if not metrics:
        return 0.0

    # 2. Filter invalid values (None, NaN, Infinity)
    valid_metrics = {
        k: v for k, v in metrics.items()
        if v is not None and math.isfinite(v)
    }

    # 3. Edge case: no usable values
    if not valid_metrics:
        return 0.0

    # 4. Happy path: weighted average
    weighted_sum = sum(
        valid_metrics[k] * WEIGHTS.get(k, 1.0)
        for k in valid_metrics
    )
    total_weight = sum(
        WEIGHTS.get(k, 1.0)
        for k in valid_metrics
    )

    return round(weighted_sum / total_weight, 2)
```

```typescript
// TypeScript equivalent
const WEIGHTS: Record<string, number> = {
  accuracy: 2.0,
  completeness: 1.5,
  relevance: 1.5,
  speed: 1.0,
};

function computeWeightedScore(metrics: Record<string, number | null>): number {
  const entries = Object.entries(metrics);

  // Edge case: nothing
  if (entries.length === 0) return 0;

  // Filter invalid values
  const valid = entries.filter(
    ([, v]) => v !== null && Number.isFinite(v)
  ) as [string, number][];

  // Edge case: no usable values
  if (valid.length === 0) return 0;

  // Happy path: weighted average
  const weightedSum = valid.reduce(
    (sum, [k, v]) => sum + v * (WEIGHTS[k] ?? 1.0), 0
  );
  const totalWeight = valid.reduce(
    (sum, [k]) => sum + (WEIGHTS[k] ?? 1.0), 0
  );

  return Math.round((weightedSum / totalWeight) * 100) / 100;
}
```

---

## 3. UI Card Component (pattern: 4 UI states)

```tsx
// src/components/SummaryCard.tsx

interface SummaryCardProps {
  item: SummaryItem;
  score: number | null;
  indicators: Indicator[];
}

export function SummaryCard({ item, score, indicators }: SummaryCardProps) {
  // Handle 4 states: loading, error, empty, populated
  if (item.status === "loading") {
    return <SummaryCardSkeleton />;
  }

  if (item.status === "error") {
    return <SummaryCardError message={item.errorMessage} />;
  }

  if (score === null) {
    return <SummaryCardEmpty item={item} />;
  }

  // Populated state
  const color = getScoreColor(score);

  return (
    <Card className={`border-l-4 border-l-${color}`}>
      <CardHeader>
        <h3>{item.name}</h3>
        <ScoreBadge score={score} color={color} />
      </CardHeader>
      <CardBody>
        <IndicatorDots indicators={indicators} />
      </CardBody>
    </Card>
  );
}

// Helper extracted -- not inline in the component
function getScoreColor(score: number): "green" | "yellow" | "red" {
  if (score >= 75) return "green";
  if (score >= 50) return "yellow";
  return "red";
}
```

---

## 4. Enums for Statuses (no magic strings)

```python
# Python
from enum import Enum

class TaskStatus(str, Enum):
    """Status lifecycle: pending -> running -> completed/failed."""
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"

class Priority(str, Enum):
    """P0 = must-have, P1 = important, P2 = nice-to-have."""
    P0 = "P0"
    P1 = "P1"
    P2 = "P2"

class StatusColor(str, Enum):
    GREEN = "green"    # >= 75
    YELLOW = "yellow"  # >= 50
    RED = "red"        # < 50
    GRAY = "gray"      # no data
```

```typescript
// TypeScript
enum TaskStatus {
  Pending = "pending",
  Running = "running",
  Completed = "completed",
  Failed = "failed",
}

enum Priority {
  P0 = "P0",  // Must-have
  P1 = "P1",  // Important
  P2 = "P2",  // Nice-to-have
}

type StatusColor = "green" | "yellow" | "red" | "gray";
```

---

## 5. Complete Endpoint Test (recommended pattern)

```python
# tests/integration/test_item_endpoint.py

import pytest
from httpx import AsyncClient


class TestGetItemDetail:
    """Tests for GET /api/v1/items/{item_id}."""

    # --- Happy path ---

    async def test_returns_item_with_history(self, client: AsyncClient, seeded_item):
        response = await client.get(f"/api/v1/items/{seeded_item.id}")
        assert response.status_code == 200
        data = response.json()
        assert data["item"]["id"] == str(seeded_item.id)
        assert isinstance(data["history"], list)
        assert data["status"] in ("healthy", "warning", "critical", "unknown")

    # --- Error cases ---

    async def test_not_found(self, client: AsyncClient):
        fake_id = "00000000-0000-0000-0000-000000000000"
        response = await client.get(f"/api/v1/items/{fake_id}")
        assert response.status_code == 404

    async def test_invalid_id_format(self, client: AsyncClient):
        response = await client.get("/api/v1/items/not-a-uuid")
        assert response.status_code == 422

    async def test_requires_auth(self, unauthenticated_client: AsyncClient):
        response = await unauthenticated_client.get("/api/v1/items/any-id")
        assert response.status_code in (401, 403)

    # --- Edge cases ---

    async def test_item_with_no_history(self, client: AsyncClient, item_no_history):
        response = await client.get(f"/api/v1/items/{item_no_history.id}")
        assert response.status_code == 200
        data = response.json()
        assert data["history"] == []
        assert data["trend"] is None

    async def test_item_with_single_history_point(self, client: AsyncClient, item_one_point):
        response = await client.get(f"/api/v1/items/{item_one_point.id}")
        data = response.json()
        assert len(data["history"]) == 1
        assert data["trend"] is None  # Not enough points for a trend

    async def test_item_with_null_score(self, client: AsyncClient, item_null_score):
        response = await client.get(f"/api/v1/items/{item_null_score.id}")
        assert response.status_code == 200
        assert response.json()["status"] == "unknown"

    # --- Security ---

    async def test_sql_injection_in_path(self, client: AsyncClient):
        response = await client.get("/api/v1/items/'; DROP TABLE--")
        assert response.status_code == 422
```

```typescript
// TypeScript equivalent (adapt to your test runner: Jest, Vitest, etc.)

describe("GET /api/v1/items/:id", () => {
  // Happy path
  it("returns item with history", async () => {
    const res = await client.get(`/api/v1/items/${seededItem.id}`);
    expect(res.status).toBe(200);
    expect(res.body.item.id).toBe(seededItem.id);
    expect(Array.isArray(res.body.history)).toBe(true);
  });

  // Error cases
  it("returns 404 for nonexistent item", async () => {
    const res = await client.get("/api/v1/items/00000000-0000-0000-0000-000000000000");
    expect(res.status).toBe(404);
  });

  it("returns 401 without auth", async () => {
    const res = await unauthClient.get("/api/v1/items/any-id");
    expect(res.status).toBeOneOf([401, 403]);
  });

  // Edge cases
  it("returns empty history when no data", async () => {
    const res = await client.get(`/api/v1/items/${itemNoHistory.id}`);
    expect(res.body.history).toEqual([]);
    expect(res.body.trend).toBeNull();
  });
});
```
