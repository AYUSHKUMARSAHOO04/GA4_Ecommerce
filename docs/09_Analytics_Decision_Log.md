**[↑ Back to README](../README.md)** &nbsp;·&nbsp; **[Repository Navigation](../README.md#-repository-structure)**

---

# 09 — Analytics Decision Log

A durable, chronological record of every assumption, validation result, and methodology decision made across this engagement — written so a new analyst inheriting this dataset does not have to rediscover any of it.

---

## 1. Purpose

Every non-trivial analytics engagement makes dozens of small methodology decisions — how to define a session, which field to trust, how to handle an anomaly. Left undocumented, those decisions live only in the head of whoever made them. This log exists so they don't have to be rediscovered, or worse, silently re-broken, by the next person who touches this dataset.

## 2. The Methodology Rule Registry (R1–R7)

| Rule | Decision | Evidence | Discovered In | Status |
|---|---|---|---|---|
| **R1** | `debug_mode` traffic is included, not filtered | 85.74% of events, 99.7%+ of sessions carry `debug_mode=1`; confirmed uniform across channels (Q2.8), so relative comparisons are unaffected | `03_Data_Validation.md` §6.4 | Permanent |
| **R2** | `transaction_id` cleaned: exclude `(not set)`, dedupe by `(transaction_id, user_pseudo_id)` | 767+ distinct users collided under a shared placeholder ID; a smaller number of real numeric IDs also collided across genuinely different users | `03_Data_Validation.md` §4.2 | Permanent |
| **R3** | Transaction-level revenue is the sole source of truth; item-level and `user_ltv` are never reconciled to it | Item-level diverges on 28.56% of individual transactions (though aggregates agree within 0.015%); `user_ltv` overstates revenue by 12.6% in aggregate | `03_Data_Validation.md` §4.3; quantified in `05_SQL_Query_Repository.md` Q11.2 | Permanent |
| **R4** | Product-level cuts on `view_item`/`begin_checkout` are scoped to items-populated rows only | `view_item` only 62.65% items-populated; `begin_checkout` 82.54% | `03_Data_Validation.md` §5.2 | Permanent |
| **R5** | `engagement_time_msec` capped at 3,600,000ms before averaging | Confirmed 10–19+ hour single-event outliers from background browser tabs | `03_Data_Validation.md` §6.5 | Permanent |
| **R6** | Engagement-rate and engagement-time metrics are not comparable before/after Dec 28, 2020 | `engagement_time_msec` presence jumped from a flat 0.0% to 42–44% exactly on that date — a confirmed tagging/config change, not real behavior | `05_SQL_Query_Repository.md` Q4.6 | Permanent |
| **R7** | `item_id` is unreliable as a cross-event-type join key; use `item_name` instead | Q9.5's `item_id`-based join returned 100% "Unmapped"; switching to `item_name` (Q9.6) resolved the mapping cleanly | `05_SQL_Query_Repository.md` Q9.5–Q9.6 | Permanent |

**Why these are "rules" and not footnotes:** each one changes the *arithmetic* of a metric, not just its caption. A dashboard built without R2 overstates transaction counts; without R6 it shows a fake engagement improvement; without R7 it shows a fake 0%-converting category. These are enforced constraints on every query in `05_SQL_Query_Repository.md`, not optional caveats.

---

## 3. Key Assumptions and Their Resolution

| # | Assumption (at time made) | Where Stated | Resolution |
|---|---|---|---|
| 1 | `event_value_in_usd` is unreliable and should be excluded from all revenue analysis | `02_Data_Understanding.md` §Field Dictionary | **Revised** — Q4.4 showed 92.1% match with real purchase revenue *specifically on `purchase` events*; narrowed to exclude only non-purchase events, not the field entirely |
| 2 | Cross-session cart persistence explains the view-to-purchase divergence (6.1% direct vs. ~3.4% naive stage product) | Raised while designing Q6.6 | **Partially overturned** — Q6.6 showed persistence explains only 3.41% of checkout sessions; the dominant factor (42.91%) is checkouts with no detectable cart-add at all (open item, not fully resolved) |
| 3 | `item_id` is a safe join key for product-level browsing-to-purchase analysis | Default assumption entering Stage 5, Section 9 | **Overturned** → became Rule R7 |
| 4 | Refund rate is a computable, trustworthy metric (Metric 5.4) | `04_Metrics_Framework.md` — flagged as an open validation item | **Resolved as a negative finding** — 0 `refund` events exist in the dataset; reported as "not tracked," not "0% refund rate" |
| 5 | `user_ltv.revenue`'s null-rate and distribution were never checked (Metric 8.1) | `04_Metrics_Framework.md` — flagged as an open validation item | **Resolved** — 0% null, but heavily zero-inflated (median = 0); confirmed to overstate real revenue by 12.6% in aggregate, feeding directly into R3 |
| 6 | Geographic analysis was in scope | `01_Project_Overview.md` | **Explicitly descoped** — no query was ever executed; stated as an acknowledged gap in `06_Analysis.md` §12, not filled with invented findings |

---

## 4. Anomalies Investigated (and How Each Was Closed)

1. **Engagement rate jump, Dec 28, 2020** (`06_Analysis.md` §4) — Diagnosed via Q4.6 by checking the underlying parameter's *presence rate*, not just its value. Confirmed as a tagging change → **R6**.
2. **"0%-converting" mega-category** (`06_Analysis.md` §9) — Diagnosed via a failed `item_id`-based fix (Q9.5) that returned 100% unmapped, then resolved via `item_name` (Q9.6). Confirmed as a join-key defect → **R7**.
3. **Cart-to-checkout naive ratio (66%) vs. session-scoped rate (39.25%)** — A 27-point gap, explained as an event-vs-session grain mismatch; documented in `06_Analysis.md` §5 as a standing example of why grain must be stated explicitly on every query.
4. **View-to-purchase direct rate (6.1%) nearly 2x the naive stage-product estimate (3.4%)** — Investigated via Q6.6; only partially explained (see Assumption #2 above). **Status: open.**
5. **Peak traffic day was Dec 8, not the Nov 27–30 Black Friday/Cyber Monday window** — Noted in `06_Analysis.md` §2 as a finding without an assumed cause; no promotional calendar was available to confirm the driver. **Status: open, flagged for follow-up.**

---

## 5. Decisions Explicitly Rejected

- **Rejected: filtering out all `debug_mode=1` traffic.** Would have removed 85.74% of events (99.7%+ of sessions), destroying statistical power for retention/cohort/geographic work, for a signal later shown to be a dataset-wide export artifact rather than genuine internal QA activity.
- **Rejected: using `item_category` as the category join field.** Shown to be tagged inconsistently between event types; superseded by the R7 `item_name` fix.
- **Rejected: reporting engagement rate as a single continuous trend line across the full 92-day window.** Would have presented a tagging artifact as a real 30+ point improvement (R6).
- **Rejected: treating the `(data deleted)` traffic segment as noise to exclude.** Instead documented as the highest-value, highest-priority open investigation item in `07_Business_Recommendations.md` (BR5), specifically because its performance (3.14% conversion, 96% engagement) was too distinctive to dismiss.
- **Rejected: padding the SQL Query Repository to a round "75–100 queries" count with repetitive per-dimension cuts.** Closed at 54 queries, prioritizing that every query either implemented a defined metric or resolved a specific, real anomaly.

---

## 6. Outstanding Open Items (Carried Forward, Not Silently Dropped)

| Open Item | First Raised | Current Status |
|---|---|---|
| 42.91% of checkouts show no detectable prior add-to-cart — "Buy Now" flow vs. tracking gap | `05_SQL_Query_Repository.md` Q6.6 | Unresolved — requires session-recording/UX investigation, not further SQL (see `07_Business_Recommendations.md` BR9) |
| `(data deleted)` segment's underlying characteristics | `06_Analysis.md` §10 | Unresolved — Legal/Privacy-gated (see BR5) |
| Root cause of the 19.97% first-session product-view rate | `06_Analysis.md` §3 | Hypothesis only (homepage/navigation friction) — not isolated by an executed query |
| Root cause of desktop's higher cart abandonment vs. mobile | `06_Analysis.md` §11 | Hypothesis only — requires session-recording/heatmap review |
| Geographic performance | `06_Analysis.md` §12 | Not assessed — no query executed; lowest-effort next analysis given confirmed clean `geo.country` data |
| Cause of the Dec 8 and Jan 6 traffic spikes | `06_Analysis.md` §2 | Not assessed — no promotional calendar available to cross-reference |

---

## 7. How to Extend This Project

A future analyst adding to this repository should:
1. Read this log before writing any new query — it prevents re-deriving R1–R7 from scratch or reintroducing an already-fixed error.
2. Add any new methodology rule discovered to the registry in §2, in the same format, with the query ID that discovered it.
3. Treat §6's open items as the natural starting point for the next phase of work — they are genuine gaps, not resolved findings dressed up as incomplete.

---

**[← Previous: 08 Executive Summary](08_Executive_Summary.md)** &nbsp;|&nbsp; **[Back to README](../README.md)** &nbsp;|&nbsp; This is the final document in the sequence.
