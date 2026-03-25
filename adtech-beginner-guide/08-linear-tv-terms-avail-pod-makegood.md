# Linear TV Terms: Avail, Pod, Makegood, Traffic Log Reconciliation

These terms come from **traditional linear television** (broadcast/cable schedules). Jobs like the WBD Staff Engineer description often bridge **linear** and **digital** ads—so beginners should **recognize** this language even if they never worked at a network.

---

## 1. Linear TV (definition)

**Linear TV** means viewers watch a **scheduled** channel at a **set time** (as opposed to fully on-demand, though **hybrids** exist). **Ad breaks** are **planned** in the **program** timeline.

---

## 2. Avail (availability)

**Definition:** An **avail** is a **block of commercial time** **available to be sold**—“we have **2 minutes** of national avails in this episode’s break.”

**Why it matters:** Sales and planners **allocate** advertiser **spots** into avails. If forecasting says you have **more** avails than reality, you **over-sell**; if **fewer**, you leave money on the table.

**Digital analogy:** Not 1:1, but think “**how many pre-roll pod seconds** can we monetize this week?”—**inventory**.

---

## 3. Pod

**Definition:** A **pod** is a **group of ads** shown **together** in one commercial break—“a **90-second pod** contains three **30s** spots.”

**Engineering tie-in:** **Podding rules** affect **rotation**, **competitive separation** (two car ads back-to-back?), and **dynamic ad insertion (DAI)** in streaming that mimics linear breaks.

---

## 4. Makegood

**Definition:** A **makegood** is **replacement inventory** (or credit) given when the original **delivery** didn’t meet the **deal**—wrong rating delivery, missed spots, brand-safety failure, technical error.

**Example:** “We promised **10M impressions** in the sports demo; we delivered **8M**—we’ll **make good** with **extra inventory** next month.”

**Staff angle:** **Operational excellence**—**reconcile** early, **automate** eligibility checks, **audit** what counts as delivered.

---

## 5. Traffic instructions (traffic)

**“Traffic” (verb/noun):** The **operational** process of **implementing** the deal in **scheduling systems**: which **spot** airs in which **break**, **version** (15s vs 30s), rotation percentages, **regional** splits.

**Traditionally** human-heavy; **modernization** pushes **APIs**, **rules engines**, and **validation** to reduce errors.

---

## 6. Traffic log (affidavit / as-run log)

**Definition:** After airing, the station/broadcast generates a **log** of **what actually ran**—times, **ISCI/ad-ID**, duration, sometimes **substitutions**.

**Why it exists:** **Proof of performance**; **billing**; **reconciliation** against what was **ordered**.

---

## 7. Traffic log reconciliation

**Definition:** Comparing **three worlds** so finance and ops agree:

1. **Order / deal** (what was sold—**booking system**)  
2. **Schedule / traffic** (what was **planned** to air)  
3. **As-run / traffic log** (what **actually** aired—may differ due to news interruption, breaking coverage, human changes)

**Discrepancy causes:** **Makegoods**, **preemptions**, **late creative swaps**, **technical** playout errors, **human** traffic mistakes.

**Engineering opportunity:** Event-driven **pipelines**, **immutable** logs, **idempotent** adjustments, **dashboards** for ops—this is classic **legacy → cloud** modernization.

---

## 8. Quick glossary

| Term | One line |
|------|----------|
| **Avail** | Sellable block of ad time in linear inventory. |
| **Pod** | Cluster of ads within a break. |
| **Makegood** | Comp inventory/credit for short delivery or errors. |
| **Traffic** | Operational implementation of spots into schedules. |
| **Traffic log / as-run** | Record of what actually aired. |
| **Reconciliation** | Match booked vs planned vs aired (+ digital metrics). |

Next: [Tradeoffs — reach, frequency, privacy, precision](./09-tradeoffs-reach-frequency-privacy-precision.md).
