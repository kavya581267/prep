# Ad Server vs SSP vs DSP — Programmatic (Beginner)

**Programmatic advertising** means **automated buying and selling** of ad impressions, often **per auction** in **milliseconds**. This file explains **who does what** so you don’t confuse the pieces in interviews.

---

## 1. Two worlds (simplified)

**Direct sold (simplified):** A human negotiates a deal: “Run my campaign on your homepage in March.” Traffickers **enter** it; the **ad server** **respects** that booking.

**Programmatic:** Software connects **buyers** and **sellers** in **real time** via protocols like **OpenRTB** (you don’t need to memorize protocol details). A **bid request** goes out; **bidders** respond; a **winner** serves an ad.

Many large publishers use **both**.

---

## 2. Ad server — what it is

An **ad server** is the **system of record** for **what campaigns exist** and for **serving decisions** in many architectures.

**Typical responsibilities:**

- Store **creatives**, **priorities**, **targeting**, **delivery rules**  
- **Decide** which ad wins a placement when multiple line items compete  
- **Track** impressions/clicks for **reporting** and **billing**  
- **Coordinate** with **programmatic** demand (sometimes by calling an SSP or exchange)

**Analogy:** The ad server is like a **traffic controller** at an intersection: it knows the **rules** and **who is allowed** to go next.

**Important:** In real companies, “ad server” might be **in-house**, **vendor** (various commercial products exist), or **split** across microservices. In interviews, describe **functions**, not a brand name.

---

## 3. DSP — Demand-Side Platform

**Who uses it:** **Advertisers** (or agencies) who want to **buy** impressions across many publishers.

**What it does (conceptually):**

- Holds **campaign goals** (budgets, bids, audiences)  
- Receives **bid opportunities** from exchanges  
- **Decides** whether to bid and how much  
- **Tracks** performance and optimization

**Analogy:** A **buying robot** that participates in auctions on behalf of brands.

**Engineering hot topics:** **Bid shading**, **forecasting** spend, **frequency** across exchanges, **identity** for audiences, **brand safety** filters.

---

## 4. SSP — Supply-Side Platform

**Who uses it:** **Publishers** who want to **sell** their impressions **programmatically**.

**What it does (conceptually):**

- Packages **inventory** (placement, size, video context) into **bid requests**  
- Invites **DSPs** / exchanges to bid  
- Applies **publisher controls**: **floor prices**, blocked categories, **deals**  
- Helps **maximize yield** for the publisher (often with **header bidding** in web contexts)

**Analogy:** An **auctioneer** representing the **seller**.

---

## 5. Ad exchange

An **exchange** is a **marketplace** connecting many DSPs and SSPs. Sometimes people say **SSP** when they imply **exchange connectivity**—the lines blur commercially.

**Beginner takeaway:** **DSP = buy side**, **SSP = sell side**, **Exchange = venue**, **Ad server = fulfillment + rules** (oversimplified but interview-useful).

---

## 6. How a single ad moment can involve multiple systems

On a news site with programmatic video:

1. **Ad server** checks **direct-sold** obligations first (**priority**).  
2. If **remnant** is open to programmatic, the **SSP** builds a bid request.  
3. **DSPs** bid; **winner** returns creative / tracking.  
4. **Events** flow to **reporting** and **billing** pipelines.

For **Staff** interviews you might be asked to **separate concerns**: **latency-critical path**, **async logging**, **fraud**, **reconciliation**.

---

## 7. Common interview mistakes (avoid)

- Saying “**DSP serves the creative**” without clarifying **who hosts assets** and **who enforces policies** (often **publisher-side** checks still apply).  
- Ignoring **direct** line items that **must** preempt programmatic.  
- Forgetting **timeouts**: if bids don’t arrive, show a **house ad** or **default**.

---

## 8. Mini glossary

| Term | One line |
|------|----------|
| **Programmatic** | Automated auction/ buying of ad impressions. |
| **Ad server** | Campaign truth + decisioning/fulfillment hub (conceptually). |
| **DSP** | Buyer-side bidding stack. |
| **SSP** | Seller-side auction/optimization to demand. |
| **Exchange** | Liquidity layer connecting buyers/sellers. |

Next: [Brand safety & verification](./06-brand-safety-ad-verification.md).
