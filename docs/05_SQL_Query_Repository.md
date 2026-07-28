# 05 : SQL Query Repository

Every query below implements a specific metric defined in `04_Metrics_Framework.md` and applies
the validation-derived rules from `03_Data_Validation.md`:

- **[R1]** debug_mode traffic included, not filtered (documented dataset limitation)
- **[R2]** `transaction_id` cleaning: exclude `'(not set)'` and dedupe by `(transaction_id, user_pseudo_id)` for per-transaction metrics
- **[R3]** transaction-level revenue is sole source of truth; item-level revenue never reconciled to it
- **[R4]** `view_item`/`begin_checkout` product-level cuts scoped to items-populated rows only
- **[R5]** `engagement_time_msec` capped at 3,600,000ms before averaging
- **[R6]** engagement-time-based metrics not comparable pre/post Dec 28, 2020 (confirmed tagging change see §4.6)
- **[R7]** `item_id` unreliable as a cross-event-type join key; use `item_name` for product-level joins between browsing and purchase events (see §9.5-9.6)

Each query specifies: **Purpose** (which metric, §-reference) → **SQL** → **Business Interpretation** → **Expected Output**.

---

## SECTION 1 - North Star Metric Queries

### Query 1.1 - Weekly Transacting Users (WTU)

**Purpose:** Implements Metric 1 (North Star). Applies [R1], [R2].

```sql
WITH cleaned_purchases AS (
  SELECT
    PARSE_DATE('%Y%m%d', event_date) AS event_dt,
    user_pseudo_id,
    ecommerce.transaction_id AS transaction_id
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE event_name = 'purchase'
    AND ecommerce.transaction_id IS NOT NULL
    AND ecommerce.transaction_id != '(not set)'
  GROUP BY event_dt, user_pseudo_id, transaction_id  -- dedupes exact (user, txn) repeats per R2
)
SELECT
  DATE_TRUNC(event_dt, WEEK(MONDAY)) AS week_start,
  COUNT(DISTINCT user_pseudo_id) AS weekly_transacting_users
FROM cleaned_purchases
GROUP BY week_start
ORDER BY week_start;
```

**Business Interpretation:** This is the single number the VP Product and Head of Analytics track every Monday. A flat or declining trend despite stable/growing session volume (Query 2.1) signals a funnel problem, not a traffic problem triggers a Stage 6 funnel deep-dive before any acquisition budget conversation.

**Expected Output:** ~13 rows (one per week across the 92-day window), with a visible spike in the week(s) covering Black Friday/Cyber Monday and again around Christmas must be annotated as seasonal, not organic growth, per the Stage 1 scope note.

---

### Query 1.2 - WTU vs. Total Sessions (North Star Efficiency Check)

**Purpose:** Cross-references Metric 1 against Metric 2.1 to test whether North Star growth is efficiency-driven or volume-driven.

```sql
WITH sessions AS (
  SELECT
    PARSE_DATE('%Y%m%d', event_date) AS event_dt,
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY event_dt, session_key
),
weekly_sessions AS (
  SELECT DATE_TRUNC(event_dt, WEEK(MONDAY)) AS week_start, COUNT(DISTINCT session_key) AS sessions
  FROM sessions
  GROUP BY week_start
),
weekly_wtu AS (
  SELECT
    DATE_TRUNC(PARSE_DATE('%Y%m%d', event_date), WEEK(MONDAY)) AS week_start,
    COUNT(DISTINCT user_pseudo_id) AS wtu
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE event_name = 'purchase'
    AND ecommerce.transaction_id IS NOT NULL
    AND ecommerce.transaction_id != '(not set)'
  GROUP BY week_start
)
SELECT
  s.week_start,
  s.sessions,
  w.wtu,
  ROUND(w.wtu * 1.0 / s.sessions * 100, 4) AS wtu_per_100_sessions
FROM weekly_sessions s
JOIN weekly_wtu w USING (week_start)
ORDER BY week_start;
```

**Business Interpretation:** `wtu_per_100_sessions` is a normalized efficiency line if it's flat while raw WTU rises, growth is purely traffic-driven (fragile); if it's rising, the product/funnel itself is converting better (durable). This is the chart that answers "are we actually getting better, or just bigger."

**Expected Output:** ~13 weekly rows; expect `wtu_per_100_sessions` in the low single digits given the ~15%×~66%×~15% funnel stage rates implied by Stage 1 raw volumes.

---

### Query 1.3 - North Star Cohort Split: New vs. Returning Contribution

**Purpose:** Decomposes WTU by `ga_session_number` to show whether North Star growth comes from new or returning buyers directly informs the Section 7/8 retention narrative.

```sql
WITH purchase_sessions AS (
  SELECT
    PARSE_DATE('%Y%m%d', event_date) AS event_dt,
    user_pseudo_id,
    ecommerce.transaction_id AS transaction_id,
    (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_number') AS session_number
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE event_name = 'purchase'
    AND ecommerce.transaction_id IS NOT NULL
    AND ecommerce.transaction_id != '(not set)'
)
SELECT
  DATE_TRUNC(event_dt, WEEK(MONDAY)) AS week_start,
  COUNT(DISTINCT IF(session_number = 1, user_pseudo_id, NULL)) AS wtu_new,
  COUNT(DISTINCT IF(session_number > 1, user_pseudo_id, NULL)) AS wtu_returning
FROM purchase_sessions
GROUP BY week_start
ORDER BY week_start;
```

**Business Interpretation:** If `wtu_new` dominates every week, the business is functioning as a continuous acquisition funnel with little repeat-purchase contribution reinforces the "brand engagement, not repeat-revenue engine" framing from `01_Project_Overview.md`, and is a direct input to the Repeat Purchase Rate finding (Metric 7.3) in Stage 6.

**Expected Output:** Two time series across ~13 weeks; given the 1.33 sessions/user baseline (§2.2 validation), expect `wtu_new` to substantially outweigh `wtu_returning` in most weeks.

---

## SECTION 2 - Acquisition Metrics Queries

### Query 2.1 - Daily & Weekly Sessions

**Purpose:** Implements Metric 2.1 (Sessions). Applies [R1].

```sql
WITH sessions AS (
  SELECT
    event_date,
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY event_date, session_key
)
SELECT
  PARSE_DATE('%Y%m%d', event_date) AS event_dt,
  COUNT(DISTINCT session_key) AS daily_sessions
FROM sessions
GROUP BY event_dt
ORDER BY event_dt;
```

**Business Interpretation:** Base traffic trend line for the Executive Dashboard. Must be read with the Black Friday/Christmas seasonal annotation a raw spike here is expected and should not be attributed to any specific Marketing action without a channel-level breakdown (Query 2.4).

**Expected Output:** 92 rows, one per day, ranging roughly from a few thousand to tens of thousands of sessions/day with a clear late-November and late-December peak.

---

### Query 2.2 - New Users (First-Ever Session)

**Purpose:** Implements Metric 2.2. Applies [R1].

```sql
SELECT
  PARSE_DATE('%Y%m%d', event_date) AS event_dt,
  COUNT(DISTINCT user_pseudo_id) AS new_users
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
WHERE event_name = 'first_visit'
GROUP BY event_dt
ORDER BY event_dt;
```

**Business Interpretation:** Baseline acquisition-volume trend, cross-checked against `ga_session_number = 1` counts (Query 2.2b) as a data-consistency sanity check before being trusted for reporting.

**Expected Output:** 92 daily rows; total across the window should be close to, but not necessarily exactly, the 270,154 unique users confirmed in §2.1 (some users' first_visit may fall outside the window if session-stitching edge cases exist worth a reconciliation note, not an assumption of exact match).

---

### Query 2.2b - New Users Cross-Check (session_number method)

**Purpose:** Validates Query 2.2 using an independent method (per the "cross-check before trusting" note in Metric 2.2's Common Mistakes).

```sql
SELECT
  PARSE_DATE('%Y%m%d', event_date) AS event_dt,
  COUNT(DISTINCT user_pseudo_id) AS new_users_by_session_number
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
WHERE (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_number') = 1
  AND (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'entrances') = 1
GROUP BY event_dt
ORDER BY event_dt;
```

**Business Interpretation:** If this materially disagrees with Query 2.2's `first_visit`-based count, that's a genuine data-quality finding worth escalating (e.g., `first_visit` double-firing under debug_mode, consistent with the §2.5 entrance-doubling finding) don't silently pick whichever number looks better.

**Expected Output:** Should closely track Query 2.2's daily totals; any divergence >5% on a given day warrants investigation before either number is published.

---

### Query 2.3 - New User Rate (Weekly)

**Purpose:** Implements Metric 2.3.

```sql
WITH weekly_totals AS (
  SELECT
    DATE_TRUNC(PARSE_DATE('%Y%m%d', event_date), WEEK(MONDAY)) AS week_start,
    COUNT(DISTINCT user_pseudo_id) AS total_users,
    COUNT(DISTINCT IF(event_name = 'first_visit', user_pseudo_id, NULL)) AS new_users
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY week_start
)
SELECT
  week_start,
  total_users,
  new_users,
  ROUND(new_users * 100.0 / total_users, 2) AS new_user_rate_pct
FROM weekly_totals
ORDER BY week_start;
```

**Business Interpretation:** Per Metric 2.3's interpretation guidance, a healthy range is 60-80%; sustained readings near 100% flag a retention problem (Section 7), readings below 40% flag stalled acquisition (Section 2/10).

**Expected Output:** ~13 weekly rows; given the low sessions-per-user baseline (1.33, §2.2), expect this rate to run on the higher end of or above the "healthy" range itself a notable finding to surface in Stage 6.

---

### Query 2.4 - Channel Session Mix (Medium + Source)

**Purpose:** Implements Metric 2.4. Correctly grouped by medium+source together, never source alone.

```sql
WITH sessions AS (
  SELECT
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    traffic_source.medium AS medium,
    traffic_source.source AS source
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY session_key, medium, source
)
SELECT
  COALESCE(medium, '(not set)') AS medium,
  COALESCE(source, '(not set)') AS source,
  COUNT(DISTINCT session_key) AS sessions,
  ROUND(COUNT(DISTINCT session_key) * 100.0 / SUM(COUNT(DISTINCT session_key)) OVER (), 2) AS pct_of_total_sessions
FROM sessions
GROUP BY medium, source
ORDER BY sessions DESC;
```

**Business Interpretation:** This is the Marketing team's primary channel-mix view. Per §6.2, expect `(none)/direct` at roughly 23% of sessions this must be labeled and reported as a real, legitimate channel segment, not folded into an "unknown" bucket or dropped from the chart.

**Expected Output:** A ranked list of medium+source combinations (e.g., `organic/google`, `(none)/(direct)`, `cpc/google`, `referral/...`), summing to 100%.

---

### Query 2.5 - Landing Page Entrance Rate (Deduplicated)

**Purpose:** Implements Metric 2.5, with the mandatory dedup fix identified in §2.5's validation finding (double-fired `entrances=1` events within a session).

```sql
WITH raw_entrances AS (
  SELECT
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_location') AS page_location,
    event_timestamp
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'entrances') = 1
),
deduped_entrances AS (
  -- Keep only the earliest entrance-flagged event per session, per the §2.5 duplicate-firing finding
  SELECT session_key, page_location,
    ROW_NUMBER() OVER (PARTITION BY session_key ORDER BY event_timestamp ASC) AS rn
  FROM raw_entrances
)
SELECT
  page_location,
  COUNT(*) AS entrance_sessions,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS pct_of_entrances
FROM deduped_entrances
WHERE rn = 1
GROUP BY page_location
ORDER BY entrance_sessions DESC
LIMIT 25;
```

**Business Interpretation:** Shows which pages act as the true front door. High concentration in 1-2 URLs (likely the homepage and a top campaign landing page) signals a resilience risk Marketing/SEO should diversify entry points rather than depend on a single page.

**Expected Output:** A ranked list of ~25 landing page URLs; expect the homepage (`https://shop.googlemerchandisestore.com/`) to dominate, consistent with the sample rows seen in Stage 1 discovery.

---

### Query 2.6 - Channel Mix Trend (Weekly, Top 5 Channels)

**Purpose:** Extends Query 2.4 with a time dimension supports the Marketing Channel dashboard's trend view, not just a point-in-time snapshot.

```sql
WITH sessions AS (
  SELECT
    event_date,
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    CONCAT(COALESCE(traffic_source.medium, '(not set)'), ' / ', COALESCE(traffic_source.source, '(not set)')) AS channel
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY event_date, session_key, channel
),
top_channels AS (
  SELECT channel FROM sessions GROUP BY channel ORDER BY COUNT(DISTINCT session_key) DESC LIMIT 5
)
SELECT
  DATE_TRUNC(PARSE_DATE('%Y%m%d', s.event_date), WEEK(MONDAY)) AS week_start,
  s.channel,
  COUNT(DISTINCT s.session_key) AS sessions
FROM sessions s
JOIN top_channels t USING (channel)
GROUP BY week_start, s.channel
ORDER BY week_start, sessions DESC;
```

**Business Interpretation:** Reveals whether a channel's share is stable, growing, or seasonal (e.g., paid channels often spike specifically around Black Friday while organic stays flat) informs whether a channel's Black Friday performance is repeatable or a one-off seasonal artifact.

**Expected Output:** ~13 weeks × 5 channels = ~65 rows, suitable for a stacked-area or multi-line chart in the dashboard.

---

### Query 2.7 - New vs. Returning Session Share by Channel

**Purpose:** Implements Metric 10.3 using acquisition-stage data included here because it's foundational to understanding *what kind* of traffic each channel brings before Section 10's deeper marketing analysis.

```sql
WITH sessions AS (
  SELECT
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    traffic_source.medium AS medium,
    traffic_source.source AS source,
    (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_number') AS session_number
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY session_key, medium, source, session_number
)
SELECT
  COALESCE(medium, '(not set)') AS medium,
  COALESCE(source, '(not set)') AS source,
  COUNT(DISTINCT IF(session_number = 1, session_key, NULL)) AS new_sessions,
  COUNT(DISTINCT IF(session_number > 1, session_key, NULL)) AS returning_sessions,
  ROUND(COUNT(DISTINCT IF(session_number > 1, session_key, NULL)) * 100.0 /
    COUNT(DISTINCT session_key), 2) AS pct_returning
FROM sessions
GROUP BY medium, source
ORDER BY new_sessions + returning_sessions DESC;
```

**Business Interpretation:** A channel with high `pct_returning` (e.g., direct/email) is functioning as a retention channel and should not be judged by new-user-acquisition KPIs; a channel with low `pct_returning` (e.g., paid search) is a discovery channel and should be judged primarily on new-user quality and first-session engagement (Metric 3.3), not repeat behavior.

**Expected Output:** One row per medium+source combination with a `pct_returning` column ranging widely (likely near-0% for pure paid-discovery channels up to 40%+ for direct/organic).

---

### Query 2.8 - Debug Traffic Share by Channel (Data Quality Cross-Check)

**Purpose:** Validation-adjacent query tests whether the [R1] debug-traffic finding (85.74% overall, §6.4) is evenly distributed across channels or concentrated in one, which would change how channel comparisons should be interpreted.

```sql
WITH sessions AS (
  SELECT
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    COALESCE(traffic_source.medium, '(not set)') AS medium,
    MAX((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'debug_mode')) AS is_debug
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY session_key, medium
)
SELECT
  medium,
  COUNT(*) AS total_sessions,
  COUNTIF(is_debug = 1) AS debug_sessions,
  ROUND(COUNTIF(is_debug = 1) * 100.0 / COUNT(*), 2) AS pct_debug
FROM sessions
GROUP BY medium
ORDER BY total_sessions DESC;
```

**Business Interpretation:** If `pct_debug` is roughly uniform (~85%) across all channels, the [R1] decision to leave debug traffic in stands without qualification for channel comparisons relative channel performance is unaffected even though absolute volumes are inflated. If one channel shows a materially different debug share, every channel-level metric in Section 10 needs a footnote specific to that channel.

**Expected Output:** A short table (one row per medium); the key check is whether `pct_debug` clusters tightly around 85% or diverges meaningfully by channel.

---

**Batch 1 complete - 11 queries across Section 1 (North Star) and Section 2 (Acquisition).**

**Batch 1 execution notes (from actual BigQuery results):** New User Rate is rising over time (82%→91%), a leading indicator of weak retention explored formally in Section 7. Peak traffic day was Dec 8, not the Nov 27–30 BF/CM window needs a specific cause, not generic seasonality. Landing-page URLs need normalization (homepage fragments into 3 separate rows, 46% of entrances combined). `(data deleted)` is a real, distinct traffic_source value (6.17% of sessions, 96% returning) open investigation item. Session-level debug-flag saturation (99.7%+) confirms and strengthens the [R1] decision.

---

## SECTION 2B - Follow-Up Queries (Prompted by Batch 1 Findings)

### Query 2.9 - Normalized Landing Page Performance (Homepage Consolidation)

**Purpose:** Corrects the URL-fragmentation issue discovered in Query 2.5 three homepage domain variants must be treated as one page before any landing-page ranking is reported.

```sql
WITH raw_entrances AS (
  SELECT
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_location') AS page_location,
    event_timestamp
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'entrances') = 1
),
deduped_entrances AS (
  SELECT session_key, page_location,
    ROW_NUMBER() OVER (PARTITION BY session_key ORDER BY event_timestamp ASC) AS rn
  FROM raw_entrances
),
normalized AS (
  SELECT
    session_key,
    CASE
      WHEN REGEXP_CONTAINS(page_location, r'^https://(shop\.|www\.)?googlemerchandisestore\.com/?$')
        THEN 'Homepage (normalized)'
      ELSE page_location
    END AS normalized_page
  FROM deduped_entrances
  WHERE rn = 1
)
SELECT
  normalized_page,
  COUNT(*) AS entrance_sessions,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS pct_of_entrances
FROM normalized
GROUP BY normalized_page
ORDER BY entrance_sessions DESC
LIMIT 20;
```

**Business Interpretation:** Once consolidated, the true homepage share is likely close to the combined 45.96% found across the three variants in Query 2.5 a much larger single-page concentration than any individual row suggested. This is the version of the metric that should actually go in front of Marketing/SEO, since the un-normalized version materially understates homepage dependency.

**Expected Output:** A cleaner top-20 landing page list with "Homepage (normalized)" now clearly the dominant single row, likely 40%+ of entrances alone.

### Query 2.10 - First-Session Activation Rate by Channel

**Purpose:** Directly tests whether channels differ in the *quality* of new users they bring, prompted by Query 2.7's finding that paid search skews almost entirely new (9.25% returning) this asks whether that paid-search traffic actually engages once it arrives.

```sql
WITH new_sessions AS (
  SELECT
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    COALESCE(traffic_source.medium, '(not set)') AS medium,
    COALESCE(traffic_source.source, '(not set)') AS source,
    MAX((SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'session_engaged')) AS session_engaged,
    (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_number') AS session_number
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY session_key, medium, source, session_number
)
SELECT
  medium,
  source,
  COUNT(*) AS new_sessions,
  COUNTIF(session_engaged = '1') AS engaged_new_sessions,
  ROUND(COUNTIF(session_engaged = '1') * 100.0 / COUNT(*), 2) AS pct_engaged
FROM new_sessions
WHERE session_number = 1
GROUP BY medium, source
HAVING new_sessions > 100
ORDER BY new_sessions DESC;
```

**Business Interpretation:** If `cpc/google` shows a materially lower `pct_engaged` than `organic/google` despite similar new-session volume, that's evidence of lower-quality paid traffic informs a real budget conversation in Section 10, not just a volume comparison.

**Expected Output:** One row per major channel restricted to first-ever sessions, `pct_engaged` likely varying meaningfully across channels (organic typically outperforms paid on engagement quality in comparable ecommerce datasets, but this must be confirmed against actual output, not assumed).

---

## SECTION 3 - Activation Metrics Queries

### Query 3.1 - First-Session Product View Rate

**Purpose:** Implements Metric 3.1. Notes [R4] does not block this query (presence-of-event check, not item-detail dependent).

```sql
WITH first_sessions AS (
  SELECT
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_number') AS session_number,
    event_name
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
),
session_flags AS (
  SELECT
    session_key,
    MAX(session_number) AS session_number,
    MAX(IF(event_name = 'view_item', 1, 0)) AS viewed_product
  FROM first_sessions
  GROUP BY session_key
)
SELECT
  COUNT(*) AS first_sessions_total,
  SUM(viewed_product) AS first_sessions_with_view,
  ROUND(SUM(viewed_product) * 100.0 / COUNT(*), 2) AS pct_viewed_product
FROM session_flags
WHERE session_number = 1;
```

**Business Interpretation:** A low rate here (relative to the overall `view_item` volume) signals a homepage/navigation problem specific to new visitors they're arriving but not reaching the catalog.

**Expected Output:** A single summary row; benchmark against the overall product-view rate across all sessions (not just first ones) to see if new visitors specifically underperform.

### Query 3.2 - Time-to-First-Add-to-Cart (Median)

**Purpose:** Implements Metric 3.2.

```sql
WITH first_touch AS (
  SELECT DISTINCT user_pseudo_id, user_first_touch_timestamp
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
),
first_cart AS (
  SELECT
    user_pseudo_id,
    MIN(event_timestamp) AS first_add_to_cart_ts
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE event_name = 'add_to_cart'
  GROUP BY user_pseudo_id
)
SELECT
  APPROX_QUANTILES(
    TIMESTAMP_DIFF(TIMESTAMP_MICROS(fc.first_add_to_cart_ts), TIMESTAMP_MICROS(ft.user_first_touch_timestamp), SECOND),
    2
  )[OFFSET(1)] AS median_seconds_to_first_cart,
  COUNT(*) AS users_with_cart_add
FROM first_cart fc
JOIN first_touch ft USING (user_pseudo_id)
WHERE TIMESTAMP_DIFF(TIMESTAMP_MICROS(fc.first_add_to_cart_ts), TIMESTAMP_MICROS(ft.user_first_touch_timestamp), SECOND) >= 0;
```

**Business Interpretation:** A short median (minutes, not days) suggests users who add to cart do so decisively within their first visit; a long median suggests a multi-visit research pattern before commitment directly relevant given Query 1.3's finding that most purchases happen on returning (not first) sessions.

**Expected Output:** A single row with `median_seconds_to_first_cart` and the count of users this applies to (should be close to, but likely below, total unique users not everyone adds to cart).

### Query 3.3 - First-Session Engagement Rate

**Purpose:** Implements Metric 3.3.

```sql
WITH sessions AS (
  SELECT
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    MAX((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_number')) AS session_number,
    MAX((SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'session_engaged')) AS session_engaged
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY session_key
)
SELECT
  COUNT(*) AS first_sessions,
  COUNTIF(session_engaged = '1') AS engaged_first_sessions,
  ROUND(COUNTIF(session_engaged = '1') * 100.0 / COUNT(*), 2) AS pct_engaged
FROM sessions
WHERE session_number = 1;
```

**Business Interpretation:** The cleanest non-monetary activation signal pairs directly with Query 2.10's channel breakdown of the same metric to identify both the overall baseline and channel-level deviations from it.

**Expected Output:** A single summary row; compare directly against Query 2.10's per-channel breakdown for consistency.

---

## SECTION 4 - Engagement Metrics Queries

### Query 4.1 - Engagement Rate (Weekly Trend)

**Purpose:** Implements Metric 4.1.

```sql
WITH sessions AS (
  SELECT
    DATE_TRUNC(PARSE_DATE('%Y%m%d', event_date), WEEK(MONDAY)) AS week_start,
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    MAX((SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'session_engaged')) AS session_engaged
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY week_start, session_key
)
SELECT
  week_start,
  COUNT(*) AS total_sessions,
  COUNTIF(session_engaged = '1') AS engaged_sessions,
  ROUND(COUNTIF(session_engaged = '1') * 100.0 / COUNT(*), 2) AS engagement_rate_pct
FROM sessions
GROUP BY week_start
ORDER BY week_start;
```

**Business Interpretation:** Track trend, not level. A multi-week decline is grounds to investigate a specific site change or content-quality issue before it shows up in the harder-to-reverse funnel/revenue metrics.

**Expected Output:** ~13-14 weekly rows; cross-reference against the Query 2.3 finding (rising new-user share) if engagement rate is stable while new-user share rises, the new visitors are at least engaging at the same rate as before, a partial mitigant to the retention concern.

### Query 4.2 - Average Engagement Time per Session (Capped, per R5)

**Purpose:** Implements Metric 4.2. [R5] mandatory cap applied before aggregation.

```sql
WITH capped_events AS (
  SELECT
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    LEAST(
      COALESCE((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'engagement_time_msec'), 0),
      3600000  -- [R5] cap at 1 hour per §6.5 outlier finding
    ) AS capped_engagement_ms
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
),
session_totals AS (
  SELECT session_key, SUM(capped_engagement_ms) AS session_engagement_ms
  FROM capped_events
  GROUP BY session_key
)
SELECT
  ROUND(AVG(session_engagement_ms) / 1000, 1) AS avg_engagement_seconds_per_session,
  APPROX_QUANTILES(session_engagement_ms, 2)[OFFSET(1)] / 1000 AS median_engagement_seconds
FROM session_totals;
```

**Business Interpretation:** Report mean AND median together per the §6.5 finding, even after capping individual events at 1 hour, a session with many capped events could still sum to an inflated total, so the median remains the more trustworthy headline number.

**Expected Output:** A single summary row; expect mean somewhat above median given the residual right-skew even after per-event capping.

### Query 4.3 - Pages per Session

**Purpose:** Implements Metric 4.3.

```sql
WITH sessions AS (
  SELECT
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    COUNTIF(event_name = 'page_view') AS page_views
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY session_key
)
SELECT
  ROUND(AVG(page_views), 2) AS avg_pages_per_session,
  APPROX_QUANTILES(page_views, 2)[OFFSET(1)] AS median_pages_per_session
FROM sessions;
```

**Business Interpretation:** Read alongside Query 4.2 (engagement time) high pages-per-session with low engagement time would suggest confused navigation rather than genuine interest; the two must always be reported as a pair, never independently.

**Expected Output:** A single summary row; median likely lower than mean given the presence of debug-traffic-linked duplicate page_view firing noted in earlier batches.

### Query 4.4 - Sessions per User

**Purpose:** Implements Metric 4.4. Confirms/refines the §2.2 validation baseline (1.33 sessions/user) with a distribution view, not just an average.

```sql
WITH user_sessions AS (
  SELECT
    user_pseudo_id,
    COUNT(DISTINCT CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)) AS session_count
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY user_pseudo_id
)
SELECT
  ROUND(AVG(session_count), 3) AS avg_sessions_per_user,
  APPROX_QUANTILES(session_count, 4) AS session_count_quartiles,
  COUNTIF(session_count = 1) AS single_session_users,
  ROUND(COUNTIF(session_count = 1) * 100.0 / COUNT(*), 2) AS pct_single_session_users
FROM user_sessions;
```

**Business Interpretation:** `pct_single_session_users` is the sharper, more decision-relevant number than the average it directly quantifies what share of the user base never comes back even once, setting up the Section 7 retention narrative with a concrete baseline figure rather than just an average.

**Expected Output:** A single summary row; given the 1.33 average, expect `pct_single_session_users` to be a large majority (likely 70%+) with a smaller tail of highly repeat-active users pulling the average up.

### Query 4.5 - Scroll Depth Distribution

**Purpose:** Uses `event_params.percent_scrolled`, a documented-but-not-yet-queried field from `02_Data_Understanding.md` §4.1 a genuine content-engagement signal worth including now that we're in the Engagement section.

```sql
SELECT
  (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'percent_scrolled') AS scroll_depth,
  COUNT(*) AS event_count
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
WHERE event_name = 'scroll'
GROUP BY scroll_depth
ORDER BY scroll_depth;
```

**Business Interpretation:** GA4's default scroll trigger fires at 90% depth, so a single dominant bucket at 90 is expected and not itself informative this query mainly exists to confirm whether any *other* scroll-depth thresholds were custom-configured (multiple distinct values would indicate a customized implementation worth knowing about for content-engagement analysis).

**Expected Output:** Likely a single dominant row at `scroll_depth = 90`; if only one value appears, this metric is not usable for granular content-engagement analysis and should be documented as such rather than forced into a chart.

---

**Batch 2 complete 9 queries across Section 2B (follow-ups), Section 3 (Activation), and Section 4 (Engagement).**

**Batch 2 execution notes (from actual BigQuery results):** Homepage normalization confirmed at 45.96% of entrances, exactly as estimated. Channel-level activation engagement is remarkably uniform (~70-71%) across organic/direct/referral/paid, with `cpc/google` the lowest of the real channels (70.11%) and `(data deleted)` a persistent outlier (96.01%, still an open investigation item). **Two decisive findings:** only 19.97% of first-ever sessions ever view a product (the real top-of-funnel leak, not checkout), and 82.47% of all users are single-session, one-and-done visitors hard confirmation of the "brand engagement, not repeat-revenue engine" framing. **One critical flag requiring resolution before further trend analysis:** Query 4.1 shows engagement rate jumping from ~47-58% to 85-91% in a single week (Dec 28) investigated below before proceeding.

### Query 4.6 - Diagnostic: Engagement Rate Step-Change Investigation (Dec 28 Anomaly)

**Purpose:** Investigates whether the Query 4.1 step-change is a real behavioral shift or an instrumentation/config change, by checking whether the underlying `session_engaged` parameter's *presence rate* (not just its value) shifts at the same point a config change would show up as a change in whether the parameter fires at all, not just what value it carries.

```sql
WITH daily_check AS (
  SELECT
    PARSE_DATE('%Y%m%d', event_date) AS event_dt,
    COUNT(*) AS total_events,
    COUNTIF((SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'session_engaged') IS NOT NULL) AS events_with_session_engaged_param,
    COUNTIF((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'engagement_time_msec') IS NOT NULL) AS events_with_engagement_time_param,
    COUNTIF((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'debug_mode') = 1) AS debug_events
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE event_name = 'page_view'
  GROUP BY event_dt
)
SELECT
  event_dt,
  total_events,
  ROUND(events_with_session_engaged_param * 100.0 / total_events, 2) AS pct_with_session_engaged_param,
  ROUND(events_with_engagement_time_param * 100.0 / total_events, 2) AS pct_with_engagement_time_param,
  ROUND(debug_events * 100.0 / total_events, 2) AS pct_debug
FROM daily_check
WHERE event_dt BETWEEN '2020-12-20' AND '2021-01-10'
ORDER BY event_dt;
```

**Business Interpretation:** If `pct_with_session_engaged_param` or `pct_debug` shows a sharp discontinuity exactly at Dec 28, that confirms a tagging/config change and the pre/post periods **cannot be compared as a single continuous trend line** without that caveat the Executive Dashboard must either split the series at this point with an annotation, or the pre-period must be treated as the more reliable baseline (it's more consistent week-to-week, which is itself evidence something changed rather than user behavior gradually shifting).

**Expected Output:** A daily table spanning Dec 20–Jan 10; the diagnostic question is a clean before/after break exactly on Dec 28 in one or more of the percentage columns that's the smoking gun for a config change rather than a real trend.

---

## SECTION 5 - Commerce Metrics Queries

### Query 5.1 - Total Revenue (Weekly, Source of Truth per R3)

**Purpose:** Implements Metric 5.1. [R3] mandatory transaction-level revenue only, never reconciled to item-level.

```sql
SELECT
  DATE_TRUNC(PARSE_DATE('%Y%m%d', event_date), WEEK(MONDAY)) AS week_start,
  ROUND(SUM(ecommerce.purchase_revenue_in_usd), 2) AS total_revenue_usd,
  COUNT(*) AS purchase_event_count
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
WHERE event_name = 'purchase'
GROUP BY week_start
ORDER BY week_start;
```

**Business Interpretation:** The Executive Dashboard hero revenue chart. Must display the Black Friday/Cyber Monday/Christmas annotation directly on the chart per `01_Project_Overview.md`, raw revenue trend without that flag will visually overstate "growth."

**Expected Output:** ~13-14 weekly rows; expect the same Nov-Dec peak/Jan decline shape already seen in Query 1.1 (WTU), since revenue and transacting-user-count should move together directionally.

### Query 5.2 - Average Order Value (Cleaned, per R2)

**Purpose:** Implements Metric 5.2. [R2] mandatory.

```sql
WITH cleaned_transactions AS (
  SELECT DISTINCT
    ecommerce.transaction_id AS transaction_id,
    user_pseudo_id,
    ecommerce.purchase_revenue_in_usd AS revenue
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE event_name = 'purchase'
    AND ecommerce.transaction_id IS NOT NULL
    AND ecommerce.transaction_id != '(not set)'
)
SELECT
  COUNT(*) AS cleaned_transaction_count,
  ROUND(SUM(revenue), 2) AS total_revenue_cleaned,
  ROUND(AVG(revenue), 2) AS aov,
  APPROX_QUANTILES(revenue, 2)[OFFSET(1)] AS median_order_value
FROM cleaned_transactions;
```

**Business Interpretation:** Report both AOV and median order value per the Metric 5.2 definition, the mean can be pulled upward by the bulk/corporate orders already confirmed in §6.5 validation (up to $1,530, 400 units).

**Expected Output:** A single summary row; AOV likely somewhat above the $48 median purchase value already observed in the §4.4 validation query, given known bulk-order outliers.

### Query 5.3 - Items per Transaction (Mean + Median, per R2)

**Purpose:** Implements Metric 5.3.

```sql
WITH cleaned_transactions AS (
  SELECT DISTINCT
    ecommerce.transaction_id AS transaction_id,
    user_pseudo_id,
    ecommerce.total_item_quantity AS qty
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE event_name = 'purchase'
    AND ecommerce.transaction_id IS NOT NULL
    AND ecommerce.transaction_id != '(not set)'
)
SELECT
  ROUND(AVG(qty), 2) AS avg_items_per_transaction,
  APPROX_QUANTILES(qty, 2)[OFFSET(1)] AS median_items_per_transaction,
  MAX(qty) AS max_items_single_transaction
FROM cleaned_transactions;
```

**Business Interpretation:** A large mean/median gap (expected, given the confirmed 400-unit bulk order) supports reporting median as the "typical customer" figure and flagging the mean separately as skewed by bulk/corporate buyers a distinct customer segment worth its own analysis, not blended into typical-shopper metrics.

**Expected Output:** Median likely 1-2 items; mean noticeably higher due to bulk-order skew.

### Query 5.4 - Refund Rate (Open Validation Item — Volume Check First)

**Purpose:** Implements Metric 5.4, but per its flagged Stage 3 open validation item, this query FIRST checks refund volume/completeness before the rate is trusted.

```sql
-- Step 1: Volume check (run first, per the open validation flag in 04_Metrics_Framework.md)
SELECT
  COUNT(*) AS refund_events,
  COUNTIF(ecommerce.refund_value_in_usd IS NOT NULL) AS refunds_with_value
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
WHERE event_name = 'refund';

-- Step 2: If volume is non-trivial, compute the rate
SELECT
  ROUND(SUM(refund.refund_value_in_usd), 2) AS total_refund_value,
  (SELECT ROUND(SUM(ecommerce.purchase_revenue_in_usd), 2)
   FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
   WHERE event_name = 'purchase') AS total_purchase_revenue,
  ROUND(SUM(refund.refund_value_in_usd) /
    (SELECT SUM(ecommerce.purchase_revenue_in_usd)
     FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
     WHERE event_name = 'purchase') * 100, 4) AS refund_rate_pct
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*` AS refund
WHERE event_name = 'refund';
```

**Business Interpretation:** Note there is no `refund` event in the 17 confirmed event types from Stage 1 discovery (`page_view` through `view_item_list`) this query is expected to return **zero rows**, which itself is the finding: refunds are either not tracked as a distinct event in this implementation, or captured only via the `ecommerce.refund_value_in_usd` field on other event types. **Do not report a 0% refund rate as "no refunds occur"** report it as "refund tracking is not observable via a dedicated event in this dataset," a data-limitation finding, not a business finding.

**Expected Output:** Likely 0 rows in Step 1 resolves the Stage 3 open validation item with a clear negative finding rather than leaving it ambiguous.

### Query 5.5 - Revenue per Session (RPS) by Device

**Purpose:** Implements Metric 5.5, cut by device directly tests whether the confirmed 58%/40%/2% desktop/mobile/tablet session split (§3.3) translates proportionally into revenue, or whether one device type over/under-monetizes relative to its traffic share.

```sql
WITH sessions_by_device AS (
  SELECT
    device.category AS device_category,
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY device_category, session_key
),
session_counts AS (
  SELECT device_category, COUNT(DISTINCT session_key) AS sessions
  FROM sessions_by_device GROUP BY device_category
),
revenue_by_device AS (
  SELECT device.category AS device_category, SUM(ecommerce.purchase_revenue_in_usd) AS revenue
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE event_name = 'purchase'
  GROUP BY device_category
)
SELECT
  s.device_category,
  s.sessions,
  ROUND(r.revenue, 2) AS revenue,
  ROUND(r.revenue / s.sessions, 4) AS revenue_per_session
FROM session_counts s
JOIN revenue_by_device r USING (device_category)
ORDER BY revenue_per_session DESC;
```

**Business Interpretation:** If desktop's revenue-per-session is disproportionately higher than its 58% session share would suggest, that's a strong, concrete argument for a mobile-checkout UX investigation in Stage 6 this single query can make or break the "mobile experience needs investment" recommendation.

**Expected Output:** Three rows (desktop/mobile/tablet); given this dataset skews desktop-majority (an atypical pattern for ecommerce generally), it's plausible desktop also converts better must be confirmed from actual output, not assumed from industry norms.

---

## SECTION 6 - Funnel Metrics Queries

### Query 6.1 - View-to-Cart Rate (Session-Scoped)

**Purpose:** Implements Metric 6.1. [R4] applies to any product-level cut, not to this session-presence version.

```sql
WITH session_flags AS (
  SELECT
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    MAX(IF(event_name = 'view_item', 1, 0)) AS viewed_item,
    MAX(IF(event_name = 'add_to_cart', 1, 0)) AS added_to_cart
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY session_key
)
SELECT
  SUM(viewed_item) AS sessions_with_view,
  SUM(IF(viewed_item = 1 AND added_to_cart = 1, 1, 0)) AS sessions_view_and_cart,
  ROUND(SUM(IF(viewed_item = 1 AND added_to_cart = 1, 1, 0)) * 100.0 / SUM(viewed_item), 2) AS view_to_cart_rate_pct
FROM session_flags;
```

**Business Interpretation:** This is the session-scoped rate that should replace the naive raw-event ratio (~15%) referenced in `02_Data_Understanding.md`. Compare the two directly a materially different session-scoped number is itself worth reporting as a methodology finding.

**Expected Output:** A single summary row; given only 19.97% of *first* sessions ever view a product (Query 3.1) but this query covers *all* sessions (including returning ones, which likely convert better), expect this overall rate to be meaningfully higher than the first-session-only activation figure.

### Query 6.2 - Cart-to-Checkout Rate (Session-Scoped)

**Purpose:** Implements Metric 6.2.

```sql
WITH session_flags AS (
  SELECT
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    MAX(IF(event_name = 'add_to_cart', 1, 0)) AS added_to_cart,
    MAX(IF(event_name = 'begin_checkout', 1, 0)) AS began_checkout
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY session_key
)
SELECT
  SUM(added_to_cart) AS sessions_with_cart,
  SUM(IF(added_to_cart = 1 AND began_checkout = 1, 1, 0)) AS sessions_cart_and_checkout,
  ROUND(SUM(IF(added_to_cart = 1 AND began_checkout = 1, 1, 0)) * 100.0 / SUM(added_to_cart), 2) AS cart_to_checkout_rate_pct
FROM session_flags;
```

**Business Interpretation:** Compare against the ~66% naive event-level ratio noted in `04_Metrics_Framework.md` Metric 6.2 session-scoping should produce a similar or slightly higher number here, since this step doesn't have the item-population coverage gap that view_item has.

**Expected Output:** A single summary row, plausibly in the 55-70% range.

### Query 6.3 - Checkout-to-Purchase Rate with Granular Step Breakdown

**Purpose:** Implements Metric 6.3, including the granular 3-step breakdown explicitly called for in its Common Mistakes guidance.

```sql
WITH session_flags AS (
  SELECT
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    MAX(IF(event_name = 'begin_checkout', 1, 0)) AS began_checkout,
    MAX(IF(event_name = 'add_shipping_info', 1, 0)) AS added_shipping,
    MAX(IF(event_name = 'add_payment_info', 1, 0)) AS added_payment,
    MAX(IF(event_name = 'purchase', 1, 0)) AS purchased
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY session_key
)
SELECT
  SUM(began_checkout) AS step1_begin_checkout,
  SUM(IF(began_checkout=1 AND added_shipping=1, 1, 0)) AS step2_shipping_info,
  SUM(IF(began_checkout=1 AND added_shipping=1 AND added_payment=1, 1, 0)) AS step3_payment_info,
  SUM(IF(began_checkout=1 AND added_shipping=1 AND added_payment=1 AND purchased=1, 1, 0)) AS step4_purchase,
  ROUND(SUM(IF(began_checkout=1 AND added_shipping=1 AND added_payment=1 AND purchased=1, 1, 0)) * 100.0 / SUM(began_checkout), 2) AS overall_checkout_completion_pct
FROM session_flags;
```

**Business Interpretation:** This is the query that pinpoints exactly which checkout sub-step (shipping-info entry, payment-info entry, or final submission) loses the most sessions the single most actionable output in the entire funnel analysis, since each stage implies a different fix (shipping-cost transparency vs. payment-method friction vs. a technical submission bug).

**Expected Output:** A monotonically decreasing 4-step waterfall; the stage with the steepest percentage drop from the prior stage is the number-one Stage 7 recommendation priority.

### Query 6.4 - Overall View-to-Purchase Conversion Rate (with Reconciliation Check)

**Purpose:** Implements Metric 6.4, including the self-consistency check called for in its SQL Logic guidance (product of stage rates should approximately equal the direct end-to-end rate).

```sql
WITH session_flags AS (
  SELECT
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    MAX(IF(event_name = 'view_item', 1, 0)) AS viewed_item,
    MAX(IF(event_name = 'purchase', 1, 0)) AS purchased
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY session_key
)
SELECT
  SUM(viewed_item) AS sessions_with_view,
  SUM(IF(viewed_item = 1 AND purchased = 1, 1, 0)) AS sessions_view_to_purchase,
  ROUND(SUM(IF(viewed_item = 1 AND purchased = 1, 1, 0)) * 100.0 / SUM(viewed_item), 3) AS view_to_purchase_rate_pct
FROM session_flags;
```

**Business Interpretation:** The single headline funnel number for leadership **must always be presented alongside Queries 6.1-6.3's breakdown**, never in isolation, per the Metric 6.4 Common Mistakes guidance.

**Expected Output:** A single summary row; sanity-check this result against the product of Query 6.1 × 6.2 × 6.3's rates they should be in the same ballpark (not identical, due to sessions that skip stages non-linearly, but a large divergence would indicate a session-key construction bug worth investigating.)

### Query 6.5 — Cart Abandonment Rate by Device

**Purpose:** Implements Metric 6.5, cut by device to directly test whether mobile shows higher abandonment than desktop a natural follow-up to Query 5.5's device-revenue finding.

```sql
WITH session_flags AS (
  SELECT
    device.category AS device_category,
    CONCAT(user_pseudo_id, '-',
      CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)
    ) AS session_key,
    MAX(IF(event_name = 'add_to_cart', 1, 0)) AS added_to_cart,
    MAX(IF(event_name = 'purchase', 1, 0)) AS purchased
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY device_category, session_key
)
SELECT
  device_category,
  SUM(added_to_cart) AS sessions_with_cart,
  SUM(IF(added_to_cart=1 AND purchased=0, 1, 0)) AS abandoned_sessions,
  ROUND(SUM(IF(added_to_cart=1 AND purchased=0, 1, 0)) * 100.0 / SUM(added_to_cart), 2) AS cart_abandonment_rate_pct
FROM session_flags
WHERE device_category IN ('desktop', 'mobile', 'tablet')
GROUP BY device_category
ORDER BY cart_abandonment_rate_pct DESC;
```

**Business Interpretation:** If mobile shows a materially higher abandonment rate than desktop, this becomes one of the highest-confidence, most specific recommendations in `07_Business_Recommendations.md` "fix mobile checkout" is a much stronger, more actionable claim when backed by this exact comparison than a generic "reduce cart abandonment" statement.

**Expected Output:** Three rows; industry pattern generally shows mobile abandonment higher than desktop, but must be confirmed against this dataset's actual desktop-majority, higher-engagement pattern rather than assumed from general ecommerce benchmarks.

---

**Batch 3 complete : 11 queries across Section 4 diagnostic follow-up, Section 5 (Commerce), and Section 6 (Funnel).**

**Batch 3 execution notes (from actual BigQuery results):** [R6] new rule established Query 4.6 confirms `engagement_time_msec` presence jumps from a flat 0.0% to 42-44% exactly at Dec 28, 2020, proving the Query 4.1 engagement-rate step-change is a tagging/config change, not real behavior; pre/post periods are not comparable without a split annotation. Revenue cross-checks cleanly ($362,165 confirmed two independent ways). [R2] cleaning removes 14.7% of revenue ($53,335) from AOV scope. Refund Rate open item resolved: 0 refund events exist. Mobile slightly out-monetizes desktop per session despite lower volume. Cart-to-checkout naive event ratio (66%) vs. true session-scoped rate (39.25%) is a 27-point gap the funnel-grain trap materializing in real data. Checkout's real bottleneck is shipping→payment info (38.6% loss), not payment→purchase. View-to-purchase (6.1%) nearly doubles the naive stage-multiplication estimate (3.4%), suggesting cross-session cart persistence tested below. Desktop shows the highest cart abandonment (81.73%), inverting the common mobile-worse assumption.

### Query 6.6 - Cross-Session Cart Persistence Test

**Purpose:** Directly tests the hypothesis raised by Query 6.4's divergence that a meaningful share of `begin_checkout` sessions had their cart built in a *prior* session by the same user, not the same session, which would explain why session-scoped funnel multiplication understates the true end-to-end rate.

```sql
WITH cart_events AS (
  SELECT user_pseudo_id, event_timestamp,
    CONCAT(user_pseudo_id, '-', CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)) AS session_key
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE event_name = 'add_to_cart'
),
checkout_events AS (
  SELECT user_pseudo_id, event_timestamp,
    CONCAT(user_pseudo_id, '-', CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)) AS session_key
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE event_name = 'begin_checkout'
),
checkout_with_prior_cart AS (
  SELECT DISTINCT
    c.session_key AS checkout_session_key,
    c.user_pseudo_id,
    IF(EXISTS (
      SELECT 1 FROM cart_events ce
      WHERE ce.user_pseudo_id = c.user_pseudo_id
        AND ce.session_key = c.session_key
    ), 'same_session_cart',
    IF(EXISTS (
      SELECT 1 FROM cart_events ce
      WHERE ce.user_pseudo_id = c.user_pseudo_id
        AND ce.event_timestamp < c.event_timestamp
        AND ce.session_key != c.session_key
    ), 'prior_session_cart', 'no_cart_found')
    ) AS cart_origin
  FROM checkout_events c
)
SELECT cart_origin, COUNT(*) AS checkout_sessions, ROUND(COUNT(*)*100.0/SUM(COUNT(*)) OVER (), 2) AS pct
FROM checkout_with_prior_cart
GROUP BY cart_origin
ORDER BY checkout_sessions DESC;
```

**Business Interpretation:** If `prior_session_cart` accounts for a meaningful share, it confirms carts persist across sessions (standard ecommerce behavior cart contents typically survive via cookies/local storage independent of GA4 session boundaries), meaning true funnel and cart-abandonment analysis should ultimately be done at the **user** grain across sessions, not purely within a single session a methodology refinement for `06_Analysis.md`. A large `no_cart_found` share would instead suggest "buy now"/direct-purchase flows that skip the cart page entirely.

**Expected Output:** Three categories; given the magnitude of the divergence in Query 6.4 (11,106 vs. 5,961), expect `prior_session_cart` and/or `no_cart_found` combined to represent a substantial minority of checkout sessions the exact split determines which explanation (cart persistence vs. buy-now flow) dominates.

---

## SECTION 7 - Retention Metrics Queries

### Query 7.1 - Day 1 / Day 7 / Day 30 Retention

**Purpose:** Implements Metric 7.1.

```sql
WITH first_touch AS (
  SELECT DISTINCT user_pseudo_id,
    DATE(TIMESTAMP_MICROS(user_first_touch_timestamp)) AS first_touch_date
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
),
user_active_dates AS (
  SELECT DISTINCT user_pseudo_id, PARSE_DATE('%Y%m%d', event_date) AS active_date
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
),
cohort AS (
  SELECT user_pseudo_id, first_touch_date
  FROM first_touch
  WHERE first_touch_date BETWEEN '2020-11-01' AND '2021-01-01'  -- ensures a full 30-day observation window
)
SELECT
  COUNT(DISTINCT c.user_pseudo_id) AS cohort_size,
  COUNT(DISTINCT IF(a.active_date = DATE_ADD(c.first_touch_date, INTERVAL 1 DAY), c.user_pseudo_id, NULL)) AS day1_retained,
  COUNT(DISTINCT IF(a.active_date = DATE_ADD(c.first_touch_date, INTERVAL 7 DAY), c.user_pseudo_id, NULL)) AS day7_retained,
  COUNT(DISTINCT IF(a.active_date = DATE_ADD(c.first_touch_date, INTERVAL 30 DAY), c.user_pseudo_id, NULL)) AS day30_retained
FROM cohort c
LEFT JOIN user_active_dates a ON a.user_pseudo_id = c.user_pseudo_id;
```

**Business Interpretation:** Given the confirmed 82.47% single-session-user rate (Query 4.4), expect all three retention figures to be low single digits this query quantifies exactly how low, replacing the qualitative expectation with a hard number for the Executive Summary.

**Expected Output:** A single summary row with `cohort_size` in the low hundreds of thousands and day1/7/30 retained counts each a small fraction of it.

### Query 7.2 - Weekly Returning User Rate

**Purpose:** Implements Metric 7.2.

```sql
WITH user_weeks AS (
  SELECT DISTINCT user_pseudo_id,
    DATE_TRUNC(PARSE_DATE('%Y%m%d', event_date), WEEK(MONDAY)) AS active_week
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
),
weekly_status AS (
  SELECT
    uw.active_week,
    uw.user_pseudo_id,
    EXISTS (
      SELECT 1 FROM user_weeks prior
      WHERE prior.user_pseudo_id = uw.user_pseudo_id AND prior.active_week < uw.active_week
    ) AS is_returning
  FROM user_weeks uw
)
SELECT
  active_week,
  COUNT(*) AS active_users,
  COUNTIF(is_returning) AS returning_users,
  ROUND(COUNTIF(is_returning) * 100.0 / COUNT(*), 2) AS returning_user_rate_pct
FROM weekly_status
GROUP BY active_week
ORDER BY active_week;
```

**Business Interpretation:** Should be the complement of Query 2.3's New User Rate (returning_rate ≈ 100% - new_user_rate) a useful internal consistency check; given Query 2.3 already showed new-user rate rising toward 90%+, expect this to show a correspondingly low and falling returning-user rate.

**Expected Output:** ~13-14 weekly rows; expect returning-user rate to decline over the window, mirroring the inverse of the already-confirmed rising new-user-rate trend.

### Query 7.3 - Repeat Purchase Rate (Cleaned, per R2)

**Purpose:** Implements Metric 7.3. [R2] mandatory.

```sql
WITH cleaned_purchases AS (
  SELECT DISTINCT user_pseudo_id, ecommerce.transaction_id AS transaction_id
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE event_name = 'purchase'
    AND ecommerce.transaction_id IS NOT NULL
    AND ecommerce.transaction_id != '(not set)'
),
user_txn_counts AS (
  SELECT user_pseudo_id, COUNT(DISTINCT transaction_id) AS txn_count
  FROM cleaned_purchases
  GROUP BY user_pseudo_id
)
SELECT
  COUNT(*) AS purchasing_users,
  COUNTIF(txn_count >= 2) AS repeat_purchasers,
  ROUND(COUNTIF(txn_count >= 2) * 100.0 / COUNT(*), 2) AS repeat_purchase_rate_pct
FROM user_txn_counts;
```

**Business Interpretation:** This is the single most important number for the "brand engagement vehicle vs. repeat-revenue business" framing question posed in `01_Project_Overview.md` now answerable with a precise figure rather than a hypothesis.

**Expected Output:** A single summary row; given the 82.47% single-session-user finding, expect this rate to be very low (likely single digits), reinforcing that repeat purchasing is the exception, not the norm, for this store.

### Query 7.4 - New vs. Returning Session Mix (Weekly Trend)

**Purpose:** Implements Metric 7.4.

```sql
WITH sessions AS (
  SELECT
    DATE_TRUNC(PARSE_DATE('%Y%m%d', event_date), WEEK(MONDAY)) AS week_start,
    CONCAT(user_pseudo_id, '-', CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)) AS session_key,
    MAX((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_number')) AS session_number
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY week_start, session_key
)
SELECT
  week_start,
  COUNT(*) AS total_sessions,
  COUNTIF(session_number = 1) AS new_sessions,
  COUNTIF(session_number > 1) AS returning_sessions,
  ROUND(COUNTIF(session_number > 1) * 100.0 / COUNT(*), 2) AS pct_returning_sessions
FROM sessions
GROUP BY week_start
ORDER BY week_start;
```

**Business Interpretation:** The fast-moving, session-grain daily/weekly monitoring proxy flags a sudden retention shift well before the slower, more rigorous Query 7.2/7.3 user-grain metrics would surface it in a monthly report.

**Expected Output:** ~13-14 weekly rows; directionally should track Query 7.2 closely, since both measure the same underlying new/returning split from different angles (session vs. user grain).

---

## SECTION 8 - Customer Metrics Queries

### Query 8.1 - LTV Snapshot Coverage & Distribution (Open Item Resolution)

**Purpose:** Implements Metric 8.1, but first resolves its Stage 3/4 open validation item a null-rate/distribution check on `user_ltv.revenue` that was never run in Stage 3.

```sql
-- Step 1: Coverage check (resolves the open validation item)
SELECT
  COUNT(DISTINCT user_pseudo_id) AS total_users,
  COUNT(DISTINCT IF(user_ltv.revenue IS NULL, user_pseudo_id, NULL)) AS users_with_null_ltv,
  COUNT(DISTINCT IF(SAFE_CAST(user_ltv.revenue AS FLOAT64) > 0, user_pseudo_id, NULL)) AS users_with_positive_ltv
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`;

-- Step 2: Distribution, using MAX per user per the mandatory Metric 8.1 rule (never AVG)
WITH user_ltv_snapshot AS (
  SELECT user_pseudo_id, MAX(SAFE_CAST(user_ltv.revenue AS FLOAT64)) AS max_ltv
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY user_pseudo_id
)
SELECT
  ROUND(AVG(max_ltv), 2) AS avg_ltv_snapshot,
  APPROX_QUANTILES(max_ltv, 4) AS ltv_quartiles,
  COUNTIF(max_ltv > 0) AS users_with_any_value
FROM user_ltv_snapshot;
```

**Business Interpretation:** Resolves the open item flagged in `04_Metrics_Framework.md` §8.1 if `users_with_null_ltv` is near 0 and the distribution is sane (not all zeros), the field is confirmed usable; if nearly all values are 0 or null, this metric must be down-weighted or dropped from the Executive Scorecard rather than reported with false confidence.

**Expected Output:** Coverage should be near-complete (this is a per-event struct, always present, just often 0 before a purchase); the quartile distribution's most useful number will likely be the 75th/max percentiles, since most users (non-purchasers) will show 0.

### Query 8.2 - Purchase Frequency

**Purpose:** Implements Metric 8.2. [R2] mandatory.

```sql
WITH cleaned_purchases AS (
  SELECT DISTINCT user_pseudo_id, ecommerce.transaction_id AS transaction_id
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE event_name = 'purchase'
    AND ecommerce.transaction_id IS NOT NULL
    AND ecommerce.transaction_id != '(not set)'
)
SELECT
  COUNT(DISTINCT user_pseudo_id) AS purchasing_users,
  COUNT(DISTINCT transaction_id) AS total_valid_transactions,
  ROUND(COUNT(DISTINCT transaction_id) * 1.0 / COUNT(DISTINCT user_pseudo_id), 3) AS purchase_frequency
FROM cleaned_purchases;
```

**Business Interpretation:** Read directly alongside Query 7.3's Repeat Purchase Rate together they answer two different questions: "do people come back at all" (7.3) vs. "how many times do they buy given that they do" (this metric).

**Expected Output:** A single summary row; given the low repeat-purchase-rate expectation, this should sit close to 1.0, with any meaningful excess above 1.0 itself worth flagging as a notable finding.

### Query 8.3 - Value Tier Segmentation

**Purpose:** Implements Metric 8.3, using LTV quartiles from Query 8.1.

```sql
WITH user_ltv_snapshot AS (
  SELECT user_pseudo_id, MAX(SAFE_CAST(user_ltv.revenue AS FLOAT64)) AS max_ltv
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY user_pseudo_id
),
tiered AS (
  SELECT
    user_pseudo_id,
    max_ltv,
    CASE
      WHEN max_ltv = 0 OR max_ltv IS NULL THEN 'Non-purchaser'
      WHEN max_ltv < 30 THEN 'Low value'
      WHEN max_ltv < 100 THEN 'Medium value'
      ELSE 'High value'
    END AS value_tier
  FROM user_ltv_snapshot
)
SELECT
  value_tier,
  COUNT(*) AS user_count,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 3) AS pct_of_users,
  ROUND(SUM(max_ltv), 2) AS tier_total_ltv
FROM tiered
GROUP BY value_tier
ORDER BY tier_total_ltv DESC;
```

**Business Interpretation:** Quantifies exactly how thin the "High value" segment is in absolute user count the critical caveat flagged in Metric 8.3's Common Mistakes guidance (never cite tier behavior without stating tier size).

**Expected Output:** Four tiers; given only 4,466 cleaned purchasing users out of 270,154 total (1.65%), expect "Non-purchaser" to represent the overwhelming majority (~98%+) of users, with "High value" a very small absolute count must be stated explicitly alongside any tier-based finding in Stage 6/7.

---

**Batch 4 complete : 8 queries across Section 6 follow-up (cart persistence), Section 7 (Retention), and Section 8 (Customer).**

**Batch 4 execution notes (from actual BigQuery results):** Query 6.6 partially overturns the cart-persistence hypothesis cross-session persistence is minor (3.41%); the dominant factor is 42.91% of checkout sessions having no detectable add_to_cart at all (buy-now flow or tracking gap, flagged as open UX/instrumentation question). Day 1/7/30 retention confirmed at 4.63% / 0.71% / 0.13% the headline retention figure for the Executive Summary. Repeat Purchase Rate came in higher than predicted at 11.85%, showing past purchasers are meaningfully warmer than the general population despite near-zero general session retention. **[R3] extended:** `user_ltv.revenue` sums to $407,626 vs. the validated $362,165 real revenue a 12.6% overstatement; usable only for relative segmentation, never as a revenue figure. Value Tier Segmentation confirms extreme concentration: 0.43% of users ("High value" tier, 1,161 people) drive ~62% of measured customer value.

---

## SECTION 9 - Product Metrics Queries

### Query 9.1 - Product View Count (Top 20, per R4)

**Purpose:** Implements Metric 9.1. [R4] mandatory scoped to items-populated `view_item` rows only (62.65% coverage per §5.2).

```sql
SELECT
  item.item_name,
  item.item_category,
  COUNT(*) AS view_count
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`,
  UNNEST(items) AS item
WHERE event_name = 'view_item'
GROUP BY item.item_name, item.item_category
ORDER BY view_count DESC
LIMIT 20;
```

**Business Interpretation:** Baseline demand-signal ranking must be captioned "among item-identifiable views (62.65% of all view_item events)" per [R4], not presented as total product-view volume.

**Expected Output:** A top-20 product list; given the entrance-page data from Query 2.5 (Apparel, YouTube-brand, Dino Game Tee were top landing categories), expect those same products/categories to dominate this ranking too.

### Query 9.2 - Product-Level View-to-Cart Rate (Top 20 by Volume, Min Threshold)

**Purpose:** Implements Metric 9.2. Applies a minimum-view threshold to avoid noisy rates from low-volume products (a direct application of the statistical-significance caution embedded in the metric's Common Mistakes guidance).

```sql
WITH views AS (
  SELECT item.item_id, item.item_name, COUNT(*) AS views
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`, UNNEST(items) AS item
  WHERE event_name = 'view_item'
  GROUP BY item.item_id, item.item_name
),
carts AS (
  SELECT item.item_id, COUNT(*) AS carts
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`, UNNEST(items) AS item
  WHERE event_name = 'add_to_cart'
  GROUP BY item.item_id
)
SELECT
  v.item_name,
  v.views,
  COALESCE(c.carts, 0) AS carts,
  ROUND(COALESCE(c.carts, 0) * 100.0 / v.views, 2) AS view_to_cart_rate_pct
FROM views v
LEFT JOIN carts c USING (item_id)
WHERE v.views >= 100  -- minimum volume threshold to avoid noisy small-sample rates
ORDER BY v.views DESC
LIMIT 20;
```

**Business Interpretation:** Products with high views but a rate well below the ~19.7% overall session-scoped baseline (Query 6.1) are pricing/presentation problems; products with low relative views but a high rate are discoverability opportunities both are actionable, opposite-direction recommendations.

**Expected Output:** Top 20 products by view volume with individual conversion rates; expect meaningful spread around the ~15-20% baseline, not uniform performance.

### Query 9.3 - Product Revenue Share by Category (Relative Only, per R3)

**Purpose:** Implements Metric 9.3 at category grain. [R3] mandatory never reconciled to transaction-level Total Revenue.

```sql
SELECT
  item.item_category,
  ROUND(SUM(item.item_revenue_in_usd), 2) AS category_revenue,
  ROUND(SUM(item.item_revenue_in_usd) * 100.0 / SUM(SUM(item.item_revenue_in_usd)) OVER (), 2) AS pct_of_item_revenue
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`, UNNEST(items) AS item
WHERE event_name = 'purchase'
GROUP BY item.item_category
ORDER BY category_revenue DESC;
```

**Business Interpretation:** Shows revenue concentration by category a small number of categories (likely Apparel, given its dominance in landing pages and views) probably account for a disproportionate share, informing merchandising/homepage placement priorities.

**Expected Output:** A ranked list of item categories; total will not exactly equal the validated $362,165 headline revenue per [R3] expected and documented, not an error to reconcile.

### Query 9.4 - Category Performance Index (Views, Conversion, Revenue Combined)

**Purpose:** Implements Metric 9.4, the composite rollup combining 9.1-9.3 at category grain for a single merchandising-review view.

```sql
WITH views AS (
  SELECT item.item_category AS category, COUNT(*) AS views
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`, UNNEST(items) AS item
  WHERE event_name = 'view_item'
  GROUP BY category
),
carts AS (
  SELECT item.item_category AS category, COUNT(*) AS carts
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`, UNNEST(items) AS item
  WHERE event_name = 'add_to_cart'
  GROUP BY category
),
revenue AS (
  SELECT item.item_category AS category, SUM(item.item_revenue_in_usd) AS revenue
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`, UNNEST(items) AS item
  WHERE event_name = 'purchase'
  GROUP BY category
)
SELECT
  v.category,
  v.views,
  COALESCE(c.carts, 0) AS carts,
  ROUND(COALESCE(c.carts, 0) * 100.0 / v.views, 2) AS view_to_cart_rate_pct,
  ROUND(COALESCE(r.revenue, 0), 2) AS revenue,
  ROUND(COALESCE(r.revenue, 0) * 100.0 / SUM(COALESCE(r.revenue, 0)) OVER (), 2) AS pct_of_revenue
FROM views v
LEFT JOIN carts c USING (category)
LEFT JOIN revenue r USING (category)
WHERE v.views >= 500
ORDER BY revenue DESC;
```

**Business Interpretation:** A single table for the merchandising planning review instantly flags any category that's high-traffic/low-conversion (fix the PDP/pricing) versus low-traffic/high-conversion (invest in visibility/promotion).

**Expected Output:** A ranked category table with 4 comparable metrics side by side, ready to drop directly into a Stage 6/7 merchandising recommendation.

---

## SECTION 10 - Marketing Metrics Queries

### Query 10.1 - Channel Conversion Rate (Session-Scoped, Volume-Qualified)

**Purpose:** Implements Metric 10.1, paired explicitly with volume (per its own Common Mistakes guidance against ranking by rate alone).

```sql
WITH sessions AS (
  SELECT
    CONCAT(user_pseudo_id, '-', CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)) AS session_key,
    COALESCE(traffic_source.medium, '(not set)') AS medium,
    COALESCE(traffic_source.source, '(not set)') AS source,
    MAX(IF(event_name = 'purchase', 1, 0)) AS purchased
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY session_key, medium, source
)
SELECT
  medium, source,
  COUNT(*) AS sessions,
  SUM(purchased) AS purchasing_sessions,
  ROUND(SUM(purchased) * 100.0 / COUNT(*), 3) AS conversion_rate_pct
FROM sessions
GROUP BY medium, source
HAVING sessions > 500
ORDER BY sessions DESC;
```

**Business Interpretation:** Must be read together, never rate alone a channel with few sessions and a lucky high rate is not a signal to reallocate budget. Compare directly against Query 2.4's session-share table for the full volume+quality picture.

**Expected Output:** A ranked table of major channels with both volume and rate visible in the same view designed specifically to prevent the volume/quality conflation the metric warns against.

### Query 10.2 - Channel Revenue Share (Attributed, per R2/R3)

**Purpose:** Implements Metric 10.2. Applies [R2] transaction cleaning before attribution.

```sql
WITH purchase_sessions AS (
  SELECT
    CONCAT(user_pseudo_id, '-', CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)) AS session_key,
    COALESCE(traffic_source.medium, '(not set)') AS medium,
    COALESCE(traffic_source.source, '(not set)') AS source,
    ecommerce.transaction_id AS transaction_id,
    ecommerce.purchase_revenue_in_usd AS revenue
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE event_name = 'purchase'
    AND ecommerce.transaction_id IS NOT NULL
    AND ecommerce.transaction_id != '(not set)'
)
SELECT
  medium, source,
  COUNT(DISTINCT transaction_id) AS transactions,
  ROUND(SUM(revenue), 2) AS revenue,
  ROUND(SUM(revenue) * 100.0 / SUM(SUM(revenue)) OVER (), 2) AS pct_of_revenue
FROM purchase_sessions
GROUP BY medium, source
ORDER BY revenue DESC;
```

**Business Interpretation:** Read directly against Query 10.1's conversion rate for the same channels a channel with a large revenue share driven purely by volume (not rate) should not be mistaken for the "best" channel in a quality sense.

**Expected Output:** Total across all rows will be close to, but slightly below, the cleaned $308,830 figure from Query 5.2 (session-attribution join may drop a small number of edge-case sessions without a resolvable session_key match) a small gap here is expected, not an error.

### Query 10.3 - New User Share by Channel

**Purpose:** Implements Metric 10.3.

```sql
WITH new_user_flags AS (
  SELECT DISTINCT
    user_pseudo_id,
    COALESCE(traffic_source.medium, '(not set)') AS medium,
    COALESCE(traffic_source.source, '(not set)') AS source
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_number') = 1
)
SELECT
  medium, source,
  COUNT(DISTINCT user_pseudo_id) AS new_users,
  ROUND(COUNT(DISTINCT user_pseudo_id) * 100.0 / SUM(COUNT(DISTINCT user_pseudo_id)) OVER (), 2) AS pct_of_new_users
FROM new_user_flags
GROUP BY medium, source
ORDER BY new_users DESC;
```

**Business Interpretation:** Distinguishes discovery channels (high new-user share) from retention/re-engagement channels (low new-user share, high returning share per Query 2.7) the two should be judged on different rubrics, not the same "did it bring new users" yardstick.

**Expected Output:** A ranked channel table; `cpc/google` and `organic/google` should dominate new-user share (consistent with Query 2.7's earlier finding that paid search skews almost entirely new).

### Query 10.4 - Organic vs. Paid Traffic Mix (Corrected Medium Classification)

**Purpose:** Implements Metric 10.4, using `traffic_source.medium` directly (not `gclid` presence, per the explicit warning in the metric's Common Mistakes guidance).

```sql
WITH sessions AS (
  SELECT
    CONCAT(user_pseudo_id, '-', CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)) AS session_key,
    CASE
      WHEN traffic_source.medium IN ('cpc', 'paid', 'display', 'ppc') THEN 'Paid'
      WHEN traffic_source.medium IN ('organic', 'referral') THEN 'Organic/Referral'
      WHEN traffic_source.medium = '(none)' THEN 'Direct'
      ELSE 'Other/Unclassified'
    END AS traffic_type
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY session_key, traffic_type
)
SELECT
  traffic_type,
  COUNT(*) AS sessions,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS pct_of_sessions
FROM sessions
GROUP BY traffic_type
ORDER BY sessions DESC;
```

**Business Interpretation:** Given Query 2.4's confirmed channel mix (`cpc/google` only 4.34% of sessions), expect this to show heavy organic/direct dependence a real growth-headroom finding: paid search is barely used relative to its typical share in comparable ecommerce sites, suggesting room for incremental paid investment as a growth lever.

**Expected Output:** Four categories; "Paid" likely in the low single digits of total sessions, consistent with the 4.34% cpc/google figure already confirmed.

---

**Batch 5 complete : 8 queries across Section 9 (Product) and Section 10 (Marketing).**

**Batch 5 execution notes (from actual BigQuery results):** **Critical data-quality finding** `item_category` is tagged inconsistently between browsing and purchase events (a full breadcrumb path like `"Home/Apparel/Men's / Unisex/"` on view/cart events vs. a clean label like `"Apparel"` on purchase events), causing high-traffic categories to falsely appear as 0%-converting in Query 9.4. Fix via `item_id` join, added below as Query 9.5. `(data deleted)` confirmed as a genuinely exceptional segment: 3.14% conversion, 12.97% of revenue from 6.17% of sessions the best-converting channel in the dataset. Paid search (`cpc/google`) converts *below* organic google (0.98% vs. 1.11%), refining last batch's "clear headroom for paid" read into "fix targeting before scaling spend." Apparel dominates item revenue at 47.42%.

### Query 9.5 - Category Performance Index, Corrected (item_id-based, not item_category-based)

**Purpose:** Fixes the inconsistent-taxonomy issue discovered in Query 9.4 by deriving each product's category from its `purchase`-event tagging (the clean version) and joining that back to browsing-stage data via the stable `item_id`, rather than trusting `item_category` directly on browsing events.

```sql
WITH canonical_category AS (
  -- Build a clean item_id -> category lookup using only purchase-event tagging (the clean label form)
  SELECT item.item_id, ANY_VALUE(item.item_category) AS clean_category
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`, UNNEST(items) AS item
  WHERE event_name = 'purchase' AND item.item_category NOT LIKE '%/%'
  GROUP BY item.item_id
),
views AS (
  SELECT item.item_id, COUNT(*) AS views
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`, UNNEST(items) AS item
  WHERE event_name = 'view_item'
  GROUP BY item.item_id
),
carts AS (
  SELECT item.item_id, COUNT(*) AS carts
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`, UNNEST(items) AS item
  WHERE event_name = 'add_to_cart'
  GROUP BY item.item_id
),
revenue AS (
  SELECT item.item_id, SUM(item.item_revenue_in_usd) AS revenue
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`, UNNEST(items) AS item
  WHERE event_name = 'purchase'
  GROUP BY item.item_id
)
SELECT
  COALESCE(cc.clean_category, 'Unmapped (no purchase-tagged category found)') AS category,
  SUM(v.views) AS views,
  SUM(COALESCE(c.carts, 0)) AS carts,
  ROUND(SUM(COALESCE(c.carts, 0)) * 100.0 / SUM(v.views), 2) AS view_to_cart_rate_pct,
  ROUND(SUM(COALESCE(r.revenue, 0)), 2) AS revenue
FROM views v
LEFT JOIN canonical_category cc USING (item_id)
LEFT JOIN carts c USING (item_id)
LEFT JOIN revenue r USING (item_id)
GROUP BY category
HAVING views >= 500
ORDER BY revenue DESC;
```

### Query 9.6 - Category Performance Index, Corrected via item_name (Diagnostic Fix for Query 9.5's Failure)

**Purpose:** Query 9.5's `item_id`-based join returned 100% "Unmapped" meaning `item_id` itself doesn't join cleanly across event types in this implementation, a second, deeper instance of the same inconsistent-tagging pattern found on `item_category`. Switching to `item_name`, which Queries 9.1/9.2 already showed matching cleanly across view/cart events.

```sql
WITH canonical_category AS (
  SELECT item.item_name, ANY_VALUE(item.item_category) AS clean_category
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`, UNNEST(items) AS item
  WHERE event_name = 'purchase' AND item.item_category NOT LIKE '%/%' AND item.item_category IS NOT NULL
  GROUP BY item.item_name
),
views AS (
  SELECT item.item_name, COUNT(*) AS views
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`, UNNEST(items) AS item
  WHERE event_name = 'view_item'
  GROUP BY item.item_name
),
carts AS (
  SELECT item.item_name, COUNT(*) AS carts
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`, UNNEST(items) AS item
  WHERE event_name = 'add_to_cart'
  GROUP BY item.item_name
),
revenue AS (
  SELECT item.item_name, SUM(item.item_revenue_in_usd) AS revenue
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`, UNNEST(items) AS item
  WHERE event_name = 'purchase'
  GROUP BY item.item_name
)
SELECT
  COALESCE(cc.clean_category, 'Unmapped') AS category,
  SUM(v.views) AS views,
  SUM(COALESCE(c.carts, 0)) AS carts,
  ROUND(SUM(COALESCE(c.carts, 0)) * 100.0 / SUM(v.views), 2) AS view_to_cart_rate_pct,
  ROUND(SUM(COALESCE(r.revenue, 0)), 2) AS revenue
FROM views v
LEFT JOIN canonical_category cc USING (item_name)
LEFT JOIN carts c USING (item_name)
LEFT JOIN revenue r USING (item_name)
GROUP BY category
HAVING views >= 500
ORDER BY revenue DESC;
```

**Business Interpretation:** If this version still shows a large "Unmapped" share, the issue isn't the join key at all it means most *products that are ever viewed* are simply never purchased at the volume needed to appear in the purchase-side lookup (plausible given the confirmed ~1.65% purchasing-user rate), which is a legitimate finding, not a bug. If it resolves cleanly with mostly-mapped categories and non-zero revenue on high-view categories, that confirms `item_id` format inconsistency (not `item_category` alone) was the real root cause an even more significant data-quality finding than originally identified, since it means **`item_id` cannot be trusted as a join key anywhere in this dataset**, a caveat that must be added prominently to `02_Data_Understanding.md`'s `items` field documentation.

**Expected Output:** Either a mostly-resolved category table (confirming the item_id-format hypothesis) or a still-large "Unmapped" share (confirming it's a genuine view-without-purchase sparsity issue) both are informative, valid outcomes; run and report whichever actually occurs rather than assuming.

**Query 9.6 execution result CONFIRMED:** switching the join key from `item_id` to `item_name` resolved the mapping cleanly (Apparel $168,987, New $26,156, Bags $23,924, etc. reconciling to within ~5% of the original `item_category`-based total, an expected residual from naming variants). Only 193,921 views/50,627 carts remain genuinely "Unmapped" a legitimate finding (products browsed but never purchased in-window, consistent with the ~1.65% purchasing rate), not a join defect. **Root cause confirmed: `item_id` does not reliably join between browsing-stage and purchase-stage events in this dataset.**

**[R7] - NEW RULE: `item_id` is unreliable as a cross-event-type join key in this dataset; use `item_name` for any product-level join between browsing and purchase events.** (Trade-off: `item_name` isn't a guaranteed-unique identifier the way `item_id` nominally should be variant collisions are theoretically possible but it is empirically the reliable choice here, confirmed by this query.) This is now a permanent addition to `02_Data_Understanding.md`'s `items` field documentation and `09_Analytics_Decision_Log.md`.

---

## SECTION 11 - Executive & Cross-Validation Queries

### Query 11.1 - Executive Scorecard (Single Rollup Query)

**Purpose:** Combines the top-line metric from each section into one query the literal backing data for the `04_Metrics_Framework.md` §11 Executive Scorecard table and the Stage 8 Executive Summary's headline numbers.

```sql
WITH sessions AS (
  SELECT
    CONCAT(user_pseudo_id, '-', CAST((SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS STRING)) AS session_key,
    MAX((SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'session_engaged')) AS session_engaged
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  GROUP BY session_key
),
cleaned_purchases AS (
  SELECT DISTINCT user_pseudo_id, ecommerce.transaction_id AS transaction_id, ecommerce.purchase_revenue_in_usd AS revenue
  FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE event_name = 'purchase' AND ecommerce.transaction_id IS NOT NULL AND ecommerce.transaction_id != '(not set)'
)
SELECT
  (SELECT COUNT(*) FROM sessions) AS total_sessions,
  (SELECT ROUND(COUNTIF(session_engaged = '1') * 100.0 / COUNT(*), 2) FROM sessions) AS engagement_rate_pct,
  (SELECT COUNT(DISTINCT user_pseudo_id) FROM cleaned_purchases) AS total_transacting_users,
  (SELECT COUNT(DISTINCT transaction_id) FROM cleaned_purchases) AS total_valid_transactions,
  (SELECT ROUND(SUM(revenue), 2) FROM cleaned_purchases) AS total_revenue_cleaned,
  (SELECT ROUND(AVG(revenue), 2) FROM cleaned_purchases) AS aov;
```

**Business Interpretation:** The single source query behind the Executive Dashboard's top-row scorecard every number here should already reconcile with a specific earlier query in this repository (traceable, auditable, not a fresh untested calculation).

**Expected Output:** One row, six columns should match: `total_sessions` ≈ Query 2.1 sum, `total_transacting_users` ≈ sum of Query 1.1, `total_revenue_cleaned`/`aov` ≈ Query 5.2 exactly.

### Query 11.2 - Cross-Validation: Revenue Reconciliation Across All Three Sources

**Purpose:** A single query that puts all three revenue figures discovered across this repository side by side transaction-level (source of truth), item-level (relative only), and `user_ltv` (segmentation only) making the [R3] distinctions visually explicit rather than scattered across many queries.

```sql
SELECT 'Transaction-level (source of truth)' AS revenue_source,
  (SELECT ROUND(SUM(ecommerce.purchase_revenue_in_usd),2) FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*` WHERE event_name='purchase') AS total
UNION ALL
SELECT 'Item-level (relative/category use only)',
  (SELECT ROUND(SUM(item.item_revenue_in_usd),2) FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`, UNNEST(items) AS item WHERE event_name='purchase')
UNION ALL
SELECT 'user_ltv.revenue (segmentation use only)',
  (SELECT ROUND(SUM(max_ltv),2) FROM (
    SELECT MAX(SAFE_CAST(user_ltv.revenue AS FLOAT64)) AS max_ltv
    FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*` GROUP BY user_pseudo_id
  ));
```

**Business Interpretation:** This query alone justifies the entire [R3] rule to a skeptical stakeholder three legitimate, correctly-computed figures that intentionally do not match, each serving a different purpose. Include this exact table in `09_Analytics_Decision_Log.md` as the evidence for that rule.

**Expected Output:** Three rows: ~$362,165 (transaction-level), a lower figure (~$308-320K range, item-level, per the confirmed 28.56% gap), and ~$407,626 (user_ltv, the confirmed overstatement).

### Query 11.3 - Data Quality Flags Summary (Living Reference)

**Purpose:** A single reference query listing every methodology rule established across Stages 3-5, for quick onboarding of anyone new to this project (a Product Manager, a new analyst) without needing to read all prior documents first.

```sql
-- Not a data query — a documentation anchor. Included here as the canonical, queryable summary.
SELECT * FROM UNNEST([
  STRUCT('R1' AS rule_id, 'debug_mode traffic included, not filtered' AS rule, '85.74% event-level / 99.7%+ session-level debug saturation; exclusion would gut the dataset' AS rationale),
  ('R2', 'transaction_id cleaned: exclude (not set), dedupe by (transaction_id, user_pseudo_id)', '(not set) collisions span 767+ distinct users; some real IDs also collide across users'),
  ('R3', 'transaction-level revenue is sole source of truth; item-level and user_ltv never reconciled to it', 'item-level gap 28.56% of transactions; user_ltv overstates revenue by 12.6%'),
  ('R4', 'view_item/begin_checkout product-level cuts scoped to items-populated rows only', 'view_item only 62.65% items-populated; begin_checkout 82.54%'),
  ('R5', 'engagement_time_msec capped at 3,600,000ms before averaging', 'confirmed 10-19+ hour single-event outliers from background-tab sessions'),
  ('R6', 'engagement-time-based metrics not comparable pre/post Dec 28 2020', 'confirmed tagging change: engagement_time_msec presence jumped 0% to 43% exactly on that date')
]);
```

**Business Interpretation:** This is a documentation device, not an analytical query but it's genuinely useful as a single "why does this project do X" reference, and belongs verbatim in `09_Analytics_Decision_Log.md`.

**Expected Output:** Six rows, one per established rule this is the complete rule set carried forward into every remaining stage of the project.

---

**Batch 6 (final) complete : 5 queries across Section 9 correction and Section 11 (Executive/Cross-Validation).**

## Repository Summary

**53 queries total across Sections 1-11**, each implementing a specific metric from `04_Metrics_Framework.md`, each tested against real BigQuery output rather than assumed, and each carrying forward the R1-R6 rules established through actual evidence rather than upfront assumption. This is intentionally short of the original 75-100 target the decision was to prioritize **query depth and cross-validation** (follow-up queries prompted by real findings, diagnostic queries resolving anomalies, correction queries fixing discovered data-quality issues) over reaching a round number with repetitive per-dimension cuts. Every one of the 6 methodology rules (R1-R6) and every major finding in this document is now traceable to a specific query and its actual output the standard a real analytics team would hold a production metrics layer to.

**If a higher query count is wanted for the portfolio's numeric claim, straightforward additions would be:** per-geography (country-level) cuts of Sections 5/6/10 metrics (~8-10 queries), per-week trend versions of the Section 9 product metrics (~5-6 queries), and day-of-week/hour-of-day seasonality cuts of Section 1/2 metrics (~5-6 queries) happy to generate any of these on request.

**Next: Stage 6 : Analysis** (Funnel, User Journey, Revenue, Retention, Cohort, Device, Geographic, Marketing, Product Performance), which synthesizes all 53 queries' actual results into the narrative analysis document.
