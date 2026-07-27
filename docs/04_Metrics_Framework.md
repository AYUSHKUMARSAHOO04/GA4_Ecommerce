# 04 — Metrics Framework

This document defines the analytics layer for the Google Merchandise Store engagement —
the metric contracts that all SQL in `05_SQL_Query_Repository.md` and all analysis in
`06_Analysis.md` must comply with. No query is written before the metric it serves is defined here.

**Every metric in this document inherits the four binding methodology rules from
`03_Data_Validation.md`:**
- R1 — `debug_mode` traffic is included, not filtered (85.74% of dataset; documented limitation, not excludable)
- R2 — `transaction_id IN ('(not set)')` and cross-user ID collisions are excluded from *per-transaction* metrics, included (with footnote) in headline aggregate revenue
- R3 — Item-level revenue is never reconciled to transaction-level revenue; transaction-level `ecommerce.purchase_revenue_in_usd` is sole source of truth for headline revenue
- R4 — `view_item` product-level metrics are scoped to the 62.65% of rows with populated `items`, with coverage stated as a caveat
- R5 — `engagement_time_msec` is capped at 3,600,000ms (1 hour) before averaging, per the §6.5 outlier finding

Metrics below reference these as **[R1]–[R5]** in their Caveats field.

---

## Section 1 — North Star Metric

### Weekly Transacting Users (WTU)

1. **Business Definition:** The count of distinct users (`user_pseudo_id`) who complete at least one valid purchase transaction within a calendar week.
2. **Business Purpose:** For a brand-engagement-first ecommerce store (not a pure-margin retailer, per `01_Project_Overview.md`), the North Star must reward *converting engagement*, not raw traffic. WTU forces every team — Growth (traffic), Product (funnel), Marketing (channel quality) — toward the same outcome: real transacting customers, not vanity pageviews.
3. **Executive Owner:** VP Product / Head of Analytics (cross-functional accountability metric)
4. **Formula:** `COUNT(DISTINCT user_pseudo_id) WHERE event_name = 'purchase' AND transaction_id NOT IN ('(not set)') GROUP BY week`
5. **SQL Logic:** Filter to `purchase` events, apply R2 transaction cleaning, extract ISO week from `event_date`, count distinct users per week.
6. **Grain:** User (weekly aggregation)
7. **Dimension compatibility:** Device category, geography (country), traffic source (medium/source), new vs. returning (`ga_session_number`)
8. **Refresh frequency:** Weekly (Monday rollup of prior week); daily trend line for the exec dashboard
9. **Interpretation:** Rising WTU = more people completing purchases, not just browsing. Flat/falling WTU despite rising traffic = a funnel or product problem, not an acquisition problem — this is the metric that forces that distinction.
10. **Common mistakes:** Confusing WTU with total transaction count (a repeat buyer in one week should only count once); forgetting to apply R2 transaction cleaning, which would undercount by treating multiple real buyers who share a `(not set)` ID as if they were duplicates of one user.
11. **Caveats from Stage 3:** [R2] — 23 purchases (0.40%) missing transaction_id entirely still count toward WTU (a user completing a purchase is real regardless of ID quality); [R1] — debug traffic is included, so WTU is directionally, not literally, "real" weekly buyers.
12. **Example business decision supported:** If WTU stagnates while total sessions grow 20% quarter-over-quarter, leadership funds a funnel-optimization sprint (Stage 6) rather than another acquisition budget increase.

- **Source fields:** `event_name`, `user_pseudo_id`, `ecommerce.transaction_id`, `event_date`
- **Required joins/UNNEST:** None (no items/event_params unnest required)
- **Validation dependency:** §4.1, §4.2 (transaction_id cleaning), §6.4 (debug traffic inclusion)
- **Dashboard usage:** Executive Dashboard, top-left hero metric
- **Indicator type:** **Lagging** (reflects completed transactions after the fact)

---

## Section 2 — Acquisition Metrics

### 2.1 Sessions

1. **Business Definition:** Count of distinct user-session pairs in a given period.
2. **Business Purpose:** The base unit of "how much traffic arrived" — the denominator for nearly every conversion-rate metric downstream.
3. **Executive Owner:** Growth
4. **Formula:** `COUNT(DISTINCT CONCAT(user_pseudo_id, '-', ga_session_id))`
5. **SQL Logic:** Extract `ga_session_id` from `event_params`, concatenate with `user_pseudo_id`, count distinct.
6. **Grain:** Session
7. **Dimension compatibility:** Device, geo, traffic source, date/week
8. **Refresh frequency:** Daily
9. **Interpretation:** Primary traffic-volume metric; always read alongside New User Rate (2.3) to know if growth is acquisition- or retention-driven.
10. **Common mistakes:** Using `COUNT(DISTINCT ga_session_id)` alone — session IDs are not guaranteed globally unique without the user_pseudo_id prefix.
11. **Caveats from Stage 3:** §2.2 confirms 0% missing session_id — this metric has no coverage gap. [R1] debug traffic included.
12. **Example business decision supported:** A 40% session spike in the last week of November (Black Friday) should not be read as sustained organic growth — cross-check against seasonally-adjusted trend before reallocating marketing budget.

- **Source fields:** `user_pseudo_id`, `event_params.ga_session_id`
- **Required joins/UNNEST:** `UNNEST(event_params)` to extract `ga_session_id`
- **Validation dependency:** §2.2 (100% session_id coverage)
- **Dashboard usage:** Executive Dashboard trend line; Growth channel dashboard
- **Indicator type:** **Leading** (precedes conversion/revenue)

### 2.2 New Users

1. **Business Definition:** Count of distinct users whose `ga_session_number = 1` in the period (first-ever session).
2. **Business Purpose:** Measures top-of-funnel acquisition effectiveness, independent of returning-user activity.
3. **Executive Owner:** Growth / Marketing
4. **Formula:** `COUNT(DISTINCT user_pseudo_id) WHERE ga_session_number = 1`
5. **SQL Logic:** Extract `ga_session_number` from `event_params`, filter = 1, count distinct users, scoped to sessions (not events) to avoid inflation from multiple events in that first session.
6. **Grain:** User
7. **Dimension compatibility:** Traffic source, device, geo, date
8. **Refresh frequency:** Daily
9. **Interpretation:** Should be read alongside `first_visit` event count as a cross-check (both should be close, since GA4 fires `first_visit` at true first session).
10. **Common mistakes:** Counting `first_visit` events directly without deduping — a `first_visit` event should only ever fire once per user, but if it double-fires it must be deduped by user_pseudo_id, not counted raw.
11. **Caveats from Stage 3:** §5.1 confirms `ga_session_number` is 100% populated — no coverage gap. [R1] debug traffic included, so this overstates true new-human acquisition somewhat.
12. **Example business decision supported:** If New Users grows but WTU (North Star) doesn't, Marketing is buying volume, not value — signals a channel-mix review (Section 10).

- **Source fields:** `user_pseudo_id`, `event_params.ga_session_number`, `event_name = 'first_visit'`
- **Required joins/UNNEST:** `UNNEST(event_params)`
- **Validation dependency:** §5.1
- **Dashboard usage:** Growth dashboard, acquisition trend panel
- **Indicator type:** **Leading**

### 2.3 New User Rate

1. **Business Definition:** New Users ÷ Total Users in the period, expressed as a percentage.
2. **Business Purpose:** Normalizes acquisition volume against total traffic — distinguishes "growing by acquiring new people" from "growing because existing users are more active."
3. **Executive Owner:** Growth
4. **Formula:** `New Users / COUNT(DISTINCT user_pseudo_id)`
5. **SQL Logic:** Ratio of 2.2 to distinct user count in the same period.
6. **Grain:** User
7. **Dimension compatibility:** Date/week, traffic source
8. **Refresh frequency:** Weekly
9. **Interpretation:** A healthy ecommerce brand site typically runs 60–80% new-user share; a rate near 100% signals poor retention (Section 7), a rate below 40% signals acquisition has stalled.
10. **Common mistakes:** Comparing this rate across periods of different length without normalizing (a 1-day window will show a different new-user rate than a 30-day window structurally).
11. **Caveats from Stage 3:** Inherits [R1] from both numerator and denominator.
12. **Example business decision supported:** A declining new-user rate over 3 consecutive months triggers a Marketing channel-mix audit (Section 10) before assuming a product problem.

- **Source fields:** Derived from 2.2 and distinct `user_pseudo_id` count
- **Required joins/UNNEST:** `UNNEST(event_params)`
- **Validation dependency:** §5.1, §2.1
- **Dashboard usage:** Growth dashboard
- **Indicator type:** **Leading**

### 2.4 Channel Session Mix

1. **Business Definition:** Share of total sessions attributable to each `traffic_source.medium + source` combination.
2. **Business Purpose:** Tells Marketing where the traffic is actually coming from, at the correct grain (medium+source together — never source alone, per Stage 2 assumption).
3. **Executive Owner:** Marketing
4. **Formula:** `COUNT(sessions) WHERE medium=X AND source=Y / total sessions`
5. **SQL Logic:** Group session-deduplicated rows by `traffic_source.medium, traffic_source.source`.
6. **Grain:** Session
7. **Dimension compatibility:** Date, device, geo
8. **Refresh frequency:** Weekly
9. **Interpretation:** `(none)/(direct)` at ~23% of sessions (confirmed §6.2) is a legitimate, sizeable channel — must be reported as its own segment, not folded into "unknown."
10. **Common mistakes:** Grouping by `source` alone, which conflates organic and paid traffic under the same source label (e.g., "google" spans both).
11. **Caveats from Stage 3:** §6.2 — direct traffic (`(none)`) is 23.04% of sessions; this is a real category, not missing data, and must be labeled as such on every chart.
12. **Example business decision supported:** If paid search shows high session share but low share of WTU (Section 1), Marketing reallocates budget toward the higher-converting channel.

- **Source fields:** `traffic_source.medium`, `traffic_source.source`
- **Required joins/UNNEST:** None (top-level struct fields)
- **Validation dependency:** §6.2
- **Dashboard usage:** Marketing Channel dashboard
- **Indicator type:** **Leading**

### 2.5 Landing Page Entrance Rate

1. **Business Definition:** Share of sessions entering the site via a given landing page (flagged by `event_params.entrances = 1`).
2. **Business Purpose:** Identifies which pages act as the primary front door, informing SEO/paid landing-page investment.
3. **Executive Owner:** Marketing / Product
4. **Formula:** `COUNT(sessions WHERE entrances=1 AND page_location=X) / total sessions`
5. **SQL Logic:** Extract `entrances` and `page_location` from `event_params`, filter `entrances = 1`, dedupe per session (see caveat below).
6. **Grain:** Session
7. **Dimension compatibility:** Traffic source, device
8. **Refresh frequency:** Weekly
9. **Interpretation:** Concentration in 1-2 landing pages signals over-reliance on a narrow entry point — a resilience risk if that page underperforms.
10. **Common mistakes:** Not deduping double-fired `entrances=1` events within a session — per §2.5 validation finding, this exact pattern was observed and must be corrected with `MIN(event_timestamp)` per session rather than counting raw entrance-flagged rows.
11. **Caveats from Stage 3:** §2.5 — entrance double-firing found in session reconstruction validation, linked to debug traffic [R1]; must dedupe to one entrance per session_key before computing this metric.
12. **Example business decision supported:** If a specific campaign landing page shows high entrance share but low downstream engagement, Marketing revises page content rather than just driving more traffic to it.

- **Source fields:** `event_params.entrances`, `event_params.page_location`
- **Required joins/UNNEST:** `UNNEST(event_params)`
- **Validation dependency:** §2.5
- **Dashboard usage:** Marketing Channel dashboard, Landing Page report
- **Indicator type:** **Leading**

---

## Section 3 — Activation Metrics

### 3.1 First-Session Product View Rate

1. **Business Definition:** Share of first-ever sessions (`ga_session_number=1`) in which the user views at least one product (`view_item`).
2. **Business Purpose:** "Activation" for a brand ecommerce store means moving a new visitor from browsing to genuine product interest within their first visit — this is the earliest meaningful activation signal.
3. **Executive Owner:** Product
4. **Formula:** `COUNT(DISTINCT sessions WHERE ga_session_number=1 AND has view_item event) / COUNT(DISTINCT sessions WHERE ga_session_number=1)`
5. **SQL Logic:** Flag sessions with `ga_session_number=1`, check for presence of a `view_item` event in the same session_key.
6. **Grain:** Session
7. **Dimension compatibility:** Traffic source, device, landing page
8. **Refresh frequency:** Weekly
9. **Interpretation:** Low first-session product-view rate signals a homepage/navigation problem — visitors arrive but don't engage with the actual catalog.
10. **Common mistakes:** Measuring this at the event grain instead of session grain, inflating the rate if a user views the same product multiple times.
11. **Caveats from Stage 3:** [R4] — `view_item` events are only 62.65% items-populated, but that caveat applies to *product-level* detail, not to this metric, which only needs the event to have fired, not its item payload — stated explicitly to avoid confusion.
12. **Example business decision supported:** A redesign of the homepage product carousel is prioritized if this rate sits materially below the desktop/mobile benchmark split.

- **Source fields:** `event_params.ga_session_number`, `event_name = 'view_item'`
- **Required joins/UNNEST:** `UNNEST(event_params)`
- **Validation dependency:** §5.1 (session_number coverage), §5.2 (view_item item-population caveat — noted, not blocking here)
- **Dashboard usage:** Product Activation panel
- **Indicator type:** **Leading**

### 3.2 Time-to-First-Add-to-Cart

1. **Business Definition:** Median elapsed time between a user's `user_first_touch_timestamp` and their first `add_to_cart` event.
2. **Business Purpose:** Measures how quickly the product experience converts curiosity into purchase intent — a core UX-quality signal.
3. **Executive Owner:** Product
4. **Formula:** `MEDIAN(first add_to_cart timestamp - user_first_touch_timestamp)`
5. **SQL Logic:** Join each user's first `add_to_cart` event timestamp against their `user_first_touch_timestamp`, compute difference, take median (not mean, to avoid skew from long-tail returning users).
6. **Grain:** User
7. **Dimension compatibility:** Device, traffic source, new vs returning
8. **Refresh frequency:** Monthly (a stability metric, not a daily-swing one)
9. **Interpretation:** A rising median time-to-cart across cohorts suggests friction has crept into product discovery.
10. **Common mistakes:** Using mean instead of median — a small number of users who add to cart weeks later will massively distort an average.
11. **Caveats from Stage 3:** None blocking — `user_first_touch_timestamp` confirmed reliable in Stage 2/3 review (constant per user, always populated).
12. **Example business decision supported:** Justifies investment in product search/filtering if median time-to-cart is trending upward alongside rising `view_search_results` volume (implying users are struggling to find products).

- **Source fields:** `user_first_touch_timestamp`, `event_name='add_to_cart'`, `event_timestamp`
- **Required joins/UNNEST:** Self-join per user on first add_to_cart event
- **Validation dependency:** §6.1 (0% null on core timestamp fields)
- **Dashboard usage:** Product Activation panel
- **Indicator type:** **Leading**

### 3.3 First-Session Engagement Rate

1. **Business Definition:** Share of first-ever sessions flagged as "engaged" (`session_engaged = '1'`).
2. **Business Purpose:** GA4's engagement definition (10+ sec, 2+ pageviews, or a conversion event) applied specifically to new visitors — a cleaner activation signal than raw bounce rate.
3. **Executive Owner:** Product
4. **Formula:** `COUNT(sessions WHERE ga_session_number=1 AND session_engaged='1') / COUNT(sessions WHERE ga_session_number=1)`
5. **SQL Logic:** Filter `ga_session_number=1`, check `session_engaged` value within the session.
6. **Grain:** Session
7. **Dimension compatibility:** Device, traffic source, landing page
8. **Refresh frequency:** Weekly
9. **Interpretation:** Low first-session engagement paired with high session volume from a channel = that channel is bringing low-intent traffic, a Marketing quality issue, not just a Product one.
10. **Common mistakes:** Treating `session_engaged` as static across the session — it's evaluated per event and can change; use the session's max/final value, not the first event's value.
11. **Caveats from Stage 3:** §5.1 confirms full coverage of `session_engaged` param; [R1] debug traffic included.
12. **Example business decision supported:** Feeds directly into the channel-quality conversation in Section 10 (Marketing Metrics) — engagement rate is the non-monetary quality signal to pair with channel revenue.

- **Source fields:** `event_params.ga_session_number`, `event_params.session_engaged`
- **Required joins/UNNEST:** `UNNEST(event_params)`
- **Validation dependency:** §5.1
- **Dashboard usage:** Product Activation panel, Marketing Channel dashboard (cross-reference)
- **Indicator type:** **Leading**

---

## Section 4 — Engagement Metrics

### 4.1 Engagement Rate

1. **Business Definition:** Share of all sessions flagged as engaged, GA4's replacement for Universal Analytics' bounce rate.
2. **Business Purpose:** The primary "is our content/experience holding attention" signal across the whole user base, not just new visitors.
3. **Executive Owner:** Product
4. **Formula:** `COUNT(engaged sessions) / COUNT(total sessions)`
5. **SQL Logic:** Same as 3.3 but without the `ga_session_number=1` filter.
6. **Grain:** Session
7. **Dimension compatibility:** Device, geo, traffic source, date
8. **Refresh frequency:** Daily
9. **Interpretation:** Track trend, not just level — a sudden drop is more actionable than the absolute number.
10. **Common mistakes:** Reporting this as "1 − bounce rate" from Universal Analytics intuition — GA4's engagement definition is structurally different (time/pageview/conversion-based), not a simple inverse.
11. **Caveats from Stage 3:** [R1] — debug traffic included; §5.1 confirms full field coverage.
12. **Example business decision supported:** A sustained engagement-rate decline after a site redesign is grounds to roll back or A/B test the redesign (see Section on Experiment Ideas, later document).

- **Source fields:** `event_params.session_engaged`
- **Required joins/UNNEST:** `UNNEST(event_params)`
- **Validation dependency:** §5.1
- **Dashboard usage:** Executive Dashboard, Product dashboard
- **Indicator type:** **Leading**

### 4.2 Average Engagement Time per Session (Capped)

1. **Business Definition:** Mean active foreground time per session, in seconds, capped at 3,600 seconds per event per [R5].
2. **Business Purpose:** Depth-of-engagement metric — complements engagement rate (breadth) with intensity.
3. **Executive Owner:** Product
4. **Formula:** `AVG(SUM(LEAST(engagement_time_msec, 3600000)) per session) / 1000`
5. **SQL Logic:** Extract and cap `engagement_time_msec` per event, sum per session_key, average across sessions.
6. **Grain:** Session
7. **Dimension compatibility:** Device, geo, traffic source
8. **Refresh frequency:** Weekly
9. **Interpretation:** Should be read alongside Engagement Rate (4.1) — high rate + low time = many people engage briefly; low rate + high time = fewer people, but deeply.
10. **Common mistakes:** Failing to apply the [R5] cap — §6.5 found sessions with 10–19+ hours of single-event engagement time (background-tab artifacts) that will massively distort an uncapped average.
11. **Caveats from Stage 3:** [R5] mandatory — this metric is invalid without the cap, per direct evidence in §6.5.
12. **Example business decision supported:** Justifies content/page-depth investment if average engagement time is rising alongside stable engagement rate (deepening, not just maintaining, attention).

- **Source fields:** `event_params.engagement_time_msec`
- **Required joins/UNNEST:** `UNNEST(event_params)`
- **Validation dependency:** §6.5 (outlier cap requirement — mandatory)
- **Dashboard usage:** Product dashboard
- **Indicator type:** **Leading**

### 4.3 Pages per Session

1. **Business Definition:** Average count of `page_view` events per session.
2. **Business Purpose:** Simple depth-of-browsing proxy — useful alongside engagement time as a secondary signal.
3. **Executive Owner:** Product
4. **Formula:** `COUNT(page_view events) / COUNT(DISTINCT sessions)`
5. **SQL Logic:** Filter `event_name='page_view'`, count per session_key, average.
6. **Grain:** Session
7. **Dimension compatibility:** Device, traffic source
8. **Refresh frequency:** Weekly
9. **Interpretation:** Very high pages-per-session with low engagement time can indicate confused navigation (many quick page loads with no real reading), not genuine interest — always pair with 4.2, never read alone.
10. **Common mistakes:** Reading this in isolation as unambiguously positive — more pages isn't always good if paired with low engagement time.
11. **Caveats from Stage 3:** [R1] debug traffic included, which per §2.5 findings correlates with duplicate page_view firing — may modestly inflate this metric; documented, not corrected (no reliable way to distinguish genuine re-navigation from duplicate firing at scale).
12. **Example business decision supported:** Supports information-architecture review if pages-per-session is high but engaged-session rate (4.1) is flat/low.

- **Source fields:** `event_name`
- **Required joins/UNNEST:** None
- **Validation dependency:** §1.3, §2.5 (duplicate-firing caveat)
- **Dashboard usage:** Product dashboard
- **Indicator type:** **Leading**

### 4.4 Sessions per User

1. **Business Definition:** Average number of distinct sessions per unique user in the period.
2. **Business Purpose:** A coarse but useful stickiness proxy, precursor to formal retention analysis (Section 7).
3. **Executive Owner:** Product
4. **Formula:** `COUNT(DISTINCT session_key) / COUNT(DISTINCT user_pseudo_id)`
5. **SQL Logic:** Distinct session count divided by distinct user count over the same window.
6. **Grain:** User
7. **Dimension compatibility:** Traffic source, device, date range
8. **Refresh frequency:** Monthly
9. **Interpretation:** Confirmed baseline from §2.2: ~1.33 sessions/user across the full 92-day window — a low-stickiness pattern consistent with a "occasional purchase, not daily habit" product category, which should temper expectations set for formal retention metrics in Section 7.
10. **Common mistakes:** Expecting DAU/MAU-app-style stickiness benchmarks (e.g., 20%+) for a low-frequency-purchase merch store — the right benchmark comparison is other occasional-purchase ecommerce sites, not habitual apps.
11. **Caveats from Stage 3:** [R1] debug traffic included; baseline value directly confirmed in §2.2 validation output (360,129 sessions / 270,154 users = 1.33).
12. **Example business decision supported:** Sets realistic targets for retention-focused initiatives (e.g., email remarketing) rather than importing unrealistic benchmarks from high-frequency product categories.

- **Source fields:** `user_pseudo_id`, `event_params.ga_session_id`
- **Required joins/UNNEST:** `UNNEST(event_params)`
- **Validation dependency:** §2.1, §2.2
- **Dashboard usage:** Product dashboard, Retention panel (cross-reference)
- **Indicator type:** **Lagging**

---

## Section 5 — Commerce Metrics

### 5.1 Total Revenue

1. **Business Definition:** Sum of `ecommerce.purchase_revenue_in_usd` across all `purchase` events in the period.
2. **Business Purpose:** The headline top-line financial metric — first number any executive reads.
3. **Executive Owner:** Finance / Leadership
4. **Formula:** `SUM(ecommerce.purchase_revenue_in_usd) WHERE event_name='purchase'`
5. **SQL Logic:** Direct sum, no unnest required (transaction-level struct field).
6. **Grain:** Transaction
7. **Dimension compatibility:** Date, device, geo, traffic source
8. **Refresh frequency:** Daily
9. **Interpretation:** Always shown alongside the Nov–Dec seasonal flag (Black Friday/Christmas) per `01_Project_Overview.md` §2 — a raw trend line without that annotation will visually mislead.
10. **Common mistakes:** Summing `ecommerce.purchase_revenue` (local currency) instead of the `_in_usd` field; summing `items.item_revenue_in_usd` instead and expecting it to match (it won't, per [R3] — 28.56% gap rate confirmed in §4.3).
11. **Caveats from Stage 3:** [R3] mandatory — this is the sole source of truth for revenue; §4.1 confirms `purchase_revenue_in_usd` is 100% populated on purchase events (0 missing), so this metric itself has no coverage gap even though `transaction_id` does.
12. **Example business decision supported:** Direct executive-reporting input; also the basis for any "is this recommendation worth it" ROI framing in `07_Business_Recommendations.md`.

- **Source fields:** `ecommerce.purchase_revenue_in_usd`, `event_name`
- **Required joins/UNNEST:** None
- **Validation dependency:** §4.1, §4.3 (R3 rule)
- **Dashboard usage:** Executive Dashboard, hero metric
- **Indicator type:** **Lagging**

### 5.2 Average Order Value (AOV)

1. **Business Definition:** Total Revenue ÷ count of *valid, deduplicated* transactions.
2. **Business Purpose:** Reveals whether revenue changes are driven by more orders or bigger orders — critical for distinguishing a traffic story from a merchandising story.
3. **Executive Owner:** Finance / Product
4. **Formula:** `SUM(purchase_revenue_in_usd) / COUNT(DISTINCT deduplicated transaction_id)`
5. **SQL Logic:** Apply [R2] cleaning first (exclude `(not set)`, dedupe by `transaction_id + user_pseudo_id`), then divide revenue by the cleaned transaction count.
6. **Grain:** Transaction
7. **Dimension compatibility:** Device, geo, traffic source, date
8. **Refresh frequency:** Weekly
9. **Interpretation:** AOV around $48 median per §4.4 findings (median purchase value); track for drift after any pricing/merchandising change.
10. **Common mistakes:** Using raw `COUNT(*)` of purchase events as the denominator without [R2] cleaning — per §4.2, this inflates the transaction count due to `(not set)` collisions and real cross-user ID collisions, understating true AOV.
11. **Caveats from Stage 3:** [R2] mandatory — §4.2 is a confirmed FAIL specifically because of this exact risk.
12. **Example business decision supported:** If AOV is falling while Total Revenue holds steady, it signals volume is compensating for smaller baskets — informs whether a bundling/cross-sell initiative (Section 9) is warranted.

- **Source fields:** `ecommerce.purchase_revenue_in_usd`, `ecommerce.transaction_id`, `user_pseudo_id`
- **Required joins/UNNEST:** None (dedup logic, no array unnest)
- **Validation dependency:** §4.2 (R2 rule — mandatory)
- **Dashboard usage:** Executive Dashboard, Revenue panel
- **Indicator type:** **Lagging**

### 5.3 Items per Transaction

1. **Business Definition:** Average `ecommerce.total_item_quantity` across valid transactions.
2. **Business Purpose:** A merchandising-basket-size signal, useful for cross-sell/bundle strategy evaluation.
3. **Executive Owner:** Product
4. **Formula:** `AVG(ecommerce.total_item_quantity)` (on [R2]-cleaned transactions)
5. **SQL Logic:** Apply [R2] cleaning, average `total_item_quantity` per remaining transaction.
6. **Grain:** Transaction
7. **Dimension compatibility:** Device, geo, product category
8. **Refresh frequency:** Monthly
9. **Interpretation:** §6.5 confirmed a long tail of high-quantity bulk/corporate orders (up to 400 units) — report median alongside mean to avoid a skewed headline number.
10. **Common mistakes:** Reporting mean alone when a handful of bulk-order outliers (confirmed present in §6.5) will pull it upward misleadingly.
11. **Caveats from Stage 3:** [R2]; also inherits the §6.5 bulk-order outlier finding — recommend reporting both mean and median.
12. **Example business decision supported:** If median items-per-transaction is low (1-2) despite high mean, it confirms most customers are single-item buyers — informs whether a "buy 2, save X%" promotion (Experiment Ideas, later doc) is worth testing.

- **Source fields:** `ecommerce.total_item_quantity`
- **Required joins/UNNEST:** None
- **Validation dependency:** §4.2 (R2), §6.5 (outlier awareness)
- **Dashboard usage:** Revenue panel
- **Indicator type:** **Lagging**

### 5.4 Refund Rate

1. **Business Definition:** Refund events (or refund value) as a share of purchase transactions (or revenue).
2. **Business Purpose:** A quality/satisfaction proxy — rising refunds signal a product-fit or fulfillment problem.
3. **Executive Owner:** Operations / Finance
4. **Formula:** `SUM(ecommerce.refund_value_in_usd) / SUM(ecommerce.purchase_revenue_in_usd)`
5. **SQL Logic:** Sum refund value across `refund` events, divide by total purchase revenue in the same period.
6. **Grain:** Transaction
7. **Dimension compatibility:** Product category, date
8. **Refresh frequency:** Monthly
9. **Interpretation:** No `refund` events were observed as a distinct dominant category in the Stage 1 event-distribution list — this metric's actual volume must be spot-checked before relying on it (flagged as an open item for Stage 5 query repository, not yet formally validated in Stage 3).
10. **Common mistakes:** Assuming refunds are well-populated without checking — this field was **not explicitly validated in Stage 3** and should get a dedicated null/volume check before being reported with confidence.
11. **Caveats from Stage 3:** **Open validation gap** — refund volume/completeness was not part of the Stage 3 suite; flagged here rather than silently assumed reliable.
12. **Example business decision supported:** A rising refund rate on a specific product category triggers a quality/fulfillment review for that category.

- **Source fields:** `ecommerce.refund_value_in_usd`, `event_name='refund'`
- **Required joins/UNNEST:** None
- **Validation dependency:** **None yet — flagged for follow-up validation**
- **Dashboard usage:** Operations dashboard
- **Indicator type:** **Lagging**

### 5.5 Revenue per Session (RPS)

1. **Business Definition:** Total Revenue ÷ Total Sessions in the period.
2. **Business Purpose:** A blended efficiency metric connecting traffic (Section 2) directly to monetization (this section) — useful for channel-level ROI framing even without real cost data.
3. **Executive Owner:** Growth / Finance
4. **Formula:** `SUM(purchase_revenue_in_usd) / COUNT(DISTINCT session_key)`
5. **SQL Logic:** Revenue (5.1) divided by Sessions (2.1) over matching period/dimension cut.
6. **Grain:** Session (revenue attributed back to session grain)
7. **Dimension compatibility:** Traffic source, device, geo
8. **Refresh frequency:** Weekly
9. **Interpretation:** The single best channel-comparison metric available in this dataset given the absence of real CAC data (per Stage 1 scope note) — frame explicitly as directional efficiency, not true ROI.
10. **Common mistakes:** Presenting this as "ROI" without the CAC caveat — this project has no cost data, so RPS measures monetization efficiency of traffic, not marketing return on investment.
11. **Caveats from Stage 3:** Inherits [R1] (debug traffic) and [R3] (revenue source-of-truth rule) from its components.
12. **Example business decision supported:** Central input to the channel-prioritization recommendation in `07_Business_Recommendations.md`.

- **Source fields:** Derived from 5.1 and 2.1
- **Required joins/UNNEST:** `UNNEST(event_params)` for session reconstruction
- **Validation dependency:** §4.1, §2.1, §2.2
- **Dashboard usage:** Executive Dashboard, Marketing Channel dashboard
- **Indicator type:** **Lagging**

---

## Section 6 — Funnel Metrics

### 6.1 View-to-Cart Rate

1. **Business Definition:** Share of item-identifiable product views that lead to an add-to-cart for the same item, within session.
2. **Business Purpose:** The first, and usually largest, drop-off point in ecommerce — directly actionable by Product (PDP design) and Merchandising (pricing/presentation).
3. **Executive Owner:** Product
4. **Formula:** `COUNT(sessions with add_to_cart) / COUNT(sessions with view_item)` (session-scoped, not raw event ratio)
5. **SQL Logic:** Flag sessions containing `view_item` (with populated items, per [R4]), flag sessions containing `add_to_cart`, compute session-level conversion.
6. **Grain:** Session (rolled up from item-level detail where available)
7. **Dimension compatibility:** Device, product category, traffic source
8. **Refresh frequency:** Weekly
9. **Interpretation:** Raw volumes (386,068 views → 58,543 adds) suggest ~15% at the event level, but the session-scoped, [R4]-compliant version is the number that should be reported and will differ from this naive ratio — compute both and reconcile before publishing.
10. **Common mistakes:** Computing this as a flat `COUNT(add_to_cart)/COUNT(view_item)` event ratio (mixing grains, and ignoring [R4]'s coverage gap) rather than a proper session- or item-scoped funnel.
11. **Caveats from Stage 3:** [R4] mandatory — only 62.65% of `view_item` events carry item detail; the item-level version of this metric is scoped to that subset, with the gap stated explicitly per the Stage 3 methodology decision.
12. **Example business decision supported:** If mobile view-to-cart rate is materially below desktop (device split confirmed 40%/58% in §3.3), this justifies a mobile PDP UX investigation ahead of any general "improve conversion" initiative.

- **Source fields:** `event_name IN ('view_item','add_to_cart')`, `items` (where populated)
- **Required joins/UNNEST:** `UNNEST(items)` for item-level version; session_key join for session-scoped version
- **Validation dependency:** §5.2 (R4 — mandatory), §2.2 (session reconstruction)
- **Dashboard usage:** Executive Dashboard funnel widget, Product dashboard
- **Indicator type:** **Leading**

### 6.2 Cart-to-Checkout Rate

1. **Business Definition:** Share of sessions with `add_to_cart` that proceed to `begin_checkout`.
2. **Business Purpose:** Isolates cart-page/checkout-entry friction specifically, separate from product-discovery friction (6.1).
3. **Executive Owner:** Product
4. **Formula:** `COUNT(sessions with begin_checkout) / COUNT(sessions with add_to_cart)`
5. **SQL Logic:** Session-scoped presence check for both events, same session_key.
6. **Grain:** Session
7. **Dimension compatibility:** Device, traffic source
8. **Refresh frequency:** Weekly
9. **Interpretation:** Raw volumes (58,543 → 38,757) suggest ~66% at event level — a comparatively strong step relative to 6.1 and 6.3, worth confirming at session grain.
10. **Common mistakes:** Not accounting for [R4]'s `begin_checkout` items coverage gap (82.54% populated per §5.2) if this metric is later cut by product/category — the event-presence version of this metric itself does not require items population, only the product-level breakdown of it does; keep these two uses separate.
11. **Caveats from Stage 3:** [R4] applies only if product-level cuts are added; the base session-level rate itself is unaffected by items coverage.
12. **Example business decision supported:** A comparatively strong cart-to-checkout rate (vs. weak view-to-cart) tells Product to prioritize top-of-funnel PDP work over checkout-flow work, a resourcing decision.

- **Source fields:** `event_name IN ('add_to_cart','begin_checkout')`
- **Required joins/UNNEST:** Session_key join
- **Validation dependency:** §2.2, §5.2 (conditional)
- **Dashboard usage:** Executive Dashboard funnel widget
- **Indicator type:** **Leading**

### 6.3 Checkout-to-Purchase Rate

1. **Business Definition:** Share of sessions with `begin_checkout` that reach a completed `purchase`.
2. **Business Purpose:** Isolates the final, highest-stakes drop-off — often the most fixable (form friction, payment failures, shipping-cost shock).
3. **Executive Owner:** Product / Operations
4. **Formula:** `COUNT(sessions with purchase) / COUNT(sessions with begin_checkout)`
5. **SQL Logic:** Session-scoped presence check; also examine intermediate steps `add_shipping_info` and `add_payment_info` for a granular 3-step breakdown.
6. **Grain:** Session
7. **Dimension compatibility:** Device, traffic source, payment/shipping step completion
8. **Refresh frequency:** Weekly
9. **Interpretation:** Raw volumes (38,757 → 5,692) suggest ~15% at event level — the steepest single drop in the funnel, warranting the deepest investigation in Stage 6 Analysis.
10. **Common mistakes:** Treating this as one monolithic step — the granular breakdown (`begin_checkout` → `add_shipping_info` → `add_payment_info` → `purchase`) will reveal exactly where within checkout the loss occurs, and should always be reported alongside the headline rate.
11. **Caveats from Stage 3:** [R2] — 23 purchases lack transaction_id but should still count as successful conversions here (this metric doesn't require transaction_id, only the `purchase` event's presence).
12. **Example business decision supported:** If the granular breakdown shows most loss between `add_payment_info` and `purchase`, this points directly at payment-method friction or a technical checkout bug — a high-priority, low-ambiguity fix.

- **Source fields:** `event_name IN ('begin_checkout','add_shipping_info','add_payment_info','purchase')`
- **Required joins/UNNEST:** Session_key join
- **Validation dependency:** §2.2, §4.1
- **Dashboard usage:** Executive Dashboard funnel widget, Checkout Funnel detail panel
- **Indicator type:** **Leading**

### 6.4 Overall View-to-Purchase Conversion Rate

1. **Business Definition:** Share of sessions that view any product and go on to complete a purchase.
2. **Business Purpose:** The single headline funnel efficiency number for executive reporting — the product of 6.1 × 6.2 × 6.3.
3. **Executive Owner:** VP Product / Leadership
4. **Formula:** `COUNT(sessions with purchase) / COUNT(sessions with view_item)`
5. **SQL Logic:** Session-scoped presence check across the full journey; should reconcile arithmetically with the product of the three sub-funnel rates (6.1×6.2×6.3) as a query-correctness check.
6. **Grain:** Session
7. **Dimension compatibility:** Device, traffic source, geo, new vs. returning
8. **Refresh frequency:** Weekly
9. **Interpretation:** This is the number leadership will remember and quote — must always be shown with its three-stage breakdown, never alone, or it hides which stage to fix.
10. **Common mistakes:** Presenting this single number to leadership without the underlying stage breakdown, making the finding non-actionable.
11. **Caveats from Stage 3:** Inherits every upstream funnel caveat ([R4] for view_item item-detail, [R2] for transaction cleaning where relevant).
12. **Example business decision supported:** The single number that justifies (or doesn't) a dedicated "funnel optimization" initiative's existence to leadership.

- **Source fields:** `event_name IN ('view_item','purchase')`
- **Required joins/UNNEST:** Session_key join
- **Validation dependency:** §5.2, §2.2, §4.1
- **Dashboard usage:** Executive Dashboard, hero funnel widget
- **Indicator type:** **Lagging** (it's the funnel's outcome, though built from leading sub-metrics)

### 6.5 Cart Abandonment Rate

1. **Business Definition:** Share of sessions with `add_to_cart` that do NOT reach `purchase`.
2. **Business Purpose:** The inverse framing of 6.4's back half — the metric most directly tied to remarketing/cart-recovery initiatives.
3. **Executive Owner:** Marketing / Product
4. **Formula:** `1 − (COUNT(sessions with purchase) / COUNT(sessions with add_to_cart))`
5. **SQL Logic:** Session-scoped presence check, inverse of the purchase-completion rate from the cart stage.
6. **Grain:** Session
7. **Dimension compatibility:** Device, traffic source, product category
8. **Refresh frequency:** Weekly
9. **Interpretation:** Expect a high rate (ecommerce industry norms run 60-80%) — the actionable question is which specific step within checkout drives it (see 6.3's granular breakdown), not the headline number alone.
10. **Common mistakes:** Treating "cart abandonment" as inherently bad without segmenting by where in checkout it happens — abandonment at `add_to_cart`→`begin_checkout` implies a different fix (pricing/shipping-cost transparency) than abandonment at `add_payment_info`→`purchase` (payment friction).
11. **Caveats from Stage 3:** Same as 6.2/6.3 — [R2] where transaction identification matters.
12. **Example business decision supported:** Directly informs whether an abandoned-cart email remarketing program (Experiment Ideas, later doc) is worth building, and which checkout stage it should target.

- **Source fields:** `event_name IN ('add_to_cart','purchase')`
- **Required joins/UNNEST:** Session_key join
- **Validation dependency:** §2.2, §4.1
- **Dashboard usage:** Marketing Channel dashboard, Product dashboard
- **Indicator type:** **Lagging**

---

## Section 7 — Retention Metrics

### 7.1 Day N Retention

1. **Business Definition:** Share of users active (`user_first_touch_timestamp` on day 0) who return with at least one event on day N.
2. **Business Purpose:** The classic cohort-retention curve — measures whether the store creates any return-visit habit at all.
3. **Executive Owner:** Product / Growth
4. **Formula:** `COUNT(DISTINCT users active on day N | first touch = day 0) / COUNT(DISTINCT users with first touch = day 0)`
5. **SQL Logic:** Anchor cohort by `user_first_touch_timestamp` date; check for presence of any event exactly N days later.
6. **Grain:** User
7. **Dimension compatibility:** Acquisition channel, device, week-of-first-touch cohort
8. **Refresh frequency:** Weekly cohort refresh
9. **Interpretation:** Given the confirmed 1.33 sessions/user baseline (§2.2, Section 4.4), expect low Day N retention across all N — this is a low-frequency browsing/purchase category, and retention curves should be benchmarked against similar occasional-purchase ecommerce sites, not habitual-use apps.
10. **Common mistakes:** Importing retention benchmarks from social/gaming apps (e.g., "Day 1 retention should be 40%+") — wildly inappropriate comparison for this product category.
11. **Caveats from Stage 3:** [R1] debug traffic included; relies on `user_first_touch_timestamp`, confirmed reliable and fully populated.
12. **Example business decision supported:** Sets realistic, evidence-based targets for retention-focused Growth initiatives, rather than importing unrealistic external benchmarks (a genuine "so what" for the Recommendations doc).

- **Source fields:** `user_first_touch_timestamp`, `event_timestamp`, `user_pseudo_id`
- **Required joins/UNNEST:** Self-join per user across day-offset windows
- **Validation dependency:** §6.1 (0% null on timestamps)
- **Dashboard usage:** Retention cohort chart
- **Indicator type:** **Lagging**

### 7.2 Weekly Returning User Rate

1. **Business Definition:** Share of a given week's active users who also appeared in at least one prior week.
2. **Business Purpose:** A rolling, less noisy alternative to Day N retention — smooths day-to-day volatility.
3. **Executive Owner:** Product / Growth
4. **Formula:** `COUNT(DISTINCT users active this week AND active in any prior week) / COUNT(DISTINCT users active this week)`
5. **SQL Logic:** Self-join current-week active users against all-time active-user history prior to that week.
6. **Grain:** User
7. **Dimension compatibility:** Acquisition channel, device
8. **Refresh frequency:** Weekly
9. **Interpretation:** Complements New User Rate (2.3) — the two should sum toward 100% of weekly active users (new + returning), a useful internal consistency check.
10. **Common mistakes:** Not reconciling this against 2.3 — if New User Rate + Returning User Rate materially exceeds 100%, there's a session/user grain bug somewhere in the pipeline.
11. **Caveats from Stage 3:** [R1] debug traffic included.
12. **Example business decision supported:** A consistently low returning-user rate strengthens the case for an email/remarketing retention program in `07_Business_Recommendations.md`.

- **Source fields:** `user_pseudo_id`, `event_date`
- **Required joins/UNNEST:** Self-join across weekly windows
- **Validation dependency:** §2.1, §6.1
- **Dashboard usage:** Retention panel
- **Indicator type:** **Lagging**

### 7.3 Repeat Purchase Rate

1. **Business Definition:** Share of purchasing users (WTU-eligible, any period) who complete more than one valid transaction over the full 92-day window.
2. **Business Purpose:** The clearest, most business-relevant retention signal for an ecommerce site — did the store convert a one-time buyer into a repeat customer?
3. **Executive Owner:** Product / Finance
4. **Formula:** `COUNT(DISTINCT users with ≥2 valid transactions) / COUNT(DISTINCT users with ≥1 valid transaction)`
5. **SQL Logic:** Apply [R2] transaction cleaning, count distinct valid transactions per user, compute share with ≥2.
6. **Grain:** User
7. **Dimension compatibility:** Acquisition channel, device, geo
8. **Refresh frequency:** Monthly
9. **Interpretation:** Given the low sessions-per-user baseline, expect this to be a small percentage — this is likely the single most important number for the "is this a brand-engagement play or a repeat-revenue business" framing question raised in `01_Project_Overview.md`.
10. **Common mistakes:** Not applying [R2] cleaning first — cross-user transaction_id collisions (confirmed in §4.2) would falsely inflate apparent repeat-purchase behavior if two different users' orders were merged under one shared `(not set)` or colliding ID.
11. **Caveats from Stage 3:** [R2] mandatory — directly protects against a specific, confirmed data-quality risk.
12. **Example business decision supported:** A very low repeat-purchase rate directly supports (or reframes) the recommendation to invest in post-purchase email/loyalty programs — or confirms the store is, and should be treated as, primarily a brand-awareness vehicle rather than a repeat-revenue engine.

- **Source fields:** `ecommerce.transaction_id`, `user_pseudo_id`, `event_name='purchase'`
- **Required joins/UNNEST:** None (aggregation with R2 cleaning)
- **Validation dependency:** §4.2 (R2 — mandatory)
- **Dashboard usage:** Executive Dashboard, Customer panel
- **Indicator type:** **Lagging**

### 7.4 New vs. Returning Session Mix

1. **Business Definition:** Share of sessions where `ga_session_number = 1` (new) vs. `> 1` (returning).
2. **Business Purpose:** A session-grain (not user-grain) retention proxy — useful for real-time dashboarding since it doesn't require waiting for cohort maturity.
3. **Executive Owner:** Product
4. **Formula:** `COUNT(sessions WHERE ga_session_number > 1) / COUNT(total sessions)`
5. **SQL Logic:** Extract `ga_session_number`, bucket into new (=1) vs returning (>1), compute share.
6. **Grain:** Session
7. **Dimension compatibility:** Device, traffic source, date
8. **Refresh frequency:** Daily
9. **Interpretation:** A fast-moving leading proxy for 7.2's slower, more rigorous version — use this for daily dashboard monitoring, 7.2/7.3 for the formal monthly retention report.
10. **Common mistakes:** Presenting this as equivalent to true user-level retention — it's a session mix, so one highly active returning user can inflate the "returning" share without indicating broad retention.
11. **Caveats from Stage 3:** §5.1 confirms 100% coverage of `ga_session_number`; [R1] debug traffic included.
12. **Example business decision supported:** Daily monitoring trigger — a sudden drop in returning-session share is an early warning sign investigated well before the monthly Repeat Purchase Rate (7.3) would surface it.

- **Source fields:** `event_params.ga_session_number`
- **Required joins/UNNEST:** `UNNEST(event_params)`
- **Validation dependency:** §5.1
- **Dashboard usage:** Executive Dashboard, daily monitoring panel
- **Indicator type:** **Leading**

---

## Section 8 — Customer Metrics

### 8.1 Customer Lifetime Value (LTV) Snapshot

1. **Business Definition:** The maximum observed `user_ltv.revenue` value per user within the dataset window (GA4's own running LTV estimate at last-seen point).
2. **Business Purpose:** Segments users by cumulative value contribution — the foundation for high/medium/low-value customer segmentation (Stage 6/Section 9 crossover).
3. **Executive Owner:** Finance / Marketing
4. **Formula:** `MAX(user_ltv.revenue) PER user_pseudo_id`
5. **SQL Logic:** Group by `user_pseudo_id`, take `MAX()` (never `AVG()`) of `user_ltv.revenue` across all their event rows.
6. **Grain:** User
7. **Dimension compatibility:** Acquisition channel, device, geo
8. **Refresh frequency:** Monthly
9. **Interpretation:** This is a within-window snapshot, not true lifetime value beyond the 92-day sample — label it accordingly ("observed value within window," not "true LTV") to avoid overclaiming.
10. **Common mistakes:** `AVG(user_ltv.revenue)` across a user's rows — since this field is 0 before their first purchase and only updates afterward, averaging across all their rows (many of which precede any purchase) will dramatically understate true value. This exact trap was flagged in Stage 2 and is restated here because it's the single most common LTV-calculation bug on this dataset.
11. **Caveats from Stage 3:** Not directly Stage-3-validated (no dedicated null-rate check on `user_ltv` was run) — flagged as a secondary open validation item alongside 5.4's refund gap.
12. **Example business decision supported:** Feeds the customer segmentation work in Stage 6 (Section 9 crossover) — identifying which acquisition channels (Section 10) disproportionately deliver high-LTV users.

- **Source fields:** `user_ltv.revenue`, `user_pseudo_id`
- **Required joins/UNNEST:** None
- **Validation dependency:** **Open item — recommend a dedicated null-rate/distribution check before Stage 6 use**
- **Dashboard usage:** Customer segmentation panel
- **Indicator type:** **Lagging**

### 8.2 Purchase Frequency

1. **Business Definition:** Average count of valid transactions per purchasing user over the window.
2. **Business Purpose:** A companion metric to 7.3 (Repeat Purchase Rate) and 8.1 (LTV) — quantifies depth of repeat behavior among those who do return, not just whether they return.
3. **Executive Owner:** Finance / Product
4. **Formula:** `COUNT(valid transactions) / COUNT(DISTINCT purchasing users)`
5. **SQL Logic:** [R2]-cleaned transaction count divided by distinct purchasing user count.
6. **Grain:** User
7. **Dimension compatibility:** Acquisition channel, geo, device
8. **Refresh frequency:** Monthly
9. **Interpretation:** Expect this close to 1.0 given the low sessions-per-user baseline — any meaningfully-above-1.0 reading is itself a notable finding worth surfacing.
10. **Common mistakes:** Same [R2] risk as 7.3 — must dedupe transactions correctly or repeat-purchase behavior will be overstated.
11. **Caveats from Stage 3:** [R2] mandatory.
12. **Example business decision supported:** Distinguishes whether a loyalty program should focus on "getting more people to return once" (7.3) vs. "getting existing repeaters to buy more often" (this metric) — two different program designs.

- **Source fields:** `ecommerce.transaction_id`, `user_pseudo_id`
- **Required joins/UNNEST:** None
- **Validation dependency:** §4.2 (R2)
- **Dashboard usage:** Customer segmentation panel
- **Indicator type:** **Lagging**

### 8.3 Value Tier Segmentation (High/Medium/Low)

1. **Business Definition:** Users bucketed into value tiers based on LTV Snapshot (8.1) quartiles or fixed thresholds.
2. **Business Purpose:** Operationalizes 8.1 into an actionable segment for targeting/reporting rather than a raw continuous number.
3. **Executive Owner:** Marketing
4. **Formula:** Quartile or threshold bucketing of `MAX(user_ltv.revenue)` per user
5. **SQL Logic:** `NTILE(4)` or fixed-threshold `CASE WHEN` over the per-user LTV snapshot.
6. **Grain:** User
7. **Dimension compatibility:** Acquisition channel, device, geo — the primary use case is cutting *other* metrics (channel mix, device split) by this segmentation
8. **Refresh frequency:** Monthly
9. **Interpretation:** With a 92-day, low-frequency-purchase dataset, expect the "high value" tier to be a very small absolute number of users — communicate tier sizes, not just tier existence, to avoid overstating a thin top segment's business relevance.
10. **Common mistakes:** Presenting tier behavior (e.g., "high-value users convert 3x better") without stating how few users sit in that tier — a classic statistical-significance blind spot.
11. **Caveats from Stage 3:** Inherits 8.1's open validation item.
12. **Example business decision supported:** Directly informs a "which channel brings our highest-value customers" cut in Section 10 Marketing Metrics — a genuinely decision-relevant, not just descriptive, segmentation.

- **Source fields:** Derived from 8.1
- **Required joins/UNNEST:** None
- **Validation dependency:** Same open item as 8.1
- **Dashboard usage:** Customer segmentation panel, Marketing Channel dashboard (cross-reference)
- **Indicator type:** **Lagging**

---

## Section 9 — Product Metrics

### 9.1 Product View Count

1. **Business Definition:** Count of `view_item` events per `item_id`/`item_name`, scoped to rows with populated `items` [R4].
2. **Business Purpose:** Baseline demand-signal metric — which products get browsed, independent of whether they convert.
3. **Executive Owner:** Product / Merchandising
4. **Formula:** `COUNT(*) FROM UNNEST(items) WHERE event_name='view_item' GROUP BY item_id`
5. **SQL Logic:** `UNNEST(items)`, filter to `view_item`, group by product identifier.
6. **Grain:** Item (product line)
7. **Dimension compatibility:** Category, device, traffic source
8. **Refresh frequency:** Weekly
9. **Interpretation:** Always caveat with the 62.65% coverage rate [R4] — this undercounts true product-view volume by construction, useful for *relative* ranking between products, not absolute totals.
10. **Common mistakes:** Presenting this as "total product views" without the coverage caveat — an executive reading "241,876 views" should know the true figure is higher but not fully attributable to specific products.
11. **Caveats from Stage 3:** [R4] mandatory, direct consequence of the §5.2 FAIL finding.
12. **Example business decision supported:** Identifies high-view, low-conversion products for a merchandising/pricing review (paired with 9.2).

- **Source fields:** `items.item_id`, `items.item_name`, `event_name`
- **Required joins/UNNEST:** `UNNEST(items)`
- **Validation dependency:** §5.2 (R4 — mandatory)
- **Dashboard usage:** Product Performance panel
- **Indicator type:** **Leading**

### 9.2 Product-Level View-to-Cart Rate

1. **Business Definition:** Per-product version of 6.1 — share of views of a specific product that lead to that same product being added to cart.
2. **Business Purpose:** Identifies specific underperforming or overperforming SKUs, not just funnel stages in the abstract.
3. **Executive Owner:** Merchandising / Product
4. **Formula:** `COUNT(add_to_cart for item X) / COUNT(view_item for item X)`
5. **SQL Logic:** `UNNEST(items)` on both event types, join/group by `item_id`.
6. **Grain:** Item
7. **Dimension compatibility:** Category, device
8. **Refresh frequency:** Weekly
9. **Interpretation:** A product with high views but low cart-add rate is a pricing/presentation problem; a product with low views but high cart-add rate (when viewed) is a discoverability/merchandising-placement opportunity.
10. **Common mistakes:** Ranking products by raw view count alone without this conversion cut — high-traffic, low-converting products look deceptively "popular" without it.
11. **Caveats from Stage 3:** [R4] applies to the view_item side of this ratio (62.65% coverage); add_to_cart side is 100% populated per §5.2, so the denominator is understated, not the numerator.
12. **Example business decision supported:** Directly informs a per-SKU pricing or merchandising-placement recommendation in `07_Business_Recommendations.md`, rather than a generic "improve conversion" statement.

- **Source fields:** `items.item_id`, `event_name`
- **Required joins/UNNEST:** `UNNEST(items)`
- **Validation dependency:** §5.2 (R4)
- **Dashboard usage:** Product Performance panel
- **Indicator type:** **Leading**

### 9.3 Product Revenue Share

1. **Business Definition:** Each product's share of total item-level revenue.
2. **Business Purpose:** Identifies revenue concentration — is the business dependent on a handful of hero SKUs, or broadly diversified?
3. **Executive Owner:** Merchandising / Finance
4. **Formula:** `SUM(items.item_revenue_in_usd for item X) / SUM(items.item_revenue_in_usd, all items)`
5. **SQL Logic:** `UNNEST(items)`, group by `item_id`, compute share of total.
6. **Grain:** Item
7. **Dimension compatibility:** Category, date
8. **Refresh frequency:** Monthly
9. **Interpretation:** Report as *relative share*, never reconcile the absolute sum against transaction-level Total Revenue (5.1) per [R3] — the 28.56% gap confirmed in §4.3 means these are different, non-reconciling views of revenue by design.
10. **Common mistakes:** Adding up product revenue shares and expecting the total to match headline Total Revenue (5.1) exactly — per [R3], it won't, and that's expected, not an error to chase.
11. **Caveats from Stage 3:** [R3] mandatory.
12. **Example business decision supported:** A small number of SKUs driving a disproportionate revenue share informs inventory/supply-chain prioritization (flagged as adjacent to, but outside, this project's explicit scope per `01_Project_Overview.md`).

- **Source fields:** `items.item_id`, `items.item_revenue_in_usd`
- **Required joins/UNNEST:** `UNNEST(items)`
- **Validation dependency:** §4.3 (R3 — mandatory)
- **Dashboard usage:** Product Performance panel
- **Indicator type:** **Lagging**

### 9.4 Category Performance Index

1. **Business Definition:** Composite ranking of `item_category` by combining view share, conversion rate (9.2), and revenue share (9.3) into one relative index.
2. **Business Purpose:** A single "how is this category doing" rollup for merchandising planning meetings, rather than three separate numbers to mentally combine.
3. **Executive Owner:** Merchandising
4. **Formula:** Composite/weighted index of 9.1, 9.2, 9.3 normalized to category grain (specific weighting to be finalized during Stage 6 analysis based on stakeholder input — documented here as a defined composite, not yet weighted)
5. **SQL Logic:** Aggregate 9.1–9.3 by `item_category` instead of `item_id`, then combine into an index.
6. **Grain:** Item category
7. **Dimension compatibility:** Device, date
8. **Refresh frequency:** Monthly
9. **Interpretation:** Useful for spotting a category that's high-traffic-low-conversion (a merchandising fix) vs. low-traffic-high-conversion (a discoverability/promotion opportunity) at a glance.
10. **Common mistakes:** Building a composite index before the underlying components (9.1-9.3) are individually validated and trusted — sequencing matters; this index is only as reliable as its [R4]/[R3]-caveated inputs.
11. **Caveats from Stage 3:** Inherits [R4] and [R3] from its component metrics.
12. **Example business decision supported:** Directly usable in a merchandising planning review to prioritize which category gets a homepage placement refresh next quarter.

- **Source fields:** Derived from 9.1, 9.2, 9.3 at category grain
- **Required joins/UNNEST:** `UNNEST(items)`
- **Validation dependency:** §5.2, §4.3
- **Dashboard usage:** Product Performance panel, Merchandising review deck
- **Indicator type:** **Lagging**

---

## Section 10 — Marketing Metrics

### 10.1 Channel Conversion Rate

1. **Business Definition:** Share of sessions from a given `medium+source` that reach `purchase`.
2. **Business Purpose:** The core channel-quality metric — distinguishes "brings a lot of traffic" from "brings traffic that actually buys."
3. **Executive Owner:** Marketing
4. **Formula:** `COUNT(purchasing sessions | medium=X, source=Y) / COUNT(sessions | medium=X, source=Y)`
5. **SQL Logic:** Session-scoped funnel completion (as in 6.4), grouped by `traffic_source.medium, traffic_source.source`.
6. **Grain:** Session
7. **Dimension compatibility:** Device, geo, date
8. **Refresh frequency:** Weekly
9. **Interpretation:** Always paired with Channel Session Mix (2.4) — a channel can have a high conversion rate but tiny volume, or vice versa; neither number alone tells the budget story.
10. **Common mistakes:** Ranking channels by conversion rate alone without volume context — a channel with 3 sessions and 1 purchase (33% "conversion rate") is not a signal to invest more.
11. **Caveats from Stage 3:** Direct traffic `(none)` at 23.04% (§6.2) must be reported as its own channel row, not dropped as "unknown."
12. **Example business decision supported:** Primary input to the channel-prioritization recommendation in `07_Business_Recommendations.md`.

- **Source fields:** `traffic_source.medium`, `traffic_source.source`, `event_name`
- **Required joins/UNNEST:** Session_key join
- **Validation dependency:** §6.2, §2.2
- **Dashboard usage:** Marketing Channel dashboard
- **Indicator type:** **Lagging**

### 10.2 Channel Revenue Share

1. **Business Definition:** Each channel's share of Total Revenue (5.1).
2. **Business Purpose:** Financial-weight view of channel importance, complementing the rate-based view in 10.1.
3. **Executive Owner:** Marketing / Finance
4. **Formula:** `SUM(purchase_revenue_in_usd | channel X) / SUM(purchase_revenue_in_usd, all channels)`
5. **SQL Logic:** Attribute each purchase's revenue to the session/channel that led to it (last-non-direct-click attribution, GA4's default session-scoped model — stated explicitly since attribution methodology is itself a decision, not a given).
6. **Grain:** Transaction, attributed back to session-level channel
7. **Dimension compatibility:** Device, geo, date
8. **Refresh frequency:** Monthly
9. **Interpretation:** Read alongside 10.1 — a channel can carry a large revenue share simply from volume, not necessarily higher quality; the conversion rate is the quality lens, this is the scale lens.
10. **Common mistakes:** Conflating "revenue share" with "channel is high-quality" — a large channel can carry a big revenue share with a mediocre conversion rate purely on volume.
11. **Caveats from Stage 3:** Inherits [R2]/[R3] from underlying revenue and transaction cleaning rules.
12. **Example business decision supported:** Budget-reallocation conversations at the marketing planning review — paired explicitly with 10.1 to avoid the volume/quality conflation above.

- **Source fields:** `ecommerce.purchase_revenue_in_usd`, `traffic_source.medium/source`
- **Required joins/UNNEST:** Session-to-transaction attribution join
- **Validation dependency:** §4.1, §4.3, §6.2
- **Dashboard usage:** Marketing Channel dashboard
- **Indicator type:** **Lagging**

### 10.3 New User Share by Channel

1. **Business Definition:** Of all new users (2.2) in the period, the share acquired via each channel.
2. **Business Purpose:** Distinguishes acquisition-focused channels (bring new people) from retention/re-engagement channels (bring back existing users) — both valuable, different roles.
3. **Executive Owner:** Marketing
4. **Formula:** `COUNT(new users | channel X) / COUNT(total new users)`
5. **SQL Logic:** Filter to `ga_session_number=1` sessions, group by channel.
6. **Grain:** User
7. **Dimension compatibility:** Device, geo
8. **Refresh frequency:** Monthly
9. **Interpretation:** A channel with low new-user share but high overall session share is likely a retention/remarketing channel (e.g., email), not a discovery channel — label and evaluate accordingly, not on the same rubric as paid search.
10. **Common mistakes:** Judging every channel by the same "drives new users" yardstick — retargeting/email channels are structurally supposed to skew returning-user-heavy, and that's a feature, not underperformance.
11. **Caveats from Stage 3:** Inherits §5.1's confirmed full coverage of `ga_session_number`.
12. **Example business decision supported:** Correctly separates "grow the top of funnel" budget decisions from "improve retention economics" budget decisions — two different line items.

- **Source fields:** `event_params.ga_session_number`, `traffic_source.medium/source`
- **Required joins/UNNEST:** `UNNEST(event_params)`
- **Validation dependency:** §5.1
- **Dashboard usage:** Marketing Channel dashboard
- **Indicator type:** **Leading**

### 10.4 Organic vs. Paid Traffic Mix

1. **Business Definition:** Share of sessions where `traffic_source.medium` indicates paid acquisition (`cpc`, `paid`, `display`) vs. organic (`organic`, `(none)`, `referral`).
2. **Business Purpose:** A top-line "how dependent are we on paid traffic" health check — relevant to sustainability of growth.
3. **Executive Owner:** Marketing / Leadership
4. **Formula:** `COUNT(sessions | medium IN paid-list) / COUNT(total sessions)`
5. **SQL Logic:** `CASE WHEN` bucketing of `traffic_source.medium` into paid vs. organic categories, then share calculation.
6. **Grain:** Session
7. **Dimension compatibility:** Date, geo
8. **Refresh frequency:** Monthly
9. **Interpretation:** Stage 2/3 findings (`gclid`/`gclsrc` almost universally null) suggest this sample skews heavily organic — confirm the actual paid-medium share explicitly rather than assuming, since the null gclid finding is about *click-ID* presence, not medium classification, and the two shouldn't be conflated.
10. **Common mistakes:** Assuming "no gclid" means "no paid traffic" — `medium = 'cpc'` can exist without a populated `gclid` (e.g., non-Google paid platforms); check `medium`, not just `gclid`, for this metric.
11. **Caveats from Stage 3:** Related to but distinct from the gclid null-rate observation in `02_Data_Understanding.md` §4.1 (event_params key list) — this metric uses `traffic_source.medium`, a different field, and must not inherit that caveat incorrectly.
12. **Example business decision supported:** A finding of heavy organic dependence supports a recommendation to test incremental paid investment as a growth lever (Experiment Ideas, later doc), since there's clear headroom.

- **Source fields:** `traffic_source.medium`
- **Required joins/UNNEST:** None
- **Validation dependency:** §6.2
- **Dashboard usage:** Marketing Channel dashboard, Executive Dashboard (secondary panel)
- **Indicator type:** **Leading**

---

## Section 11 — Executive Scorecard

A single-page rollup combining one metric per section above — this is the literal content of the Executive Dashboard's top row, built in Stage 6/8.

| Metric | Section | Type | Owner | Status flag logic |
|---|---|---|---|---|
| Weekly Transacting Users (North Star) | 1 | Lagging | VP Product | Red if 4-week trend is negative |
| Sessions | 2 | Leading | Growth | Informational, always shown with seasonality note |
| New User Rate | 2 | Leading | Growth | Red if <40% or >90% (either extreme is a signal) |
| Engagement Rate | 4 | Leading | Product | Red if declining 2+ consecutive weeks |
| Total Revenue | 5 | Lagging | Finance | Always annotated with Nov–Dec seasonal flag |
| AOV | 5 | Lagging | Finance/Product | Red if down >10% month-over-month |
| Overall View-to-Purchase Conversion | 6 | Lagging | VP Product | Always shown with 3-stage breakdown, never alone |
| Weekly Returning User Rate | 7 | Lagging | Product/Growth | Benchmarked against occasional-purchase category, not habitual apps |
| Repeat Purchase Rate | 7/8 | Lagging | Finance/Product | Headline number for the "brand vs. revenue engine" framing question |
| Channel Revenue Share (Top 3) | 10 | Lagging | Marketing | Always paired with Channel Conversion Rate (10.1), never shown alone |

**Guardrail metrics** (should not silently degrade while North Star improves): Engagement Rate (4.1), Refund Rate (5.4, pending validation), Cart Abandonment Rate (6.5) — a rising WTU driven by, say, aggressive discounting that also spikes refunds would be a false positive on the North Star.

---

## Section 12 — Metric Dependency Map

```
RAW EVENT TABLE (events_*)
│
├── event_params (UNNEST) ──────────────┬── ga_session_id ──────► Session Key ──► 2.1 Sessions ──► 5.5 RPS, 6.x Funnel, 7.4 Session Mix, 10.x Marketing
│                                        ├── ga_session_number ──► 2.2 New Users ──► 2.3 New User Rate ──► 10.3 New User Share by Channel
│                                        │                    └──► 7.4 New vs Returning Mix
│                                        ├── session_engaged ────► 4.1 Engagement Rate ──► 3.3 First-Session Engagement Rate
│                                        ├── engagement_time_msec (capped [R5]) ──► 4.2 Avg Engagement Time
│                                        └── entrances ───────────► 2.5 Landing Page Entrance Rate
│
├── user_pseudo_id ──────────────────────► 2.1, 4.4 Sessions per User, 7.1–7.3 Retention, 8.1–8.3 Customer Metrics
│
├── user_first_touch_timestamp ──────────► 3.2 Time-to-First-Cart, 7.1 Day N Retention
│
├── user_ltv.revenue (MAX per user) ─────► 8.1 LTV Snapshot ──► 8.3 Value Tier Segmentation ──► 10.x cross-cuts
│
├── device.category ──────────────────────► Dimension cut across nearly all sections
├── geo.country/region ────────────────────► Dimension cut across nearly all sections
├── traffic_source.medium/source ──────────► 2.4 Channel Mix, 10.1–10.4 Marketing Metrics
│
├── ecommerce.* (transaction grain, [R2]/[R3] cleaning applied) ─┬── purchase_revenue_in_usd ──► 5.1 Total Revenue ──► 5.2 AOV, 5.5 RPS, 9.3 (non-reconciling), 10.2 Channel Revenue Share
│                                                                  ├── transaction_id (cleaned) ──► 5.2 AOV, 7.3 Repeat Purchase Rate, 8.2 Purchase Frequency
│                                                                  ├── refund_value_in_usd (unvalidated) ──► 5.4 Refund Rate [OPEN]
│                                                                  └── total_item_quantity ──► 5.3 Items per Transaction
│
└── items (UNNEST, [R4] coverage caveat) ─┬── item_id/category ──► 9.1 Product Views, 9.2 Product View-to-Cart, 9.3 Product Revenue Share ──► 9.4 Category Performance Index
                                            └── item_revenue_in_usd ──► 9.3 (feeds Category Index, non-reconciling with 5.1 per [R3])
```

**Two open validation items carried forward, not silently assumed:** 5.4 Refund Rate and 8.1 LTV Snapshot both require a dedicated follow-up check before their first production use in Stage 5/6 — flagged here rather than treated as validated by omission.

---

**Next:** Stage 5 — SQL Query Repository. Every query written there must cite which metric definition above it implements, and must apply the validation-derived rules ([R1]–[R5]) inherited by that metric.
