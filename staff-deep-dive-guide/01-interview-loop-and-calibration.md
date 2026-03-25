# Interview Loop & Calibration — Conceptual Depth

---

## 1. What companies measure at Staff (under the rubric)

Interviews are **sampling** your future behavior. For Staff ICs, the sample is biased toward:

- **Problem decomposition:** Can you **bound** an ambiguous problem into **solvable** pieces?  
- **Explicit tradeoffs:** Do you name **what you sacrifice** for each gain—not hand-wave?  
- **Operational maturity:** Deploy, rollback, SLOs, **cost**, security—**not only** diagrams.  
- **Organizational realism:** Cross-team **friction**, **incentives**, **communication** plans.  
- **Mentorship & standards:** How quality **scales** beyond your keyboard.

**Beginner mistake:** Treating design as **architecture trivia** (“I know Kafka”).  
**Staff pattern:** “Here is **requirements**, **contract**, **failure modes**, **cost**, **rollout**.”

---

## 2. Typical round types (what each optimizes for)

### System design (often 2× at Staff)

**Evaluator mental model:** “Would I trust this person to **own** the **first** **draft** of a critical design **and** revise it when wrong?”

**Depth they probe:**

- Data **consistency** and **idempotency** on **money-adjacent** paths.  
- **Hot paths** vs **cold paths** — don’t **over-build** **async** for **synchronous** SLAs without reason.  
- **Observability:** what **breaks** customer trust vs **internal** inconvenience?

**CoderPad nuance:** Be ready to **type** JSON schema, **pseudo-SQL** for dedupe, or **hash** for experiment bucket—**LLD touch** without full implementation.

### Coding

Tests **precision under pressure** and **communication**. At Staff, **perfection** in **45 min** matters less than **structured** reasoning + **edge cases**.

### Behavioral / leadership / OE

**Principal “Operational Excellence”** threads often look for:

- **Incident** narratives with **metrics** and **systemic** fixes.  
- How you **negotiate** reliability vs **features**.  
- **Postmortem** culture you helped **shape** (templates, **training**).

### Domain (AdTech / media)

You’re not expected to be **20-year** industry veteran— but you must **map** your **distributed systems** experience to **inventory, delivery, measurement** language (use **adtech-beginner-guide**).

---

## 3. Reported WBD-style calibration (anecdotal, not contractual)

Source discussion: [Blind — Staff SDE loop at WBD](https://www.teamblind.com/post/staff-sde-interview-loop-at-warner-bros-discovery-fruanwzs).

Patterns candidates described:

- **Four rounds** total in one story; one round with **Principal** tagged **Operational & Engineering Excellence**.  
- **Two** **system design** slots; both had **CoderPad** links—confusion whether **coding** expected; reply suggested **mostly HLD** + **some** LLD.

**How to use this:** Prepare **both** **whiteboard-style narrative** and **typed** artifacts (API + event payloads). If interviewer says **don’t code**, **stop** typing and **draw**.

**Practice links (crowdsourced):** [Ad serving + A/B (PracHub)](https://prachub.com/interview-questions/design-an-ad-content-delivery-system-with-a-b-testing); [Rotation + CSV](https://prachub.com/interview-questions/implement-array-rotations-and-csv-keyword-search).

---

## 4. Senior vs Staff—interview behavior shift


| Topic        | Senior              | Staff                                                                            |
| ------------ | ------------------- | -------------------------------------------------------------------------------- |
| Design scope | Service + datastore | **Platform** + **multi-team** **contracts** + **migration**                      |
| Failure      | “We’d add retries”  | **Retry budget**, **idempotency**, **compensation**, **who owns** **DLQ** replay |
| Conflict     | Resolved locally    | **RFC**, **SLO**, **exec** **sync** when needed                                  |
| Metrics      | Feature shipped     | **Error budget**, **cost per** **request**, **toil hours**                       |


---

## 5. What to extract from recruiter **calls**

Ask:

- **Round** **titles**, **length**, **interviewer** roles (EM, **Principal**, peer IC).  
- **Tools**: CoderPad **language**, **shared** **whiteboard**.  
- **Focus** areas for **this** org (streaming vs **API** vs **linear**).  
- **Bar** for Staff: **multiplier** examples they loved/hated.

**Why:** Reduces **surprise** panic; lets you **place** **best** stories in **right** round.

---

## 6. In-room strategy (timed)

**First 5 minutes:** **Clarify** scale, **SLAs**, **money** vs **analytics** paths, **multi-region**, **privacy**—see [03](./03-system-design-ad-serving-experiments.md).

**Middle 20:** **Vertical** slice working—request **path** end-to-end **before** **digging** **tangential** **persistence** choices.

**Last 10:** **Failures**, **observability**, **cost**, **rollout**, **open risks**.

**If stuck:** “I’d **timebox** this sub-problem: for v1 we’d **assume** X; **revise** when **assumption** breaks because of Y.”

---

## 7. Interviewer red flags **you** should notice

**Good:** Pushes on **tradeoffs**, **updates** **requirements** mid-design, **collaborates**.

**Concerning (for your job choice):** Dismisses **operational** concerns entirely—may signal **immature** **prod** culture (one data point only).

---

## Related

- [02-two-day-execution-plan.md](./02-two-day-execution-plan.md)  
- [03-system-design-ad-serving-experiments.md](./03-system-design-ad-serving-experiments.md)  
- [08-operational-engineering-excellence.md](./08-operational-engineering-excellence.md)

