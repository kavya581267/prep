# Platform Modernization & Legacy — Conceptual Depth

Linear TV terms: [`../adtech-beginner-guide/08-linear-tv-terms-avail-pod-makegood.md`](../adtech-beginner-guide/08-linear-tv-terms-avail-pod-makegood.md).

---

## 1. Why legacy systems survive

**Beginner:** “Old code” isn’t the core issue—**legacy** means **tightly coupled** to business process, **low test coverage**, **manual** steps, **unknown edge cases**, and **revenue dependence**. The system is **risky to change** even if it **runs**.

**Staff:** Modernization is a **risk management** program: **incremental** value, measurable **defect** reduction, **rollback** always possible. **ROI** is **incident** reduction + **cycle time** + **new revenue** features **unblocked**.

---

## 2. Strangler fig pattern (how it actually works)

**Idea:** Place a **facade** (API gateway, routing layer, or domain service) in front of the legacy system. **New** implementations **handle a slice** of traffic:

- **By feature:** “**New** reporting API reads **new** warehouse; old still serves old UI until cut.”  
- **By segment:** “**EU** traffic uses **new** service; **NA** unchanged until parity proven.”  
- **By read vs write:** **Read** path first (safer if **stale** snapshot tolerable), **write** later with **stronger** guards.

**Mechanics:** Router checks **feature flag** + **tenant** → **delegate** to **v2** implementation or **proxy** to monolith.

**Staff mistake:** Strangler **without** **parity checks**—you discover subtle bugs **in prod** at **cutover**.

---

## 3. Dual-write vs dual-read vs CDC

### Dual-write

Application **writes** to **old** and **new** in same request.

**Pros:** Both stores **warm**.  
**Cons:** **Atomicity** across two systems is **hard**—one succeeds, one fails → **drift**. Requires **reconciliation** job + **alerting**.

**When:** Short **migration window** with **strict** ops monitoring and **automatic** repair.

### Dual-read / shadow read

Still **write** old; **async** compare **read** from new.

**Pros:** **Build** confidence without risking **writes**.  
**Cons:** New store might **never** see **latest** if not fed.

### Change Data Capture (CDC)

DB **transaction log** → stream → **new** system **materializes** read models.

**Pros:** **No** invasive dual-write in app code for reads.  
**Cons:** **Lag**; **schema** changes need **care**; **ordering** per **key** must be handled.

**Staff:** “We’d pair **CDC** to warehouse + **explicit** write path for **new** domain aggregates that old DB can’t represent.”

---

## 4. Cutover choreography

1. **Freeze** schema changes in legacy during final stretch (often painful—negotiate).  
2. **Backfill** new system from historical export + CDC **catch-up**.  
3. **Shadow** traffic: duplicate **reads** or **simulated writes** (safe sandbox).  
4. **Canary:** 1% **writes** to new, **compare** outcomes.  
5. **Promote** gradually; **rollback** = route back + **compensating** job if needed.

**Finance angle:** If **events** already **emitted** to billing, **rollback** may require **reversing** entries—not always possible; **Staff** plans **immutable** **adjustment** events.

---

## 5. Monolith → services: domain boundaries

**Eric Evans’ bounded context** (interview-safe summary): Each **context** has its **own model** of “Campaign”—not necessarily **one** shared `Campaign` table everywhere.

**Modular monolith:** **One deploy**, **modules** enforce boundaries in **code**; **one DB** but **schema** ownership per module—**pragmatic** Staff move before **20 microservices**.

**Microservices:** When teams **ship** independently and **scale** differs **wildly**. **Cost:** distributed **transactions** → **sagas**, **outbox**, **eventual** reads.

---

## 6. Linear + digital convergence (WBD angle)

**Reality:** **Linear** **traffic** and **digital** **ad servers** often started as **separate** **revenue** stacks. **Convergence** means:

- **Shared identifiers** where semantics **truly** match (don’t **force**).  
- **Unified reconciliation**: booked → trafficked → as-run + digital **delivery** → **single** **discrepancy** report.  
- **Human-in-the-loop** where **creative** or **legal** requires eyes—**automate** **validation** around them, not **replace** on day one.

**Staff:** Data model **unification** is **political**—you negotiate **canonical** **entities** with **Sales**, **Ops**, **Finance**.

---

## 7. Org patterns that make modernization succeed

- **Platform** teams ship **paved paths** (templates, golden CI, observability defaults).  
- **Product** teams adopt when **migration** is on their **OKR**.  
- **Executive** cover for **temporary** slowdown during **cutover weeks**.

---

## 8. Worked examples (migration stories you can tell)

### Example A — Strangler by **read** path first

**Legacy:** Oracle monolith `GET /reports/delivery` scans **3** joined tables, **timeouts** at month-end.

**New:** Nightly **Spark** job writes **Iceberg** table `delivery_daily` **partitioned** by `dt`. New **Reports API v2** reads **only** Iceberg for past dates; **today** still falls back to Oracle **until** **hourly** refresh lands.

**Cutover:** Feature flag **per sales region**—**EU** uses v2 first (smaller revenue **risk appetite** agreed with finance). **Rollback:** flip flag **back**, no **data loss** because v2 is **read-only** mirror.

### Example B — Dual-write failure window (why reconciliation exists)

**Operation:** Update **spot status** to `AIRED`.

| Step | Legacy DB | New Postgres |
|------|-----------|--------------|
| 1 | ✓ commit | **timeout** |
| 2 | *already changed* | retry later writes **stale** view |

**Reconciliation job:** hourly **diff** `legacy_spots` vs `new_spots` on `updated_at` + hash of fields; **auto-fix** when **confidence high**, **queue** for human when **conflict**.

### Example C — CDC lag scenario

**Debezium** emits `spot id=77` **status change** **90s** after Oracle commit.

**User** hits new UI **60s** later → still sees **OLD** status from **Postgres** replica fed by CDC.

**Mitigations:** **read-after-write** route for **same user session** hits **legacy** short term; or **SLO**: “new UI **≤ 120s** eventual consistency”; or **sync** **critical** fields via **dual-write** temporarily.

### Example D — Canary write **1%**

**Population:** hash(`traffic_id`) % 100 == 0 → **new** write service for `POST /trafficking/assign`.  
**Compare:** async job **diffs** **legacy vs new** **derived** `assignment_summary` table. **Abort** canary if **>0.01%** row mismatch.

### Example E — Linear + digital ID (political)

**Digital** `campaign_id` ≠ **linear** **traffic instruction id**—sales insists “one spreadsheet.”

**Staff compromise:** Introduce **`commercial_deal_id`** **umbrella**; map **both** systems in **reference table**. **Don’t** merge **schemas** until semantics **proven** same—avoid **silent** errors on **makegoods**.

### Example F — Rollback after finance consumed event

**Mistake:** published **`revenue.recognized`** for wrong region.

**Code rollback** **cannot** unsend Kafka. **Compensating event:** `revenue.reversal` with **pointer** to bad **event_id**; downstream **systems** implement **double-entry** style adjustments.

---

## Related

- APIs after extraction: [05](./05-api-first-platforms-contracts.md)  
- Running safely: [07](./07-ha-reliability-observability-cost.md)
