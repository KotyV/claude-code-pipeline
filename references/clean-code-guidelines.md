# Clean Code Guidelines

Shared reference for `/implement`, `/architecture`, `/qa-tests`, and `/bug-fix`.
Read this file BEFORE writing any code.

---

## 1. Single Responsibility Principle (SRP)

Every function does **ONE thing**. If the description requires "and", split into 2 functions.

```python
# --- Python example ---

# BAD: God Function -- does too many things
async def process_order(db, order_id):
    order = await db.get(Order, order_id)
    items = await fetch_items(order)
    totals = {}
    for item in items:
        result = await compute_price(item)
        totals[item.id] = apply_discount(result)
    order.total = sum(totals.values())
    order.status = "completed"
    await db.commit()
    await send_confirmation(order)
    return order

# GOOD: Decomposed -- each function has a single job
async def process_order(db, order_id: str) -> Order:
    """Orchestrate order processing (coordination only)."""
    order = await fetch_order(db, order_id)
    item_totals = await compute_all_item_totals(db, order)
    final_total = compute_order_total(item_totals)
    return await finalize_order(db, order, final_total)

async def compute_all_item_totals(db, order: Order) -> dict[str, float]:
    """Compute price for each item in the order."""
    results = {}
    for item in order.items:
        results[item.id] = await compute_item_total(item)
    return results

def compute_order_total(totals: dict[str, float]) -> float:
    """Sum all item totals with validation."""
    if not totals:
        return 0.0
    valid = {k: v for k, v in totals.items() if v is not None}
    return sum(valid.values())

async def finalize_order(db, order: Order, total: float) -> Order:
    """Persist final total and mark order as completed."""
    order.total = total
    order.status = "completed"
    await db.commit()
    return order
```

```typescript
// --- TypeScript example ---

// BAD: God Function
async function processOrder(orderId: string) {
  const order = await db.findOrder(orderId);
  const items = await fetchItems(order);
  let total = 0;
  for (const item of items) {
    const price = await computePrice(item);
    total += applyDiscount(price);
  }
  order.total = total;
  order.status = "completed";
  await db.save(order);
  await sendConfirmation(order);
  return order;
}

// GOOD: Decomposed
async function processOrder(orderId: string): Promise<Order> {
  const order = await fetchOrder(orderId);
  const itemTotals = await computeAllItemTotals(order);
  const total = computeOrderTotal(itemTotals);
  return finalizeOrder(order, total);
}
```

### Signs a function does too much

- More than **20 lines** of logic (excluding docstrings/imports) -- decompose
- More than **3 levels of indentation** -- extract the inner blocks
- The name contains "and" / "then" -- 2 functions
- More than **4 parameters** -- encapsulate in a data class / struct / options object

---

## 2. File Organization

### Size limits

| Element | Limit | Action if exceeded |
| --- | --- | --- |
| Function | 20 lines of logic | Extract into sub-functions |
| Class | 200 lines | Extract into sub-classes or mixins |
| File | 300 lines (excluding tests) | Split into sub-modules |
| Test module | 500 lines | Split by category (unit/edge/integration) |

### Structure by domain

```text
GOOD: Layer-based (cohesive)
app/
  services/order_service.py       <- business logic
  api/routes/orders.py            <- HTTP layer
  schemas/orders.py               <- validation
  db/models/order.py              <- persistence

GOOD: Domain-based (also valid)
app/
  orders/
    service.py
    routes.py
    schemas.py
    models.py

FORBIDDEN:
app/
  helpers.py        <- catch-all junk drawer
  utils.py          <- catch-all without a domain
  service.py        <- one file for the entire app
```

### Helper/utils rule

If a helper is used by **1 service only** -- put it in that service as a private function (`_helper` / internal).
If used by **2+ services** -- put it in a shared utilities folder with an explicit name (`utils/score_math.py`, NOT `utils/helpers.py`).

---

## 3. Required Design Patterns

### Repository / Service Pattern -- Isolate data access

```python
# BAD: DB access directly in the route handler
@router.get("/items/{item_id}")
async def get_item(item_id: str, db: ...):
    result = await db.execute(select(Item).where(Item.id == item_id))
    item = result.first()
    if not item:
        raise HTTPException(404)
    return item

# GOOD: DB access isolated in the service layer
@router.get("/items/{item_id}")
async def get_item(item_id: str, db: ...):
    return await item_service.get_by_id(db, item_id)

# In item_service.py
async def get_by_id(db, item_id: str) -> ItemResponse:
    item = await _fetch_item(db, item_id)
    if not item:
        raise ResourceNotFoundError(f"Item {item_id} not found")
    return ItemResponse.from_model(item)
```

```typescript
// TypeScript equivalent
// BAD: DB in the controller
app.get("/items/:id", async (req, res) => {
  const item = await db.query("SELECT * FROM items WHERE id = $1", [req.params.id]);
  res.json(item);
});

// GOOD: Service layer
app.get("/items/:id", async (req, res) => {
  const item = await itemService.getById(req.params.id);
  res.json(item);
});
```

### Strategy Pattern -- Algorithm variants

```python
# When you have different scoring, processing, or classification strategies
from typing import Protocol

class ScoringStrategy(Protocol):
    def compute(self, raw_data: dict) -> float: ...

class WeightedAverageScoring:
    def __init__(self, weights: dict[str, float]):
        self.weights = weights

    def compute(self, raw_data: dict) -> float:
        total = sum(raw_data.get(k, 0) * w for k, w in self.weights.items())
        return total / sum(self.weights.values())

class ThresholdScoring:
    def compute(self, raw_data: dict) -> float:
        passed = sum(1 for v in raw_data.values() if v >= 0.75)
        return passed / len(raw_data) * 100 if raw_data else 0.0
```

### Factory Pattern -- Complex object creation

```python
# BAD: Inline construction in a route handler
order = Order(
    id=uuid4(),
    customer_id=data.customer_id,
    status="pending",
    created_at=datetime.utcnow(),
    items=data.items,
    config=default_config(),
)

# GOOD: Factory method in the model or service
class Order:
    @classmethod
    def create(cls, customer_id: str, items: list) -> "Order":
        return cls(
            id=uuid4(),
            customer_id=customer_id,
            status="pending",
            created_at=datetime.utcnow(),
            items=items,
            config=default_config(),
        )
```

### Dependency Injection -- Always inject, never import directly

```python
# Python: framework-specific DI (FastAPI Depends, Flask inject, etc.)
async def get_item(
    item_id: str,
    db: Annotated[Session, Depends(get_db)],
    auth: Annotated[dict, Depends(verify_auth)],
) -> ItemResponse:
    ...
```

```typescript
// TypeScript: constructor injection or function parameters
class ItemService {
  constructor(
    private readonly db: Database,
    private readonly logger: Logger,
  ) {}
}
```

### Forbidden Anti-patterns

| Anti-pattern | Symptom | Fix |
| --- | --- | --- |
| God Function | 50+ lines, 5+ responsibilities | Decompose into sub-functions |
| Spaghetti Imports | Circular imports between modules | Extract common interface, lazy imports |
| Magic Numbers | `if score > 75`, `limit=10` | Named constants: `GREEN_THRESHOLD = 75` |
| Copy-Paste | 3+ occurrences of the same block | Extract into a reusable function |
| Stringly-Typed | `status = "completed"` everywhere | Enum: `class OrderStatus(str, Enum)` |
| Catch-All | `except Exception: pass` | Specific exceptions, never silent |
| Premature Abstraction | AbstractBaseFactory for 1 case | YAGNI -- abstract when 2+ cases exist |

---

## 4. Readability

### Naming

```python
# BAD: Cryptic names
d = get_data()
tmp = process(d)
res = calc(tmp, 0.75)

# GOOD: Names that tell the story
order_history = fetch_order_history(customer_id)
normalized_totals = normalize_to_currency(order_history)
average_order_value = compute_weighted_average(normalized_totals, WEIGHTS)
```

### One-liners: readability > brevity

```python
# GOOD: Simple and readable
active_items = [item for item in items if item.is_active]

# GOOD: Simple ternary
label = "high" if score >= THRESHOLD else "low"

# BAD: Too dense -- unreadable
result = {k: [x.score for x in v if x.score > threshold] for k, v in grouped.items() if len(v) > min_count}

# GOOD: Unrolled for readability
result = {}
for category, items in grouped.items():
    if len(items) <= min_count:
        continue
    result[category] = [x.score for x in items if x.score > threshold]

# BAD: Nested ternary
label = "green" if score >= 75 else "yellow" if score >= 50 else "red"

# GOOD: Lookup table or function
SCORE_LABELS = [(75, "green"), (50, "yellow"), (0, "red")]

def get_score_label(score: float) -> str:
    for threshold, label in SCORE_LABELS:
        if score >= threshold:
            return label
    return "red"
```

### Named constants

```python
# BAD: Magic numbers
if score >= 75:
    color = "green"
elif score >= 50:
    color = "yellow"

# GOOD: Explicit constants
GREEN_THRESHOLD = 75
YELLOW_THRESHOLD = 50

if score >= GREEN_THRESHOLD:
    color = "green"
elif score >= YELLOW_THRESHOLD:
    color = "yellow"
```

---

## 5. Handling Edge Cases

**RULE: Always handle edge cases FIRST, before the happy path.**

```python
def compute_average_score(scores: dict[str, float | None]) -> float:
    """Weighted average of scores on a 0-100 scale."""
    # 1. Edge case: empty input
    if not scores:
        return 0.0

    # 2. Edge case: all null/NaN/Infinity
    valid = {
        k: v for k, v in scores.items()
        if v is not None and math.isfinite(v)
    }
    if not valid:
        return 0.0

    # 3. Happy path
    weighted_sum = sum(valid[k] * WEIGHTS.get(k, 1.0) for k in valid)
    total_weight = sum(WEIGHTS.get(k, 1.0) for k in valid)
    return round(weighted_sum / total_weight, 2)
```

### Edge case checklist by type

| Input type | Cases to handle |
| --- | --- |
| Collection (list, dict, array) | Empty, null, single element, very large (10K+) |
| Number (int, float) | 0, negative, NaN, Infinity, -Infinity, boundary values |
| String | Empty `""`, null, very long (10K chars), unicode/emoji, injection attempts |
| ID (UUID, slug) | Invalid format, valid format but nonexistent in DB |
| Date/timestamp | Null, epoch (1970), far future, timezone mismatch |
| DB relation | Orphan (parent deleted), circular, null foreign key |

---

## 6. Error Handling

### Service layer pattern

```python
# Python
async def get_item_detail(db, item_id: str) -> ItemDetail:
    """Fetch item with full detail. Raises ResourceNotFoundError if absent."""
    try:
        item = await _fetch_item(db, item_id)
    except DatabaseError as e:
        logger.error("DB query failed", item_id=item_id, error=str(e))
        raise AppDatabaseError(f"Failed to fetch item: {e!s}") from e

    if item is None:
        raise ResourceNotFoundError(f"Item {item_id} not found")

    return ItemDetail.from_model(item)
```

```typescript
// TypeScript
async function getItemDetail(itemId: string): Promise<ItemDetail> {
  let item: Item | null;
  try {
    item = await db.findById(itemId);
  } catch (err) {
    logger.error("DB query failed", { itemId, error: String(err) });
    throw new DatabaseError(`Failed to fetch item: ${err}`);
  }

  if (!item) {
    throw new NotFoundError(`Item ${itemId} not found`);
  }

  return toItemDetail(item);
}
```

### Rules

- **Never** catch generic exceptions in services (`except Exception`, `catch (e)` without re-throw)
- **Never** silently swallow errors (`except: pass`, empty `catch {}`)
- **Never** `# type: ignore` / `@ts-ignore` without a comment explaining why
- **Always** `raise ... from e` (Python) to preserve the stack trace
- **Always** log the error BEFORE propagating it
- Business errors (not found, validation) should use custom exceptions, not HTTP exceptions in the service layer

---

## 7. Imports

```python
# Python: strict order -- stdlib > third-party > local
import math
import uuid
from datetime import datetime
from typing import Annotated

from fastapi import APIRouter, Depends       # [your framework]
from sqlalchemy.ext.asyncio import AsyncSession  # [your ORM]

from app.api.dependencies import get_db, verify_auth
from app.db.models.item import Item
from app.schemas.items import ItemResponse
```

```typescript
// TypeScript: strict order -- node builtins > third-party > local
import { randomUUID } from "node:crypto";

import express from "express";          // [your framework]
import { z } from "zod";               // [your validation lib]

import { ItemService } from "@/services/item-service";
import type { ItemResponse } from "@/types/items";
```

- One import per line for local imports (cleaner git diffs)
- Use `TYPE_CHECKING` (Python) or `import type` (TypeScript) for type-only imports
- No wildcard imports (`import *` / `export *`) -- always explicit

---

## 8. Secure Logging

### Rules

```text
1. NEVER use print() / console.log() in production -- use your logging library
2. NEVER log secrets, tokens, passwords, PII (emails, names)
3. ALWAYS log opaque identifiers (order_id, user_id) not raw data
4. ALWAYS log errors BEFORE propagating them
5. Use semantic log levels (not everything as info)
```

### Patterns

```python
# Python example (adapt to your logging library)
from app.logging import get_logger
logger = get_logger(__name__)

# Log levels -- when to use each
logger.debug("Query details", sql=str(stmt), params=params)    # Dev only
logger.info("Order created", order_id=str(order.id))           # Business events
logger.warning("Payment timeout", order_id=oid, attempt=2)     # Degradation
logger.error("DB query failed", error=str(e), table="orders")  # Recoverable error
logger.critical("Auth provider unreachable", url=auth_url)      # System failure

# FORBIDDEN -- sensitive data in logs
logger.info("User auth", token=jwt_token)           # Token in logs
logger.info("Login", email=user.email)               # PII
logger.error("Failed", password=user_password)        # Secret
logger.debug("API call", api_key=settings.api_key)    # API key
```

### Error messages -- client/server separation

```python
# The CLIENT receives a generic message
raise HTTPException(
    status_code=500,
    detail="An internal error occurred. Please try again.",
)

# The SERVER logs the technical detail
logger.error(
    "Score computation failed",
    order_id=str(order.id),
    error=str(e),
    item_count=len(items),
)

# FORBIDDEN -- stack trace or technical detail in the HTTP response
raise HTTPException(
    status_code=500,
    detail=f"Database error: {e}\n{traceback.format_exc()}"  # NEVER
)
```

---

## 9. Input Validation -- Defense in depth

### System boundary = strict validation

```python
# Python: every input crossing a system boundary must be validated

# HTTP boundary -> schema validation ([your validation library])
class CreateOrderRequest(BaseModel):
    customer_id: str = Field(..., min_length=1, max_length=100)
    description: str = Field(..., min_length=1, max_length=10_000)
    quantity: int = Field(default=1, ge=1, le=1000)

# LLM boundary -> parse + validate
def parse_llm_response(raw: str) -> LLMResult:
    try:
        data = json.loads(raw)
        return LLMResult.model_validate(data)
    except (json.JSONDecodeError, ValidationError) as e:
        logger.warning("Invalid LLM response", error=str(e))
        return LLMResult(score=None, status="error", reason=str(e))

# DB boundary -> ORM or parameterized queries (never raw SQL with variables)
stmt = select(Item).where(Item.id == item_id)  # OK
```

```typescript
// TypeScript: Zod, class-validator, io-ts, etc.
const CreateOrderSchema = z.object({
  customerId: z.string().min(1).max(100),
  description: z.string().min(1).max(10_000),
  quantity: z.number().int().min(1).max(1000),
});
```

### Internal code = trust what is already validated

```python
# Between internal modules, do NOT re-validate if already validated upstream
# The service trusts the schema that already validated the input

async def create_order(db, data: CreateOrderRequest) -> Order:
    # data is already validated by the schema -- no need to re-check
    order = Order.create(customer_id=data.customer_id, description=data.description)
    db.add(order)
    await db.commit()
    return order
```

---

## 10. External Dependency Safety

### LLM (Claude, OpenAI, etc.) -- treat as untrusted

```python
# LLM responses are UNTRUSTED content
# Never eval(), exec(), or interpolate into SQL/HTML

# OK -- parse and validate with a schema
parsed = json.loads(llm_response)
validated = ResponseSchema.model_validate(parsed)

# OK -- log for debug (truncated)
logger.debug("LLM raw response", response=llm_response[:500])

# CRITICAL -- direct execution
eval(llm_response)                                        # NEVER
text(f"INSERT INTO results VALUES ('{llm_response}')")    # NEVER
```

### External APIs -- resilience

```python
# Python: timeouts mandatory
async with httpx.AsyncClient(timeout=30.0) as client:
    response = await client.get(url)

# Retry with backoff
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=10))
async def call_external_api(url: str) -> dict:
    async with httpx.AsyncClient(timeout=30.0) as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.json()
```

```typescript
// TypeScript: timeouts mandatory
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 30_000);

const response = await fetch(url, { signal: controller.signal });
clearTimeout(timeout);
```
