# Alternate System Design Prompts — Walkthrough Depth

Use **35–40 minutes** per prompt: **5** clarifying, **25** design, **10** failure/observability/cost.

---

## Prompt 1 — Billions of impressions → realtime aggregates + daily truth

### Beginner goal

**Ingest** high-volume events; **serve** ops dashboards quickly; **produce** finance-grade numbers **daily**.

### Design depth

**Ingress tier:** Stateless **API** or **collector** **batches** small posts → **validate** schema, **reject** garbage fast (size limits, **known** fields). **Auth** for **trusted** publishers.

**Buffering:** Kafka (or cloud-native queue). **Partition key:** Start with **salting** of `campaign_id` to avoid hot partition; **maintain** `request_id` in payload for dedupe.

**Stream aggregate path:**  
Flink/KStreams: **tumbling** 1-min windows by `(placement, country)` maybe—not too **high** cardinality. **State backend** RocksDB; **checkpoint** to durable storage for **failure** recovery.

**Approximate** realtime is OK if contract says so.

**Batch truth path:**  
Land **raw** to **object storage** **hourly** partitions. Spark job **dedupes** `request_id`, **aggregates** to fact table **grain** required by finance (may differ from ops **grain**).  
**MERGE** into Iceberg/Delta/BQ snapshot.

**Reconciliation job:** Compare **rolling 24h** stream totals vs **batch** for same window—**threshold** alert; **finance** chooses **authority**.

### Staff tradeoffs

- **Kafka** vs **managed** pub/sub—**ops** burden vs **control**.  
- **Exactly-once** stream: usually **skip**; **dedupe** in batch **simpler** to explain.  
- **Cardinality** in realtime:** **pre-aggregate** vs **roll up** in **batch** for rare dimensions.

### Failure modes

- **Lag:** **autoscale** consumers; **dynamic** **discard** policy for **non-critical** dimensions only with **product** sign-off.  
- **Duplicate producers:** **idempotent** sink using `request_id` **unique key**.

### Mini example — one impression event (shape)

```json
{
  "request_id": "req-41c2-8891",
  "event_ts": "2025-03-25T14:30:01.120Z",
  "placement_id": "ctv_preroll_sports",
  "campaign_id": 12011,
  "country": "US",
  "price_micros": 95000
}
```

**Dedupe key** in batch: **`request_id`** only (not `(campaign_id, ts)`—**collisions** on **burst**).

---

## Prompt 2 — Multi-tenant planner + partner API

### Design depth

**Identity:** **OAuth2** client credentials for partners; **JWT** with **org_id** claim for internal.

**Authorization:** **RBAC** + **row-level** `tenant_id` on every **query**—**middleware** enforces, **no** raw SQL from handlers without **binding**.

**Noisy neighbor:** **Per-tenant** rate limits at **gateway**; **bulkhead** **connection pools** to **shared** DB.

**Data model:** **Shared** schema + **`tenant_id`** on all tenant-owned rows; **unique** indexes include `tenant_id`. **Alternative:** **schema per tenant** for **regulatory** isolation—**ops** **harder**.

**Versioning:** `/v1` **frozen** behavior; `/v2` introduced with **interop** period.

**Long operations:** **202 Accepted** + `operation_id` + **GET** status; internal **queue** (Kafka/SQS).

### Staff tradeoffs

**DB per tenant:** **isolation** vs **connection** explosion.  
**Eventual** **search index** (Elasticsearch) vs **SQL** only—**complexity**.

### Mini example — tenant isolation query

**Bad:** `SELECT * FROM line_items WHERE id = 900` → **cross-tenant** leak if IDs collide.

**Good:** `SELECT * FROM line_items WHERE tenant_id = $T AND id = 900` with **composite unique** `(tenant_id, id)`.

**JWT:** claim `"org_id":"acme"` must **match** row `tenant_id` on every **mutation**—**middleware** enforces, not each **handler** ad hoc.

---

## Prompt 3 — Notification fanout

### Design depth

**API:** Accept `notification_job` with **template id**, **channel**, **audience** spec (segment id or explicit list **bounded** size).

**Durability:** **Persist** job in DB or **log** to Kafka before **ack** “accepted.”

**Workers:** **Partition** by **channel** or **tenant** for **fairness**.  
**Provider adapters** wrap **FCM**, **APNs**, **email** vendor—each has **batch** limits and **retry** policies.

**Idempotency:** **`notification_id`** dedupe at adapter when vendors **retry**.

**Template rendering:** isolate **HTML** injection—**escape**; **sandbox** if user-supplied snippets.

### Staff tradeoffs

**Push** priority lanes (OTP vs marketing)—**separate** queues.  
**Cost:** email **cheap**, SMS **expensive**—**quota** per tenant.

### Mini example — job lifecycle

1. `POST /jobs` → persist row `jobs(id=55, state=PENDING)` → enqueue **Kafka** `notifications.jobs` **partition** `hash(tenant_id)`.  
2. Worker **consumes** → calls **FCM** batches of **500** device tokens.  
3. **Failure:** **FCM** **429** → **exponential backoff** **per tenant**; after **5** tries → DLQ `jobs.dlq` + **page** **on-call** if **OTP** lane.

---

## Prompt 4 — Rate limiter at edge

### Design depth

**Local edge limiter (token bucket in memory):** **Microseconds**; **counts** per instance only—**uneven** under load balancer **unless** **sticky** or **small** correction accepted.

**Global limiter (Redis):**

- **INCR** `key:{tenant}:{window}` with **TTL** matching window—**fixed window** approximate.  
- Or **sliding** log with **sorted set**—more accurate, more memory.

**Hybrid:** local **fast reject** + async **global** reconcile for **abuse** patterns.

**Response:** **429** + `Retry-After` + **problem+json** body with **limit** **policy** link.

### Staff tradeoffs

**Strict fairness** vs **performance**—distributed systems **rarely** **perfect** counting at **millisecond** precision **without** **coordination** cost.

### Mini example — Redis fixed window

Key `ratelimit:acme:minute:202503251430` → `INCR` each request; if **value** > **1000** → **429** `Retry-After: 42` seconds until **window** rolls.

**Race:** Two **parallel** requests both read **999** → both pass—**accept** **~0.1%** **over** for **abuse** prevention; **don’t** use for **strict** **billing** caps **without** **Lua** script or **stronger** **store**.

---

## Prompt 5 — Feature store

### Design depth

**Offline:** scheduled jobs compute **features** into **versioned** tables `feature_set_v3` keyed by `entity_id`, `as_of_ts`.

**Online:** embed **low-latency** **key-value** **lookup** in ranking path—**batch** **preload** **hot** entities if needed.

**Join:** training uses **point-in-time** join—never join future rows.

**Consistency:** online may **lag** offline by **N minutes**—document **staleness** SLO.

**Failure:** ranking uses **defaults** when feature missing—**flag** **degraded** mode metrics.

### Staff tradeoffs

**Coupled** **embedding** **updates** vs **decoupled** **batch** refresh—**freshness** vs **simplicity**.

### Mini example — online read path

**Ranking** requests feature **`user_total_impressions_7d`**.

- **Offline table** (yesterday): `7_200`.  
- **Online Redis** (streaming **deltas**): `+35` **today** → **served value** `7_235`.  
- **Redis miss:** fall back to **offline only** `7_200` + **log** **`feature_degraded=1`** metric.

**Training row** for **click** at **2025-03-20 15:00Z** uses **`AS OF`** snapshot ending **that** **timestamp**—**not** `7_235` from **March 25**.

---

## 6. Cross-prompt worked numbers (quick reference)

Assume **unless interviewer corrects:**

| Quantity | Illustrative value |
|----------|---------------------|
| Impression event size (JSON gz) | **0.5–2 KB** |
| Peak ingest | **100k–1M** events/s (company dependent) |
| Stream window for ops | **1 min** tumbling |
| Batch **SLA** to finance | **T+4h** or **daily** |
| API tenant count | **50–500** partners |
| Notification batch to FCM | **500** tokens/request (check current limits) |
| Rate limit (partner tier Gold) | **1000 RPM** burst **2000** |

Use these only as **scaffolding**; always **ask** for **real** SLOs.

---

## 7. End-to-end mini scenario — Prompt 1 only (deep)

**Goal:** Prove you can **chain** pieces with numbers.

**Ingress:** Load balancer → **100** **stateless** **collectors** (auto-scaled). Each accepts **POST /beacon** **batched** **100** events, **max body** **256KB**.

**Kafka:** Topic `impressions-raw`, **96** partitions, key = `hash(request_id)` for **uniform** spread (dedupe later by id).

**Stream:** **Flink** job: **1-min** **tumbling**, **group by** `(country, placement_type)` — **~200** groups → **feasible**.

**Storage:** **Druid** or **ClickHouse** for **ops** UI; refresh **every 30s** from **sink**.

**Batch:** **Hourly** Spark **raw** `s3://raw/dt=.../hour=...` **Parquet**; **dedupe** `request_id`; **merge** to **`fact_impressions`**.

**Reconciliation:** **`stream_24h`** sum vs **`batch_24h`** — **alert** if **relative error** > **0.25%**.

**Cost talk:** “Biggest **\$** risk is **Kafka** retention **365d** on **raw**—**tier** or **sample** **debug** topic.”

---

## Related

- Kafka detail: [04](./04-data-streaming-kafka-spark-lakehouse.md)  
- Core ad path: [03](./03-system-design-ad-serving-experiments.md)
