# Staff Leadership & Influence — Conceptual Depth

---

## 1. What “Staff” behavior is (not a title checklist)

**Staff ICs** are expected to **set technical direction**, **resolve cross-team ambiguity**, **raise quality bars**, and **develop** other engineers—often **without** reporting authority.

**Anti-pattern:** “**I’m** the smartest implementer” — Staff **multiplies** output through **clarity**, **systems**, and **trust**.

---

## 2. Influence without authority — mechanics

**Sources of influence:**

- **Competence:** shipped hard things **credibly** explained.  
- **Clarity:** **options** with **tradeoffs**, **recommendation** with **assumptions**.  
- **Relationships:** **listen** first; **credit** publicly; **challenge** privately.  
- **Process:** **RFCs** with **comment period**; **design review** **criteria** everyone knows.

**Story arc interviewers want:** **Conflict** → **data** gathered → **alignment** meeting → **decision** recorded → **follow-through** measured.

---

## 3. RFC / design doc anatomy (usable template)

1. **Context & problem** — why now, **business** trigger.  
2. **Goals & non-goals** — **explicit** scope; **non-goals** prevent endless debate.  
3. **Options** — **at least two** real alternatives, not strawmen.  
4. **Tradeoffs** — **latency, cost, operability, time-to-market**.  
5. **Recommendation** — one path; **what would change recommendation**.  
6. **Rollout** — flags, **migration**, **rollback**.  
7. **Open questions** — **decisions** needed from **which** partner team.

**Staff:** Short docs beat long **bikeshedding**; **appendix** for depth.

---

## 4. Navigating Product ambiguity

Product says: “Make **personalization** better.”

**Staff response pattern:**

1. **Clarify outcome metric** (CTR, revenue, watch time, **incrementality**?).  
2. **Constraints** (latency budget, privacy, **deadline**).  
3. **MVP** slice that **ships** in **6 weeks** vs **2026 vision**.  
4. **Risks** (bias, **cold start**) on the table **early**.

---

## 5. Conflict between engineering teams (depth)

Common causes: **shared** service **SLO** miss, **breaking** API change, **priority** dispute.

**Resolution layers:**

1. **Working group** with **engineers** + **EMs**.  
2. **Escalation** path **defined** (Dir level) **with data**.  
3. **Temporary** **decision** **owner** named—**no** endless consensus.

**Staff:** Document **precedent** so **next** conflict is cheaper.

---

## 6. STAR with Staff upgrades

Beyond STAR, add:

- **Stakeholders** map (who cared).  
- **Mechanism** you left behind (playbook, **Dashboard**, **lint rule**).  
- **What you’d do differently**—one **specific** thing.

---

## 7. Ethics & privacy escalation

Pattern: PM requests **data** that feels **wrong**.

**Steps:** Clarify **legal** position; escalate to **security/privacy** partner; propose **less invasive** alternative; **document** if overruled (**rare** but **CYA** for you and company).

---

## 8. Scaling yourself

**Stop:** being **only** **critical path** coder.  
**Start:** **enable** **review** rubrics, **templates**, **office hours**, **delegated** **ownership** with **guardrails**.

---

## 9. Worked examples (STAR snippets, RFC stub, dialogue)

### Example A — STAR snippet: influence without authority (**condensed**)

- **S:** Two teams disputed **Kafka** topic **ownership**; **releases** blocked **3 weeks**.  
- **T:** I was **tech lead** on **consumer** side, **no** **Mgr** authority.  
- **A:** Ran **1h** workshop; published **1-page** **decision matrix** (SLO, on-call, **cost**); **escalated** **single** owner to **Dir** with **options** **A/B**.  
- **R:** **Owner** assigned; **5** Teams adopted **shared** **library**; **unblock** landed in **4 days** after decision.  
- **Learning:** **Time-box** debate; **exec** **decisions** need **two** real **options**, not **seven**.

### Example B — RFC stub (outline you could type in 20 min)

```text
Title: Standardize async domain events for Trafficking
Problem: 3 duplicative integrations; ordering bugs in finance.
Non-goals: Replace sync APIs this quarter.
Options:
  A) Central event bus + outbox (recommended)
  B) DB triggers → queue (ops risk)
Recommendation: A — outbox pattern, schema registry, CI compat checks.
Rollout: Pilot team X 4 weeks; dark publish 2 weeks.
Risks: Consumer lag; mitigation autoscale + DLQ playbooks.
```

### Example C — Product ambiguity dialogue

**PM:** “We need **real-time** dashboards.”  
**You:** “**What decision** changes in **<5 minutes** if the number moves? If none, **hourly** **batch** **saves** **\$200k/year** compute—I’ll **map** options.”  
**PM:** “**Ops** **cancels** bad **flights** within **10m**.”  
**You:** “Then SLO is **≤2m** **ingest lag** **p95** for **ops** **view**; **finance** still **T+1** **authoritative**.”

### Example D — Conflict de-escalation script

“**I** **hear** Team **A** needs **schema** stability; Team **B** needs **velocity**. **Data:** **14** **breaking** changes **last** quarter caused **4** incidents. **Proposal:** **monthly** **compat** window + **Pact** tests—**can** we **trial** **60** days?”

### Example E — Ethical pushback (short)

**PM:** “Log **raw** **email** content for debugging.”  
**You:** “**PII** **minimization** policy blocks; **proposal:** **hash** + **TTL** **7d** **dev-only** **sandbox** with **access** **audit**.”

---

## Related

- Incidents & OE: [08](./08-operational-engineering-excellence.md)  
- Loop strategy: [01](./01-interview-loop-and-calibration.md)
