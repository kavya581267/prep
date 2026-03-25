# Resilience & Cost Patterns — Extended Catalog (Concepts, Not Just Bullets)

Companion to [07](./07-ha-reliability-observability-cost.md).

---

## 1. Load shedding

**Concept:** When overload is inevitable **by design** (capacity finite), **choose** which work to drop.

**Beginner example:** Web tier detects **queue depth** > threshold → **503** for **free tier** requests while keeping **paid** tier **SLA**.

**Staff depth:**

- **Classification** of requests must exist **before** crisis—**product** agrees **tier** definitions.  
- **Signals:** CPU, **thread pool** saturation, **latency** **SLO** burn, **dependency** health.  
- **Safety:** Shedding **revenue-critical** path last—**explicit** policy document.

**Tradeoff:** Temporary **user** anger vs **full** **collapse** (everyone gets 500s + **retry** storm).

---

## 2. Rate limiting vs load shedding

**Rate limit:** **Per actor** fairness cap—**prevents** abuse or accidental **loops**.  
**Load shed:** **Global** survival lever when **aggregate** **good** traffic is too heavy.

**Staff:** Stack **both**—rate limit **first line**, shed **last line**.

---

## 3. Backoff + jitter on retries

**Exponential backoff** `base * 2^n` without jitter: many clients retry **in sync** after outage ends → **thundering herd**.

**Full jitter** (AWS pattern): sleep `random(0, cap)` where cap grows—**spreads** retries.

**Staff:** **Cap** max attempts; **circuit break** dependency to **stop** pointless retries.

---

## 4. Bulkheads (resource isolation)

**Concept:** Like ship **watertight** compartments—**flood** in one doesn’t **sink** entire vessel.

**Implementation:** Separate **HTTP client** pools, **thread pools**, **bulkhead** for **S3** vs **DB** calls.

**Tradeoff:** **Lower** **peak** throughput per instance (reserved capacity) for **survivability**.

---

## 5. Cache-aside vs write-through vs write-behind

### Cache-aside (lazy loading)

App: read cache miss → read DB → set cache.

**Pros:** Simple; cache failures **don’t block** writes.  
**Cons:** **Race** on stampede; **stale** reads until TTL.

### Write-through

Write **updates** cache **and** DB together (or cache first per policy).

**Pros:** **Fresher** cache.  
**Cons:** **Write path** latency; **cache** down complicates **write** semantics.

### Write-behind

Write to cache, **async** **flush** to DB (journal required).

**Pros:** **Fast** writes.  
**Cons:** **Durability** window on crash—**Staff** only with **journaling** and **replay**.

### Cache stampede mitigation

**Single-flight** (one loader per key), **early expiration** with **probabilistic** refresh, **prefetch** on known **invalidation** events.

---

## 6. Sharding, hot keys, replicas

**Sharding:** partition data by **tenant_id** or **user_id** onto **different** **physical** stores.

**Hot key:** **One** shard gets **most** traffic—mitigate by **splitting** hot key **range** or **application-level** **subkeys**.

**Replicas:** **Async replication** → **stale reads**; **measure** **replication lag** as SLI if **risky**.

**CQRS:** **Writes** → **OLTP** model; **reads** → **denormalized** **projections** via **events**.

**Tradeoff:** **Dual** code paths; **eventual** consistency **bugs** if UI assumes **instant** read-after-write.

---

## 7. Messaging: outbox pattern (step-by-step)

1. Begin **DB transaction**.  
2. Update **business** row (`order` **status**).  
3. Insert **outbox** row `{topic, payload, not_sent}`.  
4. **Commit**.  
5. Outbox **dispatcher** reads, **produces** to Kafka, marks **sent** (or deletes).

**Why:** Without outbox, **order** committed but **message** `lost` if **producer** fails **after** commit—classic **dual-write**.

**Tradeoff:** **Latency** + **infra**; **strong** consistency **story**.

---

## 8. Saga vs choreography

**Saga:** **Multi-step** business **transaction** across services with **compensating** steps:

- Book inventory → charge payment → ship.  
- If ship fails → **refund** + **release** inventory.

**Orchestration:** Central **coordinator** explicit **state machine**.  
**Choreography:** Services **listen** and **publish**—**implicit** saga.

**Staff:** Orchestration for **tightly coupled** money flows; choreography for **wide** analytics fan-out.

---

## 9. Dead letter queues (DLQ)

Poison message **crashes** consumer repeatedly → **move** to **DLQ** after N tries.

**Requires:** **Tooling** to **inspect**, **replay** safely (maybe **transformed**), **alert**.

**Tradeoff:** **Infinite** retry **blocks** partition; **DLQ** **too fast** **hides** bugs.

---

## 10. mTLS, secrets, least privilege

**mTLS:** Both **client** and **server** present **certs**—strong **service identity**.

**Secrets:** Prefer **short-lived** tokens from **identity** service; **rotate** **certs** automatically.

**Least privilege IAM:** Scope **production** roles **per** **service**—**blast radius** of **credential** leak reduced.

**Tradeoff:** **Developer** friction—**automation** (IaC) pays off.

---

## 11. Cost patterns (expanded)

**Kafka retention:** Not all topics need **7 years**. **Tier** “**compliance**” vs **ops** topics.

**S3 lifecycle:** **Standard** → **IA** after 30 days → **Glacier** for **legal** hold—**retrieval** SLA tradeoff.

**Spark:** **Right-size** **executors**; **shuffle** heavy jobs **need** **memory headroom**; **spot** with **checkpoint** for **fault** tolerance.

**Egress:** **Cross-region** analytic **queries** pull **petabytes** → **co-locate** compute **near** data.

---

## 12. Worked scenarios (story + numbers)

### Scenario A — Load shed during dependency storm

**Symptom:** DB **CPU** **95%**; **p99** **2s**; only **budget** left **edge** **cache**.

**Policy (pre-agreed):** **Shed** **recommendation** **personalization** (**20%** traffic) → static **popular** **default** **list**.

**User-visible:** **slightly** **less** **relevant** rails; **core** **checkout** stays **up**.

**Metric:** `shed_events_total{tier="recs"}` for **postmortem**.

### Scenario B — Cache stampede math (intuition)

**Hot key TTL** **60s**; **50k** **QPS** on key; at **TTL** expiry **all** **miss** together → **50k** **DB** **queries**/**s** vs normal **500** **cache** hits.

**Single-flight** collapses to **1** **DB** read + **fan-out** fill.

### Scenario C — Bulkhead thread pools

**Pool A:** **DB** queries **max** **100** threads.  
**Pool B:** **S3** **max** **20** threads.

**S3** slow → **B** **saturates**; **A** still **serves** **read** API.

### Scenario D — Write-behind **danger**

**Accept** **write** in **20ms**; **batch flush** **every** **200ms**.  
**Crash** **150ms** after **accept** → **loss** of **unflushed** **15** writes—**unacceptable** for **payments** without **durable** **journal** **disk**.

### Scenario E — Outbox **replay**

**Bug** publishes **wrong** `payload` for **10 minutes**.

**Fix code** **then** **replay** outbox **rows** with **`published=true`**? **Danger**—**double** **side effect**.

**Instead:** **compensating** **events** or **manual** **data** **repair** + **new** **outbox** rows—**Staff** knows **replay** **isn't** **free**.

### Scenario F — Saga compensation order

**Charge** succeeded, **ship** failed → **refund** **must** **happen** **before** **inventory** **released**? **Depends** on **financial** **rules**—**orchestrator** encodes **legal** **ordering**.

---

## Related

- HA narrative: [07](./07-ha-reliability-observability-cost.md)
