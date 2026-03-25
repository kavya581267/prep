# Tradeoffs: Reach vs Frequency, Precision vs Scale, Analytics vs Privacy

This file expands **Section 10.2** from your interview prep into **beginner-friendly depth**. These are **not** math quizzes—they are **product + systems** tensions you should discuss calmly.

---

## 1. Reach vs frequency caps (brand campaign)

### Definitions (quick)

- **Reach:** **how many unique users** (or households) saw the campaign **at least once** in a period.  
- **Frequency:** **how many times**, on average, each reached person saw the ad—often summarized as **average frequency** or distribution (1×, 2×, 3×+).

A **frequency cap** **limits** repeats (e.g., max 3/day).

### The tension

**Brand campaigns** often want **broad reach**—“make sure lots of **different** people see us.”  
But **uncapped** delivery can hammer the **same** heavy viewers (**heavy tails** on streaming), **wasting** money and **annoying** users.

**Caps** protect UX and can **increase reach** efficiency by **forcing** the system to find **new** users—**but** if caps are **too tight** in a **small** audience pool, you may **fail to spend** budget or **under-deliver** promised impressions.

### Interview-style answer skeleton

1. **Clarify objective:** awareness vs response?  
2. **Propose measurement:** **unique reach** + **frequency distribution**, not only raw impressions.  
3. **Propose controls:** **caps** by device/household; **pacing**; **creative rotation** to reduce fatigue.  
4. **Tradeoff:** stricter caps → better UX & cleaner reach curves → risk **underspend** or **slower** learning in A/B tests on low-traffic placements.

### Tiny example (numbers are illustrative)

- **Universe:** 1M people.  
- **Plan A:** no cap → might show **power users** 50 times; **reach** might still look “good” in crude dashboards while **wasting** impressions.  
- **Plan B:** max 5/week → pushes the optimizer to **find fresh** users; **average frequency** drops; **reach** rises for same spend—**until** the audience is **saturated**.

---

## 2. Precision of targeting vs scale of audience

### Definitions

- **Precision targeting:** narrow segments—“**women 25–34**, **in-market for SUVs**, **in these zips**.”  
- **Scale:** enough people actually **see** the ads to meet **budget** and **statistical** needs.

### The tension

**Narrow** targets **improve relevance** (higher CTR/CPA potential) but **shrink** the pool. You might **win every auction** in a tiny segment and still **miss** the quarterly **impression** goal.

**Broad** targets **scale** easily but may **dilute** performance and **increase** waste—unless **creative**/context carries the lift.

### Engineering/system angle

- **Precision** often needs **richer** **identity** and **features** → **privacy** scrutiny, **data** pipeline complexity, **cold-start** users missing signals.  
- **Broad** targeting stresses **volume** systems: **lower** CPM inefficiency can still **dominate cost** at scale.

### Interview-style answer skeleton

Start **business constraints**: “What’s the minimum **reachable** audience this week?”  
Then **technical**: **feature availability** latency, **coverage** (% users with segment), **exploration** to find lookalikes, and **gradual** widening (**inventory expansion**) when models stall.

---

## 3. User-level analytics vs privacy / regulatory constraints

### What “user-level analytics” means

Storing and joining **per-person** events: what you saw, clicked, converted—with **long retention**—to power **attribution**, **segmentation**, **experiments**.

### The tension

More **granularity** → better optimization and clearer **experiment** reads.  
But **GDPR**, **CCPA**, **platform policies**, and **user expectations** limit **collection**, **retention**, **sharing**, and **secondary use**.

### Concepts beginners should know

- **Purpose limitation:** collect for stated purposes.  
- **Retention limits:** don’t keep PII forever “just because.”  
- **Consent / opt-out:** regions differ; **global apps** branch logic.  
- **Anonymization vs pseudonymization:** **hashed IDs** may still be **personal data** legally.  
- **Data minimization:** store **only** what you need for the job.

### Technical patterns (high level)

- **Aggregation** for reporting (cohort tables) instead of raw **row-level** exports.  
- **Differential privacy** (sometimes) for **product analytics** at massive scale—mention only if you know basics.  
- **Clean rooms** for **partner** analytics without raw PII sharing (deep only if role is data-heavy).  
- **On-device** processing for some metrics (conceptual).

### Interview-style answer skeleton

“I’d pair **legal/privacy** review early, define **data categories**, default to **minimization**, split **billing-grade** logs from **analytics** logs, enforce **TTLs**, and design **experiments** so **variants** don’t leak **sensitive** attributes in logs.”

---

## 4. Cross-link: A/B testing gotcha

Heavy **frequency** strategy changes **who** is eligible for **experiments**—can **bias** cohorts. Staff candidates **name** that risk and **mitigate** (stable assignment, **stratification**, **guardrails**).

---

## 5. One-page summary

| Tradeoff | Core conflict | What a strong candidate says |
|----------|---------------|------------------------------|
| Reach vs frequency | Unique people vs repeats | Measure **reach curves**; caps have **spend** side effects |
| Precision vs scale | Relevance vs volume | Watch **coverage**; widen with **controls** |
| Analytics vs privacy | Insight vs obligation | **Minimize**, **TTL**, separate **billing** vs **product** data |

You’ve finished the beginner track. Revisit [START HERE](./00-START-HERE.md) for navigation.
