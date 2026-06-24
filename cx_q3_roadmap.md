# CX Q3 Roadmap — Spinny
> **Agent aim:** Build the Q3 roadmap for CX at Spinny — grounded in ticket data, FCR gaps, and repeat-contact patterns.

---

## Ticket Analysis — April 2026

> **Call data source**: `cx_communication_messages` (message_type = CALL_DETAILS, meta_data JSON, dual-schema).
> **OB only** — all call metrics below count outbound calls only. IB calls excluded from attempts, connected, and distribution.
> **Closed** = `is_closed = 1 OR ever_merged_flag = 1`. Merged tickets are treated as closed.
> **FCR** = ticket has exactly 1 OB connected call AND is closed (or merged).
> **0/1/2/3/3+ Conn%** = % of tickets by OB connected call count. Closure% in brackets = % of that bucket that is closed.
> See `CX_post_sales_AIR.md` for full methodology including dual-schema JSON handling.

### SQL Query

```sql
-- Step 1: Build per-ticket OB call metrics from cx_communication_messages (dual-schema)
WITH comm_calls AS (
  SELECT
    cm.ticket_id,
    SUM(CASE
      WHEN get_json_string(cm.meta_data, '$.type') = 'outbound.manual.dial' THEN 1
      WHEN get_json_string(cm.meta_data, '$.callback_data.callType') = 'outbound.manual.dial' THEN 1
      ELSE 0
    END) AS ob_attempts,
    SUM(CASE
      WHEN get_json_string(cm.meta_data, '$.type') = 'outbound.manual.dial'
           AND get_json_string(cm.meta_data, '$.status') = 'CONNECTED' THEN 1
      WHEN get_json_string(cm.meta_data, '$.callback_data.callType') = 'outbound.manual.dial'
           AND get_json_string(cm.meta_data, '$.callback_data.callResult') = 'SUCCESS' THEN 1
      ELSE 0
    END) AS ob_connected
  FROM sp_entity_store_intermediate.cx_communication_messages cm
  WHERE cm.message_type = 'CALL_DETAILS'
    AND cm.meta_data IS NOT NULL AND cm.meta_data != ''
  GROUP BY cm.ticket_id
)

-- Step 2: Join with cx_tickets and compute all metrics per query
SELECT
  t.query,
  COUNT(*) AS tickets,

  -- Closed% (is_closed OR ever_merged)
  ROUND(SUM(CASE WHEN t.is_closed = 1 OR t.ever_merged_flag = 1 THEN 1 ELSE 0 END)
    * 100.0 / COUNT(*), 1) AS closed_pct,

  -- App% (source = SPINNY_HELP_AND_SUPPORT)
  ROUND(SUM(CASE WHEN t.source = 'SPINNY_HELP_AND_SUPPORT' THEN 1 ELSE 0 END)
    * 100.0 / COUNT(*), 1) AS app_pct,

  -- FCR% (exactly 1 OB connected call AND closed)
  ROUND(SUM(CASE WHEN COALESCE(c.ob_connected, 0) = 1
                  AND (t.is_closed = 1 OR t.ever_merged_flag = 1) THEN 1 ELSE 0 END)
    * 100.0 / COUNT(*), 1) AS fcr_pct,

  -- OB Attempts (total and per ticket)
  SUM(COALESCE(c.ob_attempts, 0)) AS ob_attempts,
  ROUND(SUM(COALESCE(c.ob_attempts, 0)) * 1.0 / COUNT(*), 1) AS ob_attempts_per_tkt,

  -- Avg OB Attempts to Close (only closed tickets in numerator and denominator)
  ROUND(SUM(CASE WHEN t.is_closed = 1 OR t.ever_merged_flag = 1
                 THEN COALESCE(c.ob_attempts, 0) ELSE 0 END) * 1.0
    / NULLIF(SUM(CASE WHEN t.is_closed = 1 OR t.ever_merged_flag = 1 THEN 1 ELSE 0 END), 0),
    1) AS avg_ob_to_close,

  -- OB Connected (total and per ticket)
  SUM(COALESCE(c.ob_connected, 0)) AS ob_connected,
  ROUND(SUM(COALESCE(c.ob_connected, 0)) * 1.0 / COUNT(*), 1) AS ob_connected_per_tkt,

  -- OB Connect% (ob_connected / ob_attempts)
  ROUND(SUM(COALESCE(c.ob_connected, 0)) * 100.0
    / NULLIF(SUM(COALESCE(c.ob_attempts, 0)), 0), 1) AS connect_pct,

  -- 0 Conn% and its closure%
  ROUND(SUM(CASE WHEN COALESCE(c.ob_connected, 0) = 0 THEN 1 ELSE 0 END)
    * 100.0 / COUNT(*), 1) AS conn_0_pct,
  ROUND(SUM(CASE WHEN COALESCE(c.ob_connected, 0) = 0
                  AND (t.is_closed = 1 OR t.ever_merged_flag = 1) THEN 1 ELSE 0 END) * 100.0
    / NULLIF(SUM(CASE WHEN COALESCE(c.ob_connected, 0) = 0 THEN 1 ELSE 0 END), 0),
    1) AS conn_0_closed_pct,

  -- 1 Conn% and its closure%
  ROUND(SUM(CASE WHEN COALESCE(c.ob_connected, 0) = 1 THEN 1 ELSE 0 END)
    * 100.0 / COUNT(*), 1) AS conn_1_pct,
  ROUND(SUM(CASE WHEN COALESCE(c.ob_connected, 0) = 1
                  AND (t.is_closed = 1 OR t.ever_merged_flag = 1) THEN 1 ELSE 0 END) * 100.0
    / NULLIF(SUM(CASE WHEN COALESCE(c.ob_connected, 0) = 1 THEN 1 ELSE 0 END), 0),
    1) AS conn_1_closed_pct,

  -- 2 Conn% and its closure%
  ROUND(SUM(CASE WHEN COALESCE(c.ob_connected, 0) = 2 THEN 1 ELSE 0 END)
    * 100.0 / COUNT(*), 1) AS conn_2_pct,
  ROUND(SUM(CASE WHEN COALESCE(c.ob_connected, 0) = 2
                  AND (t.is_closed = 1 OR t.ever_merged_flag = 1) THEN 1 ELSE 0 END) * 100.0
    / NULLIF(SUM(CASE WHEN COALESCE(c.ob_connected, 0) = 2 THEN 1 ELSE 0 END), 0),
    1) AS conn_2_closed_pct,

  -- 3 Conn% and its closure%
  ROUND(SUM(CASE WHEN COALESCE(c.ob_connected, 0) = 3 THEN 1 ELSE 0 END)
    * 100.0 / COUNT(*), 1) AS conn_3_pct,
  ROUND(SUM(CASE WHEN COALESCE(c.ob_connected, 0) = 3
                  AND (t.is_closed = 1 OR t.ever_merged_flag = 1) THEN 1 ELSE 0 END) * 100.0
    / NULLIF(SUM(CASE WHEN COALESCE(c.ob_connected, 0) = 3 THEN 1 ELSE 0 END), 0),
    1) AS conn_3_closed_pct,

  -- 3+ Conn% and its closure%
  ROUND(SUM(CASE WHEN COALESCE(c.ob_connected, 0) > 3 THEN 1 ELSE 0 END)
    * 100.0 / COUNT(*), 1) AS conn_3plus_pct,
  ROUND(SUM(CASE WHEN COALESCE(c.ob_connected, 0) > 3
                  AND (t.is_closed = 1 OR t.ever_merged_flag = 1) THEN 1 ELSE 0 END) * 100.0
    / NULLIF(SUM(CASE WHEN COALESCE(c.ob_connected, 0) > 3 THEN 1 ELSE 0 END), 0),
    1) AS conn_3plus_closed_pct

FROM internal.sp_product.cx_tickets t
LEFT JOIN comm_calls c ON c.ticket_id = t.ticket_id
WHERE t.created_at >= '2026-04-01' AND t.created_at < '2026-05-01'
GROUP BY t.query
ORDER BY 2 DESC
```

> **Notes:**
> - To get the OVERALL row, run the same query without `GROUP BY t.query` and hardcode `'OVERALL'` as the query column.
> - Change the date range in the `WHERE` clause to analyse different months.
> - The CTE handles both JSON schemas (Schema A: `$.type`/`$.status`, Schema B: `$.callback_data.callType`/`$.callback_data.callResult`).

### Query-Level Breakdown

| Query | Tickets | Closed% | App% | FCR% | OB Attempts (per tkt) | Avg OB to Close | OB Connected (per tkt) | Connect% | 0 Conn% (Closed%) | 1 Conn% (Closed%) | 2 Conn% (Closed%) | 3 Conn% (Closed%) | 3+ Conn% (Closed%) |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---|---|---|---|---|
| **OVERALL** | **24,859** | **98.6** | **59.9** | **33.1** | **88,710 (3.6)** | **3.4** | **62,041 (2.5)** | **69.9** | **22.5% (99.9%)** | **33.2% (99.7%)** | **16.7% (99.6%)** | **8.4% (99.2%)** | **19.2% (94.0%)** |
| | | | | | | | | | | | | | |
| Warranty | 7,531 | 99.7 | 66.8 | 21.2 | 36,815 (4.9) | 4.8 | 27,078 (3.6) | 73.6 | 26.0% (100%) | 21.2% (100%) | 14.0% (100%) | 8.9% (100%) | 29.9% (99.1%) |
| Payments - Seller | 3,502 | 99.8 | 64.6 | 35.2 | 8,680 (2.5) | 2.4 | 5,770 (1.6) | 66.5 | 20.0% (100%) | 35.2% (99.9%) | 28.6% (100%) | 8.7% (100%) | 7.5% (98.1%) |
| RC | 2,818 | 95.6 | 73.5 | 36.7 | 10,108 (3.6) | 3.1 | 6,652 (2.4) | 65.8 | 20.0% (100%) | 36.7% (99.9%) | 14.3% (99.8%) | 8.1% (98.2%) | 20.9% (80.0%) |
| RSA | 1,938 | 100.0 | 0.4 | 47.4 | 2,778 (1.4) | 1.4 | 1,727 (0.9) | 62.2 | 34.9% (100%) | 47.4% (100%) | 13.3% (100%) | 3.0% (100%) | 1.5% (100%) |
| RC - Seller | 1,785 | 90.0 | 61.6 | 39.8 | 5,827 (3.3) | 2.6 | 3,701 (2.1) | 63.5 | 19.0% (97.6%) | 41.2% (96.5%) | 15.6% (94.3%) | 7.5% (91.0%) | 16.7% (61.1%) |
| Insurance | 1,293 | 99.6 | 61.9 | 45.1 | 4,167 (3.2) | 3.1 | 2,979 (2.3) | 71.5 | 17.6% (100%) | 45.1% (100%) | 14.6% (100%) | 8.4% (100%) | 14.4% (97.3%) |
| Spinny Benefits | 1,015 | 99.5 | 76.8 | 18.9 | 6,336 (6.2) | 6.1 | 4,655 (4.6) | 73.5 | 21.1% (100%) | 18.9% (100%) | 9.3% (100%) | 8.0% (100%) | 42.8% (98.8%) |
| Seller-Misc | 1,040 | 99.5 | 52.4 | 42.0 | 3,014 (2.9) | 2.8 | 1,977 (1.9) | 65.6 | 15.5% (100%) | 42.0% (100%) | 18.2% (100%) | 10.1% (100%) | 14.2% (96.6%) |
| Payments | 909 | 99.9 | 62.2 | 49.9 | 1,877 (2.1) | 2.0 | 1,276 (1.4) | 68.0 | 26.1% (100%) | 49.9% (100%) | 10.5% (100%) | 5.6% (100%) | 7.9% (98.6%) |
| Miscellaneous | 865 | 99.4 | 66.5 | 39.5 | 2,425 (2.8) | 2.7 | 1,719 (2.0) | 70.9 | 16.9% (100%) | 39.5% (100%) | 19.2% (100%) | 10.5% (100%) | 13.9% (95.8%) |
| Fastag | 695 | 99.7 | 75.0 | 41.6 | 2,123 (3.1) | 3.0 | 1,456 (2.1) | 68.6 | 14.0% (100%) | 41.6% (100%) | 15.3% (100%) | 12.5% (100%) | 16.7% (98.3%) |
| Other | 592 | 100.0 | 0.5 | 26.2 | 1,726 (2.9) | 2.9 | 986 (1.7) | 57.1 | 29.2% (100%) | 26.2% (100%) | 21.6% (100%) | 11.8% (100%) | 11.1% (100%) |
| HSRP | 400 | 99.5 | 71.5 | 16.8 | 1,757 (4.4) | 4.3 | 1,306 (3.3) | 74.3 | 8.5% (100%) | 16.8% (100%) | 21.3% (100%) | 15.3% (100%) | 38.3% (98.7%) |
| Document | 360 | 100.0 | 84.7 | 37.5 | 898 (2.5) | 2.5 | 628 (1.7) | 69.9 | 16.9% (100%) | 37.5% (100%) | 24.4% (100%) | 10.8% (100%) | 10.3% (100%) |

---

## Key Signals

### 🔴 High-Volume, Low-FCR — Biggest Pain Points

| Category | Tickets | FCR% | 3+ Conn% | OB Attempts/tkt | Avg OB to Close | OB Connect% |
|---|---:|---:|---:|---:|---:|---:|
| **Warranty** | 7,531 | 21.2% | 29.9% | 4.9 | 4.8 | 73.6% |
| **Spinny Benefits** | 1,015 | 18.9% | 42.8% | 6.2 | 6.1 | 73.5% |
| **HSRP** | 400 | 16.8% | 38.3% | 4.4 | 4.3 | 74.3% |

- **Warranty** is ~30% of all tickets, consumes **37K OB attempts** in April alone (41% of all OB calls). Only 1 in 5 resolved on first connected call. 29.9% need 3+ OB connected calls. Takes 4.8 OB attempts on average to close a ticket.
- **Spinny Benefits** has the worst FCR after HSRP (18.9%) and highest call intensity — 6.2 OB attempts, 6.1 needed to close. 42.8% need 3+ OB connected calls — nearly half the volume is chronic repeat-contact.
- **HSRP** — high app%, lowest FCR (16.8%). 4.4 OB attempts per ticket, 38.3% need 3+ connected. Process-dependent (govt/RTO systems) so FCR improvement needs product workarounds (status tracking, proactive nudges).

### 🟡 High-Volume, Moderate FCR — Improvable

| Category | Tickets | FCR% | 3+ Conn% | OB Attempts/tkt | Avg OB to Close | OB Connect% |
|---|---:|---:|---:|---:|---:|---:|
| Payments - Seller | 3,502 | 35.2% | 7.5% | 2.5 | 2.4 | 66.5% |
| RC | 2,818 | 36.7% | 20.9% | 3.6 | 3.1 | 65.8% |
| RC - Seller | 1,785 | 39.8% | 16.7% | 3.3 | 2.6 | 63.5% |

- RC has a long 3+ calls tail (20.9%) and only 80% of 3+ conn tickets are closed — 1 in 5 heavy-touch RC tickets still unresolved after 2+ months.
- **RC - Seller** has the lowest overall closure at **90.0%** — 3+ conn bucket is only 61.1% closed. These are RTO/govt-dependent RC transfer tickets that stay open waiting for process completion.
- **RSA** is an outlier: 100% closed, 47.4% FCR, but driven by inbound calls (1.4 OB/tkt vs 1.9 IB/tkt) — customers call in for roadside assistance, not the other way.

### 🟢 Healthier Categories

- **Payments** (buyer): 49.9% FCR, only 7.9% 3+ connected, 2.1 OB attempts/tkt, 2.0 to close — cleanest category
- **Insurance**: 45.1% FCR, 14.4% 3+ connected
- **Seller-Misc**: 42.0% FCR, 99.5% closed

### 📌 Cross-Cutting Insights

1. **98.6% of April tickets are closed** (including merged). Only 354 genuinely pending — mostly RC-Seller tickets stuck at RTO.
2. **High app% ≠ self-serve deflection.** High `app_pct` doesn't correlate with high FCR — app tickets aren't being resolved, just channel-shifted.
3. **Overall OB connect rate is 69.9%** — ~30% of all outbound call attempts are wasted. At 89K OB attempts in April, that's **~27K failed calls** burning agent time.
4. **22.5% of tickets have zero OB connected calls** — mostly merged/chat-resolved tickets (99.9% closed). Of these, 22.7% never had an OB attempt at all (chat-only), and 3.3% had attempts but never connected.
5. **RSA is inbound-heavy** — 34.9% have 0 OB connected calls, 47.4% have exactly 1. Customers proactively call in for roadside assistance — different support model needed.
6. **3+ conn bucket closure drops sharply for RC (80%) and RC-Seller (61%)** — these are the genuinely stuck tickets, blocked on external processes (RTO, govt).

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

## 0 OB Connected Deep Dive — Why Tickets Close Without a Connected Call (April 2026)

**Total 0-OB-connected tickets: 5,684** (22.5% of all tickets) — 2,250 non-merged + 3,434 merged.

> Merged tickets concentrate overwhelmingly in this bucket (e.g. Warranty 83%, Spinny Benefits 92%, RSA 67%). Most "0 OB connected" is not a service gap — it's ticket merges. The analysis below separates non-merged (real closure patterns) from merged (duplicate-creation patterns).

### All Queries — Non-Merged: 2,250 tickets across 11 patterns

| # | Bucket | Tickets | % | Avg CX Msgs | Avg Agent Msgs | Avg AI Msgs | Avg Sys Msgs | Top Queries |
|---|---|---:|---:|---:|---:|---:|---:|---|
| 1 | **Customer Unreachable** | 348 | 15.5 | 1.7 | 0.6 | 0.1 | 2.9 | RSA (120), Payments-Seller (41), Warranty (33), RC (32) |
| 2 | **Paused → Auto-Close** | 302 | 13.4 | 2.1 | 0.8 | 0.2 | 7.2 | Warranty (65), RC (51), Other (47), Payments-Seller (42) |
| 3 | **Payment Released** | 290 | 12.9 | 2.5 | 0.9 | 0.1 | 2.5 | Payments-Seller (201), Other (45), Payments (19) |
| 4 | **RC Process** | 282 | 12.5 | 2.1 | 1.2 | 0.2 | 2.6 | RC (142), RC-Seller (140) |
| 5 | **Others** (uncategorized) | 249 | 11.1 | 2.1 | 1.1 | 0.1 | 2.0 | Payments (81), Misc (38), RC-Seller (25) |
| 6 | **Bot Auto-Denial** | 187 | 8.3 | 2.5 | 0.0 | 3.6 | 0.0 | RC (119), Warranty (68) |
| 7 | **Info Shared** | 172 | 7.6 | 1.9 | 0.8 | 0.1 | 1.9 | Payments (48), Insurance (48), Payments-Seller (39) |
| 8 | **Resolved** | 139 | 6.2 | 2.5 | 1.3 | 0.3 | 2.4 | Warranty (58), Seller-Misc (46), RC (11) |
| 9 | **Denial / Not Needed** | 102 | 4.5 | 1.7 | 0.5 | 0.1 | 1.7 | RSA (70), Warranty (24) |
| 10 | **Scheduling Exhausted** | 62 | 2.8 | 2.7 | 0.7 | 0.4 | 3.2 | Warranty (62 — 100%) |
| 11 | **Repair Completed** | 15 | 0.7 | 3.3 | 1.0 | 0.0 | 8.1 | Warranty (15 — 100%) |

**Sample ticket IDs per bucket:**

| Bucket | Sample Tickets |
|---|---|
| Customer Unreachable | `2057912`, `2057936`, `2058007`, `2058436`, `2058668` |
| Paused → Auto-Close | `2057669`, `2058050`, `2058368`, `2058413`, `2058557` |
| Payment Released | `2057630`, `2057903`, `2057949`, `2058517`, `2058575` |
| RC Process | `2057928`, `2057962`, `2057972`, `2058456`, `2058777` |
| Others | `2058610`, `2058717`, `2058845`, `2059216`, `2059229` |
| Bot Auto-Denial | `2058699`, `2058850`, `2060537`, `2061444`, `2069872` |
| Info Shared | `2057675`, `2057782`, `2057840`, `2057994`, `2058049` |
| Resolved | `2057893`, `2058507`, `2058841`, `2059221`, `2059397` |
| Denial / Not Needed | `2058341`, `2059204`, `2059601`, `2059963`, `2061500` |
| Scheduling Exhausted | `2057837`, `2061196`, `2065043`, `2065102`, `2065935` |
| Repair Completed | `2059578`, `2066916`, `2067451`, `2069013`, `2069438` |

**What's happening in each bucket (confirmed from message reads):**

1. **Customer Unreachable:** 2-3 OB attempts, none connected. System sends "We attempted to contact you" after each fail. Closed without resolution.
2. **Paused → Auto-Close:** Same unreachability, formal pause flow: ticket held → reminder sent → no resume → auto-closed. Avg 7.2 system messages.
3. **Payment Released:** Back-office action — agent releases hold amount, notifies via chat with UTR. No call needed.
4. **RC Process:** Agent shares RC transfer status / courier tracking / TAT via chat. Status updates only.
5. **Bot Auto-Denial:** AI bot (981/998) handles E2E — reads concern, issues verdict, customer doesn't reply → auto-closes. **0 agent msgs, 3.6 AI msgs.**
6. **Info Shared:** Agent shares explanation/document/TAT via chat — informational resolution, templated.
7. **Resolved:** "Resolved on call" via personal/office phones not tracked in CRM. "Resolved on chat" = messaging-only.
8. **Denial / Not Needed:** Warranty expired, out of coverage, or service not actually needed (RSA: "Cx did not require RSA").
9. **Scheduling Exhausted:** Warranty-only. 3 failed scheduling attempts for inspection/workshop → auto-closed.
10. **Repair Completed:** Workshop flow completed via system messages — process-driven, no OB needed.

### All-Query Automation Map

| Bucket | Tickets/mo | Already Automated? | Automation Potential | Key Lever |
|---|---:|---|---|---|
| Customer Unreachable | 348 | ❌ | 🟡 Medium | Deflect to WhatsApp/chat after 1st failed OB |
| Paused → Auto-Close | 302 | ⚠️ Partial | 🟢 Low | Closure is auto; reduce cycle time |
| Payment Released | 290 | ❌ | 🔴 **Very High** | Auto-verify RC → auto-release → auto-notify with UTR |
| RC Process | 282 | ❌ | 🔴 **Very High** | Auto-share RC status / courier tracking via bot |
| Others | 249 | ❌ | ⚪ Unknown | Need better reason tagging |
| Bot Auto-Denial | 187 | ✅ Yes | 🟢 Done | Already E2E bot-handled; expand to more queries |
| Info Shared | 172 | ❌ | 🔴 **High** | Bot auto-reply with templated info |
| Resolved | 139 | ❌ | 🟡 Low | CRM call tracking fix for off-system calls |
| Denial / Not Needed | 102 | ❌ | 🔴 **High** | Eligibility check at ticket creation |
| Scheduling Exhausted | 62 | ❌ | 🔴 **High** | Self-serve scheduling via app/WhatsApp |
| Repair Completed | 15 | ✅ Yes | 🟢 Done | System messages handle this |

**~1,200 tickets/month (53%) across Payment Released, RC Process, Bot Auto-Denial, Info Shared, Denial, and Scheduling Exhausted are automatable.**

---

### Payments-Seller Deep Dive — 0 OB Connected (735 tickets)

**353 non-merged + 382 merged**

#### Non-Merged: 353 tickets — 6 closure patterns

| # | Bucket | Tickets | % of 353 | Closing Reasons | Avg CX Msgs | Avg Agent | Avg Sys |
|---|---|---:|---:|---|---:|---:|---:|
| 1 | **Payment Released** | 201 | 56.9 | OTP Amount Released (118), PP Amount Released (60), Bank NOC Hold (15), Fastag Hold (4), RC Card (3), Other Hold (1) | 2.9 | 1.0 | 2.7 |
| 2 | **Paused → Auto-Close** | 42 | 11.9 | Paused Ticket Automatic Closure | 2.9 | 1.1 | 7.3 |
| 3 | **Customer Unreachable** | 41 | 11.6 | Unresponsive (40), Cx not connected (1) | 2.1 | 0.7 | 3.7 |
| 4 | **Info/TAT Shared** | 40 | 11.3 | TAT Shared (39), Deduction Informed (1) | 2.4 | 1.1 | 2.3 |
| 5 | **Others** | 21 | 5.9 | Others | 3.0 | 1.2 | 2.0 |
| 6 | **Denial** | 8 | 2.3 | Not eligible | 2.1 | 0.8 | 1.5 |

**Sample tickets:**

| Bucket | Sample Tickets |
|---|---|
| Payment Released | `2057630`, `2057949`, `2058575`, `2058655`, `2058683` |
| Paused → Auto-Close | `2058557`, `2060871`, `2061804`, `2082176`, `2087481` |
| Customer Unreachable | `2058886`, `2059508`, `2061811`, `2063313`, `2064691` |
| Info/TAT Shared | `2059543`, `2059577`, `2060464`, `2061596`, `2062392` |
| Others | `2067025`, `2067548`, `2079143`, `2087851`, `2088985` |
| Denial | `2059963`, `2067529`, `2079583`, `2099232`, `2108573` |

#### Merged: 382 tickets — Duplicate Creation Patterns

382 merged into 306 unique parents (avg 1.2 children/parent). **94% same customer** — genuine duplicates. Root cause: **seller re-raises because they can't see progress on their existing ticket.**

**Time gap from parent creation to merged child:**

| Gap | Tickets | % | App | Web | VOC | Manual |
|---|---:|---:|---:|---:|---:|---:|
| < 5 min | 137 | 35.9 | 85 | 13 | 34 | 5 |
| 5–60 min | 74 | 19.4 | 24 | 40 | 9 | 1 |
| 1–24 hrs | 81 | 21.2 | 39 | 24 | 12 | 6 |
| > 24 hrs | 90 | 23.6 | 46 | 15 | 25 | 4 |

**Key insight:** 55.3% of duplicates are created within 1 hour — the app/web doesn't prevent duplicate ticket creation when an open ticket already exists for the same seller.

**Recategorization waste:** 35/382 (9.2%) merged tickets were recategorized before merge — 21 by AI (Circuit), 14 by human agents. Most common: Payments-Seller sub-query refinement or cross-query misroutes (RC-Seller → Payments-Seller). This is wasted effort on tickets about to be merged.

**Sample merged tickets by time gap:**

| Gap | Merged Ticket | Parent Ticket | Source | Gap (min) |
|---|---|---|---|---:|
| < 5 min | `2057930` | `2057923` | App | 3 |
| < 5 min | `2058156` | `2058155` | VOC | 0 |
| 5–60 min | `2058266` | `2058044` | App | 10 |
| 5–60 min | `2061803` | `2061770` | Web | 10 |
| 1–24 hrs | `2058038` | `2055474` | VOC | 1392 |
| > 24 hrs | `2062156` | `2044719` | App | 8501 |

#### Payments-Seller Automation Map (non-merged + merged combined)

| # | Bucket | Tickets | % of 735 | Automation Potential | Key Lever |
|---|---|---:|---:|---|---|
| 1 | Payment Released | 201 | 27.3 | 🔴 **Very High** | Auto-check RC status → auto-release → auto-notify with UTR |
| 2 | Merged < 5 min | 137 | 18.6 | 🔴 **Very High** | App-level duplicate prevention — block if open ticket exists for same sell_lead |
| 3 | Merged > 24 hrs | 90 | 12.2 | 🔴 **High** | Proactive milestone notifications so seller doesn't re-raise |
| 4 | Merged 1–24 hrs | 81 | 11.0 | 🟡 Medium | Auto-status update: "We're working on it, ETA: X days" |
| 5 | Merged 5–60 min | 74 | 10.1 | 🔴 **Very High** | Same as #2 — app/web should prevent |
| 6 | Paused → Auto-Close | 42 | 5.7 | 🟢 Partial | Closure is auto; reduce pause-to-close cycle |
| 7 | Customer Unreachable | 41 | 5.6 | 🟡 Medium | Deflect to WhatsApp/chat after 1st failed OB |
| 8 | Info/TAT Shared | 40 | 5.4 | 🔴 **High** | Bot auto-reply with TAT based on agreement date + process stage |
| 9 | Others | 21 | 2.9 | ⚪ Unknown | Better reason tagging needed |
| 10 | Denial | 8 | 1.1 | 🟡 Medium | Eligibility check at ticket creation |

**~570 of 735 tickets (78%) are automatable:** 201 payment releases, 292 merged duplicates (app prevention + proactive status), 40 TAT/info shares, ~40 from Paused/Unreachable.

---

### Warranty Deep Dive — 0 OB Connected (1,947 tickets)

**331 non-merged + 1,616 merged (83%)**

#### Non-Merged: 331 tickets — 6 closure patterns

| # | Bucket | Tickets | % of 331 | What Happens |
|---|---|---:|---:|---|
| 1 | **WarrantyAI Bot auto-denial** | 68 | 20.5 | Bot 998 handles E2E. Issues verdict, customer doesn't reply → auto-closes. 0 agent msgs, 3.4 avg AI msgs. |
| 2 | **Paused → Auto-Close** | 65 | 19.6 | Agent attempts OB, can't connect → ticket paused → auto-closed. Avg 7.8 sys msgs. |
| 3 | **Scheduling Exhausted** | 62 | 18.7 | 3 failed scheduling attempts for inspection/workshop visit → auto-closed. |
| 4 | **Resolved on call** | 47 | 14.2 | Resolved via personal/office phones not tracked in CRM. Misleading "0 OB connected". |
| 5 | **3 call attempts exhausted** | 33 | 10.0 | 3 OB attempts, none connected → closed. Avg 5.6 sys msgs. |
| 6 | **Operational closures** | 56 | 16.9 | Out of Coverage (16), Resolved on chat (11), Car repaired & delivered/picked (13), Policy Expired (7), Reimbursement Done (5), others (4). |

**Sample tickets:**

| Bucket | Sample Tickets |
|---|---|
| WarrantyAI Bot auto-denial | `2119943`, `2119944`, `2120055` |
| Paused → Auto-Close | `2057669`, `2058413`, `2058591`, `2059244`, `2060589` |
| Scheduling Exhausted | `2057837`, `2061196`, `2065043`, `2065102`, `2065935` |
| Resolved on call | `2059221`, `2063892`, `2067674`, `2069252`, `2069824` |
| 3 call attempts exhausted | `2059170`, `2062160`, `2062222`, `2066528`, `2066910` |
| Operational closures | `2059578`, `2061566`, `2063991`, `2067449`, `2074351` |

#### Warranty Automation Map

| Bucket | Tickets/mo | Already Automated? | Automation Potential |
|---|---:|---|---|
| WarrantyAI Bot auto-denial | 68 | ✅ Yes | Already E2E bot-handled |
| Paused → Auto-Close | 65 | ⚠️ Partial | 🟡 Closure is auto; scale it |
| Scheduling Exhausted | 62 | ❌ | 🔴 **High** — self-serve scheduling via app/WhatsApp |
| Resolved on call (off-CRM) | 47 | ❌ | 🟡 Low — CRM call tracking fix, not automation |
| 3 call attempts exhausted | 33 | ⚠️ Partial | 🟡 Medium — deflect to chat/WhatsApp after 1st fail |
| Operational closures | 56 | Mixed | 🟡 Medium — denial/expiry checks can be bot-driven |

---

### Summary — Q3 Automation Priorities from 0-OB Analysis

| Priority | Initiative | Monthly Tickets Saved | Complexity |
|---|---|---:|---|
| **P0** | Auto-release payment holds (verify RC → release → notify with UTR) | ~290 | Medium — needs RC status API |
| **P0** | App-level duplicate ticket prevention (block if open ticket for same sell_lead) | ~211 | Low — frontend check |
| **P0** | Auto-share RC/courier status via bot | ~282 | Medium — needs delivery tracking integration |
| **P1** | Self-serve warranty scheduling (app/WhatsApp) | ~62 | Medium — needs scheduling API |
| **P1** | Bot auto-reply with templated info/TAT | ~172 | Low — template + trigger |
| **P1** | Eligibility/coverage check at ticket creation (warranty + RSA) | ~102 | Medium — needs rule engine |
| **P1** | Proactive milestone notifications for sellers (reduce >24hr duplicates) | ~90 | Medium — event-driven notifications |
| **P2** | WhatsApp/chat deflection after 1st failed OB attempt | ~348 | Low — routing change |
| **P2** | Fix CRM call tracking for off-system calls | ~139 | Low — infra fix |

**Total addressable: ~1,200 non-merged + ~3,400 merged = ~4,600 tickets/month (81% of all 0-OB tickets)**

---

## Q3 Roadmap (TBD)

> _To be built out — priorities, initiatives, owners, and timelines to follow._
