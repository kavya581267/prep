# Brand Safety and Ad Verification (Beginner)

**Brand safety** answers: “Will showing this ad **next to this content** hurt the advertiser’s reputation?”  
**Ad verification** answers: “Did the ad **actually deliver** under the **contract** (viewability, geo, invalid traffic)?”

These topics connect **trust**, **money**, and **law**—high leverage for senior engineering roles.

---

## 1. Brand safety

**Core idea:** Brands don’t want ads appearing alongside **violence**, **hate**, **adult content**, **misinformation**, or sometimes **politics** / **war news**—depends on **advertiser sensitivity** and **blocklists**.

**How it’s implemented (conceptual layers):**

1. **Publisher policy:** categories a site won’t monetize.  
2. **Contextual classification:** ML/metadata labels content (sports vs tragedy).  
3. **Keyword / URL blocklists** (fragile but common).  
4. **Third-party brand safety vendors** providing **risk scores** on bid requests or pages.

**Engineering challenges:**

- **Latency:** safety checks on the **decision path** must be **fast**.  
- **False positives:** blocking too much **loses revenue**.  
- **False negatives:** PR crises and **makegoods** (make-good — see linear TV file).

**Staff angle:** Define **SLOs** for classification delay; **human review** queues for edge cases; **audit trails** when a **bad adjacency** occurs.

---

## 2. Ad verification

**Core idea:** Independent or buyer-side **measurement** that **audits** delivery against standards.

**Common verification themes:**

- **Viewability:** was the ad actually on screen enough?  
- **Geo:** did impressions happen in **target countries**?  
- **Brand safety** incidents on live campaigns  
- **Invalid traffic (IVT):** bots, inflated counts  
- **Frequency** adherence (did we keep promises?)

**How it shows up technically:**

- **Tracking pixels** / server-to-server **beacons**  
- **Log-level data** sharing (privacy sensitive)  
- **Discrepancies** between **ad server counts** and **third-party counts**—finance teams live here

---

## 3. Invalid traffic (IVT) — short primer

**General invalid traffic (GIVT):** known crawlers, data-center traffic—some filter lists.

**Sophisticated invalid traffic (SIVT):** bots mimicking humans; harder.

**Why engineers care:** **Billing disputes**, **refunds**, **blocked accounts**. Systems use **rate limits**, **device signals**, **anomaly detection**, and **post-bid** analysis.

---

## 4. Tradeoffs (interview-ready)

| Stricter blocking | Looser blocking |
|-------------------|-----------------|
| Safer for brands | Higher fill/revenue |
| More false positives (miss legit inventory) | More brand risk |

| Heavy third-party verification | Lighter verification |
|--------------------------------|----------------------|
| Trust with buyers, fewer disputes | Faster/cheaper, more disagreement on numbers |

---

## 5. Mini glossary

| Term | One line |
|------|----------|
| **Brand safety** | Keeping ads away from harmful/undesirable context. |
| **Ad verification** | Auditing delivery vs contract (viewability, geo, IVT). |
| **IVT** | Traffic that shouldn’t count as human engagement. |
| **Makegood** | Compensation inventory when delivery/quality misses goals (see linear doc). |

Next: [Forecasting, pacing, yield](./07-forecasting-pacing-yield.md).
