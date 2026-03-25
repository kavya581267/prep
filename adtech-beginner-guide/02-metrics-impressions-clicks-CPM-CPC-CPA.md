# Metrics & Money: Impressions, Clicks, CTR, CPM, CPC, CPA

This file explains **what gets counted** in advertising and **how people pay**. These terms appear in dashboards, contracts, and interview questions.

---

## 1. Impression

**Definition:** An **impression** means an ad was **shown** (or “delivered”) to a user in a way the business agrees counts.

**Beginner nuance:** “Shown” is not trivial. Industry and products debate:

- Must **50% of the ad** be visible for **1 second**? (Display often uses **MRC**-style standards conceptually—not a test you memorize.)  
- For **video**, is it “**start**,” “**25% complete**,” “**complete**,” etc.?

**Why it matters technically:** You must **log** impressions for **billing**, **reconciliation**, **fraud detection**, and **frequency capping**.

**Retry pitfall:** If a client **retries** the same request, an naïve system could log **two** impressions for **one** view. Engineers use **request IDs** and **idempotent** writes.

---

## 2. Click

**Definition:** A **click** means the user **activated** the ad—usually tapping a banner or a “learn more” on video—often recorded when they land on a **tracking redirect** before the final URL.

**Nuances:**

- **Invalid clicks** (bots, accidents) get filtered in **ad verification** workflows.  
- Some models care about **view-through conversions** (user saw ad, bought later without clicking)—that’s advanced attribution, not needed for basics.

---

## 3. CTR (Click-Through Rate)

**Formula (conceptually):**  
`CTR = Clicks ÷ Impressions` (often shown as a **percentage**).

**Example:** 5 clicks on 1,000 impressions → CTR = 0.5%.

**How it’s used:** Measures **engagement** quality of a creative or placement. **Low CTR** doesn’t always mean “bad ad”—some campaigns optimize for **awareness**, not clicks.

---

## 4. CPM (Cost Per Mille)

**Definition:** **CPM** is **cost per thousand** impressions (“mille” = thousand).

**Example:** If you pay **$6 CPM**, you pay **$6 for 1,000 impressions** (so **$0.006 per impression**).

**Why thousand?** Historical convention; numbers are easier than tiny per-impression decimals.

**How it’s used:** Common for **brand** campaigns where **exposure** is the product.

---

## 5. CPC (Cost Per Click)

**Definition:** **CPC** is what you pay **per click** (on average or as a bid, depending on auction design).

**Example:** **$1.20 CPC** means each billable click costs about $1.20.

**How it’s used:** Common when the advertiser wants **traffic** or actions tied to clicks.

---

## 6. CPA (Cost Per Acquisition / Action)

**Definition:** **CPA** is **cost per acquisition** (or sometimes “cost per action”)—you pay when a **defined conversion** happens (purchase, signup, app install, subscription).

**Example:** **$40 CPA** might mean “we pay up to $40 per completed purchase attributed under agreed rules.”

**Complexity:** **Attribution** (“which touch gets credit?”) can be **multi-touch**, **time-windowed**, and **disputed**. For interviews, know: **CPA ties money to outcomes**, so **tracking** and **identity** matter.

---

## 7. How these metrics relate (intuition table)

| If the buyer cares most about… | You often see… | Rough optimization direction |
|--------------------------------|----------------|------------------------------|
| Reach / awareness | **CPM** | Deliver lots of **valid** impressions |
| Site traffic | **CPC** | Creative + targeting that gets **clicks** |
| Sales / leads | **CPA** | Precise **conversion** tracking + models |

**Staff-level point:** Different goals change **what you cache**, **what you log first**, and **what “success”** means in A/B tests.

---

## 8. “Billable” vs “measured”

Not every logged event is **billable**. Contracts specify:

- **Geo** eligibility  
- **Viewability** thresholds (for some display/video)  
- **Invalid traffic** removal  

Systems often maintain **raw logs** and **adjusted** reporting for finance.

---

## 9. Interview-style one-liners

- **Impression:** counted delivery of an ad under agreed rules.  
- **CTR:** clicks divided by impressions.  
- **CPM:** price per 1,000 impressions.  
- **CPC:** price per click.  
- **CPA:** price per conversion/action under attribution rules.

Next: [First-party vs third-party data](./03-data-first-party-third-party-household-device.md).
