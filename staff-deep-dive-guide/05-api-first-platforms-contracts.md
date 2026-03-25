# API-First Platforms, Contracts, Versioning — Conceptual Depth

---

## 1. What “API-first” actually means in practice

**Not:** “We have REST endpoints.”  
**Is:** **Teams agree on machine-verifiable contracts** (OpenAPI/Protobuf) **before** implementation; **consumers** rely on **documented** behavior; **changes** are **governed** like a product roadmap.

**Why Staff cares:** At scale, **integration bugs** dwarf **local** bugs—duplicate orders, silent field misinterpretation, retry storms. Your platform’s **economics** are **retry-safe**, **observable**, and **backward compatible by default**.

---

## 2. Breaking vs non-breaking changes (precise)

**Non-breaking (typical):**

- Add **optional** field with sensible default when absent.  
- Add **new** endpoint.  
- Add **new** enum value **if** clients ignore unknowns (forward-compatible client pattern).

**Breaking:**

- Remove field clients still send.  
- Change type (`string` → `int`).  
- Tighten validation (reject previously accepted garbage).  
- Change **pagination** semantics.

**Staff process:** **Deprecation window**, telemetry on `/v1` **traffic %**, **sunset** comms to partners, **contract tests** in CI.

---

## 3. Versioning mechanics

### URL versioning (`/v1/campaigns`)

**Pros:** Obvious in logs; easy routing at gateway; **CDN** rules simple.  
**Cons:** Explosion of routes; copy-paste handlers unless **adapter** layer maps v1/v2 to core domain.

### Header / content negotiation

Cleaner URLs; **harder** for naive caches; clients forget headers → **support** burden.

**Staff:** Pick one org-wide standard for **public** APIs; don’t let each team invent.

---

## 4. Idempotency — how the server should think

**Goal:** `POST /orders` with same `Idempotency-Key` returns **same resource** after duplicates.

**Server state machine (conceptual):**

1. First request: store `key → processing`, create resource, set `key → {status:done, id:123}`.  
2. Retry while **processing:** return **409** or **202** with **poll** (document clearly).  
3. Retry after done: return **201** with same `123`.

**Storage:** Redis with TTL (24–72h) or **SQL** uniqueness on `idempotency_key`. **Staff:** persist long enough for **worst-case** client retry window.

**Tradeoff:** Storage cost vs **financial** correctness—**always** worth it for **mutations** with money.

### Safe retries for clients

**Exponential backoff** with **full jitter**: `sleep = random(0, min(cap, base * 2^attempt))` avoids **thundering herd** after partial outage.

---

## 5. Pagination: offset vs cursor (why offset dies at scale)

**Offset:** `LIMIT 50 OFFSET 10000` scans **10050 rows** inside DB—**O(offset)** degradation.

**Cursor:** `WHERE (created_at, id) > (:ts, :id) ORDER BY created_at, id LIMIT 50` uses **index**—stable under concurrent inserts if tie-broken by unique `id`.

**Staff caveat:** Cursor pagination needs **documented ordering** and **opaque** cursor encoding (often base64 of tuple).

---

## 6. Rate limiting — token bucket vs sliding window

**Token bucket:** Tokens refill at rate **r**; burst up to **B**. **Pros:** smooth; allows bursts **by design**.  
**Cons:** Long idle → large burst may surprise downstream.

**Sliding window log:** Accurate per-second limits; **memory** heavier. **Approximate** **Redis** implementations exist (e.g., fixed window with small error).

**Distributed limiting:** Central **Redis** **INCR** with **TTL** window—**race** acceptable for **abuse prevention**; **not** for **exact financial** limits without stronger transaction story.

**Staff:** **Per-tenant** quotas + **cost** attribution headers for internal platforms.

---

## 7. Sync command vs async events

**Command path (sync):** “Create trafficking record now; I need **success/fail** in this request.”

**Event path (async):** “`CampaignUpdated` happened; **five** downstreams react eventually.”

**Outbox pattern (depth):** In **same DB transaction** as business update, insert row into **`outbox`**. Separate **publisher** reads outbox, produces to Kafka, marks **published**. Guarantees **no** “DB committed but message lost” without a message—closing the **dual-write** hazard.

**Choreography vs orchestration:**

- **Orchestration:** central workflow engine calls services **A** then **B**—easier to **trace**, can become **spaghetti** at scale.  
- **Choreography:** services listen—**loose**, **harder** global reasoning—needs **sagas** with **compensating** actions.

**Staff:** “We’d orchestrate **short** critical workflows; **choreograph** broad fan-out.”

---

## 8. GraphQL pitfalls (when you say “no”)

**N+1 queries:** Resolver per field hammers DB unless **DataLoader** batching.  
**Complexity limits:** Deep queries can **DoS** your own API—**cost** analysis / depth limits.  
**Caching:** CDN **edge caching** much weaker than **GET** `/resource`.

**When it fits:** Product API where many **clients need different shapes** and you **control** the graph and **resource** limits.

---

## 9. Worked examples (JSON, HTTP, SQL shapes)

### Example A — Non-breaking API evolution

**`GET /v1/campaigns/42` response v1:**

```json
{
  "id": 42,
  "name": "Spring Promo",
  "budget_micros": 5000000,
  "status": "ACTIVE"
}
```

**Additive v2 (old clients still work):** add **optional** fields; old clients **ignore** unknown JSON keys if they use a **tolerant** parser.

```json
{
  "id": 42,
  "name": "Spring Promo",
  "budget_micros": 5000000,
  "status": "ACTIVE",
  "pacing_mode": "EVEN",
  "frequency_cap_daily": 3
}
```

**Breaking mistake:** renaming `budget_micros` → `budget_usd` without **v2** path → mobile app **crashes** on parse.

### Example B — Idempotency key flow (HTTP)

**First request:**

```http
POST /v1/bookings HTTP/1.1
Idempotency-Key: 7f2a9c-...
Content-Type: application/json

{"flight_id": 404, "spots": 2}
```

**Server:** creates `booking_id=9001`, stores `7f2a9c- → {status:created, id:9001}`, returns **201** + body.

**Retry (client timeout, same key):**

```http
POST /v1/bookings HTTP/1.1
Idempotency-Key: 7f2a9c-...
```

**Server:** finds mapping, returns **201** + **same** `booking_id=9001` (or **200** with idempotent GET semantics—**pick and document**).

**Concurrent duplicate:** second in-flight with same key → **409 Conflict** or **202** + `operation_id` until first finishes—**define**.

### Example C — Cursor pagination

**First page:**

`GET /v1/line-items?limit=50`

Response:

```json
{
  "items": [ /* 50 objects */ ],
  "next_cursor": "eyJjIjoiMjAyNS0wMy0wMVQxMTozMDowMFoiLCJpZCI6OTAwfQ"
}
```

**Second page:**

`GET /v1/line-items?limit=50&cursor=eyJjIjoiMjAyNS0wMy0wMVQxMTozMDowMFoiLCJpZCI6OTAwfQ`

**Cursor decodes to** `(created_at="2025-03-01T11:30:00Z", id=900)` — next query:

```sql
WHERE (created_at, id) > ('2025-03-01 11:30:00', 900)
ORDER BY created_at, id
LIMIT 50;
```

### Example D — Token bucket numbers

**Cap** **burst** **100** req; **refill** **20** req/s.

At **T=0**: bucket **100** tokens.  
Spike **80** requests → **20** left.  
**1s** later **+20** → **40** tokens (capped at **100** if refilling forever).

**Staff:** tune **burst** so partners can **warm** caches; **refill** so sustained abuse still **Block**s.

### Example E — Outbox table rows

After booking insert, **same transaction**:

| id | topic | payload | published |
|----|-------|---------|-----------|
| 501 | `bookings.events` | `{"type":"created","id":9001}` | false |

**Dispatcher loop:** `SELECT ... FOR UPDATE SKIP LOCKED` → produce to Kafka → `published=true`.

**Crash after commit but before Kafka:** row still `published=false` → **retry** safely.

### Example F — Mini saga (orchestrated)

**Goal:** reserve inventory **I**, charge payment **P**, create traffick **T**.

1. `ReserveInventory` → OK, token **R1**  
2. `ChargePayment` → **fails**  
3. **Compensate:** `ReleaseInventory(R1)`

**Choreography version:** `InventoryReserved` event → payment service charges → on failure emits `PaymentFailed` → inventory listens and releases. **Harder** to **debug** without **central** trace id.

---

## 10. Interview closer (with reasoning)

> “Public partner APIs stay **REST + OpenAPI** with URL versioning, idempotency keys on all **POST** that spend money, cursor pagination, per-tenant rate limits, and consumer-driven contract tests in CI. Internal state changes that need reliability use **transactional outbox** to Kafka—not naive **dual write**.”

---

## Related

- Modernization / facade: [06](./06-platform-modernization-legacy-linear.md)  
- Reliability: [07](./07-ha-reliability-observability-cost.md)
