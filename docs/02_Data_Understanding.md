# 02 — Data Understanding

## 1. Dataset Architecture

The dataset is exposed as **92 daily tables** (`events_20201101` through `events_20210131`), one table per calendar day, rather than a single monolithic table. This is standard GA4 BigQuery export design.

```sql
-- Confirmed structure
SELECT table_name
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.INFORMATION_SCHEMA.TABLES`
ORDER BY table_name;
-- Returns events_20201101 ... events_20210131 (92 tables)
```

**Why daily tables, not one table:** partition-pruning at the table level (instead of relying on a partition column) keeps queries cheap and fast at BigQuery scale — querying one day scans one table, not a filtered slice of a multi-year table. All cross-day analysis in this project uses the wildcard pattern:

```sql
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
```

with `_TABLE_SUFFIX` used for date filtering when needed, to control cost and scan volume.

**Confirmed dataset facts (from validation, detailed in `03_Data_Validation.md`):**
- Total events: 4,295,584
- Platform: 100% WEB (no native app traffic)
- 17 distinct event types
- Window spans a major seasonal shopping period (Black Friday → Christmas → New Year)

## 2. Event-Driven Data Model

**The single most important fact about this dataset: one row = one event.** Not one user. Not one session. Not one transaction.

A single user's single session can generate 15–20+ rows — `page_view`, `scroll`, `view_item`, `add_to_cart`, `begin_checkout`, `purchase`, and more, each a separate row with its own timestamp. Every "unique users," "sessions," or "conversion rate" metric in this project is a **derived aggregation** on top of this raw event grain — never a direct row count.

**Standard event vocabulary observed in this dataset, ranked by volume:**

| event_name | Count | Funnel stage |
|---|---|---|
| page_view | 1,350,428 | Awareness / browsing |
| user_engagement | 1,058,721 | Engagement signal (not a page action) |
| scroll | 493,072 | Engagement signal |
| view_item | 386,068 | Product discovery |
| session_start | 354,970 | Session boundary marker |
| first_visit | 257,462 | New-user marker |
| view_promotion | 190,104 | Merchandising exposure |
| add_to_cart | 58,543 | Purchase intent |
| begin_checkout | 38,757 | Checkout entry |
| select_item | 31,007 | Product discovery |
| view_search_results | 26,172 | Search behavior |
| add_shipping_info | 19,722 | Checkout step |
| add_payment_info | 13,899 | Checkout step |
| select_promotion | 9,450 | Merchandising interaction |
| purchase | 5,692 | Conversion |
| click | 1,446 | Outbound/UI interaction |
| view_item_list | 71 | Product discovery (rare — flagged for validation) |

**Immediate observation worth flagging in the Executive Summary later:** `view_item` (386K) → `add_to_cart` (58.5K) → `purchase` (5.7K) is already a visibly steep funnel before any formal analysis — roughly 15% view-to-cart and 10% cart-to-purchase at raw volume. Formal funnel construction (session-scoped, not just raw counts) happens in `06_Analysis.md`.

**Identity model:** `user_id` (authenticated ID) is almost entirely null — this is a guest-checkout store, no login required to browse or buy. `user_pseudo_id` (device/browser pseudonymous ID) is the de facto user key for this entire project. This is stated explicitly rather than assumed, because it directly limits what "unique customer" claims this project can honestly make (see Analytics Assumptions, §6).

**Session model:** GA4 does not give you a `session_id` column directly — it is embedded inside `event_params` as `ga_session_id`. The true session grain used throughout this project is:

```
session_key = user_pseudo_id + ga_session_id
```

concatenated together, since `ga_session_id` alone is not guaranteed globally unique across different users.

## 3. Data Dictionary

### 3.1 Top-Level Fields (Quick Reference)

| Field | Type | Tier | One-line meaning |
|---|---|---|---|
| event_date | STRING (YYYYMMDD) | Essential | Calendar date of event; must be `PARSE_DATE`'d before date math |
| event_timestamp | INT64 (microseconds) | Essential | Exact moment of event; use `TIMESTAMP_MICROS()` |
| event_name | STRING | Essential | The behavioral action — the entire funnel vocabulary |
| event_previous_timestamp | INT64 (microseconds) | Optional | Timestamp of user's previous event of the *same* event_name |
| event_value_in_usd | FLOAT64 | Optional — use with caution | Generic event value; found unreliable in this dataset (see §6) |
| event_bundle_sequence_id | INT64 | Optional | Client batching metadata — not analytical |
| event_server_timestamp_offset | INT64 | Optional | Clock-drift correction metadata — not analytical |
| user_id | STRING | Essential to understand, low coverage | Authenticated ID; near-always null here |
| user_pseudo_id | STRING | **Essential** | De facto unique-user key for this entire project |
| privacy_info | STRUCT | Optional | Consent-mode metadata; mostly null in this sample |
| user_properties | ARRAY\<STRUCT\> | Optional (empty here) | Custom user-scoped traits; unused in this dataset |
| user_first_touch_timestamp | INT64 (microseconds) | **Essential** | True first-ever-seen moment; anchors cohort analysis |
| user_ltv | STRUCT | **Essential** | GA4's running lifetime-value snapshot; must take MAX, not AVG, per user |
| device | STRUCT | **Essential** | Device/browser/OS context |
| geo | STRUCT | **Essential** | Location context |
| app_info | STRUCT | Optional — confirmed unused | Native app metadata; null (web-only stream) |
| traffic_source | STRUCT | **Essential** | Acquisition channel context |
| stream_id / platform | INT64 / STRING | Useful | Confirms 100% WEB platform |
| event_dimensions | STRUCT | Optional — confirmed unused | Hostname; null here |
| ecommerce | STRUCT | **Essential** | Transaction-level financial record |
| items | ARRAY\<STRUCT\> | **Essential** | Product-level detail per event |

Full field-by-field documentation — including business meaning, technical meaning, why GA4 stores it, example values, business questions answered, KPIs served, SQL considerations, common interview questions, and common analyst mistakes — is maintained for every field above. (Reference version of this deep-dive is preserved in project history and will be re-published as an appendix if useful; the table above is the canonical GitHub-facing summary to keep this document scannable.)

## 4. Important Nested Fields — Structural Deep Dive

GA4 BigQuery export uses a **"wide event, narrow row"** design: rather than hundreds of sparsely-populated top-level columns, context is nested into flexible key-value or struct arrays. Understanding *why* this design exists is itself an interview-relevant insight (it keeps the outer schema stable while the vocabulary of possible parameters grows over time).

### 4.1 `event_params` — the most-used field in this entire project

`ARRAY<STRUCT<key STRING, value STRUCT<string_value, int_value, float_value, double_value>>>`

Only **one** of the four `value.*` sub-fields is ever populated per key, depending on that parameter's data type. The standard access pattern used throughout this project's SQL repository:

```sql
(SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS session_id,
(SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_location') AS page_location
```

**Key children used repeatedly in this project:**

| Key | Value field | What it drives |
|---|---|---|
| `ga_session_id` | int_value | True session identifier (combine with user_pseudo_id) |
| `ga_session_number` | int_value | New (=1) vs. returning (>1) session flag |
| `page_location` | string_value | Full URL — funnel path, category parsing |
| `page_title` | string_value | Human-readable page name for reporting |
| `session_engaged` / `engaged_session_event` | string/int_value | GA4's engagement definition (replaces UA "bounce rate") |
| `engagement_time_msec` | int_value | Active foreground time — not total wall-clock session duration |
| `entrances` | int_value | Flags the landing page of a session |
| `percent_scrolled` | int_value | Content-engagement signal on scroll events |
| `gclid` / `gclsrc` | string_value | Paid Google Ads click ID — sparsely populated in this sample |
| `debug_mode` | int_value | QA/test-traffic flag — **filtered out** in production-grade queries |
| `clean_event` | string_value | Internal GTM bookkeeping — not business-relevant |

### 4.2 `items` — the product-performance array

~26-field STRUCT array (`item_id`, `item_name`, `item_category` through `category5`, `price_in_usd`, `quantity`, `item_revenue_in_usd`, `item_list_name/index`, `promotion_id/name`, etc.), empty `[]` on non-commerce events.

**Critical structural warning carried through every query in this project:** unnesting `items` **multiplies row count** for multi-item transactions. Any event-level aggregate (e.g., session count, purchase count) computed *after* an items-unnest in the same query will be silently wrong unless grain is explicitly managed (aggregate items separately, or use `COUNT(DISTINCT transaction_id)` rather than `COUNT(*)`). This is the single most common structural SQL bug in GA4 ecommerce analysis and is treated as a first-class validation check in `03_Data_Validation.md`.

### 4.3 `device`

`category` (mobile/desktop/tablet), `mobile_brand_name`, `operating_system` (often literally `"Web"` for web streams — not a data bug, a real GA4 quirk), `web_info.browser` / `browser_version`. `device.category` is the primary grouping field for all device-split KPIs; model-level fields are too sparse for aggregate analysis.

### 4.4 `geo`

`continent`, `sub_continent`, `country`, `region`, `city`, `metro`. `country` is reliable at this sample size; `city`/`metro` become statistically noisy fast — this project caps geographic granularity at country/region level for any claim-bearing analysis, and treats city-level cuts as descriptive only.

### 4.5 `traffic_source`

`medium`, `source`, `name`. Always grouped as `medium + source` together, never `source` alone — `source = "google"` spans both free organic search and paid CPC, two entirely different economics; collapsing them is a real analytical error this project explicitly avoids.

### 4.6 `ecommerce`

`purchase_revenue_in_usd`, `refund_value_in_usd`, `shipping_value_in_usd`, `tax_value_in_usd`, `transaction_id`, etc. — populated only on `purchase`/`refund` events. This project always uses the `_in_usd`-suffixed fields (not the local-currency fields) and validates `COUNT(DISTINCT transaction_id)` against `COUNT(*)` on purchase events to catch duplicate-fire logging before trusting any revenue total.

## 5. ER-Style Relationship Explanation

This dataset is **not relational** in the traditional sense — it is a single denormalized event stream. There is no separate `users`, `sessions`, `products`, or `transactions` table to join against. Instead, those entities are *reconstructed* from the event stream using derived keys:

```
events_* (one row per event)
   │
   ├── user_pseudo_id ─────────────► reconstructs "USER" entity
   │        └── user_first_touch_timestamp anchors user's cohort
   │        └── user_ltv gives running lifetime value snapshot
   │
   ├── user_pseudo_id + event_params.ga_session_id
   │        └────────────────────────► reconstructs "SESSION" entity
   │             └── ga_session_number flags new vs. returning
   │             └── entrances flags landing page
   │
   ├── ecommerce.transaction_id ─────► reconstructs "TRANSACTION" entity
   │        └── one purchase event = one transaction (validated for duplicates)
   │
   └── UNNEST(items) ────────────────► reconstructs "PRODUCT LINE ITEM" entity
            └── many-to-one with the parent event (multi-item carts/orders)
```

**Conceptual relationship summary:**

| Reconstructed entity | Derived from | Cardinality to raw event row |
|---|---|---|
| User | `user_pseudo_id` | Many events → 1 user |
| Session | `user_pseudo_id` + `ga_session_id` | Many events → 1 session |
| Transaction | `ecommerce.transaction_id` | 1 purchase event → 1 transaction (validate for dupes) |
| Product line item | `UNNEST(items)` | 1 event → many items |

This is why nearly every non-trivial query in `05_SQL_Query_Repository.md` begins by explicitly defining its grain (event, session, user, or item-line) before aggregating — grain confusion is the single largest source of incorrect metrics on this dataset.

## 6. Analytics Assumptions

Stated explicitly, per senior-analyst practice, rather than left implicit:

1. **`user_pseudo_id` is treated as "unique user."** This overcounts true humans (same person on phone + laptop = 2 IDs) and can undercount over time if cookies clear (1 person becomes multiple IDs). All "unique user" language in this project carries this caveat.
2. **`user_id` (authenticated ID) is not usable** for user-level analysis due to near-total sparsity — this is a guest-checkout site.
3. **`event_value_in_usd` is excluded from revenue calculations.** Its values did not reconcile against `ecommerce.purchase_revenue_in_usd` during validation and are treated as an unreliable/obfuscation artifact rather than real per-event revenue.
4. **`debug_mode = 1` traffic is treated as test traffic** and excluded from production-grade metrics after validation confirms its presence and volume.
5. **City/metro-level geographic cuts are descriptive only**, not decision-bearing, due to small sample sizes at that granularity.
6. **No real advertising cost or CAC data exists in this schema.** Any "ROI" or "channel efficiency" language elsewhere in this project is directional and qualitative, never a literal cost-based calculation.
7. **The Nov–Dec seasonal spike (Black Friday/Cyber Monday/Christmas) is called out explicitly** wherever raw trend lines are shown, to avoid presenting seasonal lift as organic growth.
8. **This is a sample, obfuscated dataset.** All findings are framed as directional and methodology-driven, never as literal claims about the real Google Merchandise Store's actual business performance.

---

**Next:** `03_Data_Validation.md` will run and document the validation queries that support assumptions 3–6 above with actual query evidence (null-rate checks, debug_mode volume, transaction dedup check, platform confirmation), producing a PASS/WARNING/FAIL data quality report.
