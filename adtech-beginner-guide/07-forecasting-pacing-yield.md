# Forecasting, Pacing, and Yield (Beginner)

These three words sound similar but mean **different jobs** in advertising operations. Interviews (especially **ad platforms + linear**) love this distinction.

---

## 1. Forecasting

**Question it answers:** “**How much inventory** do we expect to have—**and sell**—in the future?”

**Examples:**

- “How many **18–49** impressions will we likely have next quarter on **CTV app X**?”  
- “If we add **frequency caps**, how does **available supply** shrink?”

**Inputs (conceptual):** historical traffic, seasonality (sports, holidays), audience growth, **technical** changes (new app version), **sold** commitments already booked.

**Outputs:** **Avail** projections (see linear doc), **capacity** for sales, **pricing** hints.

**Engineering:** Data pipelines, **ML** or statistical models, **feature stores**, **backtesting**. Accuracy vs explainability tradeoffs for sales trust.

**vs. pacing/yield:** Forecasting is **mostly about the future supply picture**, not minute-by-minute spend control.

---

## 2. Pacing

**Question it answers:** “Given a **budget** and **time window**, **how fast** should we spend so we **neither burn out early** nor **finish the flight under-delivered**?”

**Simple story:** $10,000 for 10 days ≈ **$1,000/day** in an even (**even**) pacing model—reality has **intra-day curves** (primetime spikes).

**Why it’s hard:**

- Traffic is **uneven** (weekends vs weekdays).  
- **Auction dynamics**: you might win more bids at certain times.  
- **Latency** of spend feedback—**you’re steering with a slightly old speedometer**.

**Failure modes:**

- **Overspend** early → budget gone, campaign stops when you still had valuable days ahead.  
- **Underspend** → unhappy advertiser; **makegoods** or lost revenue.

**Engineering:** High-speed **counters**, **control loops**, sometimes **probabilistic** throttles; must handle **distributed consistency** carefully.

---

## 3. Yield (yield management / yield optimization)

**Question it answers:** “**Which demand** should we accept to **maximize value** for the publisher’s inventory—often **revenue** subject to **constraints**?”

**Intuition:** Not every dollar is equal:

- A **direct** deal might have **priority** or **guaranteed** components.  
- A **programmatic** impression might bring **variable** CPM.  
- **Accepting a low bid** might **block** a better opportunity **milliseconds** later (depending on auction design—don’t overclaim without clarifying setup).

**Yield** is the **portfolio** decision: **who wins the slot** when multiple valid options exist and rules allow choice.

**Engineering:** Optimization under **latency** caps; **exploration vs exploitation**; **A/B tests** on ranking functions—your experimentation doc ties here.

---

## 4. Three-way comparison (memorize this table)

| | **Forecasting** | **Pacing** | **Yield** |
|---|-----------------|------------|-----------|
| **Time horizon** | Future days/weeks/quarters | Hours/days within a flight | Each decision moment |
| **Main output** | Expected supply/capacity | Spend rate vs budget curve | Which demand wins / price accepted |
| **Analogy** | Weather prediction | Cruise control on a hilly road | Picking the best fare for one taxi ride |

---

## 5. How they connect

- **Forecasting** informs **pricing** and **sales** (“we can promise X”).  
- **Pacing** ensures the **campaign actually spends** as sold.  
- **Yield** decides **who** gets the impression when **multiple** suitors qualify.

---

## 6. Mini glossary

| Term | One line |
|------|----------|
| **Forecasting** | Predict future inventory/supply and sellable capacity. |
| **Pacing** | Control spend speed against budget and time. |
| **Yield** | Maximize inventory value under rules (revenue vs constraints). |

Next: [Linear TV terms](./08-linear-tv-terms-avail-pod-makegood.md).
