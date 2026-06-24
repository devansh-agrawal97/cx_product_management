# CX Domain — AIR OS Context

<!-- Local draft by Arpit Dhankani — not yet merged into spinny/spinny-air -->
<!-- All schema facts confirmed via live Doris queries — 2026-05-28 -->

This file captures CX-domain knowledge for eventual contribution to `context/domains/CX_AIR.md` in AIR OS.

---

## Table Hierarchy

CX ticket data lives across four layers — from raw operational DB to analytics-ready views:

```
iceberg.sp_ticket_master.*          ← raw operational DB (Django app backend)
         ↓
sp_entity_store_intermediate.cx_*   ← intermediate (per-ticket aggregates, mappings)
         ↓
sp_entity_store.support_tickets     ← entity store (fully denormalized, one row per ticket)
         ↓
sp_product.cx_tickets               ← analytics layer (pre-computed metrics: FCR, FRT, resolution time)
```

**Default for ad-hoc queries:** `sp_entity_store.support_tickets` — freshest, most complete, no joins needed.

**For message-level or raw event data:** use `iceberg.sp_ticket_master.*` intermediate tables.

---

## Schema & Tables

## ⭐ Operating rule — try `support_tickets` first, always

Per Arpit Dhankani 2026-05-29: **for any CX question, default to `sp_entity_store.support_tickets`**.
Only reach for other tables when this one doesn't have what you need.

### Decision tree

```
Question about a CX ticket?
├─ Default: sp_entity_store.support_tickets WHERE service_unit_id = 1
│  ├─ Unique only? AND ticket_merged_ts IS NULL
│  ├─ Closed only? AND ticket_closed_ts IS NOT NULL
│  └─ Use array columns for message text / call detail / status history / multi-concern
│
└─ Fall back ONLY when this table can't answer:
   ├─ FCR / is_dropped / is_reopened / is_circuit_resolved / same_month_closed_flag / ever_merged_flag
   │   → sp_product.cx_tickets (precomputed flags)
   │
   ├─ Warranty verdict per concern (warranty_status / warranty_status_reason)
   │   → iceberg.sp_ticket_master.ticket_query_mapping
   │
   ├─ Activity-log events NOT in support_tickets arrays
   │   (Recategorization, Add more concerns, Ticket Attributes Updated, …)
   │   → iceberg.sp_ticket_master.activity_log
   │
   ├─ Voicebot raw payload (transcript, extracted_data, telephony_data, cost_breakdown)
   │   → iceberg.sp_ticket_master.call_record
   │
   ├─ Days since car event (buyer delivery / seller stockin)
   │   → JOIN sp_product.demand_deal_masterdata (deal_id) for BUYER
   │   → JOIN iceberg.sp_web.listing_lead (sell_lead_id) for SELLER
   │
   ├─ Callback dispatch detail (preference type, scheduled slot)
   │   → support_tickets has first/latest_callback_slot_* directly; only need
   │     iceberg.sp_ticket_master.customer_callback_history for the full per-request list
   │
   └─ Recategorization payload (prev_query / updated_query / sub_query mappings)
       → iceberg.sp_ticket_master.activity_log (Recategorization / Ticket Attributes Updated)
```

### Use-case → column cheatsheet (all on `support_tickets`)

| Need | Column / pattern |
|---|---|
| Ticket volume | `COUNT(*)` (filter `service_unit_id = 1`) |
| Unique tickets | `COUNT(CASE WHEN ticket_merged_ts IS NULL THEN 1 END)` |
| Closed tickets | `COUNT(CASE WHEN ticket_closed_ts IS NOT NULL THEN 1 END)` |
| Same-month-closed | `DATE_TRUNC('month', ticket_created_ts) = DATE_TRUNC('month', ticket_closed_ts) AND ticket_merged_ts IS NULL` (or use the flag on `cx_tickets` if exactness matters) |
| CSAT | `(COUNT(customer_rating IN (4,5)) * 100.0) / NULLIF(COUNT(customer_rating > 0), 0)` |
| Rated % | `COUNT(customer_rating > 0 AND closed AND non-merged) / COUNT(closed AND non-merged)` |
| Avg resolution time | `AVG(SECONDS_DIFF(first_ticket_resolved_ts, ticket_created_ts)/3600.0)` — ⚠️ ~30% higher than `cx_tickets.resolution_time_hrs` for matured months because of timestamp-source differences. For exact dashboard parity, use `cx_tickets.resolution_time_hrs`. |
| Reopen rate | `COUNT(first_ticket_reopened_ts IS NOT NULL) / COUNT(*)` |
| First-response time | `SECONDS_DIFF(first_agent_assigned_ts, ticket_created_ts)/60.0` minutes |
| First-call response | `SECONDS_DIFF(first_connected_outbound_call_start_ts, ticket_created_ts)/3600.0` hrs |
| Outbound pickup rate | `total_outbound_connected_calls_count / total_outbound_attempted_calls_count` (per ticket) |
| Inbound pickup rate | `total_inbound_connected_calls_count / total_inbound_attempted_calls_count` |
| AI involvement on ticket | `ai_text_messages_count > 0` OR `first_agent_assigned_id IN (<bot_ids>)` |
| Bot solo-resolved | `first_agent_assigned_id = latest_agent_assigned_id AND first_agent_assigned_id IN (<bot_ids>)` |
| Recategorisation count (AI-only) | `query_updated_by_ai_count` (direct integer) |
| Total recat count | `query_updated_count` |
| Multi-concern warranty | `LATERAL VIEW EXPLODE(sub_query_metadata)` |
| NDOA breach | `LATERAL VIEW EXPLODE(indicators_metadata) WHERE GET_JSON_STRING(elem,'$.indicator_name') = 'NDOA_BREACHED'` |
| Callback requested | `first_callback_requested_ts IS NOT NULL` OR `callback_requests_created_count > 0` |
| Callback preference type | `first_callback_slot_preference_type_name` |
| Callback dispatched in slot | Compare `first_connected_outbound_call_start_ts` to `first_callback_slot_ts` |
| Days since delivery (BUYER) | `DATEDIFF(ticket_created_ts, dmd.delivery_time)` after JOIN on `deal_id` |
| Days since stockin (SELLER) | `DATEDIFF(ticket_created_ts, DATE_ADD(ll.delivery_time, INTERVAL 330 MINUTE))` after JOIN on `sell_lead_id` |
| Read message text | `LATERAL VIEW EXPLODE(communications_metadata) → GET_JSON_STRING(elem, '$.text')` |
| Read call subject / purpose | `LATERAL VIEW EXPLODE(calls_ai_summary) → GET_JSON_STRING(elem, '$.call_purpose')` |
| Read status change history | `LATERAL VIEW EXPLODE(ticket_status_metadata)` |
| Agent assignment history | `LATERAL VIEW EXPLODE(assigned_agents_metadata)` |
| KAM on the underlying lead | `permanent_kam_email` (and `temporary_kam_email` if active) |
| Customer city | `deal_hub_city_name` (not `car_registration_city_name` — that's RTO location) |

### What `support_tickets` does NOT have (must fall back)

| Column / metric | Where to find |
|---|---|
| `fcr_flag` ('FCR' / 'NON_FCR' / 'flag' / 'dropped' / NULL) | `sp_product.cx_tickets.fcr_flag` |
| `is_dropped` | `sp_product.cx_tickets.is_dropped` (or `fcr_flag = 'dropped'` on `cx_tickets`) |
| `is_reopened` (precomputed flag) | `sp_product.cx_tickets.is_reopened` — but support_tickets has `first_ticket_reopened_ts` which is equivalent |
| `is_circuit_resolved` | `sp_product.cx_tickets.is_circuit_resolved` |
| `same_month_closed_flag` | `sp_product.cx_tickets.same_month_closed_flag` — or derive on support_tickets |
| `ever_merged_flag` | `sp_product.cx_tickets.ever_merged_flag` — or use `ticket_merged_ts IS NOT NULL` on support_tickets |
| `resolution_time_hrs` (ETL-precomputed) | `sp_product.cx_tickets.resolution_time_hrs` — for exact dashboard parity; otherwise derive on support_tickets |
| Warranty verdict per concern | `iceberg.sp_ticket_master.ticket_query_mapping.warranty_status` |
| Voicebot transcript / extracted_data | `iceberg.sp_ticket_master.call_record.call_data` JSON |
| `Add more concerns` event count | `iceberg.sp_ticket_master.activity_log` |

### `sp_entity_store.support_tickets` — Primary detailed CX Table (169 columns)

One row per ticket. Fully denormalized — includes customer, agent, car, lead, deal, and message-aggregate columns. **Use this for most CX analysis.**

**Full schema grouped by functional area** (confirmed via `DESC` on 2026-05-29):

#### 1. Ticket identity & lifecycle (11 cols)
`ticket_id` (PK · int) · `ticket_created_ts` · `ticket_closed_ts` · `ticket_resolved_ts` · `first_ticket_closed_ts` · `latest_ticket_closed_ts` · `first_ticket_resolved_ts` · `latest_ticket_resolved_ts` · `first_ticket_reopened_ts` · `latest_ticket_reopened_ts` · `ticket_merged_ts`
> Use `first_*` for the canonical first event; `latest_*` reflects reopens. `ticket_merged_ts IS NOT NULL` is the per-ticket merge flag.

#### 2. Query taxonomy (8 cols)
`query_name` · `query_category_name` · `primary_sub_query_id` · `primary_sub_query_name` · `primary_query_heading_id` · `primary_query_heading_name` · **`sub_query_metadata`** (array) · `ticket_priority_code`
> `primary_*` = the first / dominant sub_query. **`sub_query_metadata`** is a JSON-string array with the full multi-concern picture: `[{"sub_query_id","sub_query_name","query_headings":[{"query_heading_id","query_heading_name"}]}, …]`. Use this instead of `ticket_query_mapping` joins for ad-hoc analysis.

#### 3. Ticket attributes (3 cols)
`service_unit_id` (must be 1 for CX) · `ticket_source_name` · `ticket_status_name` · `ticket_status_metadata` (array)
> **`ticket_status_metadata`** is an array of all status changes: `[{"status_updated_ts","previous_state","new_state","status_updated_by_id","status_updated_by_username","status_updated_by_email", …}, …]`. Full audit trail without needing `activity_log`.

#### 4. Parent/case hierarchy (6 cols)
`parent_ticket_id` · `case_tickets_count` · `first_case_ticket_id` · `case_created_ts` · `case_closed_ts` · `case_resolved_ts`
> A "case" groups related tickets (e.g. follow-ups, merges). Use the case-level fields when reporting at customer-issue granularity.

#### 5. Agent assignment (12 cols)
`assigned_agents_count` · `assigned_agents_metadata` (array) · `first_agent_assigned_id` · `first_agent_assigned_username` · `first_agent_assigned_full_name` · `first_agent_assigned_email` · `first_agent_assigned_ts` · `latest_agent_assigned_id` · `latest_agent_assigned_username` · `latest_agent_assigned_full_name` · `latest_agent_assigned_email` · `latest_agent_assigned_ts`
> **`assigned_agents_metadata`** = `[{"agent_id","assigned_ts","agent_username","agent_fullname","agent_email"}, …]` — full reassignment history. Use `first_agent_assigned_id IN (bot_ids)` for "first-touch by bot" analyses.

#### 6. Query updates (8 cols)
`first_query_updated_ts` · `latest_query_updated_ts` · `first_query_updated_by_id` · `first_query_updated_by_username` · `latest_query_updated_by_id` · `latest_query_updated_by_username` · `query_updated_count` · `query_updated_by_ai_count`
> Recategorization/add-more-concerns events. `query_updated_by_ai_count` directly counts the Circuit-bot recategorizations on this ticket (cross-check against the AI vs Manual recat split from `activity_log`).

#### 7. CSAT (2 cols)
`customer_rating` (1–5 int, NULL if not rated) · `customer_rating_submitted_ts`

#### 8. Closing (2 cols)
`ticket_closing_reason_id` · `ticket_closing_reason_name`
> ⚠️ **`ticket_closing_reason_name` is STALE** — denormalized from an old snapshot of the reason table; many names no longer match the live reason. **Use `ticket_closing_reason_id` only for joins**, then get the current name from `iceberg.sp_ticket_master.ticket_reason.display_name`. See Known Trap #8.

#### 9. Car info (7 cols)
`car_id` · `lead_category_name` (assured / auction / budget / luxury / bcm) · `car_registration_number` · `car_registration_city_id` · `car_registration_city_name` · `car_registration_parent_city_id` · `car_registration_parent_city_name`
> `car_registration_*` is the RTO location, not the customer location. For customer-location use the deal_hub fields.

#### 10. Customer info (5 cols)
`customer_id` · `customer_type_name` (BUYER / SELLER / DEALER / OTHER) · `customer_name` · `customer_phone_number` · `customer_alternate_phone_number`
> These are **plaintext** here (entity store), while the raw `iceberg.sp_ticket_master.customer.{name,phone,alternate_phone}` columns are MD5-hashed. Use these.

#### 11. KAM linkage (8 cols)
`permanent_kam_id` · `permanent_kam_username` · `permanent_kam_email` · `permanent_kam_fullname` · `temporary_kam_id` · `temporary_kam_username` · `temporary_kam_email` · `temporary_kam_fullname`
> The KAM is the long-term account manager on the underlying lead, *not* the CX agent handling the ticket. KAM ≠ ticket assignee.

#### 12. Funnel linkage (4 cols)
`sell_lead_id` · `deal_id` · `buy_lead_id` · `dealer_id`
> Direct PKs into supply / demand / auction funnels. `sell_lead_id` joins to `iceberg.sp_web.listing_lead.id` (use for seller days-since-stockin). `deal_id` joins to `sp_product.demand_deal_masterdata`.

#### 13. Deal hub (6 cols)
`deal_hub_id` · `deal_hub_name` · `deal_hub_city_id` · `deal_hub_city_name` · `deal_hub_parent_city_id` · `deal_hub_parent_city_name`
> Hub where the deal was transacted — the canonical "city" for CSAT/volume splits.

#### 14. Social media & feedback channels (4 cols)
`social_media_platform_name` · `social_media_post_url` · `feedback_source_name` · `feedback_form_url`

#### 15. Action / task tracking (7 cols)
`next_date_of_action_ts` (the **NDOA**!) · `assigned_tasks_count` · `open_tasks_count` · `total_self_tasks_count` · `open_self_tasks_count` · `all_tasks_deadline_ts` · `open_tasks_deadline_ts`
> `next_date_of_action_ts` is the SLA commitment. NDOA breach = current time past this ts and ticket not closed. The `indicators_metadata` array also carries `NDOA_BREACHED` / `NDOA_TODAY` events.

#### 16. Indicators (5 cols)
**`indicators_metadata`** (array) · `indicators_created_count` · `indicators_resolved_count` · `callback_requests_created_count` · `callback_requests_resolved_count`
> **`indicators_metadata`** = `[{"indicator_name","indicator_created_ts","indicator_removed_ts"}, …]`. Indicator names seen: `NDOA_BREACHED`, `NDOA_TODAY`, `CALLBACK_REQUESTED`, `CUSTOMER_MESSAGE`, `UNREAD`, `UNREAD_TASKS`, `BREACHED_TASKS`, `REMINDER`.

#### 17. Callback slots (6 cols)
`first_callback_requested_ts` · `latest_callback_requested_ts` · `callback_slots_metadata` (array) · `first_callback_slot_preference_type_name` · `first_callback_slot_ts` · `latest_callback_slot_preference_type_name` · `latest_callback_slot_ts`
> `preference_type_name` = `'Next 90 Minutes'` or `'Scheduled Slot'`. Use this section for callback-SLA analyses without joining to `iceberg.sp_ticket_master.customer_callback_history`.

#### 18. Voice notes (3 cols + 1 metadata)
`communications_metadata` (array — see below) · `total_voice_notes_count` · `first_voice_note_ts` · `latest_voice_note_ts`

#### 19. Media messages (7 cols)
`total_media_messages_count` · `customer_media_messages_count` · `agent_media_messages_count` · `first_customer_media_message_ts` · `latest_customer_media_message_ts` · `first_agent_media_message_ts` · `latest_agent_media_message_ts`
> "Media" = image / file attachments.

#### 20. Text messages (14 cols)
Counts: `total_text_messages_count` · `customer_text_messages_count` · `agent_text_messages_count` · `ai_text_messages_count` · `system_text_messages_count`
First/latest per sender type: `first_text_message_ts` · `latest_text_message_ts` · `first_customer_text_message_ts` · `latest_customer_text_message_ts` · `first_agent_text_message_ts` · `latest_agent_text_message_ts` · `first_ai_text_message_ts` · `latest_ai_text_message_ts` · `first_system_text_message_ts` · `latest_system_text_message_ts`
> **AI-text-message volume per ticket is here** — most direct metric for bot-touch.

#### 21. Communications metadata (full text!) (1 col)
**`communications_metadata`** (array) = `[{"message_id","message_created_ts","message_type","sender_user_id","sender_type","agent_fullname","agent_email","text"}, …]`
> ⚡ **The `text` here is the actual unhashed message body**, including customer messages, agent replies, INTERNAL_NOTE, MESSAGE, system templates. This is the cleanest path to bot↔customer conversations without joining to the SQLMesh `cx_communication_messages` table. The array can be 20+ messages per ticket on chatty cases.

#### 22. Calls — ticket-level (18 cols + 2 arrays)
**`calls_metadata`** (array) · **`calls_ai_summary`** (array)
Counts: `total_outbound_calls_count` · `total_outbound_attempted_calls_count` · `total_outbound_connected_calls_count` · `total_inbound_calls_count` · `total_inbound_attempted_calls_count` · `total_inbound_connected_calls_count`
Timestamps: `first_attempted_outbound_call_start_ts` · `first_connected_outbound_call_start_ts` · `first_attempted_outbound_call_post_assignment_ts` · `latest_attempted_outbound_call_start_ts` · `latest_connected_outbound_call_start_ts` · `first_attempted_inbound_call_start_ts` · `first_connected_inbound_call_start_ts` · `latest_attempted_inbound_call_start_ts` · `latest_connected_inbound_call_start_ts`
> ⚠️ **The `*_calls_count` columns are integers here** (unlike the fractional ones on `supply_workflow_tasks`). Confirmed integer values for outbound and inbound.
> **`calls_metadata`** = `[{"call_id","call_type_name","call_status_name","call_start_ts","call_end_ts","agent_id","agent_fullname","agent_email","campaign_id","campaign_name", …}, …]`
> **`calls_ai_summary`** = `[{"ai_summary_generated_ts","call_id","call_subject","call_action_items","call_purpose","call_customer_queries","call_discussion_points"}, …]` — LLM-extracted summary of each call.

#### 23. AI ticket summary (2 cols)
`ticket_summary_generated_ts` · `ticket_summary` (text)
> LLM-generated overall ticket summary, regenerated as the conversation evolves. Pairs with the chart-448 / Overview metric flows.

#### 24. Calls — case-level (parent-ticket aggregates) (15 cols + 1 array)
Same shape as section 22, prefixed with `case_*` — aggregates calls across all tickets in the same case (parent_ticket_id family). Use these when you want the customer-issue-level call story instead of single-ticket.

#### 25. Record freshness (1 col)
`record_updated_ts` — batch refresh time. Check this if you suspect staleness.

---

### Quick reference: which array to use for what

| Need | Array column | Sub-keys |
|---|---|---|
| All sub-queries on this multi-concern ticket | `sub_query_metadata` | `sub_query_id`, `sub_query_name`, `query_headings[]` |
| Full status change history | `ticket_status_metadata` | `status_updated_ts`, `previous_state`, `new_state`, `status_updated_by_*` |
| All agent assignments + reassignments | `assigned_agents_metadata` | `agent_id`, `assigned_ts`, `agent_username/fullname/email` |
| Indicator flags (NDOA, callback, breach) | `indicators_metadata` | `indicator_name`, `indicator_created_ts`, `indicator_removed_ts` |
| Callback slot history | `callback_slots_metadata` | (slot type + scheduled ts) |
| **Full message text on the ticket** | **`communications_metadata`** | `message_id`, `message_type`, `sender_type`, `agent_*`, **`text`** |
| All calls with timing + agent | `calls_metadata` | `call_id`, `call_type_name`, `call_status_name`, `call_start/end_ts`, `agent_*`, `campaign_*` |
| LLM summaries of calls | `calls_ai_summary` | `call_subject`, `call_purpose`, `call_action_items`, `call_customer_queries`, `call_discussion_points` |

⚠️ Array columns are stored as `array<text>` where each element is a JSON-string. To unpack:
```sql
-- Doris pattern: explode + parse the JSON elements
SELECT
  ticket_id,
  GET_JSON_STRING(elem, '$.indicator_name') AS indicator_name,
  GET_JSON_STRING(elem, '$.indicator_created_ts') AS created_ts
FROM sp_entity_store.support_tickets
LATERAL VIEW EXPLODE(indicators_metadata) t AS elem
WHERE service_unit_id = 1
```

⚠️ **MANDATORY: filter on `service_unit_id = 1`** when using `support_tickets` for CX analysis. Unlike
`sp_product.cx_tickets` (which is CX-scoped at the ETL layer), `support_tickets` carries **all six
service units** — including FEEDBACK auto-collection (~48K/mo) and AUCTION dealer tickets (~11K/mo).
Without the filter, your denominators will be ~4× the true CX volume and all rate metrics
(CSAT / FCR / resolution %) will be heavily diluted.

```sql
-- ALWAYS apply this filter when querying support_tickets for CX:
FROM sp_entity_store.support_tickets
WHERE service_unit_id = 1
  AND <other filters>
```

Recommended pattern for detailed analyses requiring columns that exist on `support_tickets` but not
on `sp_product.cx_tickets` (e.g. `primary_sub_query_name`, agent IDs, message-aggregate counts):
```sql
-- Option A: start from cx_tickets (auto-scoped), enrich with support_tickets columns
FROM sp_product.cx_tickets cxt
LEFT JOIN sp_entity_store.support_tickets st ON st.ticket_id = cxt.ticket_id

-- Option B: start from support_tickets, apply service_unit_id = 1 explicitly
FROM sp_entity_store.support_tickets st
WHERE st.service_unit_id = 1
```
Option A is what the dashboard's dataset 206 does. Option B is fine for ad-hoc; just don't forget
the filter.

Key columns (confirmed):

| Column | Type | Notes |
|---|---|---|
| `ticket_id` | int | PK |
| `ticket_created_ts` | datetime(3) | Ticket creation time (IST) |
| `ticket_closed_ts` | datetime(3) | Latest close time |
| `ticket_resolved_ts` | datetime(3) | Latest resolve time |
| `first_ticket_closed_ts` | datetime(3) | First close (use for FCR TAT calc) |
| `first_ticket_resolved_ts` | datetime(3) | First resolve time |
| `first_ticket_reopened_ts` | datetime(3) | Non-null = ticket was reopened at least once |
| `query_name` | varchar | Specific query (e.g. "Payments - Seller", "RC", "Welcome call") |
| `query_category_name` | varchar | Top-level category — see enum below |
| `primary_sub_query_name` | varchar | Sub-category of query |
| `primary_query_heading_name` | varchar | Query heading |
| `ticket_priority_code` | varchar | Severity — see enum below |
| `ticket_source_name` | varchar | Origination channel — see enum below |
| `ticket_status_name` | varchar | Current status — see enum below |
| `ticket_closing_reason_name` | varchar | Why it was closed — see top values below |
| `customer_type_name` | varchar | BUYER / SELLER / DEALER / OTHER |
| `customer_id` | int | Join to customer master |
| `customer_rating` | double | CSAT score (same as `cx_rating` in intermediate) |
| `customer_rating_submitted_ts` | datetime(3) | When CSAT was submitted |
| `car_id` | int | Associated car |
| `lead_category_name` | varchar | assured / auction / budget / luxury / bcm — ~50% null |
| `sell_lead_id` | int | Bridge to supply funnel (sell_leads.lead_id) |
| `buy_lead_id` | int | Bridge to demand funnel |
| `deal_id` | int | Associated deal |
| `dealer_id` | int | Dealer (for auction/dealer tickets) |
| `deal_hub_id` / `deal_hub_name` | int/varchar | Hub location |
| `first_agent_assigned_email` | varchar | First agent who handled the ticket |
| `latest_agent_assigned_email` | varchar | Most recent agent |
| `first_agent_assigned_ts` | datetime(3) | When first agent was assigned (use for FRT) |
| `permanent_kam_email` | varchar | Permanent KAM on the related lead |
| `service_unit_id` | int | Service unit / team bucket |
| `total_outbound_calls_count` | double | ⚠️ Stores FRACTIONS, not integers — see trap below |
| `total_inbound_calls_count` | double | ⚠️ Same — use connected timestamp columns for "at least one call" checks |
| `first_connected_outbound_call_start_ts` | datetime(3) | Safe proxy for "at least one outbound connected call" |
| `total_text_messages_count` | double | Total messages on ticket |
| `ai_text_messages_count` | double | Messages handled by AI agent |
| `ticket_summary` | text | AI-generated ticket summary |
| `ticket_summary_generated_ts` | datetime(3) | When AI summary was generated |
| `record_updated_ts` | datetime(3) | Batch refresh timestamp — check freshness |

---

### `iceberg.sp_ticket_master` — Raw Operational DB

Django app backend. Use when entity store doesn't have what you need (per-message metadata,
multi-query fanout, activity history, warranty verdict codes).

Key tables confirmed:

| Table | What it holds |
|---|---|
| `ticket` | Raw ticket rows: `id`, `status` (code), `priority` (code), `source` (code), `customer_type` (code), `deal_id`, `car_id`, `closing_reason_id`, `cx_rating`, `created_at`, `resolved_at`, `closed_at` |
| `communication` | Maps ticket→task: `ticket_id`, `task_id`, `status`, `unread_message_count` |
| `communication_message` | Individual messages: `id`, `communication_id`, `message_type`, `text` (⚠️ HASHED — see below), `is_played`, `is_summary_generated` |
| `ticket_query_mapping` | **One row per customer-raised query on a ticket** — `ticket_id`, `query_heading_id`, `sub_query_id`, `free_text`, `warranty_status`, `warranty_status_reason`. Warranty tickets have up to **33 queries on a single ticket** (avg 1.64); all other ticket types average ~1.0. |
| `activity_log` | Full audit log of everything that happens on a ticket: `activity` (event name), `data` (JSON payload), `user_id` (bot or human), `object_id` (the ticket). |
| `call_record` | Raw call events per ticket |
| `user` | User master incl. AI bots — `is_bot = true` flag. ⚠️ `username` and `email` are **hashed**, only `first_name` / `last_name` are human-readable. |
| `customer` | Customer master (most PII fields hashed) |
| `car` | Car details |
| `ticket_meta_data` | Additional ticket metadata |
| `ticket_status_logs` | Full status change log (point-in-time history) |
| `ticket_closing_reason` | ~~Old lookup table — superseded~~ |
| `ticket_reason` | **New** reason lookup: `id`, `name`, `display_name`, `category` (`TICKET_CLOSING` / `TICKET_PAUSE` / `KAM_CHANGE_REQUEST_BY_CUSTOMER` / etc.). Join: `ticket.closing_reason_id = ticket_reason.id WHERE category = 'TICKET_CLOSING'` |
| `voicebot_call_response` | Voicebot interaction logs |

⚠️ **`iceberg.sp_ticket_master` timestamps are UTC** — add 330 minutes when comparing to entity store (IST).

### Masked → Unmasked column audit (re-run 2026-05-29)

The earlier mass-hashing in `iceberg.sp_ticket_master` has been largely **rolled back**. Re-sample of
all 21 previously-masked columns:

- **17 columns are now plaintext / parseable JSON**
- **3 columns remain hashed** (32-char MD5): `customer.name`, `customer.phone`, `customer.alternate_phone`
- **1 column appears empty** in recent rows (no values to sample): `django_custom_admin_log.username`, `user.username`/`email`/`agent_id` returned empty samples in the test window

⚠️ **The unmasking re-introduced customer PII into ~11 columns.** A masked column should never expose
customer phone/email/name/contact in any form. The PII Audit table below flags every leak.

---

### Per-column content reference

#### `agent_change_request_mapping.meta_data` — JSON, **clean**
Tracks change-request context. **Keys:** `car_reg_no`, `car_delivery_date`, `procurement_category`,
`previous_request_city_id`, `previous_kam_id`. No customer PII.

#### `ai_response_log.response` — JSON, **clean (structured fields)**
LLM verdict payloads. **Keys (43 total):** `brief_summary`, `overall_ticket_summary`,
`ticket_summary`, `proposed_timeline.{timeline_text,date,evidence_message_ids}`,
`actions_pending.{Customer,Support_Agent,Internal_Ops}[].{task,evidence_message_ids}`,
`action_pending_{internal,support,customer}`, `reason`, `decision`, `filter_matched`,
`escalation_alert`, `social_media_escalation`, `legal_escalation`, `empathy_concerns`,
`sla_violation_risk`, `confidence_score`, `final_csat_estimate`, `sentiment_trend`,
`customer_requested_ticket_closure`, `customer_pending_actions`, `agent_pending_actions`,
`top_3_recent_concerns`, `ai_response_templates`, `recategorised`, `new_query`, `new_subquery`,
`new_query_heading`, `extra_details`, `ticket_id`. None of the **keys** are PII fields. ⚠️ But the
free-text values (e.g. `overall_ticket_summary`, `top_3_recent_concerns`) can echo whatever the
customer wrote.

#### `ai_response_log.request_data` — plaintext, **contains customer message text** ⚠️
Format: `Request-ID: ...\nCustomer message: <raw customer text>\n...\nLast agent message or call
summary: <agent text>`. No structured PII field, but the customer's exact message body is
preserved verbatim — and customers do paste phone numbers, registration numbers, and names into
their messages.

#### `call_record.call_data` — JSON, **PII** ⚠️
Voicebot/CallKaro per-call records (Plivo/Bolna providers).
**Top-level keys (180 nested):** `id`, `status`, `summary`, `agent_id`, `batch_id`, `provider`,
`user_number`*, `agent_number`, `created_at`, `updated_at`, `scheduled_at`, `initiated_at`,
`rescheduled_at`, `total_cost`, `cost_breakdown.*`, `transcript`, `retry_config`, `retry_history`,
`retry_count`, `workflow_retries`, `campaign_id`, `smart_status`, `error_message`,
`answered_by_voice_mail`, `lid_data`, `external_client_name`*, `extracted_data`,
`custom_extractions`, `agent_extraction`, `tool_call_logs`, `agent_context_details`,
`transfer_call_data`, `context_details.recipient_phone_number`*,
`context_details.recipient_data.{customer_name*, car_details, request_id, ticket_id, success, todo_id}`,
`telephony_data.{hangup_reason, hosted_telephony, recording_url, duration, ring_duration,
post_dial_delay, provider, call_type, hangup_provider_code, hangup_by, from_number, to_number,
to_number_carrier, provider_call_id}`, `batch_run_details`.
* = **PII fields**: `user_number` (customer's actual phone, e.g. `+919818769127`),
`context_details.recipient_phone_number`, `context_details.recipient_data.customer_name`,
`external_client_name`.

#### `chat_template.template` — plaintext, **clean**
Canned message templates (e.g. *"Hope your car has been safely delivered to your location and
everything is running smoothly"*). No PII.

#### `common_filemodelrelation.file_url` — plaintext, **clean**
Call recording URLs. Two patterns:
- `https://spinnynew.ameyoemerge.in:8887/ameyowebaccess/command?command=downloadVoiceLog&data={'crtObjectId':'d247-69f8ff7b-vce-daf-525803','targetFormat':'mp3'}` (Ameyo)
- `sp-call-recording/ameyo/312258974/call_recording.mp3` (S3 path)

#### `common_filemodelrelation.file_name` — plaintext, **PII** ⚠️
Format: `<10-digit-prefix>-<numeric_id>`. The 10-digit prefix is a phone number (e.g.
`7814668382-19424775`, `8130557776-16724172`). Customer phone is leaked through the file naming
convention.

#### `communication_message.text` — plaintext, **may contain PII**
Raw message body (customer ↔ agent ↔ bot ↔ system). Could be: `"Kitna time lagega"`,
`"CX said no one had called him about the welcome kit..."`. Customer text is verbatim — anything
the customer types lands here.

#### `communication_message.meta_data` — JSON, **PII** ⚠️
Per-message metadata, primarily wrapping the call/CRM payload.
**Keys (84 total):** `campaign_group`, `campaign_name`*, `account_id`, `phone_number`*, `city`,
`operator_name`*, `metadata.{service, contextType, contextId}`, `call_log_id`, `business_name`*,
`category`, `agent_id`, `operator_campaign_id`, `is_query_message`, `files`, `ticket_id`,
`dealer_id`, `is_audio_note`, `customer_phone`*, `is_system_generated`, `message_description`,
`callback_data.{customerCRTId, talkTime, lastStatus, hangupFirst, customerId, userAssociations[].
{userId, associtionType, associationId}, recordingFileUrl, userId, callback_time, preferred_language_input,
comment, ringingTime, campaignId, userCrtObjectId, callId, language_input, systemDisposition,
holdTime, leadId, phoneNumber*, hangUpOnHold, ivrTime, dispositionCode, call_token_id, dialedTime,
phone*, callStartTime, callResult, numAttempts, setupTime, transfer_call_status, ivr_input,
dstPhone*, callType, VoiceFileURL, callEndTime, campaign_id, callbackTime, srcPhone*, totaltimespent, dtmf}`.
* = **PII**: `customer_phone`, `phone_number`, `callback_data.{phone, phoneNumber, dstPhone, srcPhone}`,
plus agent identifiers (`campaign_name`, `operator_name`, `business_name`).

#### `customer.name` — **STILL HASHED** (32-char MD5) ✓
#### `customer.phone` — **STILL HASHED** (32-char MD5) ✓
#### `customer.alternate_phone` — **STILL HASHED** (32-char MD5) ✓
#### `customer.email` — plaintext, **PII** ⚠️
Actual customer email addresses (e.g. `asheeshkataria@gmail.com`). Inconsistent with the hashing of
the other PII fields on the same table.

#### `customer.meta_data` — JSON, **PII** ⚠️
Tiny payload. **Keys:** `contact_number`* (the customer's plain phone, e.g. `9818769127`), `city`,
`positive_post_event_sent`. The `contact_number` here bypasses the `customer.phone` hashing.

#### `deal.warranty_data` — JSON, **clean** (vehicle-only, no customer PII)
The warranty-evaluation snapshot. **Keys (23 total):** `file_url` (PDF link), `fetched_at`,
`plan_version`, `rsa_chargable`, `is_assured_plus`, `listing_details`, `registration_no`,
`is_protect_opted`, `is_protect_plus_opted`, `is_super_protect_opted`, `is_protect_star_opted`,
`car_delivery_date`, `odometer_reading_at_purchase`, `comprehensive_warranty`,
`powertrain_warranty.{status, date}`, `key_systems_warranty.{status, date}`,
`functional_warranty.{status, date}`.

#### `external_statuses.message` — plaintext, **may contain PII**
Agent-written notes. Sample: *"The seller has agreed with the transfer process, Please connect
with the seller. Seller details ..."* — agents often paste seller names/phones here.

#### `feedback_voicebot_data.customer_phone_number` — plaintext, **PII** ⚠️
Raw 10-digit phone numbers, e.g. `9833838815`. Column name says exactly what it is.

#### `feedback_voicebot_response.call_back_data` — JSON, **clean**
**Keys (6):** `request_id`, `call_status`, `call_duration`, `call_start_time`, `call_end_time`,
`call_recording_url` (S3 path). No PII.

#### `missed_call_detection.phone_number` — plaintext, **PII** ⚠️
Raw 10-digit phone numbers, e.g. `9861351317`.

#### `sub_query.name` — plaintext, **clean**
Warranty/support sub-query labels: `Engine`, `My issue is not listed here`, etc. (Previously
mistakenly hashed; now matches the readable `support_tickets.primary_sub_query_name`.)

#### `ticket.extra_data` — JSON, **PII** ⚠️
Original creation payload. **Keys (63 total):** `user_id`, `reference`, `is_escalated`,
`feedback_source`, `feedback_queue`, `request_payload.{source, comment, car_reg_no, customer_type,
customer_contact_no*, sub_query_to_query_heading_mapping.<sub_query_id>}`, `social_post_url`,
`social_media_source`, `survey_monkey_link`, `is_created_manually`, `is_consumer`, `deal_id`,
`lead_id`, `visit_id`, `transaction_id`, `city_name`*, `description`, `procurement_category`,
`satisfy_auto_response_criteria`, `is_odometer_photo_mandatory`, `inspection_outcome`,
`last_callback_requested_at`, `escalated_to_management`, `is_concern_marked_unresolved`,
`missed_call_count`.
* = **PII**: `request_payload.customer_contact_no`, `city_name`.

#### `ticket_communication_call_log_mapping.meta_data` — JSON, **PII** ⚠️
Same schema as `communication_message.meta_data` (CRM payload). Same PII fields: `phone_number`,
`callback_data.{phone, phoneNumber, dstPhone, srcPhone}`, plus agent fields. ⚠️ This is the column
the broken callback dashboard (Superset 223) tries to read — it's no longer hashed, so that
dashboard could now be fixed using the JSON path.

#### `ticket_test.extra_data` — JSON, **PII** ⚠️
Same shape as `ticket.extra_data` (subset of keys). Also leaks `customer_contact_no` and `city_name`.

#### `todo.call_config` — JSON, **clean**
Outbound call configuration. **Keys (21):** `from_phone_number` (internal Spinny virtual number,
e.g. `+918031454111` — NOT customer), `retry_config.{enabled, max_retries, retry_on_statuses,
retry_on_voicemail, retry_intervals_minutes}`, `min_gap_between_call`, `min_trigger_time`,
`max_trigger_time`, `ob_scheduler_time_gap`, `pickup_scores` (48-element int array of
hour-of-day pickup probabilities), `similarity_window_min`, `decay_horizon_days`,
`call_increment_trigger`, `is_feedback_todo`, `carry_over`, `number_of_retries`,
`gap_between_retries`, `priority`, `language`. No customer PII.

#### `user.username | email | agent_id` — empty in recent samples (cannot confirm unmasking state)
#### `voicebot_call_response.phone_number` — plaintext, **PII** ⚠️
Raw 10-digit phone numbers, e.g. `8238444324`.

---

### PII Leakage Summary — columns that need re-masking

| Column | What leaks | Severity |
|---|---|:--:|
| `customer.email` | Raw customer email addresses | 🔴 high |
| `customer.meta_data.contact_number` | Raw customer phone (bypasses `customer.phone` MD5) | 🔴 high |
| `feedback_voicebot_data.customer_phone_number` | Raw 10-digit phone | 🔴 high |
| `missed_call_detection.phone_number` | Raw 10-digit phone | 🔴 high |
| `voicebot_call_response.phone_number` | Raw 10-digit phone | 🔴 high |
| `common_filemodelrelation.file_name` | Phone prefix in file IDs | 🔴 high |
| `call_record.call_data` | `user_number`, `recipient_phone_number`, `customer_name`, `external_client_name` | 🔴 high |
| `communication_message.meta_data` | `customer_phone`, `phone_number`, `callback_data.*phone*` | 🔴 high |
| `ticket_communication_call_log_mapping.meta_data` | `phone_number`, `callback_data.*phone*` | 🔴 high |
| `ticket.extra_data` | `request_payload.customer_contact_no` | 🔴 high |
| `ticket_test.extra_data` | `request_payload.customer_contact_no` | 🟡 medium (test table) |
| `ai_response_log.request_data` | Customer message body (free-text — may contain phone/email) | 🟡 medium |
| `communication_message.text` | Customer message body | 🟡 medium |
| `external_statuses.message` | Agent notes (may contain seller/buyer details) | 🟡 medium |

A masked column should never expose customer information in any form. The 11 🔴-marked columns above
violate this. The 🟡-marked columns leak via free-text content rather than a structured PII field —
also worth tightening.

### Detection heuristic (for future audits)

Masked columns previously stored **pure hex strings** of fixed length: `^[0-9a-f]{32,}$` matched all
sampled non-null values. Most fields were 128-char (SHA-512); a few customer-PII fields used 32-char
MD5. After the rollback, only `customer.name`/`phone`/`alternate_phone` retain the 32-char MD5 form.

### Multi-query tickets (warranty-specific) — and how to count them correctly

Warranty tickets can carry **multiple distinct customer concerns on a single ticket** (A/C + Brake +
Lights together). The schema is `ticket_query_mapping`, with one row per concern.

⚠️ **`ticket_query_mapping` rows are NOT one-per-concern.** Each time the customer clicks "Add more
concerns" in the app, the system **re-writes ALL active concerns as new rows** — not just the new
ones. So raw row count overstates concern count.

**Real example (ticket 2192369):** 7 `Add more concerns` activity_log events between 18:42 and 18:51
on May 18 created 22 tqm rows, but many were re-saves of the same (query_heading_id, sub_query_id)
tuples from a few seconds earlier. Activity_log confirmed each "Add more concerns" event corresponded
to a batch of tqm-row creates within the same second.

**Canonical de-dup pattern:**
```sql
COUNT(DISTINCT CONCAT(CAST(query_heading_id AS STRING),'/',CAST(sub_query_id AS STRING)))
   -- distinct concerns per ticket
```

**Corrected multi-query ratio (Apr–May '26, deduped on (query_heading_id, sub_query_id)):**

| Query type | Tickets | Avg raw rows | Avg distinct concerns | Inflation |
|---|--:|--:|--:|--:|
| **Warranty** | 14,439 | 1.64 | **1.50** | 8.8% |
| Insurance | 2,513 | 1.00 | (mostly NULL keys) | – |
| RC / RC-Seller / RSA / Document / Payments | – | 1.00 | (mostly NULL keys) | – |

**Two takeaways:**

1. **Warranty is the only category that uses the multi-concern schema meaningfully.** All other
   ticket types have `query_heading_id` and `sub_query_id` mostly NULL on tqm — they're free-text
   issues with no structured tagging. So when you see "1.0 queries/ticket" for RC/Document/etc, it's
   because the structured field is unused, not because there's exactly one concern.
2. **For warranty, dedup before counting.** Raw row count inflates concerns by 8.8% on average due
   to "Add more concerns" re-writes. Use the DISTINCT pattern above for any analysis.

### Distinguishing new concerns vs query updates — the activity_log signal

`activity_log.activity` distinguishes the four intents:

| Activity | What it means | Actor | tqm side-effect |
|---|---|---|---|
| `Ticket Created` | Initial ticket creation | App / customer / agent | First tqm row(s) with original concern(s) |
| `Add more concerns` | Customer added/edited concerns via the app picker | Customer | Re-writes the full active set as new rows (causes 8.8% inflation) |
| **`Recategorization`** | **AI reclassification** of an existing concern | **Bot 696 (Circuit)** — 100% | New row with corrected tuple |
| **`Ticket Attributes Updated`** | **Manual reclassification** of an existing concern | **Human agents** — 100% | New row with corrected tuple |

⚠️ **Recategorization happens on two paths, both important.** Earlier we said `Recategorization` is
the recat signal — that's only the AI path. Manual recats are logged as `Ticket Attributes Updated`.

### AI vs Manual recategorization volume (Jan–May 2026)

| Month | AI (Circuit) | Manual (Agent) | AI % share |
|---|--:|--:|--:|
| Jan | 2,498 | 4,288 | 37% |
| Feb | 2,077 | 4,515 | 31% |
| Mar | 2,122 | 3,849 | 36% |
| Apr | 1,990 | 4,367 | 31% |
| May | 1,874 | 4,849 | **28%** |

~6,700 recategorizations per month — 30% AI, 70% manual. AI share is **declining** as manual volume
grows. Either humans are catching more mis-categorizations than the AI is, or the AI is being routed
fewer candidates.

### Data payload structure differs between the two paths

AI (`Recategorization`):
```json
{
  "prev_query": "Miscellaneous", "updated_query": "Warranty",
  "prev_sub_query": "My issue is not listed here", "updated_sub_query": "Gear or Clutch",
  "prev_query_heading": null, "updated_query_heading": "Hard to shift the gears"
}
```
Single-concern, scalar fields.

Manual (`Ticket Attributes Updated`):
```json
{
  "prev_query": "Warranty", "updated_query": "Warranty",
  "prev_sub_query": ["Air conditioning"], "updated_sub_query": ["Air conditioning","Interiors"],
  "is_warranty_update": true, "num_issues": 2,
  "prev_query_heading": ["My AC is not cooling..."], "updated_query_heading": [...]
}
```
Array-formatted (supports multi-concern updates), with `is_warranty_update` flag and `num_issues`
field. Use `is_warranty_update = true` to isolate warranty-specific manual recats.

### Recategorization flow patterns (May 2026)

**Where AI most often reclassifies:**
- Miscellaneous → Warranty (161) — catching warranty issues filed in catchall
- Spinny Benefits → Warranty (54)
- Document → RC (59)
- Miscellaneous → RC (40)
- (Many same-query refinements — e.g. RC → RC = 277, refining only the sub_query)

**Where humans most often reclassify:**
- Miscellaneous → Warranty (473) — agents catching the same pattern at 3× the AI's rate
- RSA → Warranty (271)
- Spinny Benefits → Warranty (92)
- Other → Warranty (68)
- RC - Seller → Payments - Seller (167)
- Seller-Miscellaneous → Payments - Seller (106)

⚠️ Same-query refinements (e.g. `prev_query = updated_query`) are common in both paths — they're
**sub-query reclassifications**, not top-level moves. Don't filter them out unless you're
specifically measuring top-level moves.

### Circuit's actual current role

Bot 696 (Circuit) was previously catalogued as a "legacy general-purpose router" whose volume
collapsed from 4,583/month (Sep '25) to 18/month (May '26) as a *first-touch* agent. It is **not
dead** — it has been **repurposed as the AI recategorizer**, doing ~1,874 recats per month.
Update its bot-inventory description accordingly: Circuit is now the post-creation
classification cleanup bot, not a general router.

### Warranty verdict columns — Canonical Mapping

Source-of-truth: `TicketWarrantyStatusChoices` / `TicketWarrantyReasonChoices` in the sp-ticket-master
Django app. Confirmed by Arpit Dhankani, 2026-05-28. **These codes are NOT in any lookup table in
Doris** — they are hardcoded enums in the application code.

**`ticket_query_mapping.warranty_status`** — per-query coverage verdict after evaluating warranty expiry:

| Code | Verdict | Volume (Apr–May '26) |
|---|---|--:|
| NULL | (Non-warranty query — no verdict applicable) | 160K rows |
| 1 | **Covered** | 8,130 rows / 5,298 tix |
| 2 | **Not covered** | 1,048 rows / 897 tix |
| 3 | **Coverage depends on inspection** | 4,071 rows / 2,850 tix |
| 4 | **Not covered under warranty** | 7,872 rows / 5,656 tix |

**Denial = `warranty_status IN (2, 4)`** — both codes mean "not covered" with a small semantic split
(2 is plain "not covered"; 4 is "not covered under warranty"). Aggregate them as the denial cohort
for most reporting.

**`ticket_query_mapping.warranty_status_reason`** — encodes `"<warranty_type> <active|expired>"`:

| Code | Reason |
|---|---|
| 0 | (None) |
| 1 | Comprehensive active |
| 2 | Comprehensive expired |
| 3 | Powertrain active |
| 4 | Powertrain expired |
| 5 | Key systems active |
| 6 | Key systems expired |
| 7 | Functional active |
| 8 | Functional expired |

Pairs naturally with `warranty_status`: a `2/3` row means "Not covered, Powertrain active" — i.e.
the customer's powertrain warranty is in force, but the specific issue they raised isn't covered
under it.

### How WarrantyAI (bot 998) actually works — confirmed via activity_log + cx_communication_messages

Sequence on a typical warranty ticket (real example, ticket 2229302, May 24 '26):

1. **Customer submits** a structured concern list via the help-and-support flow (HTML payload with
   itemised concerns like "Broken or damaged light(s)").
2. **WarrantyAI (`AGENT`)** responds **within seconds** with a per-item verdict + reasoning, e.g.:
   *"the issue(s) you have raised are not covered under your warranty plan. 1. Broken or damaged
   light(s) — Vehicle lights are exterior parts that usually get damaged due to outside impact or
   environmental factors rather than a manufacturing defect…"*
3. Bot auto-creates an indicator (`Indicator Created` activity, e.g. `CALLBACK_REQUESTED` or
   `CUSTOMER_MESSAGE`) and updates ticket status (`Status Updated` activity with
   `reason_for_action = "AI_resolved_no_response"` or `"Warranty Denial Bot - Inactivity"`).
4. **SYSTEM** schedules a callback window and sends standard messages.
5. If the customer doesn't respond, SYSTEM auto-closes with the boilerplate denial:
   *"The service you have requested is unfortunately not covered under the Spinny Warranty policy.
   Your ticket will now be closed."*
6. Customer can re-open by raising more concerns (`Add more concerns` activity).
7. If escalated, a **human AGENT** writes a personalised denial confirming the bot's verdict.

So the bot is doing the **first verdict + indicator + status update**; the rest is system
templates + optional human confirmation. The 47% CSAT covers the entire flow, not just the bot's
message.

**WarrantyAI verdict reason codes** (from `activity_log.data.reason_for_action`, May '26):
| Reason | Count | Meaning |
|---|--:|---|
| `AI_resolved_no_response` | 225 | Customer didn't reply → auto-closed |
| `Warranty Denial Bot - Inactivity` | 5 | Same idea, alt label |
| (empty) | 46 | No reason logged |

### Activity log — durable audit trail

`iceberg.sp_ticket_master.activity_log` is the canonical audit log. Top activity types (May '26):

| Activity | Volume | Notes |
|---|--:|---|
| Indicator Created / Removed | 545K | UI flags raised/cleared (CALLBACK_REQUESTED, UNREAD, BREACHED_TASKS, REMINDER, etc.) |
| Status Updated | 196K | Every status change. `data` JSON has `new_state`, `previous_state`, `reason_for_action` |
| Task Created / Closed / Assigned / Reassigned | ~175K | Task lifecycle |
| Task Deadline Updated / Breached | ~190K | SLA tracking |
| Ticket Prioritised / Deprioritised | ~85K | Priority changes |
| **Reassigned to Human from AI** | **1,348** | **Direct AI escalation metric** — count this to measure bot deflection failure |
| **Assigned to AI Agent** | **1,129** | Bot first-assignment events (independent of `first_agent_assigned_id`) |
| Add more concerns | 1,031 | Customer appending more queries to existing ticket — drives warranty multi-query |
| Recategorization | 1,866 | Ticket query reclassified |
| Permanent / Temporary Remapping | 637 | Ownership changes |

`activity_log.data` is a JSON string — use `GET_JSON_STRING(data, '$.field')` to extract fields.

**Join chain to ticket-level data:**
```sql
activity_log al
  JOIN ticket t ON t.id = al.object_id           -- object_id = ticket id
  -- al.user_id = the user (or bot) who performed the action
  -- al.activity = event name (see table above)
  -- al.data = JSON payload
```

---

### `sp_entity_store_intermediate.cx_*` — Intermediate Tables

Pre-aggregated per-ticket, used internally to build `support_tickets`. Useful for joins and point-in-time status.

| Table | Key columns | Use for |
|---|---|---|
| `cx_ticket_meta` | `ticket_id`, `created_at`, `closed_at`, `resolved_at`, `sub_query_id`, `query_heading_id`, `ticket_priority`, `customer_type`, `ticket_source`, `ticket_status`, `cx_rating`, `closing_reason`, `permanent_kam_id/email` | Ticket metadata without the full denorm |
| `cx_ticket_status_logs` | `ticket_id`, `first_closing_time`, `latest_closing_time`, `first_resolved_time`, `first_reopened_time`, `status_updation_logs` (array) | Reopen analysis, FCR tracking |
| `cx_ticket_lead_mapping` | `ticket_id`, `lead_id` | **Bridge: ticket → sell_lead** (supply tickets) |
| `cx_ticket_deal_mapping` | `ticket_id`, `dealer_id`, `deal_id`, `buy_lead_id`, `deal_hub_id/name/city` | **Bridge: ticket → deal** (buyer/dealer tickets) |
| `cx_ticket_call_mapping` | `call_id`, `ticket_id`, `call_start/end_time`, `call_type`, `call_status`, `agent_email`, `campaign_name`, `operator_name`, `call_weightage` | Per-call analysis against tickets |
| `cx_message_tickets` | `ticket_id`, aggregated message counts by type (customer/agent/AI/system), first/latest message timestamps, voice note counts, media counts, `call_id_purpose_mapping` (array) | Message volume / response time analysis |
| `cx_ticket_ai_summary` | `ticket_id`, `ticket_summary`, `ticket_summary_time` | AI-generated summaries |
| `auc_tickets` | Auction-domain tickets | Tickets related to auction process |
| `des_tickets` | Delivery-related tickets | Tickets related to delivery/dispatch |

**Standard join: ticket → supply lead**
```sql
JOIN sp_entity_store_intermediate.cx_ticket_lead_mapping m ON m.ticket_id = t.ticket_id
JOIN sp_entity_store.sell_leads sl ON sl.lead_id = m.lead_id
```

**Standard join: ticket → messages (SQLMesh versioned)**
```sql
-- Use the latest hash (verify with SHOW TABLES LIKE 'cx_ticket_lead_mapping%' in sp_sqlmesh_onedata)
sp_sqlmesh_onedata.cx_ticket_lead_mapping__3214137628  -- ticket_id → lead_id bridge
sp_sqlmesh_onedata.cx_communication_messages__2814946861  -- message content
-- Key fields in cx_communication_messages: text, is_customer, call_purpose,
-- call_action_items, call_discussion_points, call_customer_queries (AI summary fields)
```

---

### `sp_product.cx_tickets` — Analytics Layer (Dashboard Source)

Pre-computed metrics table — likely the source for the CX Master Dashboard on Superset.

Key computed columns (confirmed):

| Column | Type | Notes |
|---|---|---|
| `ticket_id` | varchar | Note: varchar here, int in entity store |
| `query` / `sub_query` / `query_heading` | text | Category hierarchy |
| `fcr_flag` | text | First Call Resolution flag |
| `is_circuit_resolved` | tinyint | Whether resolved at circuit level |
| `is_reopened` | tinyint | Reopened flag |
| `is_dropped` | tinyint | Dropped/abandoned flag |
| `csat_submitted` | tinyint | 1 = CSAT survey was submitted |
| `is_csat_positive` | tinyint | 1 = positive CSAT |
| `resolution_time_hrs` | decimal | TAT in hours (creation → resolution) |
| `pendency_hours` | decimal | Hours ticket was pending |
| `frt1` | bigint | First Response Time (minutes to first agent response) |
| `num_tasks` / `num_messages` / `num_calls` | bigint | Engagement counts |
| `connected_calls` | bigint | ⚠️ **Misleading name** — see Call Measurement below |

---

### Call Measurement — `cx_tickets` vs `cx_communication_messages`

> **Confirmed via ticket-level cross-validation, June 2026.**

#### `cx_tickets` fields (pre-aggregated on ticket)
- **`num_calls`** = count of `CALL_DETAILS` rows in `cx_communication_messages` for that ticket. Includes both inbound + outbound. Reliable for total call attempts.
- **`connected_calls`** = ⚠️ **NOT connected phone calls.** This field counts total customer interactions/touchpoints (calls + messages + events), NOT connected phone calls. Evidence:
  - 20.5% of Spinny Benefits tickets have `connected_calls > 0` but `num_calls = 0`
  - 39.1% of Spinny Benefits tickets have `connected_calls > num_calls`
  - 24.1% of Warranty tickets have `connected_calls > num_calls`
  - **Do not use this field for call connect rate analysis.**

#### `cx_communication_messages` — the correct source for call analysis
Table: `sp_entity_store_intermediate.cx_communication_messages`

Filter: `message_type = 'CALL_DETAILS'` and `meta_data IS NOT NULL AND meta_data != ''`

##### Dual JSON Schema (CRITICAL)
Two different JSON schemas exist in `meta_data` for CALL_DETAILS records. **Both must be handled** — using only one misses ~25% of call records.

| | Schema A (~279K records) | Schema B (~75K records) |
|---|---|---|
| Call direction | `$.type` | `$.callback_data.callType` |
| Call outcome | `$.status` | `$.callback_data.callResult` |
| OB value | `outbound.manual.dial` | `outbound.manual.dial` |
| Connected value | `CONNECTED` | `SUCCESS` |
| IB value | `inbound.call.dial` | _(no IB records in Schema B)_ |
| Failed values | `NO_ANSWER`, `BUSY`, etc. | `FAILURE` |

~871 records match neither schema — excluded from all counts.

**How to detect**: If `get_json_string(meta_data, '$.type')` returns a call direction value → Schema A. If `get_json_string(meta_data, '$.callback_data.callType')` returns a value → Schema B.

Additional Schema A fields:
- **`$.duration`** — call duration (e.g. `"00:02:29.895"`)
- **`$.hangup_by`** — who ended call (`AgentHangup` / `UserHangup`)
- **`$.status`** values: `CONNECTED`, `NO_ANSWER`, `BUSY`, `CALL_NOT_PICKED`, `CALL_HANGUP`, `CALL_DROP`, `FAILED`, `PROVIDER_FAILURE`, `ATTEMPT_FAILED`, `NUMBER_FAILURE`, `PROVIDER_TEMP_FAILURE`

##### Correct OB call counting logic (dual-schema)
```sql
-- OB attempt (either schema):
SUM(CASE
  WHEN get_json_string(meta_data, '$.type') = 'outbound.manual.dial' THEN 1
  WHEN get_json_string(meta_data, '$.callback_data.callType') = 'outbound.manual.dial' THEN 1
  ELSE 0
END) AS ob_attempts

-- OB connected (either schema):
SUM(CASE
  WHEN get_json_string(meta_data, '$.type') = 'outbound.manual.dial'
       AND get_json_string(meta_data, '$.status') = 'CONNECTED' THEN 1
  WHEN get_json_string(meta_data, '$.callback_data.callType') = 'outbound.manual.dial'
       AND get_json_string(meta_data, '$.callback_data.callResult') = 'SUCCESS' THEN 1
  ELSE 0
END) AS ob_connected
```

**Note on IB calls**: Inbound calls (`$.type = 'inbound.call.dial'`) almost never have `status = CONNECTED` in meta_data — they use a different status tracking mechanism. All IB rows can be treated as connected. However, for FCR and call distribution analysis, **only OB calls are counted** (IB excluded from attempts, connected, and distribution buckets).

##### Ticket Status — Closed Definition
A ticket is considered **closed** if either:
- `is_closed = 1` — explicitly closed/resolved/dropped
- `ever_merged_flag = 1` — merged into another ticket (treated as closed)

Status combinations in `cx_tickets`:

| is_closed | ever_merged_flag | is_dropped | pending_flag | Effective Status |
|---:|---:|---:|---:|---|
| 1 | 0 | 0 | 0 | Closed |
| 0 | 1 | 0 | 0 | Merged (→ treat as closed) |
| 1 | 0 | 1 | 0 | Dropped (closed) |
| 0 | 0 | 0 | 1 | Pending (genuinely open) |

April 2026: 98.6% of tickets are closed (including merged). Only 354 genuinely pending.

##### FCR Definition
**FCR (First Call Resolution)** = ticket has exactly 1 OB connected call AND is closed (is_closed=1 OR ever_merged_flag=1).

```sql
-- FCR%:
ROUND(
  SUM(CASE WHEN ob_connected = 1 AND (is_closed = 1 OR ever_merged_flag = 1) THEN 1 ELSE 0 END)
  * 100.0 / COUNT(*),
1) AS fcr_pct
```

Why this definition:
- The DB `fcr_flag` field is based on internal CX rules, not call-based
- Open tickets with 1 OB connected call should NOT count as FCR — the issue isn't resolved yet
- Merged tickets should count — the issue was resolved via the parent ticket

##### OB Connected Call Distribution (0/1/2/3/3+)
Tickets are bucketed by their per-ticket OB connected call count. Within each bucket, closure% = % of tickets in that bucket that are closed (is_closed=1 OR ever_merged_flag=1).

##### Benchmarks (April 2026, OB only)
- **OB connect rate**: 69.9% (62K connected / 89K OB attempts)
- **Avg OB per ticket**: 3.6 attempts, 2.5 connected
- **Avg OB to close**: 3.4 attempts per closed ticket
- **FCR**: 33.1% (1 OB connected & closed)
- **0 OB conn tickets**: 22.5% (99.9% closed — mostly chat-resolved or merged)
- **3+ OB conn tickets**: 19.2% (94.0% closed — chronic repeat-contact)

##### Query-level OB distribution with merge % (April 2026, all tickets incl. merged)

Merged tickets concentrate heavily in the 0 OB connected bucket. The `(M%)` in each cell shows what
share of that bucket is merged tickets — essential context when interpreting "0 OB connected" rates.

| Query | Tickets | Closed % | FCR % | App % | OB Att | OB Conn | 0 (M%) | 1 (M%) | 2 (M%) | 3 (M%) | 3+ |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| **Warranty** | 7,464 | 99.7 | 25.6 | 66.8 | 36,067 | 26,537 | 1,947 (83%) | 1,586 (12%) | 1,051 (4%) | 664 (2%) | 2,216 |
| **Payments-Seller** | 3,493 | 99.8 | 39.4 | 64.6 | 8,649 | 5,753 | 698 (55%) | 1,227 (5%) | 1,003 (0%) | 303 (0%) | 262 |
| **RC** | 2,780 | 95.5 | 42.7 | 73.3 | 9,904 | 6,529 | 558 (30%) | 1,028 (8%) | 398 (2%) | 223 (0%) | 573 |
| **RSA** | 1,924 | 100.0 | 61.8 | 0.4 | 2,754 | 1,710 | 675 (67%) | 908 (8%) | 255 (4%) | 57 (2%) | 29 |
| **RC-Seller** | 1,783 | 90.1 | 51.9 | 61.6 | 5,821 | 3,696 | 339 (43%) | 735 (6%) | 279 (2%) | 133 (1%) | 297 |
| **Insurance** | 1,274 | 99.6 | 49.1 | 61.9 | 4,007 | 2,873 | 225 (52%) | 579 (7%) | 186 (1%) | 105 (2%) | 179 |
| **Seller-Misc** | 1,027 | 99.5 | 47.8 | 52.5 | 2,953 | 1,936 | 158 (37%) | 434 (4%) | 188 (1%) | 103 (0%) | 144 |
| **Spinny Benefits** | 1,004 | 99.5 | 11.3 | 76.8 | 6,260 | 4,599 | 211 (92%) | 191 (53%) | 92 (16%) | 81 (5%) | 429 |
| **Payments** | 900 | 99.8 | 65.4 | 62.1 | 1,842 | 1,253 | 237 (33%) | 454 (2%) | 91 (1%) | 48 (2%) | 70 |
| **Miscellaneous** | 834 | 99.5 | 42.1 | 66.1 | 2,271 | 1,606 | 144 (50%) | 338 (5%) | 158 (2%) | 84 (1%) | 110 |
| **Fastag** | 689 | 99.7 | 40.5 | 74.9 | 2,112 | 1,445 | 96 (44%) | 288 (5%) | 103 (2%) | 87 (0%) | 115 |
| **Other** | 581 | 100.0 | 35.3 | 0.5 | 1,664 | 949 | 171 (34%) | 154 (6%) | 126 (1%) | 69 (0%) | 61 |
| **HSRP** | 392 | 99.5 | 12.0 | 72.2 | 1,711 | 1,276 | 34 (53%) | 67 (19%) | 83 (4%) | 58 (0%) | 150 |
| **Document** | 350 | 100.0 | 38.0 | 84.6 | 859 | 604 | 60 (38%) | 131 (2%) | 85 (0%) | 39 (0%) | 35 |

**Key patterns:**
- **Merged tickets are overwhelmingly in the 0-conn bucket** — Warranty 83%, Spinny Benefits 92%, RSA 67%. Most "0 OB connected" is not a service gap but ticket merges.
- **Spinny Benefits has unusually high merge rates even in 1-conn (53%) and 2-conn (16%)** — suggests aggressive merging of duplicate benefit requests even after calls are made.
- **Merge rates drop to <10% in the 1+ OB connected buckets** for most queries, confirming merged tickets rarely have outbound calls made before absorption.
- **RSA is 99.6% non-app** (0.4% app) — almost all RSA tickets originate from calls/other channels, unlike most other queries (60–85% app).
- **Highest FCR:** Payments (65.4%), RSA (61.8%) — simpler, single-call resolution queries.
- **Lowest FCR:** Spinny Benefits (11.3%), HSRP (12.0%) — multi-touch, process-heavy queries requiring 3+ calls.
- **Most call-heavy:** Spinny Benefits (avg 6.2 OB att/ticket), Warranty (avg 4.8 OB att/ticket).

---

##### 0 OB Connected Deep Dive — Why Tickets Close Without a Connected Call (April 2026)

**Total 0-OB-connected tickets: 5,684** (2,250 non-merged + 3,434 merged). Analysis below covers
non-merged tickets (the real closure patterns) and merged tickets (duplicate-creation patterns).

###### All Queries — Non-Merged: 2,250 tickets across 11 patterns

| # | Bucket | Tickets | % | Avg CX Msgs | Avg Agent Msgs | Avg AI Msgs | Avg Sys Msgs | Top Queries |
|---|---|---:|---:|---:|---:|---:|---:|---|
| 1 | **Customer Unreachable** (Unresponsive / Cx not connected / 3 call attempts exhausted) | 348 | 15.5 | 1.7 | 0.6 | 0.1 | 2.9 | RSA (120), Payments-Seller (41), Warranty (33), RC (32) |
| 2 | **Paused → Auto-Close** | 302 | 13.4 | 2.1 | 0.8 | 0.2 | 7.2 | Warranty (65), RC (51), Other (47), Payments-Seller (42) |
| 3 | **Payment Released** (OTP/PP/NOC/Fastag/RC Card hold) | 290 | 12.9 | 2.5 | 0.9 | 0.1 | 2.5 | Payments-Seller (201), Other (45), Payments (19) |
| 4 | **RC Process** (RC Transferred / Dispatched / TAT Shared / couriered) | 282 | 12.5 | 2.1 | 1.2 | 0.2 | 2.6 | RC (142), RC-Seller (140) |
| 5 | **Others** (uncategorized) | 249 | 11.1 | 2.1 | 1.1 | 0.1 | 2.0 | Payments (81), Misc (38), RC-Seller (25) |
| 6 | **Bot Auto-Denial** (NULL closing reason, bot first-touch 981/998) | 187 | 8.3 | 2.5 | 0.0 | 3.6 | 0.0 | RC (119), Warranty (68) |
| 7 | **Info Shared** (Explanation / TAT / Docs / Fastag / Insurance) | 172 | 7.6 | 1.9 | 0.8 | 0.1 | 1.9 | Payments (48), Insurance (48), Payments-Seller (39) |
| 8 | **Resolved** (on call / on chat / Issue Resolved) | 139 | 6.2 | 2.5 | 1.3 | 0.3 | 2.4 | Warranty (58), Seller-Misc (46), RC (11) |
| 9 | **Denial / Not Needed** (Out of Coverage / Cx did not require RSA / Policy Expired / Not eligible) | 102 | 4.5 | 1.7 | 0.5 | 0.1 | 1.7 | RSA (70), Warranty (24) |
| 10 | **Scheduling Exhausted** (3 scheduling attempts) | 62 | 2.8 | 2.7 | 0.7 | 0.4 | 3.2 | Warranty (62 — 100%) |
| 11 | **Repair Completed** (workshop / car delivered / car picked) | 15 | 0.7 | 3.3 | 1.0 | 0.0 | 8.1 | Warranty (15 — 100%) |

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

**Pattern descriptions (confirmed by reading communications_metadata on sample tickets):**

1. **Customer Unreachable:** Agent made 2-3 OB attempts, none connected. Internal notes: *"Unresponsive / RNR / No. switch off"*. System sends "We attempted to contact you" after each fail. Closed without resolution.
2. **Paused → Auto-Close:** Same unreachability, but goes through the formal pause flow: ticket put on hold with a resume deadline → system sends reminder → customer doesn't resume → auto-closed. Avg 7.2 system messages from the pause chain.
3. **Payment Released:** Agent processes hold-amount release as a **back-office action** (no call needed), then informs customer via chat with payment screenshot + UTR: *"Hold amount has been released. Kindly check."*
4. **RC Process:** Agent shares RC transfer status, courier tracking ID, or TAT commitment via chat. No call required for status updates.
5. **Bot Auto-Denial:** AI bot (981=RC Agent, 998=WarrantyAI) handles end-to-end. Bot reads concern, issues verdict, customer doesn't reply → auto-closes. **0 agent messages, 3.6 avg AI messages.** No human ever touches.
6. **Info Shared:** Agent shares explanation, document, or TAT via chat — informational resolution. Templated responses.
7. **Resolved:** "Resolved on call" tickets often used **personal/office phones** not tracked in CRM (agent notes: *"call done by social media office phone"*). "Resolved on chat" = fully resolved via messaging.
8. **Denial / Not Needed:** Warranty expired, out of coverage, or customer didn't actually need the service (RSA: *"Cx did not require RSA"*).
9. **Scheduling Exhausted:** Warranty-only. Agent tried to schedule inspection/workshop visit 3 times, customer didn't respond to scheduling requests.
10. **Repair Completed:** Workshop flow completed via system messages (inspection → repair → delivery). Process-driven, no OB call needed.

**All-query automation map:**

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

###### Payments-Seller Deep Dive — 0 OB Connected (April 2026)

**Total: 735 tickets** (353 non-merged + 382 merged)

**Non-merged: 353 tickets — 6 closure patterns:**

| # | Bucket | Tickets | % of 353 | Closing Reasons | Avg CX Msgs | Avg Agent | Avg Sys |
|---|---|---:|---:|---|---:|---:|---:|
| 1 | **Payment Released** | 201 | 56.9 | OTP Amount Released (118), PP Amount Released (60), Bank NOC Hold (15), Fastag Hold (4), RC Card (3), Other Hold (1) | 2.9 | 1.0 | 2.7 |
| 2 | **Paused → Auto-Close** | 42 | 11.9 | Paused Ticket Automatic Closure | 2.9 | 1.1 | 7.3 |
| 3 | **Customer Unreachable** | 41 | 11.6 | Unresponsive (40), Cx not connected (1) | 2.1 | 0.7 | 3.7 |
| 4 | **Info/TAT Shared** | 40 | 11.3 | TAT Shared (39), Deduction Informed (1) | 2.4 | 1.1 | 2.3 |
| 5 | **Others** | 21 | 5.9 | Others | 3.0 | 1.2 | 2.0 |
| 6 | **Denial** | 8 | 2.3 | Not eligible | 2.1 | 0.8 | 1.5 |

**Sample ticket IDs per bucket:**

| Bucket | Sample Tickets |
|---|---|
| Payment Released | `2057630`, `2057949`, `2058575`, `2058655`, `2058683` |
| Paused → Auto-Close | `2058557`, `2060871`, `2061804`, `2082176`, `2087481` |
| Customer Unreachable | `2058886`, `2059508`, `2061811`, `2063313`, `2064691` |
| Info/TAT Shared | `2059543`, `2059577`, `2060464`, `2061596`, `2062392` |
| Others | `2067025`, `2067548`, `2079143`, `2087851`, `2088985` |
| Denial | `2059963`, `2067529`, `2079583`, `2099232`, `2108573` |

**Merged: 382 tickets — duplicate creation patterns:**

382 merged into 306 unique parents (avg 1.2 children/parent). **94% same customer** — genuine
duplicates. Root cause: seller re-raises because they don't see progress on their existing ticket.

**Time gap from parent creation to merged child:**

| Gap | Tickets | % | App | Web | VOC | Manual |
|---|---:|---:|---:|---:|---:|---:|
| < 5 min | 137 | 35.9 | 85 | 13 | 34 | 5 |
| 5–60 min | 74 | 19.4 | 24 | 40 | 9 | 1 |
| 1–24 hrs | 81 | 21.2 | 39 | 24 | 12 | 6 |
| > 24 hrs | 90 | 23.6 | 46 | 15 | 25 | 4 |

**Sample merged tickets by time gap:**

| Gap | Merged Ticket | Parent Ticket | Source | Gap (min) |
|---|---|---|---|---:|
| < 5 min | `2057930` | `2057923` | App | 3 |
| < 5 min | `2058156` | `2058155` | VOC | 0 |
| < 5 min | `2058521` | `2058519` | App | 0 |
| < 5 min | `2058706` | `2058720` | Web | -3 |
| < 5 min | `2059167` | `2059180` | App | -3 |
| 5–60 min | `2058266` | `2058044` | App | 10 |
| 5–60 min | `2061449` | `2061374` | App | 16 |
| 5–60 min | `2061803` | `2061770` | Web | 10 |
| 5–60 min | `2061890` | `2061869` | Web | 6 |
| 5–60 min | `2063341` | `2063335` | App | 21 |
| 1–24 hrs | `2058038` | `2055474` | VOC | 1392 |
| 1–24 hrs | `2058280` | `2055474` | App | 1410 |
| 1–24 hrs | `2059547` | `2057928` | Manual | 435 |
| 1–24 hrs | `2062094` | `2060916` | Web | 291 |
| 1–24 hrs | `2063561` | `2063246` | Web | 753 |
| > 24 hrs | `2060945` | `2055474` | App | 2829 |
| > 24 hrs | `2062156` | `2044719` | App | 8501 |
| > 24 hrs | `2063617` | `2059315` | VOC | 2580 |
| > 24 hrs | `2063887` | `2060822` | App | 1540 |
| > 24 hrs | `2064174` | `2057815` | VOC | 3101 |

**Recategorization on merged tickets:** 35/382 (9.2%) were recategorized before merge — 21 by AI
(Circuit), 14 by human agents. Most common flows: Payments-Seller sub-query refinement (*"My issue
is not listed here"* → *"Refund of hold amount"*), and cross-query misroutes (RC-Seller → Payments-Seller,
Seller-Misc → Payments-Seller). This is wasted effort — the ticket is about to be merged.

**Payments-Seller automation map (combined non-merged + merged):**

| # | Bucket | Tickets | % of 735 | Automation Potential | Key Lever |
|---|---|---:|---:|---|---|
| 1 | Payment Released | 201 | 27.3 | 🔴 **Very High** | Auto-check RC status → auto-release → auto-notify with UTR |
| 2 | Merged < 5 min | 137 | 18.6 | 🔴 **Very High** | App-level duplicate prevention — block if open ticket exists for same sell_lead |
| 3 | Merged > 24 hrs | 90 | 12.2 | 🔴 **High** | Proactive milestone notifications so seller doesn't re-raise |
| 4 | Merged 1–24 hrs | 81 | 11.0 | 🟡 Medium | Auto-status update: *"We're working on it, ETA: X days"* |
| 5 | Merged 5–60 min | 74 | 10.1 | 🔴 **Very High** | Same as #2 — app/web should prevent |
| 6 | Paused → Auto-Close | 42 | 5.7 | 🟢 Partial | Closure is auto; reduce pause-to-close cycle |
| 7 | Customer Unreachable | 41 | 5.6 | 🟡 Medium | Deflect to WhatsApp/chat after 1st failed OB |
| 8 | Info/TAT Shared | 40 | 5.4 | 🔴 **High** | Bot auto-reply with TAT based on agreement date + process stage |
| 9 | Others | 21 | 2.9 | ⚪ Unknown | Better reason tagging needed |
| 10 | Denial | 8 | 1.1 | 🟡 Medium | Eligibility check at ticket creation |

**~570 of 735 tickets (78%) are automatable:** 201 payment releases (auto-verify + auto-release +
auto-notify), 292 merged duplicates (app duplicate prevention + proactive status updates), 40 TAT/info
shares (bot auto-reply), ~40 from Paused/Unreachable (channel deflection to chat/WhatsApp).

---

###### Warranty Deep Dive — 0 OB Connected, Non-Merged (April 2026)

**Total 0-OB-connected warranty tickets: 1,947** (331 non-merged + 1,616 merged (83%))

**Non-merged: 331 tickets — 6 closure patterns:**

| # | Bucket | Tickets | % of 331 | What Happens |
|---|---|---:|---:|---|
| 1 | **WarrantyAI Bot auto-denial** | 68 | 20.5 | Bot 998 handles E2E. Issues verdict, customer doesn't reply → auto-closes. 0 agent msgs, 3.4 avg AI msgs. |
| 2 | **Paused → Auto-Close** | 65 | 19.6 | Agent attempts OB, can't connect → ticket paused → auto-closed. Avg 7.8 sys msgs. |
| 3 | **Scheduling Exhausted** | 62 | 18.7 | 3 failed scheduling attempts for inspection/workshop visit → auto-closed. |
| 4 | **Resolved on call** | 47 | 14.2 | Resolved via personal/office phones (not tracked in CRM) or IB calls. Misleading "0 OB connected". |
| 5 | **3 call attempts exhausted** | 33 | 10.0 | 3 OB attempts, none connected → closed. Avg 5.6 sys msgs. |
| 6 | **Operational closures** | 56 | 16.9 | Out of Coverage (16), Resolved on chat (11), Car repaired & delivered/picked (13), Policy Expired (7), Reimbursement Done (5), Workshop completed (2), others (2). |

**Sample tickets per bucket:**

| Bucket | Sample Tickets |
|---|---|
| WarrantyAI Bot auto-denial | `2119943`, `2119944`, `2120055` (+ any from `first_agent_assigned_id = 998` with NULL closing reason) |
| Paused → Auto-Close | `2057669`, `2058413`, `2058591`, `2059244`, `2060589` |
| Scheduling Exhausted | `2057837`, `2061196`, `2065043`, `2065102`, `2065935` |
| Resolved on call | `2059221`, `2063892`, `2067674`, `2069252`, `2069824` |
| 3 call attempts exhausted | `2059170`, `2062160`, `2062222`, `2066528`, `2066910` |
| Operational closures | `2059578` (Car repaired), `2061566` (Out of Coverage), `2063991` (Resolved on chat), `2067449` (Resolved on chat), `2074351` (Out of Coverage) |

**Warranty automation map:**

| Bucket | Tickets/mo | Already Automated? | Automation Potential |
|---|---:|---|---|
| WarrantyAI Bot auto-denial | 68 | ✅ Yes | Already E2E bot-handled |
| Paused → Auto-Close | 65 | ⚠️ Partial | 🟡 Closure is auto; scale it |
| Scheduling Exhausted | 62 | ❌ | 🔴 **High** — self-serve scheduling via app/WhatsApp |
| Resolved on call (off-CRM) | 47 | ❌ | 🟡 Low — CRM call tracking fix, not automation |
| 3 call attempts exhausted | 33 | ⚠️ Partial | 🟡 Medium — deflect to chat/WhatsApp after 1st fail |
| Operational closures | 56 | Mixed | 🟡 Medium — denial/expiry checks can be bot-driven |

| `AOH_creation` | text | After office hours flag at creation |
| `car_category` | text | Car category |
| `city` | text | City |
| `first_user` / `latest_user` | text | First and latest assigned agents |
| `same_month_closed_flag` | tinyint | Closed in same calendar month as creation |
| `pending_flag` | tinyint | Currently pending |

---

## Enum Values (Confirmed via Live Queries, Jan–May 2025)

### `query_category_name`
| Value | Description |
|---|---|
| `FEEDBACK` | Customer feedback / NPS (largest volume: ~680K tickets since 2025) |
| `SUPPORT` | General support queries |
| `AUCTION` | Auction-related queries (dealer-side) |
| `WARRANTY` | Warranty / EW claims |
| `SERVICING` | Servicing requests |
| `SOCIAL_MEDIA` | Social media complaints/mentions |

### `ticket_status_name`
| Value | Notes |
|---|---|
| `RESOLVED` | Resolved (largest bucket) |
| `NEW` | Just created, unassigned |
| `IN_PROGRESS` | Agent working on it |
| `MERGED` | Merged into another ticket |
| `ATTEMPT_DONE` | Attempt made, awaiting response |
| `AWAITING_CUSTOMER_RESPONSE` | Ball in customer's court |
| `CLOSED` | Closed (distinct from RESOLVED) |
| `PAUSED` | On hold |
| `REFURB_IN_PROGRESS` | Car in refurb pipeline (CX tracking) |
| `REFURB_COMPLETED` | Refurb done |
| `CAR_IS_WITH_CX` | Physical car with CX team |
| `IN_PROCESS` | Generic in-process state |
| `REPAIR_UNDER_PROCESS` | Repair ongoing |
| `INSPECTION_REQUIRED` | Requires inspection |
| `INSPECTION_SCHEDULED` | Inspection scheduled |
| `REOPENED` | Reopened after close/resolve |
| `ISSUE_RESOLVED` | Issue resolved (separate from RESOLVED) |
| `CAR_RETURNED` | Car returned to customer |

### `ticket_source_name`
| Value | Notes |
|---|---|
| `FEEDBACK` | Automated feedback collection |
| `SPINNY_HELP_AND_SUPPORT` | In-app help flow |
| `DEALER_APP` | Raised from dealer app |
| `PHONE_CALL` | Inbound phone call |
| `CREATED_MANUALLY` | Agent-created manually |
| `WEBSITE_INBOUND` | Website contact form |
| `VOC` | Voice of Customer |
| `RSA` | Roadside Assistance |
| `SERVICING_SUPPORT` | Servicing-specific support |
| `DEMAND_INTERNAL` | Internal demand team tickets |
| `CONTACT_US` | Contact Us form |
| `SP_INSURANCE` | Insurance-related |
| `DEALER_CRM` | From dealer CRM |

### `customer_type_name`
`BUYER` · `SELLER` · `DEALER` · `OTHER`

### `ticket_priority_code`
`FAIR` · `SEVERE` · `POOR` · `MEDIOCRE` · `NEUTRAL`
(Severity ranking: SEVERE > POOR > MEDIOCRE > FAIR > NEUTRAL — not confirmed, verify against product definition)

### Top `ticket_closing_reason_name` values ⚠️ STALE — names shown here are from old entity store snapshot; see Known Trap #8 for correct names
`Response not received` · `Others` · `3 call attempts exhausted` · `REFUND_COMPLETED_BY_SALES` · `Insurance correction denied` · `Document correction` · `3 attempts exhausted` · `RC TAT Shared` · `Unresponsive` · `Dropped` · `RC Transferred` · `DEALER_FINANCING_APPROVED` · `Items not found` · `Positive review` · `TAT Shared` · `Issue resolved at DSI` · `Explanation Shared` · `Delivery Note Shared`

### `lead_category_name` (in support_tickets)
`assured` · `auction` · `budget` · `luxury` · `bcm` · null (~50% of tickets have no linked lead)

---

## Known Traps

**1. `*_calls_count` columns store FRACTIONS, not integers.**
Same gotcha as `supply_workflow_tasks`. Columns like `total_outbound_calls_count` in `support_tickets` store rate-like values in 0.0–1.0 range, not integer call counts. Use `first_connected_outbound_call_start_ts IS NOT NULL` to detect "at least one connected call."

**2. `ticket_id` type mismatch across layers.**
- `sp_entity_store.support_tickets.ticket_id` → **int**
- `sp_product.cx_tickets.ticket_id` → **varchar**
Cast when joining: `CAST(cx.ticket_id AS INT) = st.ticket_id`

**3. `iceberg.sp_ticket_master` is UTC; entity store is IST.**
All `iceberg.sp_ticket_master.*` timestamps have `WITH_TIMEZONE` (stored UTC). `sp_entity_store.support_tickets` timestamps are IST. Add +330 minutes when joining across layers.

**4. FEEDBACK category ≠ CSAT.**
`query_category_name = 'FEEDBACK'` (~45% of all tickets) is an automated collection bucket, not a manually raised support issue. Always filter it out when measuring support volume or agent workload: `WHERE query_category_name != 'FEEDBACK'`.

**5. MERGED tickets are counted in raw volume but shouldn't be counted as independent tickets.**
`ticket_status_name = 'MERGED'` means the ticket was absorbed into another. Exclude with `WHERE ticket_status_name != 'MERGED'` for unique ticket counts.

**6. `lead_category_name` is ~50% null.**
Tickets are not always linked to a sell/buy lead. Don't assume a ticket has a lead. Use `sell_lead_id IS NOT NULL` / `buy_lead_id IS NOT NULL` to filter to lead-linked tickets only.

**8. `ticket_closing_reason_name` in `support_tickets` is stale — use `ticket_reason` table.**
`ticket_closing_reason` is the old lookup table; the reason now lives in `iceberg.sp_ticket_master.ticket_reason`.
The entity store denormalized the name from a stale snapshot — many high-volume reasons are now **wrong**:

| `ticket_closing_reason_id` | Name in `support_tickets` ❌ | Actual `display_name` in `ticket_reason` ✓ |
|---|---|---|
| 272 | Items not found | Paused Ticket Automatic Closure |
| 48 | 3 call attempts exhausted | Resolved on call |
| 71 | TAT Shared | OTP Amount Released |
| 39 | RSA provided | Cx did not require RSA |
| 70 | Document correction | TAT Shared |
| 244 | DEALER_FINANCING_APPROVED | Out of Coverage |
| 72 | OTP Amount Released | PP Amount Released |

**Correct pattern** — always fetch the reason name from `ticket_reason` directly:
```sql
-- From iceberg (raw)
SELECT t.id AS ticket_id, tr.display_name AS closing_reason
FROM iceberg.sp_ticket_master.ticket t
JOIN iceberg.sp_ticket_master.ticket_reason tr
    ON t.closing_reason_id = tr.id
WHERE tr.category = 'TICKET_CLOSING'

-- From entity store (use id for join, get name from ticket_reason)
SELECT st.ticket_id, tr.display_name AS closing_reason
FROM sp_entity_store.support_tickets st
JOIN iceberg.sp_ticket_master.ticket_reason tr
    ON st.ticket_closing_reason_id = tr.id
WHERE tr.category = 'TICKET_CLOSING'
  AND st.service_unit_id = 1
  AND st.ticket_merged_ts IS NULL
```

⚠️ **Entity store team needs to re-ETL `ticket_closing_reason_name`** from `ticket_reason.display_name` — flagged 2026-06-01.

**7. SQLMesh versioned table hashes go stale.**
`sp_sqlmesh_onedata.cx_ticket_lead_mapping__<hash>` — the hash suffix rotates. Before relying on a hardcoded hash, run:
```sql
SHOW TABLES LIKE 'cx_ticket_lead_mapping%' -- in sp_sqlmesh_onedata
```
Pick the table with the latest `event_date` MAX.

---

## Metrics — Canonical Definitions

**Source-of-truth:** all definitions below are extracted directly from the CX Master Dashboard
(Superset ID 126) charts: "Overview dashboard" (448), "last 30 days" (449), "Reporting numbers"
(463), and the KPI tiles (474–478). Dataset: virtual table `cx business` (Superset dataset 206)
which is `sp_product.cx_tickets cxt LEFT JOIN sp_entity_store.support_tickets st ON cxt.ticket_id = st.ticket_id`.

### Unique tickets (denominator for almost everything)
```sql
COUNT(CASE WHEN ever_merged_flag = 0 THEN ticket_id END)
```
Always exclude merged tickets — they were absorbed into another ticket. % merged typically ~30–40%.

### Closed tickets
```sql
COUNT(CASE WHEN is_closed = 1 AND ever_merged_flag = 0 THEN 1 END)
```

### Pending tickets
```sql
COUNT(DISTINCT CASE WHEN ever_merged_flag = 0 AND is_closed = 0 THEN ticket_id END)
```

### Same-month closed
Used to scope monthly performance metrics — only count tickets that were both *created* and
*resolved* within the same calendar month:
```sql
COUNT(CASE WHEN DATE_TRUNC('month', created_at) = DATE_TRUNC('month', closing_time)
            AND ever_merged_flag = 0 THEN 1 END)
-- Or use the pre-computed flag:
SUM(same_month_closed_flag)
```

### Same-month resolved %
```sql
ROUND(100.0 *
  COUNT(CASE WHEN same_month_closed_flag = 1 AND ever_merged_flag = 0 THEN ticket_id END)
  / NULLIF(COUNT(CASE WHEN ever_merged_flag = 0 THEN ticket_id END), 0), 1)
```

### CSAT — canonical definition (source-of-truth)

**Default view (running numbers — quote these unless asked otherwise):**
```sql
SELECT
  ROUND(100.0
    * COUNT(CASE WHEN cx_rating IN (4,5) THEN 1 END)
    / NULLIF(COUNT(CASE WHEN cx_rating > 0 THEN 1 END), 0), 2) AS csat_pct
FROM sp_product.cx_tickets
WHERE ever_merged_flag = 0
  AND created_at >= '<start>' AND created_at < '<end>'
-- For "app tickets only", add: AND source = 'SPINNY_HELP_AND_SUPPORT'
-- For "same-month-closed", add: AND same_month_closed_flag = 1
```

**Two equivalent ways to query CSAT, both confirmed by Arpit Dhankani 2026-05-28:**

| Path | Scope-filter needed? |
|---|---|
| **`sp_product.cx_tickets.cx_rating`** | **No — table is already CX-scoped at ETL** (99.3% SUPPORT + 1.5% LUXURY; excludes FEEDBACK / AUCTION / SOCIAL_MEDIA / SERVICING) |
| **`iceberg.sp_ticket_master.ticket.cx_rating`** | **Yes — apply `WHERE service_unit_id = 1`** (raw table contains all service units) |

⚠️ When in doubt, prefer `sp_product.cx_tickets` — it's the table the dashboard reads from, and its CX
scope is enforced upstream in the SQLMesh ETL. There's no risk of accidentally including FEEDBACK
auto-collection rows or AUCTION dealer tickets.

**Service unit ID → name mapping** (from `iceberg.sp_ticket_master.service_unit`):
| id | name | May '26 in `iceberg.ticket` | May '26 in `sp_product.cx_tickets` |
|--:|---|--:|--:|
| **1** | **SUPPORT** | 23,839 | **23,663 (99.3%)** |
| 2 | SOCIAL_MEDIA | 349 | 4 |
| 3 | SERVICING | 226 | 1 |
| 4 | FEEDBACK | 48,200 | **0** (excluded by ETL) |
| 5 | AUCTION | 11,000 | **0** (excluded by ETL) |
| 6 | LUXURY | 358 | 353 |

So `sp_product.cx_tickets` ≈ SUPPORT with a small LUXURY tail. The dashboard reads from this table,
which is why the Overview metrics aren't polluted by FEEDBACK / AUCTION volume despite having no
explicit `service_unit_id` filter on the Superset side.

**Canonical CSAT formula (works against either table):**
```sql
ROUND(100.0
  * COUNT(CASE WHEN ever_merged_flag = 0 AND cx_rating IN (4,5) THEN 1 END)
  / NULLIF(COUNT(CASE WHEN ever_merged_flag = 0 AND cx_rating > 0 THEN 1 END), 0), 1) AS csat_pct
FROM sp_product.cx_tickets                           -- preferred — already CX-scoped
WHERE created_at >= '<start>' AND created_at < '<end>'

-- Equivalent against the raw iceberg table:
-- FROM iceberg.sp_ticket_master.ticket
-- WHERE service_unit_id = 1                         -- mandatory if querying raw
--   AND merged_at IS NULL                           -- equivalent of ever_merged_flag = 0
```

Rating scale: 1–5. `cx_rating IN (4,5)` = positive. `cx_rating > 0` = any rating submitted.
"Non-merged" via `ever_merged_flag = 0` on `sp_product.cx_tickets`, or `merged_at IS NULL` on
the raw iceberg ticket table.

### Rated %
```sql
100.0 * COUNT(DISTINCT CASE
    WHEN cx_rating > 0 AND is_closed = 1 AND ever_merged_flag = 0
    THEN ticket_id END)
/ COUNT(DISTINCT CASE
    WHEN is_closed = 1 AND ever_merged_flag = 0
    THEN ticket_id END)
```

### Avg / Median / P90 resolution time
```sql
-- Avg (hours)
ROUND(AVG(CASE WHEN ever_merged_flag = 0 AND is_closed = 1 THEN resolution_time_hrs END), 2)
-- Median
percentile_approx(CASE WHEN ever_merged_flag = 0 AND is_closed = 1 THEN resolution_time_hrs END, 0.5)
-- P90
percentile_approx(CASE WHEN ever_merged_flag = 0 AND is_closed = 1 THEN resolution_time_hrs END, 0.9)
```

### FCR % (First Call Resolution)
```sql
100.0 * SUM(CASE WHEN ever_merged_flag = 0 AND fcr_flag = 'FCR' THEN 1 ELSE 0 END)
      / NULLIF(SUM(CASE WHEN ever_merged_flag = 0 AND fcr_flag IN ('FCR','NON_FCR') THEN 1 ELSE 0 END), 0)
```

**`fcr_flag` has FIVE values, not three.** Confirmed by Arpit Dhankani 2026-05-29. The FCR denominator
`IN ('FCR', 'NON_FCR')` deliberately excludes three additional cohorts:

| `fcr_flag` value | Apr–May '26 vol | Meaning | In FCR denom? |
|---|--:|---|:--:|
| `FCR` | 13,309 | First-call resolved | ✅ (numerator + denom) |
| `NON_FCR` | 19,363 | Resolved, but not on first contact | ✅ (denom) |
| `flag` | 3,542 | Closed but flagged for review (exception cases) | ❌ |
| `dropped` | 937 | Customer abandoned — **1:1 with `is_dropped = 1`** | ❌ |
| (empty/NULL) | 11,752 | In-flight (0 closed) or merged (8,000 of these) | ❌ |

So the FCR scope captures ~67% of Apr–May tickets (`32,672 / 48,903`). The 33% excluded breaks down
as: in-flight (8%), merged (16%), flagged (7%), dropped (2%).

**Corollary:** `fcr_flag = 'dropped'` ≡ `is_dropped = 1` — either is sufficient to identify abandoned tickets.

### Reopened tickets
```sql
SUM(is_reopened)
```

### Circuit-resolved CSAT (sub-cohort)
```sql
(COUNT(CASE WHEN is_circuit_resolved = 1 AND cx_rating IN (4,5) THEN 1 END) * 100.0)
/ NULLIF(COUNT(CASE WHEN is_circuit_resolved = 1 AND cx_rating > 0 THEN 1 END), 0)
```

### AI-handled tickets
```sql
-- AI-assigned at first touch:
COUNT(CASE WHEN first_agent_assigned_id IN (981, 987, 998) THEN ticket_id END)
-- AI-closed:
COUNT(CASE WHEN latest_agent_assigned_id IN (981, 987, 998) THEN ticket_id END)
```
`first_agent_assigned_id` / `latest_agent_assigned_id` live in `sp_entity_store.support_tickets`,
joined to `sp_product.cx_tickets` on ticket_id.

⚠️ **The dashboard's hardcoded `(981, 987, 998)` is stale** — as of May 2026 there are 3 newer bots
in production (1010, 1012, 1015). AI ticket KPIs on the Overview chart undercount until this is fixed.

### Automated bot monitor

**`~/claude_test/cx/bot_monitor.py`** runs every morning at 11 AM IST (cron).
- Queries `iceberg.sp_ticket_master.user WHERE is_bot = true`
- Compares against `~/claude_test/cx/known_bots.json` (state file — all 10 known bots seeded 2026-06-02)
- On new bot detected → emails `arpit.dhankani@spinny.com` + appends a placeholder row to the AI Bot Inventory table above
- Log: `~/claude_test/cx/bot_monitor.log`

When you get the alert, fill in the **use case**, **customer side**, and **launched date** in the table row.

### Canonical source-of-truth for bot identity
```sql
SELECT id, first_name, last_name, date_joined
FROM iceberg.sp_ticket_master.user
WHERE is_bot = true
```
Note: `username` and `email` columns in `iceberg.sp_ticket_master.user` are **hashed** (hex-encoded);
only `first_name` / `last_name` are human-readable. Use those for identification.

### AI Bot Inventory (confirmed May 2026)

| Bot ID | Name | Launched | Primary Use Case | Customer Side |
|---|---|---|---|---|
| 633 | System | Dec 2024 | System / auto-generated tickets | – |
| 696 | Circuit | Apr 2025 | **Repurposed as AI recategorizer** (post-creation classification cleanup). Originally a general-purpose first-touch router peaking at 4,583 tix/mo (Sep '25); first-touch volume collapsed to ~20/mo (May '26) as specialised bots took over. Current workload: ~1,874 `Recategorization` activities/month, fixing mis-categorized tickets (e.g. Miscellaneous → Warranty). | mixed |
| 829 | Meesho User | Aug 2025 | (low volume — purpose unclear) | – |
| 940 | Meesho Feedback | Jan 2026 | **Hub Visit feedback collection** — automated feedback, NOT support (4,973 tix in Apr). All `customer_type = OTHER` | OTHER |
| 981 | AI Agent | Mar 2026 | **RC queries (buyer)** — primarily RC Status + Shipment of physical RC. 60% CSAT, 37% solo-resolved | BUYER |
| 987 | CallKaro | Mar 2026 | **"Other > Pre-delivery"** queries. 93% CSAT but only 7.7% solo-resolved (hands off fast) | OTHER |
| 998 | Warranty AI Agent | Apr 2026 | **Warranty denial bot** — takes the issue + parts raised by the customer, checks against the active warranty's coverage, issues a verdict, and replies to the customer directly. Covers the full warranty taxonomy as the *input* surface (A/C, Engine, Brakes, Suspension, etc.) but the output is a coverage decision, not a fix. 47% CSAT is *expected* — many tickets end in "not covered". 43% solo-resolved. | BUYER |
| 1010 | Car Procured Voicebot | May 2026 | **Seller post-procurement voicebot** — "Car Procured" flow, 100% solo, 21h avg resolution | SELLER |
| 1012 | Insurance AI Agent | May 2026 | **Insurance queries** — just launched, ramp-up phase | BUYER |
| 1015 | Deal.Rejected.Voicebot | May 2026 | **Deal rejection follow-up voicebot** — just launched, no volume yet | (TBD) |

**Strategy observed:** Spinny is replacing the single-router Circuit bot with a fleet of **specialised
bots per use case** (RC, Warranty, Insurance, Car-Procured, Deal-Rejected). Each new bot owns one
sub-query family — easier to tune prompts/handoffs per domain.

**Performance benchmarks (Apr–May 2026 cohort):**
| Bot | Tickets | Solo-resolved % | CSAT % | Avg res (hrs) | Reopen % |
|---|--:|--:|--:|--:|--:|
| AI Agent (981, RC) | 823 | 36.6% | 60.2% | 49.3 | 7.7% |
| Warranty AI (998) | 767 | 43.4% | 46.9% | 54.1 | 7.6% |
| Car Procured VB (1010) | 233 | 100.0% | 5.4% (low rating volume) | 21.1 | 0.0% |
| CallKaro (987, Pre-delivery) | 208 | 7.7% | 93.3% | 108.3 | 1.4% |

Use "Solo-resolved %" = `first_agent_assigned_id = latest_agent_assigned_id` as the deflection metric:
the bot opened the ticket AND closed it without escalating to a human.

---

## `iceberg.sp_ticket_master.ai_response_log` — AI Audit Table

The **primary table for auditing all AI pipelines** in CX. Every time an AI model processes
something (summarise a call, recategorise a ticket, prioritise a message, read an odometer),
one row is written here. As of May 2026: **787K rows, Sep 2025 → present**.

### Schema

| Column | Type | Notes |
|---|---|---|
| `id` | bigint | PK |
| `content_id` | int | **= `ticket_id`** for ticket-level pipelines; maps to `support_tickets.ticket_id` |
| `content_type` | text | Which AI pipeline produced this row (see table below) |
| `request_id` | text | UUID — unique run identifier (correlate with LLM logs) |
| `response` | text | **JSON or plain text** — AI output; structure varies by `content_type` |
| `request_data` | text | **Full LLM prompt verbatim** ⚠️ may contain customer message text (PII) |
| `is_feedback_positive` | boolean | Agent thumbs-up/down on AI quality; NULL = no feedback given |
| `token_used` | bigint | Total tokens (input + output) |
| `input_token_used` | bigint | Input tokens only |
| `is_active` | boolean | Soft-delete flag |
| `created_at` | datetime(6) | When the AI ran |

**Join pattern — varies by `content_type`:**

| `content_id` maps to | Pipelines |
|---|---|
| `communication_message.id` | `CALL_SUMMARY`, `CUSTOMER_MESSAGE_PRIORITISER`, `VOICE_NOTE_SUMMARY`, `ODOMETER_READING`, `PROJECT_RELAY`, `PROJECT_RELAY_EXPERIMENT` |
| `ticket_id` (= `support_tickets.ticket_id`) | `TICKET_SUMMARY`, `RECATEGORIZATION`, `INTERNAL_SUMMARY`, `OVERALL_SUMMARY`, `Warranty_TICKET_CLOSURE` |

**Rule of thumb:** message-level pipelines (those that fire per incoming message / call / voice note) join on `communication_message.id`; ticket-level pipelines (those that assess the full ticket) join on `ticket_id` directly.

To get back to `ticket_id` from a `communication_message`-based row:
```sql
-- communication_message → communication → ticket_id
arl.content_id = cm.id                  -- ai_response_log → communication_message
AND cm.communication_id = c.id          -- communication_message → communication
AND c.ticket_id = <your_ticket_id>      -- communication → ticket
```

For the **`CALL_SUMMARY` pipeline**, the full chain with audio link extraction:
```sql
WITH call_summaries AS (
    SELECT
        c.ticket_id,
        cm.meta_data,
        airl.response AS call_summary,
        ROW_NUMBER() OVER (
            PARTITION BY c.ticket_id
            ORDER BY cm.created_at ASC NULLS LAST
        ) AS rn
    FROM iceberg.sp_ticket_master.communication c
    LEFT JOIN iceberg.sp_ticket_master.communication_message cm
        ON cm.communication_id = c.id
    LEFT JOIN iceberg.sp_ticket_master.ai_response_log airl
        ON cm.id = airl.content_id
        AND airl.content_type = 'CALL_SUMMARY'
    WHERE airl.response IS NOT NULL
)
SELECT
    ticket_id,
    -- Audio file URL (two possible locations in meta_data):
    COALESCE(
        GET_JSON_STRING(meta_data, '$.audio_file'),
        GET_JSON_STRING(meta_data, '$.callback_data.VoiceFileURL')
    )                                                                AS audio_link,
    -- S3-hosted audio (Cloudfront CDN):
    CASE
        WHEN GET_JSON_STRING(meta_data, '$.audio_file_s3_key') IS NOT NULL
        THEN CONCAT(
            'https://d3sw8mcxejlukg.cloudfront.net/',
            REGEXP_REPLACE(GET_JSON_STRING(meta_data, '$.audio_file_s3_key'), '^/', '')
        )
    END                                                              AS audio_s3_url,
    call_summary
FROM call_summaries
WHERE rn = 1
```

---

### Content types — all 11 pipelines (as of May 2026)

| content_type | Total rows | ~Monthly vol | Active since | Response format | Purpose |
|---|--:|--:|---|---|---|
| `CALL_SUMMARY` | 403K | ~60K | Nov 2025 | JSON | Post-call AI summary for agents |
| `TICKET_SUMMARY` | 218K | 21–50K | Nov 2025 | JSON | Full ticket analysis (main audit target) |
| `PROJECT_RELAY_EXPERIMENT` | 59K | ~18K | Sep 2025 | Plain text | AI chatbot relay — active A/B experiment |
| `RECATEGORIZATION` | 33K | ~3–5K | Oct 2025 | JSON | Circuit (696) auto-recategorization |
| `CUSTOMER_MESSAGE_PRIORITISER` | 19K | ~19K | **May 2026** | JSON | Auto-prioritization *(brand new)* |
| `INTERNAL_SUMMARY` | 17K | ~17K | **May 2026** | JSON | Internal team summary *(brand new)* |
| `OVERALL_SUMMARY` | 15K | ~15K | **May 2026** | JSON | Leadership-level ticket summary *(brand new)* |
| `ODOMETER_READING` | 10K | ~1.5–2K | Dec 2025 | JSON | Vision AI reading odometer from customer image |
| `VOICE_NOTE_SUMMARY` | 8K | ~1.2K | Nov 2025 | Plain text | AI transcription/summary of voice notes |
| `PROJECT_RELAY` | 333 | ~35 | Sep 2025 | Plain text | Older/stable relay bot (pre-experiment) |
| `Warranty_TICKET_CLOSURE` | 104 | ~80 | Apr 2026 | JSON | AI closure-decision on warranty tickets |

⚠️ **Three new pipelines launched simultaneously in May 2026**: `CUSTOMER_MESSAGE_PRIORITISER`,
`INTERNAL_SUMMARY`, `OVERALL_SUMMARY` — no historical data before May 2026.

ℹ️ A 12th pipeline **`RSA_CALL_SUMMARY`** appeared **2026-06-10** (AI summary of Roadside-Assistance calls; structured JSON `{eta_minutes, call_summary}`). The ~170K-row count is mostly a Jun 10–11 backfill; real run-rate is only a few hundred/day.

---

### `iceberg.sp_ticket_master.message_gate_log` — Message Gate (agent message quality checker) 🆕

**Launched 2026-06-17.** A real-time AI **quality gate on agent outbound messages** — *not* part of `ai_response_log`. It sits **in the agent's compose loop**: when an agent drafts a reply, the model checks it against quality dimensions, flags any failures, and offers a rewrite, logging original → suggested → final + token cost. This is the "Message Quality Checker" feature.

| Column | Notes |
|---|---|
| `agent_id` | agent whose draft was gated (→ `iceberg.sp_ticket_master.user.id`) |
| `ticket_id` | bigint → `support_tickets.ticket_id` |
| `original_message` | the agent's draft ⚠️ may contain PII |
| `dimensions_failed` | JSON array of failed checks; `NULL`/`''`/`[]` = passed clean |
| `suggested_message` | the AI's improved rewrite ⚠️ may contain PII |
| `final_message` | what was actually sent — `final_message = suggested_message` ⇒ agent **accepted** the rewrite |
| `url_list` | links in the message |
| `prompt_tokens` / `completion_tokens` / `cached_tokens` | LLM usage |
| `created_at` | **UTC** (`+330` → IST), like all `iceberg.sp_ticket_master` timestamps |

**Quality dimensions gated (3):** `repetition`, `tone_grammar`, `missed_update`.

**Day-1 baseline (2026-06-17 — partial day, 51 agents, treat as preliminary):** 568 messages gated across 407 tickets; **81% failed ≥1 dimension** (most-flagged `repetition`, then `tone_grammar`, then `missed_update`); **~83% suggestion-acceptance** (`final_message = suggested_message` in 471/568).

- **Acceptance rate** = `SUM(CASE WHEN final_message = suggested_message THEN 1 END) / COUNT(*)`.
- **Fail rate** = share of rows where `dimensions_failed` is non-empty (`NOT IN ('', '[]', '{}')` and `NOT NULL`).

---

### Response JSON keys — per pipeline

#### `TICKET_SUMMARY` ← main audit target
Full ticket analysis run after each significant update. Contains everything needed for SLA and
sentiment monitoring without reading raw messages.

```json
{
  "ticket_summary": "narrative summary of the customer's issue and resolution status",
  "sentiment_trend": "Worsening | Stable | Improving",
  "confidence_score": "High | Medium | Low",
  "empathy_concerns": ["<ISO timestamp>", ...],        // timestamps of negative customer msgs
  "escalation_alert": "Yes | No",
  "legal_escalation": "Yes | No",
  "sla_violation_risk": "Yes | No",
  "social_media_escalation": "Yes | No",
  "final_csat_estimate": "High | Medium | Low",
  "agent_pending_actions": ["action 1", "action 2", ...],
  "customer_pending_actions": ["action 1", ...],
  "ai_response_templates": ["suggested reply 1", "suggested reply 2"],
  "top_3_recent_concerns": ["concern 1", "concern 2", "concern 3"],
  "customer_requested_ticket_closure": "true | false"
}
```

**Feedback:** `is_feedback_positive` is populated here (and almost nowhere else).
Out of 218K rows: 127 thumbs-up, 48 thumbs-down (175 rated total — ~0.08% feedback rate).
Positive rate: **72.6%** (127/175). Use this for AI quality trend audits.

#### `CALL_SUMMARY`
Generated per call for agent reference.
```json
{
  "Subject": "one-line call topic",
  "Call Purpose": "why customer called",
  "Call Outcome": "what agent will do next",
  "Customer Queries": ["question 1", "question 2", ...],
  "Discussion Points": ["point 1", "point 2", ...],
  "Action Items": ["action 1", ...]
}
```

#### `RECATEGORIZATION`
Written by Circuit (bot 696) when it re-classifies a mis-categorised ticket.
```json
{
  "ticket_id": 1647427,
  "recategorised": true,
  "new_query": "Warranty",
  "new_subquery": "Lights",
  "new_query_heading": "Warning Lights are ON",
  "extra_details": true
}
```
Cross-check with `ticket_activity_log` (`activity_type = 'Recategorization'`) to confirm
the system applied the change.

#### `CUSTOMER_MESSAGE_PRIORITISER` *(May 2026)*
Runs on each incoming customer message. Output drives ticket prioritisation queue.
```json
{
  "decision": "prioritise | deprioritise",
  "reason": "narrative reason for the decision",
  "filter_matched": "none | <rule name>"
}
```

#### `INTERNAL_SUMMARY` *(May 2026)*
Concise internal briefing for ops/manager view.
```json
{
  "brief_summary": "1–2 sentence situation summary",
  "actions_pending": {
    "Customer": ["action 1", ...],
    "Internal_Ops": ["action 1", ...],
    "Support_Agent": ["action 1", ...]
  },
  "proposed_timeline": {"timeline_text": "31st May"},
  "evidence_message_ids": [19425715, 19431610, ...]
}
```

#### `OVERALL_SUMMARY` *(May 2026)*
Leadership-facing ticket summary combining all threads.
```json
{
  "overall_ticket_summary": "full narrative",
  "action_pending_support": "bullet list as plain text",
  "action_pending_customer": "bullet list as plain text",
  "action_pending_internal": "bullet list as plain text"
}
```

#### `ODOMETER_READING`
Vision AI (multimodal) reads odometer reading from customer-submitted image.
```json
{
  "final_reading_km": "43686",
  "confidence_score": 0.98,
  "flags": ["trip meter detected"],
  "Reasoning": "The odometer reading '43686 km' was clearly identified ...",
  "suggested_action": "auto-approve | manual-review",
  "customer_message": "",
  "request_id": "uuid"
}
```
`suggested_action = "auto-approve"` means confidence is high enough for auto-acceptance.

#### `VOICE_NOTE_SUMMARY`
Plain text paragraph summarising a voice note sent by the customer.
No JSON structure — the full response is the summary text.

#### `PROJECT_RELAY` / `PROJECT_RELAY_EXPERIMENT`
The AI relay chatbot's generated response text (what the bot sent to the customer).
Plain text — no JSON. The "EXPERIMENT" variant is the A/B test version; base `PROJECT_RELAY`
is the stable legacy version (~35/month).

Volume anomaly: `PROJECT_RELAY_EXPERIMENT` dropped near-zero in Feb–Mar 2026
(635 rows in Apr '26) then shot back to 18K in May '26 — indicates active experimentation.

#### `Warranty_TICKET_CLOSURE`
AI decision on whether to auto-close a warranty ticket.
```json
{
  "should_close": true,
  "reason": "Customer explicitly requested to close the ticket"
}
```

---

### Standard audit queries

**1. AI quality by pipeline (all time)**
```sql
SELECT
    content_type,
    COUNT(*)                                                           AS total_runs,
    COUNT(CASE WHEN is_feedback_positive = true  THEN 1 END)          AS thumbs_up,
    COUNT(CASE WHEN is_feedback_positive = false THEN 1 END)          AS thumbs_down,
    ROUND(
        COUNT(CASE WHEN is_feedback_positive = true THEN 1 END) * 100.0
        / NULLIF(COUNT(CASE WHEN is_feedback_positive IS NOT NULL THEN 1 END), 0), 1
    )                                                                  AS positive_pct,
    ROUND(AVG(token_used), 0)                                         AS avg_tokens
FROM iceberg.sp_ticket_master.ai_response_log
GROUP BY content_type
ORDER BY total_runs DESC
```

**2. TICKET_SUMMARY monthly trend with feedback**
```sql
SELECT
    DATE_FORMAT(created_at, '%Y-%m')                                   AS month,
    COUNT(*)                                                            AS summaries,
    COUNT(CASE WHEN is_feedback_positive = true  THEN 1 END)           AS thumbs_up,
    COUNT(CASE WHEN is_feedback_positive = false THEN 1 END)           AS thumbs_down
FROM iceberg.sp_ticket_master.ai_response_log
WHERE content_type = 'TICKET_SUMMARY'
GROUP BY month
ORDER BY month
```

**3. TICKET_SUMMARY for a specific ticket (latest run)**
```sql
SELECT
    created_at,
    GET_JSON_STRING(response, '$.sentiment_trend')         AS sentiment,
    GET_JSON_STRING(response, '$.escalation_alert')        AS escalation,
    GET_JSON_STRING(response, '$.sla_violation_risk')      AS sla_risk,
    GET_JSON_STRING(response, '$.final_csat_estimate')     AS csat_est,
    GET_JSON_STRING(response, '$.ticket_summary')          AS summary
FROM iceberg.sp_ticket_master.ai_response_log
WHERE content_type = 'TICKET_SUMMARY'
  AND content_id = <ticket_id>
ORDER BY created_at DESC
LIMIT 1
```

**4. RECATEGORIZATION accuracy check — did the AI actually change the category?**
```sql
-- Rows where AI said recategorised=true
SELECT
    COUNT(*)                                                               AS ai_claimed_recategorised,
    COUNT(CASE WHEN GET_JSON_STRING(response,'$.recategorised') = 'true'
               THEN 1 END)                                                 AS json_flag_true
FROM iceberg.sp_ticket_master.ai_response_log
WHERE content_type = 'RECATEGORIZATION'
  AND DATE_FORMAT(created_at,'%Y-%m') = '2026-05'
```

**5. Escalation alerts — open tickets flagged HIGH by AI in last 7 days**
```sql
-- TICKET_SUMMARY: content_id = ticket_id (direct join)
SELECT
    t.query_category_name,
    t.primary_sub_query_name,
    COUNT(*)                                                            AS escalation_count
FROM iceberg.sp_ticket_master.ai_response_log arl
JOIN sp_entity_store.support_tickets t ON arl.content_id = t.ticket_id
WHERE arl.content_type = 'TICKET_SUMMARY'
  AND GET_JSON_STRING(arl.response, '$.escalation_alert') = 'Yes'
  AND t.ticket_status_name NOT IN ('Closed', 'Resolved')
  AND t.service_unit_id = 1
  AND arl.created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY t.query_category_name, t.primary_sub_query_name
ORDER BY escalation_count DESC
LIMIT 20
```

**6. Odometer reading — auto-approve rate by month**
```sql
SELECT
    DATE_FORMAT(created_at, '%Y-%m')                                   AS month,
    COUNT(*)                                                            AS total,
    COUNT(CASE WHEN GET_JSON_STRING(response,'$.suggested_action') = 'auto-approve'
               THEN 1 END)                                             AS auto_approved,
    ROUND(
        COUNT(CASE WHEN GET_JSON_STRING(response,'$.suggested_action') = 'auto-approve'
                   THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0), 1
    )                                                                   AS auto_approve_pct
FROM iceberg.sp_ticket_master.ai_response_log
WHERE content_type = 'ODOMETER_READING'
GROUP BY month
ORDER BY month
```

---

### Closing-reason taxonomy — query-specific

Closing reasons are **scoped to the query type**, not universal. The same word ("Others", "Items not found") appears across queries but with different operational meaning. Per Arpit Dhankani 2026-05-29 the canonical interpretation is query-by-query.

Apr–May '26 closing-reason frequencies for the major query types (SU=1, non-merged, ≥30 tix shown), grouped into my best-guess sentiment buckets. **Marked ⚠️ where my classification needs confirmation.**

Legend: ✅ positive resolution · 🟡 process / awaiting · 🔴 denied / not eligible · ⚫ abandoned / no contact · ⚪ uncategorised ("Others")

**Warranty** (9,534 closed)
| Reason | Volume | Bucket |
|---|--:|:--:|
| 3 call attempts exhausted | 2,337 | ⚫ |
| Items not found | 1,874 | ⚫ |
| **DEALER_FINANCING_APPROVED** ⚠️ | 1,215 | ?? (likely cross-flow leakage from a finance flow — not a real warranty outcome) |
| Issue resolved at DSI | 1,190 | ✅ |
| Workshop repair completed | 989 | ✅ |
| Resolved on call | 539 | ✅ |
| Resolved on chat | 395 | ✅ |
| Non warranty denial | 252 | 🔴 |
| Car taken by customer | 208 | ✅ (car returned for repair) |
| Car delivered to customer | 196 | ✅ (post-repair delivery) |
| REJECTED | 186 | 🔴 |
| Delivery Note Shared | 153 | 🟡 |

⚠️ Note: only ~438 warranty closings are explicit denials (Non warranty denial + REJECTED), but `warranty_status` says ~50% of warranty queries are denied. The remaining denials likely close as "3 call attempts exhausted" or "Items not found" after the customer doesn't engage with the denial verdict.

**Payments - Seller** (5,795 closed)
| Reason | Volume | Bucket |
|---|--:|:--:|
| TAT Shared | 1,854 | 🟡 |
| Document correction | 1,207 | 🟡 |
| OTP Amount Released | 1,148 | ✅ |
| Others | 481 | ⚪ |
| Challan Hold Amount | 333 | ✅ |
| Unresponsive | 140 | ⚫ |
| Items not found | 139 | ⚫ |
| RC Card Amount Released | 134 | ✅ |
| REFUND_COMPLETED | 121 | ✅ |
| Bank NOC Hold Amount | 81 | ✅ |
| Fastag Hold Amount | 69 | ✅ |
| Other Hold Amount Released | 58 | ✅ |
| 2nd Key Hold Amount | 31 | ✅ |

**RC** (3,714)
| Reason | Volume | Bucket |
|---|--:|:--:|
| RC Transferred | 832 | ✅ |
| RC Dispatched courier details shared | 816 | ✅ |
| RC TAT Shared | 653 | 🟡 |
| RC couriered by RTO | 323 | ✅ |
| Others | 295 | ⚪ |
| RC Delivery Done | 243 | ✅ |
| RC Correction Done | 237 | ✅ |
| Items not found | 205 | ⚫ |
| RC correction Not possible | 110 | 🔴 |

**RC - Seller** (2,713) · skewed to operational closes: RC Transferred 716 ✅, RC TAT Shared 1,418 🟡, Others 499 ⚪, Unresponsive 49 ⚫, Items not found 31 ⚫

**RSA** (2,779) · RSA provided 1,263 ✅, HSRP dispatched 555 ✅, Unresponsive 378 ⚫, Others 352 ⚪, HSRP Sticker NA 138 🔴, Cx did not require RSA 48 🔴, Other Issue 45 ⚪

**Insurance** (1,433) · Docs Shared 555 ✅, Mobile Number updated 203 ✅, Routed to insurance company 198 ✅, Others 134 ⚪, Items not found 124 ⚫, Buyer is not ready for the liability 119 🔴, NCB Claim 52 ✅, Insurance amount reimbursed 48 ✅

**Payments** (1,375) · Others 716 ⚪, Insurance Renewal Raised 316 ✅, Amount Released 268 ✅, Items not found 44 ⚫, No additional amount charged 31 ✅

**Spinny Benefits** (953) · REQUEST_REJECTED 441 🔴 (largest — product-team rejection of benefit request), ACCOUNT_RE-ACTIVATED 222 ✅, ACCOUNT_CLOSED 128 ✅ (closure as intended outcome), Not responding 70 ⚫, Mobile Number updated 48 ✅, Others 44 ⚪

**Other** (789) · Reschedule test drive 601 🟡 (test-drive-coordination tickets), Items not found 157 ⚫, Others 31 ⚪

**Seller-Miscellaneous** (762) · Others 395 ⚪, Items not found 122 ⚫, TAX_PAID_BY_SPINNY-POST_DELIVERY 106 ✅, NO_TAX_PENDING 54 ✅, Unresponsive 43 ⚫, DEALER_AGREED_FOR_REFUND 42 ✅

**HSRP** (610) · HSRP Sticker provided 267 ✅, Not eligible 241 🔴, Unresponsive 66 ⚫, Items not found 36 ⚫

**Fastag** (1,029) · Explanation Shared 764 ✅ (most queries answered via written explanation), New Fastag Activated 127 ✅, Items not found 75 ⚫, Others 32 ⚪, Old Fastag Deactivated 31 ✅

**Document** (525) · Booking Agreement Shared 220 ✅, DOCUMENT_HANDED_OVER 135 ✅, RSA not provided 101 🔴, Others 39 ⚪, CHALLAN_CLEARED 30 ✅

**Miscellaneous** (1,000) · Others 909 ⚪ (mostly uncategorised), Items not found 91 ⚫

### Cross-query patterns

| Reason | Appears in | Universal meaning |
|---|---|---|
| **Items not found** | almost every query | Agent couldn't locate the customer's claimed item/issue → **⚫ abandoned** (treat as non-resolution, not denial) |
| **Unresponsive / 3 call attempts exhausted / Not responding** | most | Customer didn't engage → **⚫ abandoned** |
| **TAT Shared / RC TAT Shared** | Payments-Seller, RC, RC-Seller | Agent committed a turnaround time and closed pending → **🟡 process** (not truly resolved at close time) |
| **Others** | all | Agent didn't pick a specific reason → **⚪ uncategorised** (large in Miscellaneous, Payments, RC-Seller — these are mostly successful closes that didn't fit a specific bucket) |

### Implication for analyses

For any "% positive close" or "% denied" analysis, **do not aggregate closing reasons across queries** — the same label means different things. Build a per-query reason → bucket mapping (use the tables above as a starting point) and aggregate at the bucket level.

⚠️ **`DEALER_FINANCING_APPROVED` is a stale label** — the actual `display_name` in `ticket_reason` for id 244 is `"Out of Coverage"`. This is a legitimate warranty outcome (not a finance flow misroute as previously suspected). All closing-reason names in the taxonomy tables above should be re-read against `ticket_reason.display_name` via Known Trap #8.

### Standard support-volume filter (from Overview chart)
The Overview chart filters to "real" support tickets only, excluding feedback and luxury-specific noise:
```sql
WHERE query IN (
    'HSRP', 'Welcome call', 'RC - Seller', 'Warranty', 'Missed Call - Existing', 'RSA',
    'Spinny Benefits', 'Insurance', 'Seller-Miscellaneous', 'Fastag', 'Miscellaneous',
    'RC', 'Payments - Seller', 'Document', 'Other', 'Missed Call - New', 'Payments')
  AND source IN (
    'SPINNY_HELP_AND_SUPPORT', 'CREATED_MANUALLY', 'WEBSITE_INBOUND', 'PHONE_CALL', 'VOC',
    'RSA', 'CONTACT_US', 'DEMAND_INTERNAL', 'SERVICING_SUPPORT', 'SP_INSURANCE', 'EMAIL')
```

### Buyer ticket funnel — separate dataset (chart 464)
Computed at the **deal level**, not ticket level. Uses columns:
- `deal_id`, `delivery_time` — one row per delivered car
- `number_of_tickets` — tickets raised against that deal
- `days_to_first_ticket` — gap from delivery to first ticket

Buckets: 0–15d, 15–30d, 30–60d, 60+d. Used to track post-delivery support intensity per delivery cohort.

### Seller ticket funnel — separate dataset (chart 470)
Computed at the **stockin level**: keyed by `registration_no` + `stockin_date`. Same `number_of_tickets`
and `days_to_first_ticket` columns, bucketed identically. Used for post-procurement seller support.

### NDOA breach (chart 469 — Agent wise NDOA)
Uses a different dataset with `NDOA_count` and `agent_email`. NDOA = Next Date Of Action (the SLA the
agent committed to). `NDOA_count > 0` = at least one breach. Joined to CSAT to measure customer
impact of breached SLAs.
```sql
COUNT(CASE WHEN NDOA_count > 0 THEN ticket_id END) * 100.0 / COUNT(ticket_id)  -- breach %
```

---

### How the dashboard slices ticket volume

| Chart | Dimension | Time scope |
|---|---|---|
| Query wise split (482) | `query` | current month |
| Source split (479) / source wise split (453) | `source` | current month / last month |
| Customer split (480) / Customer wise split (452) | `customer_type` | current month / last month |
| City wise (461) | `CITY` | last month |
| KAM wise (462) | `latest_user` | last month |
| Monthly summarised view (451) | `created_at` (P1M time grain) | full history |
| Reporting numbers (463) | `created_at` (P1M time grain) | full history |
| tickets per customer (460) | `created_at` | full history |

---

## Playbooks

### Standard CX Ticket Query Template
```sql
SELECT
    DATE(ticket_created_ts)             AS date,
    query_category_name,
    customer_type_name,
    ticket_source_name,
    ticket_priority_code,
    COUNT(*)                            AS tickets,
    COUNT(CASE WHEN first_ticket_reopened_ts IS NOT NULL THEN 1 END) AS reopened,
    COUNT(CASE WHEN customer_rating_submitted_ts IS NOT NULL THEN 1 END) AS csat_submitted,
    AVG(customer_rating)                AS avg_csat,
    SECONDS_DIFF(first_ticket_resolved_ts, ticket_created_ts)/3600.0 AS avg_resolution_hrs
FROM sp_entity_store.support_tickets
WHERE service_unit_id = 1                  -- MANDATORY: CX SUPPORT scope only
  AND ticket_created_ts >= '2025-01-01'
  AND ticket_status_name != 'MERGED'       -- exclude absorbed tickets
GROUP BY 1,2,3,4,5
ORDER BY 1 DESC
```
⚠️ The `query_category_name != 'FEEDBACK'` filter I used in earlier analyses is **not sufficient** —
it only excludes service_unit_id = 4 (FEEDBACK), but still lets in AUCTION (5), LUXURY (6),
SOCIAL_MEDIA (2), SERVICING (3). Use `service_unit_id = 1` directly.

### Stitching Tickets to Supply Leads
```sql
SELECT
    sl.lead_id,
    sl.lead_created_ts,
    st.ticket_id,
    st.ticket_created_ts,
    st.query_category_name,
    st.ticket_status_name
FROM sp_entity_store.support_tickets st
JOIN sp_entity_store_intermediate.cx_ticket_lead_mapping m ON m.ticket_id = st.ticket_id
JOIN sp_entity_store.sell_leads sl ON sl.lead_id = m.lead_id
WHERE st.ticket_created_ts >= '2025-01-01'
```

---

## Days-since-car-event per ticket (post-delivery / post-stockin)

For every CX ticket, the canonical "freshness" measurement is **days from the car event to ticket creation** — for BUYER tickets that's days since car delivery; for SELLER tickets that's days since stockin. This is the per-ticket version of the deal/stockin-level metric the dashboard computes in charts 464 (Buyer funnel) and 470 (Seller breakdown).

### Source-of-truth columns per customer side

| Customer type | Event = | Source table | Join key | IST offset needed? |
|---|---|---|---|:--:|
| **BUYER** | Car delivery | `sp_product.demand_deal_masterdata` (filter `status_description = 'Delivery Done'`) | `ticket.deal_id = dmd.deal_id` | No (delivery_time already IST in this table) |
| **SELLER** | Car stockin | `iceberg.sp_web.listing_lead` | **`sp_entity_store.support_tickets.sell_lead_id = listing_lead.id`** — direct PK join | **Yes** — `listing_lead.delivery_time` is UTC; add 330 minutes to get IST stockin date |
| DEALER / OTHER | — | No clean event anchor | — | — |

⚠️ Note the schema overload: `listing_lead.delivery_time` is when the **seller delivered the car TO Spinny** (i.e. stockin). `demand_deal_masterdata.delivery_time` is when **Spinny delivered the car to the buyer**. Same column name, opposite ends of the funnel.

⚠️ **The dashboard's seller funnel (dataset 214) uses a reg-no string join** through the `stockin_lead_listing_leadprofile` bridge. This **undercounts seller tickets by ~9%** vs the direct `sell_lead_id` PK join — typically because of `extra_data.car_reg_no` being NULL, format-mismatched, or absent. **Always prefer `sell_lead_id`.**

### Canonical per-ticket query (use this — not the reg-no path)

```sql
WITH buyer_tix AS (
  SELECT
    t.id AS ticket_id,
    'BUYER' AS side,
    DATEDIFF(t.created_at, dmd.delivery_time) AS days_since_event
  FROM iceberg.sp_ticket_master.ticket t
  JOIN sp_product.demand_deal_masterdata dmd ON dmd.deal_id = t.deal_id
  LEFT JOIN iceberg.sp_ticket_master.sub_query sq ON sq.id = t.sub_query_id
  WHERE t.customer_type = 'BUYER'
    AND t.service_unit_id = 1
    AND dmd.status_description = 'Delivery Done'
    AND (sq.name IS NULL OR sq.name != 'New delivery')
),
seller_tix AS (
  SELECT
    t.id AS ticket_id,
    'SELLER' AS side,
    DATEDIFF(t.created_at, DATE_ADD(ll.delivery_time, INTERVAL 330 MINUTE)) AS days_since_event
  FROM iceberg.sp_ticket_master.ticket t
  JOIN sp_entity_store.support_tickets st ON st.ticket_id = t.id           -- ← bridge for sell_lead_id
  JOIN iceberg.sp_web.listing_lead ll ON ll.id = st.sell_lead_id           -- ← clean PK join
  LEFT JOIN iceberg.sp_ticket_master.sub_query sq ON sq.id = t.sub_query_id
  WHERE t.customer_type = 'SELLER'
    AND t.service_unit_id = 1
    AND ll.delivery_time IS NOT NULL
    AND (sq.name IS NULL OR sq.name != 'New delivery')
    AND t.created_at > DATE_ADD(ll.delivery_time, INTERVAL 330 MINUTE)
)
SELECT * FROM buyer_tix UNION ALL SELECT * FROM seller_tix;
```

### Standard 4-bucket convention (matches dashboard charts 464 / 470)
- `0–15 days` (early — usually delivery-related concerns)
- `16–30 days` (post-honeymoon — first warranty issues surface)
- `31–60 days` (RC / document / payment workflow tail)
- `60+ days` (long-tail — RC transfer delays, repeat warranty, durable issues)

### Per-ticket baseline (Apr–May 2026, service_unit_id = 1) — sell_lead_id PK join

| Side | Tickets | Avg days | Median | P90 | 0–15d | 16–30d | 31–60d | 60+d |
|---|--:|--:|--:|--:|--:|--:|--:|--:|
| **BUYER** | 31,872 | 108.2 | 49 | 288 | 27% | 12% | 17% | **45%** |
| **SELLER** | 12,175 | 141.4 | 72 | 297 | 12% | 11% | 21% | **56%** |

**Reads:**
- Buyer-ticket median is at **~7 weeks post-delivery**; ~45% land 60+ days out (the warranty/RC long tail).
- Seller-ticket median at **~10 weeks post-stockin**; P90 = 297 days. Sellers raise tickets a long time after their car has left their hands — mostly RC transfer / payment / fastag-deactivation issues that surface slowly.
- **Earlier numbers using the reg-no join were biased ~16% high on seller days** (avg 166.9 vs 141.4) because malformed/missing reg-numbers correlate with older, slower-resolving tickets. `sell_lead_id` PK gives the cleaner read.

### Sub-query-level analysis pattern

To pinpoint *which* sub-queries dominate each bucket (e.g. "what kinds of buyer issues come in within 15 days vs after 60?"):

```sql
SELECT
  t.customer_type, sq.name AS sub_query,
  COUNT(*) AS tix,
  AVG(days_since_event) AS avg_days,
  percentile_approx(CAST(days_since_event AS DOUBLE), 0.5) AS median_days
FROM <above CTE>
JOIN iceberg.sp_ticket_master.sub_query sq ON sq.id = t.sub_query_id
GROUP BY 1, 2
```

This is the canonical way to compute "what tickets come in *when* relative to delivery / stockin"
— useful for product/process owners deciding which interventions to staff at which time-from-event.

---

## RC Transfer Date — canonical lookup from a CX ticket

RC-related tickets (query = `RC`, sub-queries: RC Status, RTO Receipt, Shipment of physical RC, etc.)
often need the **date RC transfer was completed** to compute TAT or identify breach.

### Starting point: `support_tickets.deal_id`

Every buyer RC ticket has a `deal_id`. Use it to walk the RC transfer chain:

```
support_tickets.deal_id
    └─► rc_transfer_deal.deal_id          (one-to-one with deal)
            └─► rc_transfer_deal.id
                    └─► rto_dashboard_rtoverification.deal_id  ← (this is rc_transfer_deal.id, NOT the actual deal_id)
                                └─► rto_dashboard_rtoverification.id
                                        └─► rto_dashboard_rctransferstage.rto_verification_id
                                                WHERE rc_stage = 'complete'
```

All three tables live in `iceberg.sp_rc_transfer.*`.

### ⚠️ Critical: `rtoverification.deal_id` ≠ actual deal_id

`rto_dashboard_rtoverification.deal_id` stores `rc_transfer_deal.id` (the transfer record's own PK),
**not** the demand deal ID. The join must be:
```sql
drv.deal_id = td.id          -- ✅ correct
drv.deal_id = td.deal_id     -- ❌ wrong
```

### Why there are multiple `complete` rows per verification

`rto_dashboard_rctransferstage` writes one row per **document type** that completes:
OT (Original Transfer), HPA, HPT, NOC, DRC, HPT_RC — all with `rc_stage = 'complete'`,
typically at the same timestamp. Use `MIN(row_created_at)` to get the completion timestamp;
`is_valid` is always 1 for complete rows (no filter needed).

### Canonical SQL — RC transfer completion date from a CX ticket

```sql
-- For a set of RC tickets, get delivery date and RC transfer completion date
WITH rc_done AS (
    SELECT
        td.deal_id,
        MIN(DATE_ADD(rts.row_created_at, INTERVAL 330 MINUTE)) AS first_rc_transfer_ts  -- UTC → IST
    FROM iceberg.sp_rc_transfer.rc_transfer_deal td
    JOIN iceberg.sp_rc_transfer.rto_dashboard_rtoverification drv
        ON drv.deal_id = td.id                    -- join on transfer PK, not actual deal_id
    JOIN iceberg.sp_rc_transfer.rto_dashboard_rctransferstage rts
        ON rts.rto_verification_id = drv.id
    WHERE rts.rc_stage = 'complete'
    GROUP BY td.deal_id
)
SELECT
    st.ticket_id,
    st.deal_id,
    st.query_name,
    st.primary_sub_query_name,
    st.ticket_created_ts,
    dmd.delivery_time,
    rc.first_rc_transfer_ts,
    DATEDIFF(DATE(rc.first_rc_transfer_ts), DATE(dmd.delivery_time))  AS delivery_to_rc_days,
    DATEDIFF(DATE(st.ticket_created_ts),    DATE(rc.first_rc_transfer_ts)) AS ticket_after_rc_days
FROM sp_entity_store.support_tickets st
JOIN sp_product.demand_deal_masterdata dmd
    ON dmd.deal_id = st.deal_id
LEFT JOIN rc_done rc
    ON rc.deal_id = st.deal_id
WHERE st.service_unit_id = 1
  AND st.ticket_merged_ts IS NULL
  AND st.query_name = 'RC'
  AND st.ticket_created_ts >= '2025-01-01'
```

`ticket_after_rc_days` < 0 means the ticket was raised **before** RC transfer completed (customer
chasing status). > 0 means it was raised after — likely a post-transfer complaint (wrong RC details,
physical delivery, etc.).

### Also available: registration number and RTO code

```sql
-- Get reg number for the car on the ticket (via sell_lead_id → listing_lead → leadprofile)
LEFT JOIN iceberg.sp_web.listing_lead ll
    ON ll.id = st.sell_lead_id
LEFT JOIN internal.sp_web_mv.stockin_lead_listing_leadprofile llp
    ON llp.lead_id = ll.id
-- llp.registration_no → LEFT(lower(registration_no), 4) = rto_code, LEFT(..., 2) = state_code
```

### Table quick-reference

| Table | DB | Key join column | What it holds |
|---|---|---|---|
| `rc_transfer_deal` | `iceberg.sp_rc_transfer` | `.deal_id` → demand deal | One row per RC transfer case |
| `rto_dashboard_rtoverification` | `iceberg.sp_rc_transfer` | `.deal_id` → `rc_transfer_deal.id` | Verification record per transfer |
| `rto_dashboard_rctransferstage` | `iceberg.sp_rc_transfer` | `.rto_verification_id` → `rtoverification.id` | Stage history; filter `rc_stage = 'complete'` for completion |

---

## Calls, Response Time & Callbacks

### Which call table to use — and which to avoid

**Rule (set 2026-06-04): always map calls → tickets via `ticket_communication_call_log_mapping` (communication_call_log). For AI / voicebot calls, use `call_record` (call_records) — that's where the AI voice pipeline logs.**

| Table | Use for | Notes |
|---|---|---|
| **`iceberg.sp_ticket_master.ticket_communication_call_log_mapping`** (communication_call_log) | **Canonical call→ticket mapping for all human / general calls.** `ticket_id`, `created_at` (the call event time; `+330` → IST), `meta_data` JSON. | **Unhashed & readable.** `GET_JSON_STRING(meta_data, '$.callback_data.systemDisposition')` returns real dispositions (`CONNECTED`/`NO_ANSWER`/`BUSY`/`CALL_DROP`/`CALL_HANGUP`/`CALL_NOT_PICKED`) — this is the *actual dialer event*, so it's the source of truth for pickup-rate, response-time and callback-TAT (~97% coverage). ✏️ **Corrected 2026-06-04:** an earlier note here wrongly claimed this path was hashed-NULL and the dashboard "silently returns 0" — false; it returns valid data (verified via Superset dataset 365 / chart 813). |
| **`iceberg.sp_ticket_master.call_record`** (call_records) | **AI / voicebot calls** — CallKaro (bot 987), Bolna voice provider, voicebots (1010 / 1012 / 1015). | The AI voice pipeline logs its calls here; use it for any bot-call analysis. *Not* populated for the general human-agent pipeline (earlier marked "do not use" — that only meant "not for human calls"). |
| `sp_entity_store_intermediate.cx_ticket_call_mapping` | Human-agent **manual** dials only (`outbound.manual.dial`), with rich agent fields: `call_status`, `agent_email`, `campaign_name`, `operator_name`, `call_weightage`. | ⚠️ Does **NOT** capture the callback dialer or automated calls. Using it for callback-TAT understated in-slot **~16×** (its "first manual call after the request" averages ~34h later — an unrelated downstream agent call). Use only for agent-level manual-dial analysis — never for callback adherence or as the canonical call map. |
| `sp_entity_store.support_tickets` | Ticket-level aggregates: `first_connected_outbound_call_start_ts`, `total_outbound_calls_count` (⚠️ stores 0–1 fractions, not integers), etc. | One row per ticket |
| `iceberg.sp_ticket_master.customer_callback_history` | Callback request events | Unhashed: `ticket_id`, `preference_type`, `slot_hour`, `scheduled_date`, `created_at` |
| `iceberg.sp_ticket_master.customer_callback_preference` | Customer's stored callback preference | – |

### `call_type` and `call_status` enums — confirmed values

`call_type`:
- `outbound.manual.dial` — agent-initiated call to customer (>90% of volume)
- `inbound.call.dial` — customer-initiated call to Spinny
- `transferred.to.campaign.dial` — call routed into an outbound campaign (rare)

`call_status` (Apr–May '26 distribution):
| Status | Meaning | Connected? |
|---|---|:-:|
| `CONNECTED` | Agent and customer talked | ✅ |
| `NO_ANSWER` | No pickup | ❌ |
| `CALL_HANGUP` | Call ended via hangup — **normal end** for inbound, premature end for outbound | depends |
| `BUSY` | Line busy | ❌ |
| `CALL_DROP` | Call dropped mid-conversation | ❌ |
| `CALL_NOT_PICKED` | Customer didn't pick up | ❌ |
| `ATTEMPT_FAILED` / `PROVIDER_TEMP_FAILURE` / `PROVIDER_FAILURE` / `NUMBER_FAILURE` | Telecom issue | ❌ |
| `INVALID_PHONE_NUMBER` | Bad number | ❌ |
| `USER_UNAVAILABLE` / `VOICEMAIL_DETECTED` / `USER_REJECTED` | Customer-side failure | ❌ |
| `FAILED` | Generic failure | ❌ |

⚠️ **Inbound pickup rate is misleading.** Inbound calls predominantly show `CALL_HANGUP` (~13.7K/month) — that's the normal end of a successful inbound conversation. Filtering inbound on `status='CONNECTED'` returns ~1% which is wrong. For inbound success, treat `CALL_HANGUP` as a success signal (with care).

### Outbound pickup rate (Jan–May 2026, SUPPORT / `service_unit_id = 1`)

✏️ **Rebuilt 2026-06-04** on `communication_call_log` using `$.callback_data.systemDisposition` (the dashboard's own success signal — chart 840 defines pickup as `systemDisposition = 'CONNECTED'`), scoped to `service_unit_id = 1` via `support_tickets`. Outbound = `$.callback_data.callType = 'outbound.manual.dial'`. Two denominators:
- **Raw** = `CONNECTED / all outbound attempts` (matches dashboard methodology).
- **Clean** = excludes provider/telecom failures (`PROVIDER_TEMP_FAILURE`, `PROVIDER_FAILURE`, `NUMBER_FAILURE`, `ATTEMPT_FAILED`) from the denominator — these never reached the customer, so this is true *reachable-customer* pickup.

| Month | Outbound attempts | Connected | Provider/telecom fails | Pickup % (raw) | Pickup % (clean) |
|---|--:|--:|--:|--:|--:|
| Jan | 95,161 | 64,333 | 6,795 | 67.6% | **72.8%** |
| Feb | 84,249 | 58,212 | 5,041 | 69.1% | **73.5%** |
| Mar | 83,457 | 57,865 | 5,363 | 69.3% | **74.1%** |
| Apr | 89,289 | 62,094 | 5,795 | 69.5% | **74.4%** |
| May | 94,848 | 66,387 | 5,945 | 70.0% | **74.7%** |

- **Raw pickup ~68–70% (rising); clean reachable-customer pickup ~73–75%.** ~6–7% of dial attempts die at the provider/telecom layer before reaching the customer.
- ✏️ **Supersedes the earlier 62–65% table** (from `cx_ticket_call_mapping`): that source pooled *all* support lines (Auctions Dealer Support, Post Sales feedback campaigns) which dial colder lists and drag pickup to ~64%. **Cross-check:** computing raw pickup on `communication_call_log` across *all* support lines reproduces 62–64% — confirming the gap is **scope, not source** (the two call tables agree on general outbound pickup; they only diverged on callbacks). CX-support-only (`service_unit_id = 1`) is genuinely ~68–70%.
- The `$.callback_data.systemDisposition` path serves **every** call — the `callback_data` envelope is misnamed; it wraps all Ameyo call metadata, not just callbacks. `callType` and `systemDisposition` live there for inbound, outbound and callback alike.

### Callback TAT & slot adherence — rebuilt on communication_call_log (Superset dataset 365 / chart 813 logic)

**Canonical callback metric reconstruction.** ✏️ **Corrected 2026-06-04** — the earlier version of this query used `cx_ticket_call_mapping` + `outbound.manual.dial`, which captures a *later, unrelated* manual agent call (~34h after the request) and understated in-slot ~16×. The real callback dial is the first `communication_call_log` row carrying a callback `systemDisposition`. The callback request time is the `ticket_indicatormapping` (indicator_id = 3) event matched to the callback within ±5 min (this is what the dashboard keys on, not `customer_callback_history.created_at` directly).

```sql
WITH cb AS (
  SELECT
    map.id AS callback_id, ch.ticket_id, ch.preference_type, ch.slot_hour, ch.scheduled_date,
    MINUTES_ADD(map.created_at, 330) AS req_ist,
    CASE WHEN ch.preference_type = 'Next 90 Minutes'
         THEN MINUTES_ADD(map.created_at, 330)
         ELSE SECONDS_ADD(CAST(ch.scheduled_date AS DATETIME), ch.slot_hour * 3600) END AS slot_start,
    CASE WHEN ch.preference_type = 'Next 90 Minutes'
         THEN MINUTES_ADD(MINUTES_ADD(map.created_at, 330), 90)
         ELSE SECONDS_ADD(CAST(ch.scheduled_date AS DATETIME), (ch.slot_hour+1) * 3600) END AS slot_end
  FROM iceberg.sp_ticket_master.customer_callback_history ch
  JOIN iceberg.sp_ticket_master.ticket_indicatormapping map
    ON map.content_id = ch.ticket_id AND map.content_choice = 'TICKET' AND map.indicator_id = 3
   AND ABS(TIMESTAMPDIFF(MINUTE, ch.created_at, map.created_at)) <= 5
),
calls AS (   -- the ACTUAL callback dial, from communication_call_log
  SELECT clm.ticket_id, MINUTES_ADD(clm.created_at, 330) AS call_ist,
         GET_JSON_STRING(clm.meta_data, '$.callback_data.systemDisposition') AS disposition
  FROM iceberg.sp_ticket_master.ticket_communication_call_log_mapping clm
  WHERE GET_JSON_STRING(clm.meta_data, '$.callback_data.systemDisposition')
        IN ('BUSY','NO_ANSWER','CALL_NOT_PICKED','CONNECTED','CALL_DROP','CALL_HANGUP')
),
first_call AS (
  SELECT cb.*, c.call_ist, c.disposition,
         ROW_NUMBER() OVER (PARTITION BY cb.callback_id ORDER BY c.call_ist) AS rn
  FROM cb
  LEFT JOIN calls c ON c.ticket_id = cb.ticket_id AND c.call_ist >= cb.req_ist
)
SELECT *,
       CASE WHEN call_ist >= slot_start AND call_ist < slot_end THEN 1 ELSE 0 END AS is_in_slot,
       TIMESTAMPDIFF(MINUTE, req_ist, call_ist) AS tat_min
FROM first_call WHERE rn = 1
```

**Callback adherence — Mar–May 2026** (rebuilt on communication_call_log; ✏️ **corrected 2026-06-04** — supersedes the earlier 5%-in-slot table, which used the wrong call source):

| Month | Preference | Callbacks | Attempted % | Pickup % | In-slot % | After-slot % | Never-called % | Avg TAT (min) | P80 TAT (min) |
|---|---|--:|--:|--:|--:|--:|--:|--:|--:|
| Mar | Next 90 Minutes | 5,009 | 97.3% | 75.7% | **79.7%** | 17.6% | 2.7% | 92 | 84 |
| Apr | Next 90 Minutes | 13,210 | 98.0% | 77.7% | **79.1%** | 18.9% | 2.0% | 115 | 87 |
| May | Next 90 Minutes | 15,490 | 98.1% | 77.5% | **74.8%** | 23.2% | 1.9% | 134 | 103 |
| Mar | Scheduled Slot | 1,335 | 96.4% | 72.4% | 32.4% | 30.9% | 3.6% | 520 | 901 |
| Apr | Scheduled Slot | 3,287 | 96.9% | 72.3% | 31.4% | 31.5% | 3.1% | 490 | 915 |
| May | Scheduled Slot | 3,924 | 97.9% | 73.1% | 33.6% | 38.3% | 2.1% | 561 | 936 |

**Key findings:**
- **"Next 90 Minutes" is largely honoured: ~75–80% of callbacks are dialled inside the 90-minute window**, ~97–98% are attempted at all, and only ~2% are never called. Median dial is well within the window (P80 ≈ 84–103 min); the mean (92–134 min) is pulled up by the ~18–23% "after-slot" tail.
- **Watch the May slip:** in-slot fell 79.7 → 74.8% and after-slot rose 17.6 → 23.2% as Next-90 volume tripled (5K → 15.5K). Slot adherence is capacity-sensitive — worth a pacing alert if volume keeps climbing.
- **"Scheduled Slot" adherence is the genuinely weak flow (~32% in-slot)** — calls split roughly evenly before / in / after the chosen hour, with a long tail (P80 ≈ 15h). This, not the 90-min flow, is the real slot-adherence gap.
- Source of truth = `ticket_communication_call_log_mapping` callback `systemDisposition` (Superset dataset 365 / chart 813 "callback pickup rate"). Do **not** use `cx_ticket_call_mapping` / `outbound.manual.dial` for callback timing.

### General response time (ticket created → first outbound **attempt**, business-hours clocked)

✏️ **Redefined 2026-06-04.** The earlier table ("ticket created → first **connected** outbound call", wall-clock) was conceptually broken and bundled four different things into one number: (1) **non-operational hours** — a ticket born at 05:46 books 4hr of dead overnight time before agents start; (2) **connect not attempt** — it blamed us for the *customer* not picking up; (3) **outbound-only blindness** — ignored inbound/message first contacts; (4) it was built off `support_tickets.first_connected_outbound_call_start_ts`, a proxy that also pulls in **non-manual connects (AI / voicebot / campaign dials)**, so its "reach" looked inflated (76–90%). Three different cuts of it (proxy 2hr P50, manual-connect 19hr, raw first-connect ~7hr) all disagreed — a sign the metric was never well-defined.

**Corrected definition:** response time = **business-minutes** from `ticket_created` to the **first outbound `manual.dial` attempt** (any *customer-facing* disposition — `CONNECTED`/`NO_ANSWER`/`BUSY`/`CALL_NOT_PICKED`/`CALL_HANGUP`/`CALL_DROP`; provider/telecom failures excluded) **after** ticket creation. **Business hours = 10:00–18:00 IST, 7 days/week (480 min/day)** — the clock pauses outside that window. This measures *agent-controllable latency* (did we **try** within working hours), not customer pickup. Source = `ticket_communication_call_log_mapping` (communication_call_log), scoped `service_unit_id = 1`. "Outbound-touch %" = share of created tickets that got ≥1 such attempt (tickets handled purely on inbound / message / AI never get an outbound dial, so this is a touch rate, not a contact-success rate).

⚠️ **Timezone anchor (read before recomputing):** `ticket_created_ts` is on `sp_entity_store.support_tickets` and is **already IST** — anchor on it **directly, no `MINUTES_ADD`**. Only the `iceberg.sp_ticket_master` call tables are UTC, so `+330` applies **only** to `clm.created_at` (the call time). See the UTC/IST rule earlier in this doc.

| Month (IST) | Query | Outbound-touch % | P50 (bus-hrs) | P90 (bus-hrs) |
|---|---|--:|--:|--:|
| May | Warranty | 79.6% | 1.1 | 3.2 |
| May | RC | 85.4% | 1.0 | 2.1 |
| May | RC - Seller | 93.5% | 0.9 | 1.8 |
| May | Payments - Seller | 86.9% | 1.0 | 1.8 |
| May | Insurance | 90.9% | 0.9 | 2.1 |
| May | Spinny Benefits | 72.8% | 1.5 | 4.1 |
| May | RSA | 83.2% | 0.7 | 2.3 |
| May | **All CX** | **85.5%** | **1.0** | **2.6** |

**Reading the numbers** (1 business day = 8 bus-hrs): during operating hours CX dials **fast** — median first-attempt ~**1.0 bus-hr**, overall P90 ~**2.6 bus-hrs** (≈ a third of a working day). Independently cross-checked: of tickets *created in-hours*, ~**90% (RC) / 75% (Warranty)** get a first attempt within **2 wall-clock hours**. The large *wall-clock* P90 (~15–16h) is almost entirely **overnight waiting** — tickets born after 18:00 sit until 10:00 — which the business-hours clock correctly strips out. Lowest outbound-touch: **Spinny Benefits 72.8%**, **Warranty 79.6%**; highest **RC-Seller 93.5% / Insurance 90.9%**. ✏️ The *callback* flow is **not** the gap (~75–80% of Next-90 callbacks in-slot); the remaining gap is the weaker **Scheduled-Slot** callback flow (~32% in-slot).

✏️ **Corrected 2026-06-04 (v2 — timezone fix).** An earlier version of *this* table (touch **58.3%**, P50 **3.8**, P90 **15.7** bus-hrs) was wrong: the ticket-creation anchor used `MINUTES_ADD(ticket_created_ts, 330)` on the **already-IST** entity-store column, double-shifting it +5.5h. That pushed the "first attempt **after** creation" cutoff 5.5h late — fast-handled tickets fell to "unreached" (depressing touch %) and slower follow-up calls got picked (inflating P50/P90). Fixed by anchoring on `ticket_created_ts` directly. The `iceberg`-side response-time/pickup/callback queries elsewhere in this doc are unaffected (they only shift `created_at`, which is genuinely UTC).

---

## CSAT Deep-Dive — May 2026 baseline (service_unit_id = 1 only)

Source: `iceberg.sp_ticket_master.ticket.cx_rating`, non-merged, **`service_unit_id = 1` (SUPPORT)**,
ticket creation Apr–May '26.

### Overall trend (Jan–May 2026, SUPPORT only)

| Month | Non-merged tix | Rated | Rated % | CSAT % |
|---|--:|--:|--:|--:|
| Jan | 21,149 | 2,642 | 12.5% | 67.6% |
| Feb | 18,943 | 2,284 | 12.1% | 65.5% |
| Mar | 19,223 | 2,465 | 12.8% | 66.4% |
| Apr | 20,298 | 2,504 | 12.3% | 69.3% |
| May | 19,716 | 1,728 | 8.8% | **75.1%** |

CSAT trending UP **+7.5 pts in 5 months** (67.6 → 75.1). Rated % is stable around 12% Jan–Apr,
drops to 8.8% in May due to the methodology effect (May ratings still arriving for late-May tickets).

### CSAT by query type (Apr–May '26, rated ≥ 30)

| Query | May tix | Rated % | May CSAT | Trend (Mar→May) |
|---|--:|--:|--:|---|
| **Payments - Seller** | 2,933 | 11.0% | **87.0%** | 78.5 → 87.0 (+8.5) |
| HSRP | 371 | 9.7% | 83.3% | 81.0 → 83.3 |
| Insurance | 1,069 | 8.1% | 80.5% | **58.3 → 80.5 (+22.2)** |
| Fastag | 558 | 10.6% | 78.0% | 77.8 → 78.0 |
| Other | 418 | 7.7% | 78.1% | 72.1 → 78.1 |
| Document | 336 | 9.2%* | 77.4%* | 78.4 → 77.4 (Apr*) |
| Spinny Benefits | 682 | 10.4% | 74.6% | 75.4 → 74.6 |
| Miscellaneous | 912 | 8.3% | 73.7% | 69.9 → 73.7 |
| RSA | 1,431 | 6.1% | 73.9% | 69.8 → 73.9 |
| Seller-Miscellaneous | 1,060 | 6.9% | 72.6% | 64.6 → 72.6 |
| Payments | 660 | 9.2% | 72.1% | 65.8 → 72.1 |
| RC - Seller | 1,771 | 5.9% | 72.4% | **52.7 → 72.4 (+19.7)** |
| RC | 2,479 | 8.6% | 71.7% | 62.1 → 71.7 (+9.6) |
| **Warranty** | 5,402 | 9.5% | **67.7%** | 61.4 → 67.7 |

**Standouts:**
- **Insurance** is the biggest gainer (+22 pts in 2 months) — driven by the Insurance AI Agent launch (bot 1012, May '26)?
- **RC-Seller** +20 pts — explained by the RC pipeline overhaul (resolution time 308h → 38h, see earlier section)
- **Warranty** is the volume leader and lowest CSAT (67.7%) — pulls the overall number down

### CSAT by handler (Apr–May, service_unit_id=1, named bots)

| Handler | Tix | Rated % | CSAT % |
|---|--:|--:|--:|
| 987 CallKaro (Pre-delivery) | 197 | 7.1% | **92.9%** |
| 696 Circuit | 40 | 2.5% | 100.0% (n=1) |
| 1012 Insurance AI | 4 | 75.0% | 100.0% (n=3) |
| **Human** | 38,190 | 10.6% | **72.2%** |
| 981 AI Agent (RC) | 791 | 10.9% | 60.5% |
| 998 Warranty AI | 743 | 8.6% | **46.9%** |

### Warranty CSAT by handler × verdict — the key finding (corrected)

| Handler × verdict | Rated | CSAT % |
|---|--:|--:|
| Human × **denied** | 402 | **69.7%** |
| Human × covered | 603 | 63.2% |
| Human × needs_inspection | 186 | 65.1% |
| Human × no_verdict | 141 | 58.2% |
| **WarrantyAI × denied** | 63 | **46.0%** |

When humans deliver a denial: **70% CSAT**. When the bot delivers the same denial: **46% CSAT**.
**~24-point gap.** The denial isn't the problem — the *bot's denial delivery* is the problem.

Counter-intuitively, **covered tickets have LOWER CSAT than denied tickets** when handled by
humans (63% vs 70%). Likely because covered → repair workflow → time + failure modes; denied → fast,
clear answer that the customer accepts.

### CSAT by resolution speed

| Resolution bucket | Rated | CSAT % |
|---|--:|--:|
| 0–1 hr | 487 | 68.4% |
| 1–6 hr | 525 | 75.6% |
| **6–24 hr** | 610 | **78.0%** ← peak |
| 1–3 days | 969 | 77.2% |
| 3–7 days | 928 | 69.5% |
| >7 days | 766 | 62.3% |
| Unresolved | 9 | 66.7% |

**CSAT peaks at 6–24h resolution.** Too-fast (<1h) and too-slow (>7d) both depress CSAT. Sub-1h
resolutions may be "marked resolved without really helping" cases.

### CSAT by customer type (service_unit_id=1)

| Customer | Tix | Rated % | CSAT % |
|---|--:|--:|--:|
| BUYER | 28,126 | 11.0% | 68.3% |
| **SELLER** | 11,290 | 9.8% | **81.1%** |
| OTHER | 549 | 8.6% | 70.2% |

⚠️ **DEALER tickets are essentially absent from CX SUPPORT.** When we restrict to
`service_unit_id = 1`, dealer tickets effectively drop out — their 21,876 tickets/2mo live in the
AUCTION service unit (`service_unit_id = 5`), not SUPPORT. Earlier "DEALER 0.1% rated" finding was a
filter artefact, not a survey gap. Treat dealer feedback as an AUCTION-domain analysis, not CX.

**SELLER CSAT (81%) vs BUYER (68%) — 13-point gap, real and consistent.** Sellers experience CX
support more positively. Possible drivers: seller tickets are mostly RC/payment/document workflow
(clearer outcomes), while buyer tickets skew warranty (denials, repair workflow).

### CSAT by source

| Source | Tix | Rated % | CSAT % |
|---|--:|--:|--:|
| SPINNY_HELP_AND_SUPPORT | 26,082 | 11.3% | 72.2% |
| VOC | 4,787 | 10.9% | 70.8% |
| WEBSITE_INBOUND | 2,898 | 9.1% | 70.8% |
| RSA | 3,434 | 8.0% | 72.8% |
| DEMAND_INTERNAL | 472 | 8.7% | 73.2% |
| CREATED_MANUALLY | 4,509 | 5.4% | **66.9%** |
| CONTACT_US | 404 | 11.1% | 66.7% |

Manually-created tickets (agent-initiated, often outbound issues) have the lowest CSAT, suggesting
agent-side issue identification doesn't map well to what customers actually want addressed.

---

## CX Master Dashboard (Superset 126) — Chart Reference

URL: https://superset.mngt.ispinnyworks.in/superset/dashboard/126/ · 26 charts · 5 datasets.

### Datasets feeding this dashboard

| ID | Name | What it is | Primary keys |
|--:|---|---|---|
| **206** | `cx business` | **Main ticket dataset.** Virtual: `sp_product.cx_tickets cxt LEFT JOIN sp_entity_store.support_tickets st ON cxt.ticket_id = st.ticket_id`. **Already CX-scoped** because `sp_product.cx_tickets` is built upstream from `service_unit_id = 1` (+ ~1.5% LUXURY tail); FEEDBACK and AUCTION are excluded at the ETL layer. One row per ticket with `ever_merged_flag`, `is_closed`, `cx_rating`, `resolution_time_hrs`, `fcr_flag`, `is_circuit_resolved`, `is_reopened`, `same_month_closed_flag`, `query`, `sub_query`, `source`, `customer_type`, `CITY`, `latest_user`, `first_agent_assigned_id`, `latest_agent_assigned_id`, `customer_phone`, `created_at`, `closing_time`. Powers ~20 of the 26 charts. |
| 202 | `cx master 1` | **Deal-level buyer funnel.** Virtual: aggregates tickets per delivered deal. Columns: `deal_id`, `delivery_time`, `number_of_tickets`, `days_to_first_ticket`. ⚠️ Excludes the "New delivery" sub-query from the ticket count. |
| 203 | `CX 3` | **Monthly cars-procured volume.** Virtual: `COUNT(DISTINCT lead_id)` from `sp_web.listing_lead` grouped by `stock_in_date`. Simple counts only. |
| 213 | `Agents ndoa` | **Agent-level NDOA per ticket.** Virtual: joins `sp_ticket_master.ticket` to `ticket_indicatormapping` filtering on `indicator_id = 15` to count NDOA breaches. Has `ticket_id`, `cx_rating`, `agent_email`, `NDOA_count`, `created_ist`. |
| 214 | `Seller tickets breakdown` | **Stockin-level seller funnel.** Virtual: aggregates tickets per stocked-in car. Columns: `registration_no`, `stockin_date`, `number_of_tickets`, `days_to_first_ticket`. |

### Default ("running numbers") — quote these unless asked otherwise

⚠️ **Default convention for any CX question, unless explicitly overridden:**

```sql
FROM sp_product.cx_tickets
WHERE ever_merged_flag = 0                          -- unique (non-merged) tickets only
  AND created_at >= '<month_start>' AND created_at < '<next_month_start>'
-- NO same_month_closed_flag filter
-- NO source filter
```

This is the **"all unique support tickets created in that month"** view. Confirmed by Arpit
Dhankani 2026-05-29. Always quote these numbers as the default response to any CX question.

**Use the special-cut filters ONLY when explicitly asked:**

| If user asks for… | Add this filter |
|---|---|
| Same-month-closed numbers / efficiency view | `AND same_month_closed_flag = 1` |
| **App tickets** (raised through the Spinny Help & Support app flow) | `AND source = 'SPINNY_HELP_AND_SUPPORT'` |

### The dual-tracking convention — every CX metric has TWO views

⚠️ **Every metric is tracked two ways** (confirmed by Arpit Dhankani, 2026-05-29):

| View | Gating clause | What it answers |
|---|---|---|
| **Same-month-closed** | `same_month_closed_flag = 1` | "Of tickets created **AND** closed within the same calendar month, how did we do?" — efficiency view |
| **All-created-in-month** | (no closure-timing gate) | "Of tickets created in this month, what's the final outcome (whenever it closes)?" — outcome view |

`same_month_closed_flag = 1` ⇔ `DATE_TRUNC('month', created_at) = DATE_TRUNC('month', closing_time) AND ever_merged_flag = 0`.

**Which dashboard chart uses which view:**
- **Chart 463 (Reporting numbers)**: same-month-closed view for ALL its metrics — this is the leadership reporting view.
- **Chart 448 (Overview)**: all-created-in-month view.
- **KPI tiles (474–478)**: all-created-in-month view, scoped to current month via dashboard time filter.

**For matured months the two views diverge significantly** (March 2026 example):

| Metric | All-created | Same-month-closed | Δ |
|---|--:|--:|--:|
| Unique tix | 19,550 | 17,382 | -2,168 (11%) |
| Resolution % | **98.32%** | 88.91% | -9.4 pts |
| Avg resolution hrs | 104.88 | **65.34** | -39.5 hrs |
| Rated % | 12.97% | 12.70% | -0.3 |
| CSAT | 66.63% | 67.56% | +0.9 |
| FCR % | 38.38% | 41.80% | +3.4 |

Mechanism: ~11% of March tickets took longer than 30 days to close. Those slow-resolution tickets:
- Drag down "Resolution %" in the same-month view but get included in "All-created"
- Inflate "Avg resolution hrs" massively in the all-created view (long tail)
- Push down FCR in the all-created view (slow tickets are rarely first-call-resolved)

**In-progress months converge** (May 2026, partial through May 28):

| Metric | All-created | Same-month-closed | Δ |
|---|--:|--:|--:|
| Resolution %, Avg res, Rated %, CSAT, FCR | 84.27 / 63.06 / 10.45 / 75.07 / 43.25 | (identical) | 0 |

Because for in-progress months almost no tickets have closed beyond the month boundary yet — so `is_closed = 1` is functionally equivalent to `same_month_closed_flag = 1` for the rate-metric denominators.

**Rule of thumb:**
- For **current-month operational tracking**, either view works (they converge).
- For **historical reporting / trend analysis**, the two views tell different stories — pick deliberately:
  - "How efficient was the team this month?" → same-month-closed (chart 463 view)
  - "How well did this month's intake ultimately get resolved?" → all-created (chart 448 / KPI-tile view)

### Canonical metric formulas — both variants (from dataset 206)

All formulas exclude merged tickets via `ever_merged_flag = 0`. Dataset 206 is **already CX-scoped**
upstream (via `sp_product.cx_tickets` excluding FEEDBACK / AUCTION at ETL) — so no `service_unit_id`
filter is needed at the chart layer. The `query IN (17 values) AND source IN (11 values)` filter on
the Overview chart is a *further* restriction within CX (excludes legacy / deprecated query types
and edge-case channels).

For every metric below, two variants are listed: **(A) all-created-in-month**, **(B) same-month-closed**.

| Metric | Formula |
|---|---|
| **Unique tickets** | `COUNT(CASE WHEN ever_merged_flag = 0 THEN 1 END)` |
| **% merged** | `100 * COUNT(DISTINCT CASE WHEN ever_merged_flag = 1 THEN ticket_id END) / COUNT(DISTINCT ticket_id)` |
| **Same-month closed** | `COUNT(CASE WHEN DATE_TRUNC('month', created_at) = DATE_TRUNC('month', closing_time) AND ever_merged_flag = 0 THEN 1 END)` — or the pre-computed `same_month_closed_flag = 1` |
| **Same-month resolved %** | `ROUND(100.0 * COUNT(CASE WHEN same_month_closed_flag = 1 AND ever_merged_flag = 0 THEN ticket_id END) / NULLIF(COUNT(CASE WHEN ever_merged_flag = 0 THEN ticket_id END), 0), 1)` |
| **Closed tickets** | `COUNT(CASE WHEN is_closed = 1 AND ever_merged_flag = 0 THEN 1 END)` |
| **Resolution %** | `ROUND(100.0 * COUNT(CASE WHEN is_closed = 1 AND ever_merged_flag = 0 THEN ticket_id END) / NULLIF(COUNT(CASE WHEN ever_merged_flag = 0 THEN ticket_id END), 0), 1)` |
| **Pending tickets** | `COUNT(DISTINCT CASE WHEN ever_merged_flag = 0 AND is_closed = 0 THEN ticket_id END)` |
| **CSAT** | `(COUNT(CASE WHEN cx_rating IN (4,5) THEN 1 END) * 100.0) / NULLIF(COUNT(CASE WHEN cx_rating > 0 THEN 1 END), 0)` |
| **CSAT (circuit-resolved subset)** | Same numerator/denominator both filtered to `is_circuit_resolved = 1` — measures CSAT specifically for the Circuit-handled flow |
| **Rated %** | `100.0 * COUNT(DISTINCT CASE WHEN cx_rating > 0 AND is_closed = 1 AND ever_merged_flag = 0 THEN ticket_id END) / COUNT(DISTINCT CASE WHEN is_closed = 1 AND ever_merged_flag = 0 THEN ticket_id END)` |
| **Rated tickets (count)** | `COUNT(CASE WHEN cx_rating >= 1 THEN 1 END)` |
| **Avg / Median / P90 resolution time (hrs)** | `AVG`, `percentile_approx(..., 0.5)`, `percentile_approx(..., 0.9)` all applied to `CASE WHEN ever_merged_flag = 0 AND is_closed = 1 THEN resolution_time_hrs END` |
| **FCR % (Overview variant)** | `100.0 * SUM(CASE WHEN ever_merged_flag = 0 AND fcr_flag = 'FCR' THEN 1 ELSE 0 END) / NULLIF(SUM(CASE WHEN ever_merged_flag = 0 AND fcr_flag IN ('FCR','NON_FCR') THEN 1 ELSE 0 END), 0)` |
| **FCR % (KPI-tile variant, chart 478)** | Same numerator/denominator but gated on `is_closed = 1` instead of `ever_merged_flag = 0` — **subtly different metric, possible source of dashboard inconsistency** |
| **Reopened tickets** | `SUM(is_reopened)` |
| **Circuit-resolved tickets** | `SUM(is_circuit_resolved)` |
| **AI assigned tickets** | `COUNT(CASE WHEN first_agent_assigned_id IN (981, 987, 998) THEN ticket_id END)` ⚠️ Bot ID list is stale (missing 1010, 1012, 1015) |
| **AI closed tickets** | `COUNT(CASE WHEN latest_agent_assigned_id IN (981, 987, 998) THEN ticket_id END)` |

### Standard support-volume filter (applied on Overview + last-30-days charts)
```sql
WHERE query IN ('HSRP','Welcome call','RC - Seller','Warranty','Missed Call - Existing','RSA',
    'Spinny Benefits','Insurance','Seller-Miscellaneous','Fastag','Miscellaneous','RC',
    'Payments - Seller','Document','Other','Missed Call - New','Payments')
  AND source IN ('SPINNY_HELP_AND_SUPPORT','CREATED_MANUALLY','WEBSITE_INBOUND','PHONE_CALL','VOC',
    'RSA','CONTACT_US','DEMAND_INTERNAL','SERVICING_SUPPORT','SP_INSURANCE','EMAIL')
```

### All 26 charts — purpose and formula

| # | ID | Title | Viz | DS | Time scope | Group by | What it answers |
|--:|--:|---|---|--:|---|---|---|
| 1 | 448 | **Overview dashboard** | pivot_table | 206 | All time, P1M | `created_at` (P1M) | The big monthly KPI grid (19 metrics × month): volume mix (total/merged/unique/% merged), resolution mix (closed/same-month/avg/median/P90), quality (CSAT, rated %, FCR %, reopened, circuit-resolved, pending), AI exposure (AI assigned/closed). Standard support-volume filter applied. |
| 2 | 449 | **last 30 days** | pivot_table | 206 | Last month | `query` | Same metric set as Overview, but sliced by `query` for one month — to spot which query types pull CSAT/resolution/FCR up or down. Standard support-volume filter applied. |
| 3 | 450 | **Cars sold per month** | timeseries bar | 202 | All time | `delivery_time` | Monthly delivered cars (denominator for buyer-ticket funnel). Filters by month list. |
| 4 | 451 | **Monthly summarised view** | timeseries bar | 206 | All time | `created_at` (P1M) | Per-month: Total Tickets (raw), unique tickets (non-merged), resolved tickets (non-merged + closed). NO support-volume filter — wider denominator than Overview. |
| 5 | 452 | **Customer wise ticket split** | pie | 206 | Last month | `customer_type` | BUYER / SELLER / DEALER / OTHER share of last month's tickets. |
| 6 | 453 | **source wise split** | pie | 206 | Last month | `source` | Which channels generated last month's tickets. |
| 7 | 454 | **Cars procured per month** | timeseries bar | 203 | All time | `stock_in_date` | Monthly cars stocked in (denominator for seller-ticket funnel). Different scope from chart 3 — supply-side, not demand-side. |
| 8 | 460 | **tickets per customer** | pivot_table | 206 | All time | `created_at` | Monthly: # tickets, # unique customers (by `customer_phone`), tickets-per-customer ratio. Repeat-contact intensity metric. |
| 9 | 461 | **City wise breakdown** | treemap | 206 | Last month | `CITY` | City-share of ticket volume (treemap rectangles sized by ticket count). |
| 10 | 462 | **KAM wise breakdown** | pivot_table | 206 | Last month | `latest_user` | Per-agent (latest assignee): total / merged / unique / closed tickets, avg rating (raw 1–5), CSAT %, avg resolution time. Agent-performance scorecard. |
| 11 | 463 | **Reporting numbers** | pivot_table | 206 | All time | `created_at` (P1M) | The reporting-cadence KPIs — **all gated on `same_month_closed_flag = 1`** (only count tickets created and resolved in the same calendar month). Metrics: Total / Merged / Same-month-closed, Same-month-resolved %, Avg resolution time (hrs), Rated %, CSAT, FCR %. This is the chart the leadership reports off — same-month-closed is the canonical resolution scope. |
| 12 | 464 | **Buyer tickets funnel** | pivot_table | 202 | All time | `delivery_time` | **Deal-level post-delivery support intensity.** Per delivery month: Cars sold, total tickets, Avg/Median/P90 tickets-per-car, Avg/Median/P90 days-to-first-ticket, % deals with 0 tickets, count of deals whose first ticket landed in 0–15 / 15–30 / 30–60 / 60+ day buckets. Excludes the "New delivery" sub-query from the count (per dataset 202's SQL). |
| 13 | 469 | **Agent wise NDOA** | pivot_table | 213 | All time | `agent_email` | Per agent: total tickets, rated, overall CSAT, **% NDOA-breached tickets**, count of NDOA-breached tickets, rated NDOA-breached count, **CSAT on NDOA-breached subset**. Filters out `test.user@spinny.com` and NULL emails. NDOA = "Next Date Of Action" SLA the agent commits to (`indicator_id = 15`). |
| 14 | 470 | **Seller tickets breakdown** | pivot_table | 214 | All time | `stockin_date` | **Stockin-level post-procurement support intensity.** Mirrors chart 12 but for the seller funnel — Total cars procured, total tickets, Avg/Median/P90 tickets-per-car, Avg/Median/P90 days-to-first-ticket, % cars with 0 tickets, 0–15 / 15–30 / 30–60 / 60+ day buckets. |
| 15 | 471 | **cars sold till date (this month)** | big number | 202 | (no filter → all time aggregate) | – | Total `COUNT(deal_id)` headline. Despite the title it's NOT filtered to current month at the chart level — the filter must be coming from the dashboard's time selector. |
| 16 | 472 | **Total tickets till date (same month)** | big number | 206 | (chart-level: no filter) | – | Total ticket count headline. Same caveat re: month-scope. |
| 17 | 473 | **Total unique tickets till date (same month)** | big number | 206 | (chart-level: no filter) | – | Unique (non-merged) ticket count headline. |
| 18 | 474 | **CSAT** | big number | 206 | (no filter) | – | KPI tile: `(COUNT(cx_rating IN (4,5)) * 100) / NULLIF(COUNT(cx_rating > 0), 0)`. ⚠️ **This tile does NOT exclude merged tickets** — slightly looser denominator than the Overview chart's CSAT metric. Watch the small drift. |
| 19 | 475 | **Rated %** | big number | 206 | (no filter) | – | KPI tile: rated-among-closed-non-merged. |
| 20 | 476 | **Avg resolution time** | big number | 206 | (no filter) | – | KPI tile: `AVG(resolution_time_hrs)` — ⚠️ does NOT exclude merged / open tickets in the formula. Looser than the Overview's avg resolution metric. |
| 21 | 477 | **Resolution percentage** | big number | 206 | (no filter) | – | KPI tile: closed-over-unique. |
| 22 | 478 | **FCR%** | big number | 206 | (no filter) | – | KPI tile: ⚠️ **uses `is_closed = 1` gating** instead of the Overview's `ever_merged_flag = 0` gating. Different cohort — beware of inconsistency between this tile and the Overview's FCR. |
| 23 | 479 | **Source split (current month)** | pie | 206 | (current month via dashboard filter) | `source` | Same dimension as chart 6 but for current month. |
| 24 | 480 | **Customer split (current month)** | pie | 206 | (current month) | `customer_type` | Same dimension as chart 5 but for current month. |
| 25 | 481 | **Cars procured (this month)** | big number | 214 | (current month) | – | `COUNT(registration_no)` headline from dataset 214. |
| 26 | 482 | **Query wise split (current month)** | pie | 206 | (current month) | `query` | Top-level query mix for the current month. |

### Validation — my queries vs dashboard (May 2026, run 2026-05-29)

Reconciled every headline metric against the live dashboard. **All values match exactly.**

**KPI tiles + Reporting numbers (May '26, no literal-filter — chart 463, 472–478):**

| Metric | Dashboard | My query | Δ |
|---|--:|--:|--:|
| Total tickets (chart 472) | 24,044 | 24,044 | 0 |
| Unique tickets (chart 473) | 20,310 | 20,310 | 0 |
| Merged (chart 463) | 3,734 | 3,734 | 0 |
| Same-month-closed (chart 463) | 17,115 | 17,115 | 0 |
| Same-month-resolved % (chart 463) | 84.3% | 84.3% | 0 |
| Resolution % (chart 477) | 84.3% | 84.3% | 0 |
| Avg resolution hrs (chart 476) | 63.06 | 63.06 | 0 |
| Avg resolution hrs (chart 463 — same-month-gated) | 63.06 | 63.06 | 0 |
| Rated % (chart 475) | 10.45% | 10.45% | 0 |
| CSAT (chart 474) | 75.07% | 75.07% | 0 |
| FCR % (chart 478) | 43.25% | 43.25% | 0 |

**Overview chart 448 (May '26, with literal-filter on query + source):**

| Metric | Dashboard | My query | Δ |
|---|--:|--:|--:|
| Total | 23,961 | 23,961 | 0 |
| Merged | 3,730 | 3,730 | 0 |
| Unique | 20,231 | 20,231 | 0 |
| CSAT | 75.08% | 75.08% | 0 |
| Rated % | 10.48% | 10.48% | 0 |
| Avg resolution | 63.27 | 63.27 | 0 |
| Median resolution | 24.59 | 24.59 | 0 |
| P90 resolution | 175.01 | 175.01 | 0 |
| FCR % | 43.24% | 43.24% | 0 |
| Reopened | 1,295 | 1,295 | 0 |
| Circuit resolved | 0 | 0 | 0 |
| AI assigned | 1,310 | 1,310 | 0 |
| AI closed | 459 | 459 | 0 |

**Chart 463 — full Jan-May 2026 time-series reconciliation** (all values matched exactly):

| Month | Total | Merged | SMC | Res % | Avg res | Rated % | CSAT | FCR % |
|---|--:|--:|--:|--:|--:|--:|--:|--:|
| Jan '26 | 25,463 | 4,327 | 18,919 | 89.4 | 68.61 | 12.4 | 69.0 | 42.97 |
| Feb '26 | 22,714 | 3,784 | 16,924 | 89.4 | 63.82 | 11.9 | 66.5 | 41.54 |
| Mar '26 | 23,537 | 3,987 | 17,382 | 88.9 | 65.34 | 12.7 | 67.6 | 41.80 |
| Apr '26 | 24,859 | 4,266 | 18,267 | 88.7 | 63.96 | 12.4 | 70.6 | 41.56 |
| May '26 | 24,044 | 3,734 | 17,115 | 84.3 | 63.06 | 10.5 | 75.1 | 43.25 |

### Observations from reconciliation

1. **Overview chart 448 vs Reporting chart 463**: differ by **83 tickets** in May (23,961 vs 24,044) — that gap is exactly the tickets in `sp_product.cx_tickets` whose `query` or `source` is outside the 17 / 11 literal lists on the Overview chart (e.g. `FASTag Recharge`, `Missed Call - Luxury`).
2. **CSAT drift between charts is negligible**: chart 474 (no `ever_merged_flag` filter), chart 448 (no filter on CSAT either, despite my earlier doc note — verified by SQL inspection), chart 463 (same-month-closed-gated) all show 75.07–75.10 for May. Drift is < 0.05 pts because in practice `cx_rating > 0` is only populated on closed-and-rated tickets, which are mostly non-merged AND same-month-closed.
3. **Avg resolution time drift is real but small**: chart 476 (no filter) = 63.06, chart 448 (non-merged + closed) = 63.27. ~0.2-hour difference (~0.3%) — explained by merged tickets that still have `resolution_time_hrs` populated.
4. **The same-month-resolved % declines steeply in May (88.7 → 84.3)** — partly real ageing of recent tickets that haven't yet closed by the time we measured (May was through May 28), partly a real trend in same-month closure rates.
5. **CSAT trend is going UP across the leadership view too** (chart 463): 69.0 → 66.5 → 67.6 → 70.6 → **75.1** (Jan→May). Real, +6 pts in 5 months.
6. **FCR is the only metric trending DOWN**: 42.97 → 41.54 → 41.80 → 41.56 → 43.25. Roughly flat. Worth a deeper look — the AI bots' lower solo-resolve rates may be diluting it.

### Dashboard quirks worth knowing

1. **The "AI assigned" / "AI closed" metrics are stale.** Hardcoded bot IDs `(981, 987, 998)` miss the 3 newer bots launched May '26 (1010, 1012, 1015). These KPIs undercount AI exposure by ~250 tickets/month and growing.
2. **CX scope is enforced upstream, not via Superset filters.** `sp_product.cx_tickets` is built from `service_unit_id = 1` only (+ tiny LUXURY tail) at the SQLMesh ETL layer; dataset 206 inherits this scope. The `query` + `source` literal lists on the Overview chart are a *further* restriction within CX (legacy/deprecated query types and edge-channel exclusions), not the primary scope filter. Implication: if a new query name is added to the support flow, it'll show up correctly on most charts but will be excluded from the Overview / last-30-days charts until those literal lists are updated.
3. **CSAT KPI tile (chart 474) vs Overview CSAT — small drift.** The tile does NOT apply `ever_merged_flag = 0`; the Overview metric does. Same for chart 476 (Avg resolution time).
4. **FCR formula disagreement.** Chart 463 (Reporting numbers), chart 448 (Overview), and chart 478 (KPI tile) use **three different gating clauses** for FCR: `same_month_closed_flag = 1 AND ever_merged_flag = 0`, `ever_merged_flag = 0`, and `is_closed = 1` respectively. Numbers shown on the dashboard for FCR can therefore vary by 1–3 pts depending on which chart you read.
5. **Chart 451 (Monthly summarised view) has no support-volume filter** — so its "Total Tickets" line is the full superset including FEEDBACK / AUCTION query types, larger than the Overview chart's denominator.
6. **Datasets 202 and 214 use deal/stockin keys, not ticket_id.** Joining their numbers to ticket-level analyses requires matching on `delivery_time` / `stockin_date` cohorts, not row-level merges.

---

## GA4 — Help & Support Click Events (`iceberg.sp_google_analytics.ga4_master_data`)

### Event naming convention

```
help_supp_{side}_help_{level}_issue_click
             │              │
             │              └─ l1 = first concern selection
             │                 l2 = second (deeper) selection
             └─── buy  = buyer
                  sell = seller
```

### Events that exist (confirmed May 2026)

| Event | Side | Level | Monthly vol (May '26) | Exists? |
|---|---|---|--:|---|
| `help_supp_buy_help_l1_issue_click` | Buyer | L1 | 38,590 | ✅ |
| `help_supp_buy_help_l2_issue_click` | Buyer | L2 | 5,849 | ✅ |
| `help_supp_sell_help_l1_issue_click` | Seller | L1 | 19,798 | ✅ |
| `help_supp_sell_help_l2_issue_click` | Seller | L2 | — | ❌ **Does not exist** |

⚠️ **Sellers have no L2 event.** For seller query analysis, `help_supp_sell_help_l1_issue_click` is
the only click signal available.

### `eventvalue` → ticket column mapping (confirmed via live data)

| GA4 event | `eventvalue` field | Maps to ticket column | Notes |
|---|---|---|---|
| L1 click (`$.type`) | `"RC"` | `query_name` | Seller prefix: `"Seller-RC"` → `"RC - Seller"`, `"Seller-Payments"` → `"Payments - Seller"` |
| L1 click (`$.concern`) | `"RC Status"` | `primary_sub_query_name` | Strings match exactly |
| L2 click (`$.concern`) | `"RTO receipt not received"` | `primary_query_heading_name` | Strings match exactly |

**`_seen` events carry only `$.type`** — no concern dimension; not used for click/ticket comparisons.

### Primary use case — clicks vs app tickets at query and sub-query level

The standard analysis tracks **GA4 clicks alongside app tickets** for the same query or
sub-query bucket — not as a join, but as two parallel series read in the same view:

| Analysis level | GA4 signal | Ticket signal (app only) |
|---|---|---|
| **Query** | L1 clicks grouped by `$.type` | `WHERE ticket_source_name='SPINNY_HELP_AND_SUPPORT'` grouped by `query_name` |
| **Sub-query** | L2 clicks grouped by `$.concern` | Same filter, grouped by `primary_query_heading_name` |

**What you infer from the comparison:**
- Both rising → growing pain point
- Clicks spike, tickets lag → early warning (GA4 leads ticket creation by days)
- Clicks high, tickets low → customers abandoning after clicking, not raising tickets (friction or self-serve)
- Tickets high, clicks low → tickets coming from non-app channels (calls, WhatsApp) — the app isn't surfacing this issue

### Query-level: L1 clicks vs app tickets (May 2026)

| Query | L1 clicks (GA4) | App tickets | Tickets / click |
|---|--:|--:|--:|
| RC | 20,216 | 1,899 | 0.09 |
| Seller-RC / RC - Seller | 9,863 | 1,357 | 0.14 |
| Seller-Payments / Payments - Seller | 7,272 | 2,232 | 0.31 |
| Insurance | 5,868 | 728 | 0.12 |
| Spinny Benefits | 3,446 | 607 | 0.18 |
| Payments | 2,890 | 468 | 0.16 |
| Seller-Miscellaneous | 2,663 | 590 | 0.22 |
| Miscellaneous | 2,265 | 643 | 0.28 |
| Document | 1,854 | 298 | 0.16 |
| Fastag | 1,283 | 459 | 0.36 |
| HSRP | 662 | 275 | 0.42 |

### Full buyer help funnel (May 2026, app)

| Step | Event | Count |
|---|---|--:|
| Help screen seen | `help_supp_buy_get_help_l1_seen` | 144,647 |
| L1 concern clicked | `help_supp_buy_help_l1_issue_click` | 38,590 |
| L2 screen seen | `help_supp_buy_get_help_l2_seen` | 10,980 |
| L2 concern clicked | `help_supp_buy_help_l2_issue_click` | 5,849 |
| Ticket opened | `help_supp_buy_ticket_opened` | 6,263 |

Seller: L1 only — no L2 click event exists for sellers.

### Key columns

| Column | Use |
|---|---|
| `eventvalue` | JSON → `$.type` (query) and `$.concern` (sub-query) |
| `event_date` | `date` in `YYYYMMDD` format — always filter here (partition key) |
| `user_id` | customer UUID |
| `platformtype` | `app_android` / `app_iphone` / web |
| `city` | city dimension |
| `dealid`, `sellleadid` | ⚠️ **Do not use for joins** — unreliable |

GA4 = click/funnel volumes only. No row-level joins to `support_tickets`.

### Standard queries

**Query-level: L1 clicks vs app tickets side by side**
```sql
SELECT
    COALESCE(ga.query_name, t.query_name)  AS query_name,
    ga.l1_clicks,
    t.app_tickets
FROM (
    SELECT
        GET_JSON_STRING(eventvalue, '$.type') AS query_name,
        COUNT(*)                              AS l1_clicks
    FROM iceberg.sp_google_analytics.ga4_master_data
    WHERE event_name IN (
        'help_supp_buy_help_l1_issue_click',
        'help_supp_sell_help_l1_issue_click'
    )
      AND event_date >= '20260501'
    GROUP BY query_name
) ga
FULL OUTER JOIN (
    SELECT
        query_name,
        COUNT(*) AS app_tickets
    FROM sp_entity_store.support_tickets
    WHERE ticket_source_name = 'SPINNY_HELP_AND_SUPPORT'
      AND service_unit_id = 1
      AND ticket_merged_ts IS NULL
      AND DATE(ticket_created_ts) >= '2026-05-01'
    GROUP BY query_name
) t ON (
    ga.query_name = t.query_name
    OR (ga.query_name = 'Seller-RC'       AND t.query_name = 'RC - Seller')
    OR (ga.query_name = 'Seller-Payments' AND t.query_name = 'Payments - Seller')
)
ORDER BY l1_clicks DESC NULLS LAST
```

**Sub-query level: L2 clicks vs app tickets (by `primary_query_heading_name`)**
```sql
SELECT
    COALESCE(ga.query_name, t.query_name)         AS query_name,
    COALESCE(ga.heading_name, t.heading_name)     AS sub_query_heading,
    ga.l2_clicks,
    t.app_tickets
FROM (
    SELECT
        GET_JSON_STRING(eventvalue, '$.type')    AS query_name,
        GET_JSON_STRING(eventvalue, '$.concern') AS heading_name,
        COUNT(*)                                 AS l2_clicks
    FROM iceberg.sp_google_analytics.ga4_master_data
    WHERE event_name = 'help_supp_buy_help_l2_issue_click'
      AND event_date >= '20260501'
    GROUP BY query_name, heading_name
) ga
FULL OUTER JOIN (
    SELECT
        query_name,
        primary_query_heading_name AS heading_name,
        COUNT(*)                   AS app_tickets
    FROM sp_entity_store.support_tickets
    WHERE ticket_source_name = 'SPINNY_HELP_AND_SUPPORT'
      AND service_unit_id = 1
      AND ticket_merged_ts IS NULL
      AND primary_query_heading_name IS NOT NULL
      AND DATE(ticket_created_ts) >= '2026-05-01'
    GROUP BY query_name, heading_name
) t ON ga.query_name = t.query_name
   AND ga.heading_name = t.heading_name
ORDER BY l2_clicks DESC NULLS LAST
```

**Month-on-month trend (query level — leading indicator)**
```sql
SELECT
    DATE_FORMAT(event_date, '%Y-%m')             AS month,
    GET_JSON_STRING(eventvalue, '$.type')        AS query_name,
    COUNT(*)                                     AS l1_clicks
FROM iceberg.sp_google_analytics.ga4_master_data
WHERE event_name IN (
    'help_supp_buy_help_l1_issue_click',
    'help_supp_sell_help_l1_issue_click'
)
  AND GET_JSON_STRING(eventvalue, '$.type') IS NOT NULL
GROUP BY month, query_name
ORDER BY month DESC, l1_clicks DESC
```

---

## Reference

- **CX Master Dashboard:** https://superset.mngt.ispinnyworks.in/superset/dashboard/126/ (dashboard ID: 126)
- **CX Callback Dashboard:** https://superset.mngt.ispinnyworks.in/superset/dashboard/223/ — ⚠️ **broken**: depends on `meta_data` JSON path in iceberg, which is hashed; all pickup-rate metrics return 0. Use the SQL above instead.
- **Source DB:** `iceberg.sp_ticket_master` (Django app: sp-ticket-master)
- **Legacy ticketing:** `iceberg.sp_support_tickets.onedirect_ticket` — older system (Onedirect/Freshdesk integration), lower volume
- **Entity store table:** `sp_entity_store.support_tickets` — refreshed on batch cycle (check `record_updated_ts`)
