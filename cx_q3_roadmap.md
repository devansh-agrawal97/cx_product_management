# CX Q3 Roadmap — Spinny
> **Agent aim:** Build the Q3 roadmap for CX at Spinny — grounded in ticket data, FCR gaps, and repeat-contact patterns.

---

## Ticket Analysis — April & May 2026

### Query-Level Breakdown

| Query | Tickets | App% | FCR% | 0 Calls% | 1 Call% | 2 Calls% | 3 Calls% | 3+ Calls% |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Warranty | 15,058 | 64.5 | 18.1 | 7.8 | 20.0 | 14.4 | 10.4 | **47.4** |
| Payments - Seller | 7,310 | 68.6 | 30.9 | 9.6 | 35.3 | 29.8 | 11.1 | 14.0 |
| RC | 5,858 | 72.8 | 29.7 | 14.9 | 31.6 | 15.9 | 9.4 | 28.2 |
| RSA | 3,979 | 0.6 | 39.3 | 9.2 | 44.7 | 19.2 | 7.4 | 19.5 |
| RC - Seller | 3,912 | 67.1 | 35.0 | 11.7 | 38.5 | 17.1 | 9.5 | 23.2 |
| Insurance | 2,641 | 63.8 | 37.9 | 8.8 | 40.4 | 17.4 | 9.1 | 24.2 |
| Spinny Benefits | 2,311 | 78.3 | 8.0 | 3.2 | 9.8 | 7.8 | 8.9 | **70.4** |
| Seller-Miscellaneous | 2,288 | 52.9 | 36.0 | 11.8 | 39.6 | 18.9 | 10.1 | 19.7 |
| Miscellaneous | 1,984 | 68.2 | 33.0 | 9.0 | 35.5 | 18.8 | 11.0 | 25.7 |
| Payments | 1,691 | 64.7 | 46.4 | 18.0 | 49.0 | 12.2 | 7.6 | 13.2 |
| Fastag | 1,381 | 77.0 | 33.4 | 5.8 | 35.7 | 17.2 | 13.3 | 28.0 |
| Other | 1,088 | 0.5 | 28.3 | 20.2 | 32.5 | 21.3 | 11.5 | 14.4 |
| HSRP | 852 | 71.6 | 12.6 | 3.6 | 16.3 | 16.0 | 12.2 | **51.9** |
| Document | 748 | 87.6 | 32.2 | 10.2 | 35.0 | 21.4 | 13.2 | 20.2 |
| Missed Call - Luxury | 145 | 0.0 | 2.1 | 3.4 | 62.8 | 25.5 | 4.1 | 4.1 |
| Welcome call | 74 | 0.0 | 16.2 | 16.2 | 43.2 | 21.6 | 9.5 | 9.5 |
| FASTag Recharge Transaction | 32 | 100.0 | 0.0 | 96.9 | 3.1 | 0.0 | 0.0 | 0.0 |
| FASTag Recharge | 27 | 100.0 | 7.4 | 77.8 | 7.4 | 3.7 | 7.4 | 3.7 |

---

## Key Signals

### 🔴 High-Volume, Low-FCR — Biggest Pain Points

| Category | Tickets | FCR% | 3+ Calls% |
|---|---:|---:|---:|
| **Warranty** | 15,058 | 18.1% | 47.4% |
| **Spinny Benefits** | 2,311 | 8.0% | 70.4% |
| **HSRP** | 852 | 12.6% | 51.9% |

- **Warranty** is ~29% of all tickets. <1 in 5 resolved on first contact, nearly half need 4+ touchpoints — likely a mix of claim complexity + agent knowledge gaps.
- **Spinny Benefits** has the worst FCR in the dataset (8.0%) and worsening — 3+ calls jumped to 70.4%. Customers clearly don't understand what they've bought or can't self-serve. High app usage (78.3%) but still broken.
- **HSRP** — high app %, low FCR. Process-dependent (govt systems) so FCR improvement needs product workarounds (status tracking, proactive nudges).

### 🟡 High-Volume, Moderate FCR — Improvable

| Category | Tickets | FCR% | 3+ Calls% |
|---|---:|---:|---:|
| Payments - Seller | 7,310 | 30.9% | 14.0% |
| RC | 5,858 | 29.7% | 28.2% |
| RC - Seller | 3,912 | 35.0% | 23.2% |

- Decent FCR but room to push — RC categories have a notable 3+ calls tail.

### 🟢 Healthier Categories

- **Payments** (buyer): 46.4% FCR, only 13.2% 3+ calls
- **RSA**: 39.3% FCR despite near-zero app usage (0.6%)
- **Insurance**: 37.9% FCR

### 📌 Cross-Cutting Insight
**High app% ≠ self-serve deflection.** High `app_pct` doesn't correlate with high FCR — app tickets aren't being resolved, just channel-shifted.

---

## Product Goals — Status vs Targets

### 1. Ticket Volume Landing with Agents

| | Aug Target | Dec Target | April Actual | May Actual |
|---|---:|---:|---:|---:|
| Tickets to agents | 18K | 15K | 20K | 21.7K (~500 closed by AI) |

🔴 **Going the wrong direction.** Volume is *rising* (20K → 21.7K) while the target demands it *fall* to 18K by August. Only ~500 tickets closed by AI in May. The chatbot rollout (payments, loans, RC, insurance etc.) needs to deflect significantly more.

### 2. AI Automation on Agent-Handled Tickets

| Metric | Aug Target | Dec Target | April Actual | May Actual | Gap to Aug |
|---|---:|---:|---:|---:|---:|
| % messages to customer by AI | 25% | 50% | 10.3% | 15.6% | **-9.4pp** |
| % tasks created automatically | 20% | 40% | 2.4% | — | **-17.6pp** |
| % calls made by AI | 10% | 20% | — | ~4.5% | **-5.5pp** |
| % messages to internal team by AI | 30% | 60% | 0% | 0% | **-30pp** |

- **AI messages to customer**: Best-performing automation metric — improving (10.3% → 15.6%) but needs to nearly double by August. The **warranty relay experiment** is projected to add ~305 AI messages/day (+9,150/month), lifting this to an estimated **39.3%** — surpassing the Aug target of 25% and approaching the Dec target of 50%. _(Basis: May actuals = 3,657 AI messages on a base of ~23,442 total messages/month; +305/day added to both numerator and denominator, assuming 30-day month.)_
- **Auto task creation**: Near zero. At 2.4% vs 20% target — needs urgent focus.
- **AI calls**: Pre-delivery bot is the only live piece (~4% feedback + 0.5%). Long way from 10%.
- **Internal team AI messages**: Nothing live. 0% → 30% by August is an extremely steep ask.

### 3. Agent Response Times (p80, Office Hours)

| Metric | Aug Target | Dec Target | April Actual | May Actual | Trend |
|---|---:|---:|---:|---:|---:|
| Callback request response | 45 mins | 30 mins | 87 mins | 103 mins | 🔴 Worsening |
| Customer message response | 60 mins | 30 mins | 72 mins | ~111 mins | 🔴 Worsening |

- **Both response times are moving in the wrong direction** — callback went from 87 → 103 mins, customer messages from 72 → 111 mins.
- Callback fulfilment rate also declining: **32% → 27%** (slot-based requests are being missed).

### Summary Assessment

| Goal | Status |
|---|---|
| Ticket deflection | 🔴 Behind — volume rising |
| AI customer messages | 🟡 Improving but slow — warranty relay projected to push to 39.3% (vs 25% Aug target) |
| Auto task creation | 🔴 Critical gap |
| AI calls | 🟡 Early, pre-delivery bot only |
| AI internal messages | 🔴 Nothing live |
| Callback response time | 🔴 Worsening |
| Customer message response time | 🔴 Worsening |

**Bottom line:** 6 out of 7 goals are behind, and 2 are actively regressing. August targets are ~10 weeks away — deflection and response time need the most urgent attention.

---

## Warranty FCR — Human Agent Analysis (May 2026 onwards)

> ⚠️ **Not picked for Q3 roadmap** — impact assessed as low relative to other priorities. Documented here for reference.

**Scope:** Warranty tickets resolved by a human agent (bot IDs excluded, full list of 10 applied) since May 1 2026. Total: 2,085 tickets.

### By Customer Complaint

| Complaint Category | Share |
|---|---:|
| Gear / Clutch | 13.0% |
| Engine / Powertrain | 11.1% |
| Suspension / Noise / Brakes | 10.3% |
| Mileage | 7.4% |

- **Mileage complaints (7.4%)** are a structural mis-expectation problem tied to ARAI vs real-world figures — not a genuine warranty issue.
- **Gear/Clutch tickets** frequently involved customers who had attempted a self-repair before raising the issue, or were paused pending customer action.
- In several cases the customer raised the wrong issue category — the actual fault was not covered under warranty, making the FCR a denial rather than a resolution.

### By Agent Action

| Agent Action | Tickets | Share |
|---|---:|---:|
| Workshop visit scheduled | 595 | 28.5% |
| Warranty denied | 505 | 24.2% |
| On hold — pending customer action | 341 | 16.4% |
| General warranty terms explained | 222 | 10.6% |
| Unreachable customer | 94 | 4.5% |
| Genuinely resolved on call | 52 | 2.5% |

- **Workshop scheduling (28.5%):** Agents shared workshop address, booked drop-off slots, or arranged pickup for non-drivable cars. Recurring friction: Spinny's limited workshop network forces customers in some cities to OEM brand service centres.
- **Denial (24.2%):** Agents cited specific exclusions — clutch disc, battery, tyres, AC compressor, electrical items — or informed customers their warranty had expired. A subset of these were raised within 30 days of delivery (customers hitting an exclusion very early in ownership).
- **On hold (16.4%):** Waiting on repair estimates, invoices, or proof of purchase before the claim could proceed.
- **Terms explained (10.6%):** No active repair rejected — customer had a coverage question only.
- **Unreachable (4.5%):** Closed after 3+ unanswered call attempts.
- **Truly resolved (2.5%):** DIY fix guided by agent, or a misunderstanding cleared up.

### Key Findings on Denials

- **237 tickets** had an explicit agent-stated denial confirmed from call summaries (floor estimate; holistic reading puts the true count at ~505).
- **16 of 237 (6.7%)** were raised within 30 days of car delivery — the highest-risk window for customer dissatisfaction escalation.
- A separate cut looked at tickets where the customer used the "Other concern with…" or "My issue is not listed here" complaint path and still received a denial:

| Month | Tickets |
|---|---:|
| April 2026 | 89 |
| May 2026 | 106 |
| June 2026 (partial) | 56 |

May-over-April increase suggests this pattern is growing. June is on pace to exceed May on a full-month basis.

---

## Smart Message Prioritiser — NDOA Update Analysis (15–21 Jun 2026)

> ⚠️ **Not picked for Q3 roadmap** — only ~192 messages/week (2.2%) would benefit from suppressing the NDOA update. Volume too low to justify the investment.

**Scope:** 8,723 "do not prioritise" messages over 7 days (15–21 Jun 2026).

| Segment | Count | Share |
|---|---:|---:|
| NDOA should be updated | ~8,531 | 97.8% |
| NDOA should NOT be updated (pure acks: "ok", "thanks", "N/A", emojis) | ~192 | 2.2% |

**Current behaviour of updating NDOA for all do-not-prioritise messages is correct in almost every case.**

### Breakdown — Messages Requiring NDOA Update (Pareto)

| Category | Share | Cumulative |
|---|---:|---:|
| Status check / follow-up | 30% | 30% |
| Request / ask | 20% | 50% |
| Document / info sharing | 16% | 66% |
| Issue / complaint report | 10% | 76% |
| Question | 8% | 84% |
| Acknowledgment with embedded ask | 7% | 91% |
| Other | 5% | 96% |
| Escalation / frustration | 3% | 99% |
| Scheduling | 2% | 100% |

The top 4 buckets — status checks, requests, document sharing, and issue reports — cover **~76%** of all messages requiring NDOA updates.

> ⚠️ **Watch list:** The ~3% escalation/frustration messages classified as "do not prioritise" may be misclassifications — worth a closer look.

---

## Q3 Roadmap (TBD)

> _To be built out — priorities, initiatives, owners, and timelines to follow._
