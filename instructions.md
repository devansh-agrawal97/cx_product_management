# CX Q3 Roadmap — Working Instructions

## Scope
Build the Q3 (July–September 2026) roadmap for CX Post Sales at Spinny.

## Approach
- **Do NOT** factor in tech bandwidth, team capacity, or engineering effort.
- Focus purely on **impact** and **sizing of the problem statement**.
- Every item should be evaluated on: how big is the problem, and how much does solving it move the needle on product goals.

## Inputs Available
1. **Ticket data (April–May 2026)** — query-level breakdown with FCR%, app%, and call distribution. Stored in `cx_q3_roadmap.md`.
2. **Product goals & actuals** — Aug and Dec targets vs April/May actuals. Stored in `cx_q3_roadmap.md`.
3. **Q1 & Q2 roadmap history** — what shipped, what's in progress, what spilled, what dropped. Source: [Google Sheet](https://docs.google.com/spreadsheets/d/1xlfmLMo4b4kNHG10zPnGoUVf3hPzzRnaUimaXVdNhLs/edit?gid=646934706#gid=646934706).
4. **CX domain context** — `CX_post_sales_AIR.md` in this directory.

## Product Goals (North Star)
| Goal | Aug Target | May Actual | Direction |
|---|---|---|---|
| Tickets landing with agents | 18K | 21.7K | 🔴 Rising (wrong way) |
| % messages to customer by AI | 25% | 15.6% | 🟡 Improving |
| % tasks created automatically | 20% | 2.4% | 🔴 Critical gap |
| % calls made by AI | 10% | ~4.5% | 🟡 Early |
| % messages to internal team by AI | 30% | 0% | 🔴 Cold start |
| Callback response time (p80) | 45 mins | 103 mins | 🔴 Worsening |
| Customer message response time (p80) | 60 mins | ~111 mins | 🔴 Worsening |

## Evaluation Criteria for Q3 Items
For each candidate item, capture:
1. **Problem statement** — what's broken and for whom
2. **Sizing** — volume of tickets/customers/calls affected per month
3. **Impact on product goals** — which goal(s) does this move and by how much
4. **Current state** — is anything live today, what was tried in Q1/Q2
5. **Dependencies** — external teams, APIs, third parties needed

## Git & Push Protocol
- **Remote**: https://github.com/devansh-agrawal97/cx_product_management
- **On every push request**:
  1. Run `git diff` to check all uncommitted and staged changes
  2. Show a **merge summary** — list of files changed, lines added/removed, and a short description of each change
  3. **Wait for explicit permission** before pushing
  4. Only push after the user confirms

## What This Roadmap is NOT
- Not a sprint plan or tech spec
- Not acceptance criteria or user stories
- Not bounded by eng capacity — that's a separate conversation
