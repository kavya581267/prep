# Operational & Engineering Excellence — Conceptual Depth

---

## 1. What this round is really testing

Interviewers are building a **belief** that you **won’t** be a **reliability liability** at Staff level: you **anticipate** failure, **measure** quality, **teach** others, and **balance** speed with **safety** in **political** situations.

This is **not** “name Kubernetes.” It is **judgment** under uncertainty.

---

## 2. The production incident lifecycle (mechanical depth)

### Detection

Sources: **monitoring** alerts, **customer** tickets, **internal** dogfood, **chaos** findings.

**Staff gap to close:** **Mean time to detect (MTTD)** — blind spots mean **Sev-1** discovered on **Twitter**.

### Triage

On-call **declares severity** using a **rubric** (revenue, data loss, **regulatory**, user % affected). **Communications** channel **frozen** format: status, impact, ETA unknown/rough, next update in **X min**.

### Mitigation first, root cause second

**Stop bleeding:** **rollback**, **feature kill**, **traffic shift**, **scale up**, **disable** bad dependency.  
**Staff:** Resist **hero debugging** during **fire**—parallelize: one **IC** drives mit, one **collects** timeline.

### Resolution & post-incident

**Blameless** doesn’t mean “no accountability”—means **systems** failed, **humans** acted rationally with bad **incentives** or **missing** safeguards.

**Five Whys** is a **tool**, not religion—stop when **actionable** systemic lever found.

### Follow-through

Track **action items** like any **product** backlog: **owner**, **due date**, **verification metric**.

---

## 3. SLOs in org practice (beyond definitions)

**Picking an SLI:** Must be **customer-centric** or **revenue-centric**, not “CPU low.” For ads: “**successful** decision requests **with** **p99 latency** under **Y**.”

**Toil budget:** **Google SRE** idea—if team spends **>50%** on manual ops, **automation** project gets priority.

**Error budget policy examples:**

- Budget **healthy** → risky launches OK with **usual** gates.  
- Budget **low** → **freeze** **non-essential** releases; **focus** reliability.  
- Budget **exhausted** → **only** fixes—**VP** conversation.

**Staff:** Tie budget to **business quarters**—Finance understands **tradeoff** language.

---

## 4. Quality gates (what “ready for prod” contains)

Conceptual layers:

1. **Design review** for **blast radius**.  
2. **Automated tests** (unit + contract + **load** for critical paths).  
3. **Progressive delivery:** **canary** (% traffic), **auto** rollback on **SLO** breach.  
4. **Runbooks** + **dashboards** + **alerts** **exist before** launch (“definition of ready”).

**Staff at WBD-scale:** For **revenue-critical** path, argue for **shadow** traffic or **parallel** read validation during **migration**.

---

## 5. Mentoring Staff vs seniors vs juniors

- **Junior:** **skills** + **confidence** + **safety** (pairing, small tasks).  
- **Senior:** **ownership** of ambiguous projects; teach **tradeoff** articulation.  
- **Staff-candidate senior:** **multiply**—coach on **influence**, **writing**, **RFCs**, **cross-org** alignment.

---

## 6. Tech debt quantification (credible formulas)

Examples interviewers accept:

- **Incident hours** × **eng hourly** + **customer** impact **$** (rough).  
- **Cycle time** delay: **feature** blocked **N weeks** × **opportunity cost** (even directional).  
- **Interest** metaphor: **velocity** down **X%** after each release due to **flaky** tests.

No need for fake precision—**order of magnitude** + **assumption** transparency.

---

## 7. Strong incident story checklist

- Clear **timeline** with **UTC** and **decision** points.  
- **Customer** impact **scoped**.  
- **Mitigation** time vs **total** outage.  
- **Root** cause at **system** level.  
- **Prevention** with **metrics** proving improvement **later**.

---

## 8. Worked examples (timelines, numbers, scripts)

### Example A — Fictional incident timeline (how to narrate)

| Wall time (UTC) | Event |
|-----------------|-------|
| **14:02** | Pager: `decision_api_p99 > 800ms` for **5m** |
| **14:04** | IC joins; confirms **only** **us-east**; error budget **burn** visible |
| **14:08** | **Mitigation:** toggle **feature flag** `USE_SEGMENT_V2=false` (suspect) |
| **14:11** | Latency **recovers**; **Sev-1** declared |
| **14:45** | **Root:** bad deploy **N+1** to segment service **cold cache** |
| **Day+1** | **PRC:** add **load test** + **cache warm** on deploy |

**Numbers you’d substitute:** **%** requests affected, **$** **estimated** underdelivery (even rough).

### Example B — Comms cadence template

**T+0:** status.internal “**Investigating** latency; next update **15m**.”  
**T+15:** “**Mitigation** applied: disabled **feature X**; p99 **back** to **120ms**; **root cause** unknown.”  
**T+60:** external status page (if customer-visible).

**Staff:** Never **silent** > agreed window—**trust** loss **worse** than imperfect info.

### Example C — Postmortem action items (measurable)

| Action | Owner | Verifier |
|--------|-------|----------|
| Add **integration test** covering **N+1** path | Team A | CI job **fails** if query count > **3** |
| **SLO** on segment **dependency** + **burn alert** | Team B | **Dashboard** linked |
| **Runbook** update: **flag** **location** | IC | **Dry run** |

### Example D — Error budget policy (story you tell)

“We **paused** **non-critical** releases for **two weeks** after we **burnt** **60%** of monthly budget in **48h**. **VP Eng + PM** agreed; we **paid down** **paging** debt. **Post:** incident **rate** down **40%** quarter over quarter—I’d caveat **seasonality**.”

### Example E — Toil calculation (rough)

**On-call** spends **6h/week** resetting stuck **Kafka** consumer groups × **4** engineers × **\$150/hr** loaded cost ≈ **\$3.6k/week** → **\$180k/year** **toil**—**automation** project with **2 week** build pays back **fast** on paper.

### Example F — Mentoring contrast

**Junior:** “Pair on **unit test** patterns for **async**; **review** within **4h**.”  
**Senior:** “You **own** **RFC** draft; I’ll **challenge** assumptions in **section 4** only—**practice** **writing**.”

---

## Related

- SLO math & observability: [07](./07-ha-reliability-observability-cost.md)  
- Leadership overlap: [09](./09-staff-leadership-influence.md)
