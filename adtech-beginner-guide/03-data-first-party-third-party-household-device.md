# Data: First-Party vs Third-Party, Household vs Device

Advertising “targeting” needs **signals** about who might see an ad. Those signals come from different **sources**, with different **privacy** and **accuracy** properties.

---

## 1. First-party data

**Definition:** Data collected **directly** by the **publisher or advertiser** through **their own** relationship with the user.

**Examples:**

- **Streaming service:** watch history, subscription tier, login account.  
- **Retailer app:** past purchases, loyalty ID, app sessions.  
- **News site:** reading topics when logged in.

**Why it’s valuable:** Often **high quality** and **permissioned** under privacy policies and regulations.

**Engineering note:** First-party IDs (e.g., **logged-in user ID**) are stable for **frequency capping**, **experiment bucketing**, and **personalization**—when users are logged in.

---

## 2. Third-party data

**Definition:** Data brought in from **outside** the direct relationship—historically often via **third-party cookies** on the web or **data brokers’** audience segments.

**Examples (conceptual):**

- “Likely in market for a car” segment bought from a vendor.  
- Cross-site browsing profiles (being **deprecated / restricted** on modern browsers).

**Why it’s changing:** Privacy regulations (**GDPR**, **CCPA**), browser **cookie** limits, and platform **ATT** on mobile pushed the industry toward **first-party** and **contextual** targeting.

**Beginner takeaway:** In modern interviews, saying “we need a **first-party strategy**” is more current than assuming infinite third-party cross-site tracking.

---

## 3. Second-party data (bonus)

**Definition:** **Another company’s first-party data** shared through a **direct partnership** (e.g., a retailer + a CPG brand in a **clean room**).

You don’t need depth unless the role emphasizes **data partnerships**.

---

## 4. Device-level identity

**What it is:** An identifier tied to a **single device** or browser instance.

**Examples (conceptual, vary by platform):**

- Mobile **device advertising IDs** (users can reset/limit).  
- **Cookies** on web (first-party vs third-party).  
- **Device fingerprinting** is discouraged/unreliable ethically and technically—don’t propose it as a solution in interviews.

**Use cases:** **Frequency caps** per phone, **geo** at device level, **CTV device** graphs.

---

## 5. Household-level identity

**What it is:** A **group** of people/devices in one home (roughly), used so you don’t show the same brand’s ad **too many times** across **mom’s phone**, **dad’s tablet**, and **streaming box**.

**How it’s approximated:** **Probabilistic** or **deterministic** **graphs** matching devices to a **household cluster** using signals (IP stability—imperfect—, logged-in accounts across devices, partner data).

**Tradeoff:** Better **frequency control** vs **privacy** sensitivity and **accuracy** limits.

---

## 6. Why interviewers care

- **Experiment assignment:** Stable **user** vs **device** unit changes metrics.  
- **Deduping impressions:** Same ad shown on two devices—**one or two** impressions for caps?  
- **Privacy:** Explain **data minimization**, retention limits, consent—at least at principle level.

---

## 7. Quick comparison

| | First-party | Third-party (classic) |
|---|-------------|------------------------|
| Source | Your product relationship | External broker / cross-site |
| Trend | **Increasing** reliance | **Restricted** in many environments |
| Interview stance | Emphasize consent + value | Acknowledge limits |

| | Device | Household |
|---|--------|-----------|
| Granularity | One phone/browser | Group of devices |
| Frequency capping | Simpler | Harder but can reduce **annoyance** |
| Privacy | Still sensitive | More sensitive |

Next: [Frequency cap, daypart, geofence](./04-targeting-frequency-daypart-geofence.md).
