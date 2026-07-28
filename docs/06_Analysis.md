# 06 : Executive Product Analytics Report

**Google Merchandise Store 92-Day Performance Review (Nov 1, 2020 - Jan 31, 2021)**
**Prepared for:** CEO, VP Product, Head of Analytics, Growth, Marketing, Finance, Product Leadership
**Basis:** 54 validated SQL queries executed against `bigquery-public-data.ga4_obfuscated_sample_ecommerce`, cross-checked internally, with all findings traceable to a specific query ID and methodology rule (R1-R7). No figure in this report is estimated or invented where evidence doesn't exist, this report says so explicitly rather than filling the gap.

---

# 1. Executive Summary

This store converts visitors and generates revenue, but it does not build customers. Every top-of-funnel and monetization number looks like a normal, functioning ecommerce operation reasonable engagement, a coherent checkout flow, a healthy average order value. The moment you look at what happens *after* a visit, the picture changes: 82% of users never return a second time, Day 30 retention is 0.13%, and the business is structurally a continuous acquisition machine, not a repeat-revenue engine. That single fact should reframe how every other number in this report is read a "declining engagement rate" or a "weak channel" matters differently in a business that has no organic retention floor to fall back on.

**Top 10 Findings**

1. **Only 19.97% of first-time visitors ever view a single product** [Q3.1] the single largest, earliest leak in the entire customer journey, upstream of the funnel entirely.
2. **82.47% of all users are single-session, one-and-done visitors** [Q4.4]; Day 1/7/30 retention is 4.63% / 0.71% / 0.13% [Q7.1] this is a brand-engagement vehicle, not a repeat-revenue business, as a matter of measured fact, not framing.
3. **The steepest checkout drop-off is shipping→payment info entry (38.6% loss)**, not the final purchase step (28.9% loss) [Q6.3] the fix is in payment UX, not general "reduce abandonment" messaging.
4. **0.43% of users (1,161 people) account for ~62% of measured customer value** [Q8.3] extreme concentration with no evident targeted program built around it.
5. **A privacy-redacted traffic segment, `(data deleted)`, converts at 3.14%, 2.8x the store's next-best channel and contributes 12.97% of revenue from just 6.17% of sessions** [Q10.1, Q10.2] the store's best-performing segment is currently invisible to standard channel reporting.
6. **Paid search (`cpc/google`) underperforms organic on every quality metric** lower conversion (0.98% vs. 1.11%), lower engagement (70.11%, the lowest of all real channels), and only 4.34% of total traffic [Q10.1, Q2.10, Q10.4] current paid spend is inefficient, not merely small.
7. **A confirmed instrumentation change on Dec 28, 2020 makes engagement-rate trend data before and after that date non-comparable** [Q4.6, R6] a material measurement-integrity finding that could have led to a false "engagement doubled" narrative if undetected.
8. **`item_id` cannot be trusted as a join key across event types in this dataset** [Q9.5, R7] a foundational data-quality issue that would have produced a materially wrong merchandising conclusion (a "0%-converting" mega-category that doesn't actually exist) had it gone unchecked.
9. **Mobile modestly outperforms desktop on both revenue-per-session (1.025 vs. 0.999) and cart abandonment (80.58% vs. 81.73%)** [Q5.5, Q6.5], despite carrying only 40% of session volume the inverse of the industry-default assumption that desktop-majority sites also convert best on desktop.
10. **43% of checkout sessions show no detectable prior add-to-cart event**, and only 3.4% are explained by cross-session cart persistence [Q6.6] a large, currently unexplained share of the purchase path that is either a "Buy Now" UX pattern or a tracking gap, and is an open item requiring product/engineering investigation, not a resolved finding.

**Business Health Assessment:** This is a **operationally sound, strategically under-leveraged** business. Traffic, checkout mechanics, and revenue processing all function normally. The gaps are concentrated in (a) converting first-time interest into product engagement, (b) retaining any of the acquired audience, and (c) actively working the two segments high-value customers and the `(data deleted)` channel that already demonstrate the store is capable of much better economics than its blended averages suggest.

**Biggest Strength:** A confirmed, working checkout mechanism (43.62% of sessions that begin checkout complete it [Q6.3]) and a small but real base of repeat purchasers who convert at meaningfully higher rates than average once acquired (11.85% repeat purchase rate [Q7.3]) proof the product experience works when someone re-engages, the problem is that few do.

**Biggest Weakness:** Near-total absence of retention infrastructure or behavior (Finding #2) combined with zero visible investment in the specific segments (Findings #4, #5) that the data shows are disproportionately valuable.

**Immediate Executive Actions:** (1) Commission a homepage/PDP activation redesign targeting the 19.97% product-view rate the single highest-leverage fix available. (2) Freeze or reallocate incremental `cpc/google` spend pending a targeting review. (3) Instruct Analytics/Engineering to resolve the R6 tagging discontinuity and R7 item_id join failure before any further trend or merchandising reporting is built on top of them. (4) Commission a compliant investigation into the `(data deleted)` segment's characteristics.

---

# 2. Acquisition Analysis

**Traffic volume and seasonality.** Daily sessions ranged from a low of 2,170 (Nov 8) to a peak of 7,588 (Dec 8) [Q2.1]. Notably, **the single highest-traffic day of the entire window was Dec 8 not the Nov 27–30 Black Friday/Cyber Monday period**, which peaked at a comparatively modest 4,593 sessions (Nov 30). A second, smaller spike occurred Jan 6 (6,002 sessions). Neither spike has a confirmed cause in the executed queries this report does not assume "Black Friday" as the explanation for a spike that demonstrably peaked elsewhere. **Recommended follow-up, not yet executed:** correlate these two specific dates against any known promotional calendar or PR event before attributing the pattern.

**Channel mix** [Q2.4]: organic/google (31.29% of sessions), direct (23.17%), an aggregated `<Other>` bucket (14.46%), external referral (9.63%), internal/self-domain referral (8.01%), the privacy-redacted `(data deleted)` segment (6.17%), and paid search `cpc/google` (4.34%). This is an **organic- and direct-dependent acquisition mix** paid channels are a rounding error in volume terms.

**New user trend a warning sign, not a growth signal.** New User Rate rose steadily across the window, from 82.28% in the earliest observed week to a peak of 90.84% (week of Dec 28), settling at 86.45% by the final week [Q2.3]. In isolation this looks like healthy acquisition. Read against Section 7's retention findings, it means the opposite: **the store is not accumulating a stable repeat audience the traffic mix is getting *more*, not less, first-time-only over the course of the window.** This is the earliest leading indicator of the retention problem detailed later in this report, visible before any retention metric is even calculated.

**Landing pages concentrated, with a data-quality catch.** Raw entrance data initially showed the homepage fragmented across three URL variants (`shop.`, bare domain, and `www.` prefixes); once normalized [Q2.9], the true homepage share is **45.96% of all entrances** nearly half of all sessions enter through a single page. The next-largest entrances are the Apparel category page (11.8%) and the YouTube shop-by-brand page (7.06%). **This concentration is a resilience risk**: any homepage issue (outage, redesign misstep, broken promo banner) has an outsized blast radius on total traffic quality.

**Channel quality, not just volume** [Q2.10]: first-session engagement rate is remarkably uniform across real channels (organic 70.68%, direct 70.92%, referral 70.89–71.44%, `cpc/google` 70.11%), with `cpc/google` the lowest of all real channels and `(data deleted)` a dramatic outlier at 96.01%. Combined with Section 10's conversion-rate findings, **paid search is the one channel showing weakness on every quality dimension simultaneously** not just small, but the least efficient use of the traffic it does generate.

**Traffic quality anomaly requiring disclosure, not exclusion:** 85.74% of all events, and 99.7%+ of all sessions, carry `debug_mode = 1` [§6.4 Data Validation, Q2.8]. Query 2.8 confirmed this saturation is essentially uniform across every channel (99.7–99.97%), meaning **relative channel comparisons in this report are not distorted by this issue**, even though absolute volumes should be read as directional, per **R1**.

**Growth efficiency** [Q1.2]: Weekly Transacting Users per 100 sessions peaked at 1.84 (week of Dec 7) and fell to a low of 0.38 (week of Jan 25) a **nearly 5x swing in acquisition efficiency** across the window, moving inversely with the seasonal traffic curve. This confirms that the Nov–Dec traffic surge was not merely bigger, it was *more efficient* traffic a distinction that matters for whether post-holiday spend should chase volume or quality.

**Key SQL references:** Q1.1, Q1.2, Q2.1–Q2.10.

---

# 3. Activation Analysis

This is the section that reframes the entire report. **Activation, not checkout, is this store's primary constraint.**

**Observation:** Only 19.97% of first-ever sessions (52,171 of 261,238) ever view a single product [Q3.1].
**Supporting Metrics:** Metric 3.1 (First-Session Product View Rate).
**Business Interpretation:** Four out of five brand-new visitors leave without ever reaching a product detail page. Whatever happens between landing and the catalog homepage design, navigation clarity, category-page friction is losing more potential customers than every downstream funnel stage combined.
**Root Cause Hypothesis:** Given the homepage's 45.96% entrance concentration (Section 2) and this store's category-page-heavy top landing pages (Apparel, YouTube brand), the likely failure point is homepage-to-catalog navigation, not the catalog itself but this is a hypothesis, not yet isolated by an executed query, and should be the subject of a dedicated navigation-path analysis.
**Confidence Level:** High (the 19.97% figure itself is directly measured; the root cause is a reasoned hypothesis, not yet directly tested).
**Recommended Action:** A homepage/navigation UX audit and A/B test specifically targeting time-to-first-product-view, prioritized above any checkout-stage initiative.
**Priority:** Critical.
**Expected Business Outcome:** Given that only 19.97% of new sessions reach the point where the rest of the funnel (6.1% view-to-purchase, Q6.4) can even begin, even a modest improvement here compounds through every downstream stage this is the highest-leverage single fix identified in this report.

**A supporting, important nuance:** first-session *engagement rate* is 70.84% [Q3.3] GA4's own definition of "engaged" (10+ seconds on-site, 2+ pageviews, or a conversion event). **These two numbers together are the finding:** 71% of new visitors technically "engage" by GA4's time-based definition, but fewer than 1 in 5 ever look at a product. GA4's "engagement rate" is being read internally, almost certainly, as a proxy for interest this report shows explicitly that it is not one. Any team using engagement rate as a health metric for this store should pair it with product-view rate, never report it alone.

**Time-to-cart, for the minority who do engage with product:** among the 12,222 users (4.52% of all users) who ever add an item to cart, the median time from first touch to first add-to-cart is 248 seconds (~4.1 minutes) [Q3.2] fast and decisive. This tells us the activation problem is not "people take too long to decide" it's "people never reach the decision point at all." This further supports prioritizing the homepage/discovery layer over cart or product-page optimization.

**Key SQL references:** Q3.1, Q3.2, Q3.3, Q2.10 (cross-reference).

---

# 4. Engagement Analysis

**This section requires a measurement-integrity disclosure before any trend can be discussed.** Query 4.6 confirms that `engagement_time_msec` the parameter underlying both Average Engagement Time and, indirectly, the Engagement Rate calculation was **entirely absent (0.0% presence) on every single day before Dec 28, 2020**, then jumped to 18.15% presence that day and stabilized around 42-44% from Dec 29 onward. This is not a gradual behavioral shift; it is a clean, dated instrumentation/tagging change (**R6**). Its direct consequence: weekly Engagement Rate [Q4.1] shows 47-58% in the Oct-Dec period, then an abrupt jump to 85-91% starting the week of Dec 28. **This is a measurement artifact, not a doubling of real user engagement, and must never be presented as a continuous trend line without a split annotation at that date.** This is the clearest example in this project of why validating an anomaly before reporting it matters a report built without this check would have told leadership engagement improved dramatically in January, when in fact nothing about user behavior changed.

**With that caveat fixed in place, what can be said with confidence:**

- **Pages per session:** mean 3.75, median 2.0 [Q4.3]. The gap between mean and median (consistent with duplicate page_view firing under the R1 debug-traffic pattern) means the median is the more trustworthy "typical session" figure.
- **Average engagement time per session (capped per R5):** mean 69.7 seconds, **median only 8.1 seconds** [Q4.2] an 8.6x gap between mean and median, meaning a small number of highly-engaged sessions are pulling the average far above what a typical session looks like. Any engagement-time KPI reported to leadership should lead with the median.
- **Sessions per user:** mean 1.333, but the 25th/50th/75th percentiles are *all exactly 1* [Q4.4] meaning at least three-quarters of all users never return for a second session in this 92-day window. This is the engagement-layer confirmation of the retention crisis detailed fully in Section 7.
- **Scroll depth** [Q4.5] is only ever recorded at the single value of 90% GA4's default single-threshold trigger. This field is **not usable for granular content-engagement analysis** in this implementation and should not be forced into a dashboard as if it carries more resolution than it does.

**Business Interpretation:** Once the R6 artifact is excluded, this store shows shallow, high-variance engagement most sessions are brief, and the sessions that are deep are a distinct minority pulling up the averages. This is consistent with, not contradictory to, the Section 3 finding that most sessions never even reach a product page.

**Key SQL references:** Q4.1-Q4.6.

---

# 5. Funnel Analysis

**The executive funnel, stage by stage** (all session-scoped per the metric grain rules in `04_Metrics_Framework.md`):

| Stage | Rate | Query | Naive event-level estimate | Divergence |
|---|---|---|---|---|
| First session → views a product | 19.97% | Q3.1 | - | - |
| View → Add to cart | 19.7% | Q6.1 | ~15% (raw event ratio) | Close agreement |
| Add to cart → Begin checkout | **39.25%** | Q6.2 | ~66% (raw event ratio) | **27-point gap** |
| Begin checkout → Purchase (overall) | 43.62% | Q6.3 | - | - |
| View → Purchase (direct, overall) | **6.1%** | Q6.4 | ~3.4% (product of stage rates) | **~1.8x gap** |

**Two of these divergences are the most important methodological findings in the funnel analysis, and both are explained, not just flagged:**

1. **Cart-to-checkout: the naive event-count ratio (66%) overstated the true session-scoped rate (39.25%) by 27 points.** This is the textbook event-vs-session grain trap multiple `add_to_cart` events can fire within sessions that never proceed to checkout, inflating the naive denominator's apparent conversion. **Any report using the raw event ratio for this stage is materially wrong and should be corrected to 39.25%.**

2. **View-to-purchase direct measurement (6.1%) is nearly double what multiplying the three stage rates predicts (3.4%).** Query 6.6 tested and resolved this: cross-session cart persistence explains only 3.41% of the gap; the dominant explanation is that **42.91% of checkout sessions have no detectable `add_to_cart` event anywhere in that user's history** most consistent with either a "Buy Now" direct-purchase UX path that bypasses the cart event, or an add_to_cart tracking gap on certain entry points. **This remains an open item** requiring a product/engineering walkthrough of the actual checkout entry points before it can be resolved with confidence.

**Granular checkout breakdown where the money is actually lost** [Q6.3]:

| Step | Sessions | Loss from prior step |
|---|---|---|
| Begin checkout | 11,106 | - |
| Add shipping info | 11,104 | 0.02% (negligible) |
| Add payment info | 6,812 | **38.65%** |
| Purchase | 4,844 | 28.89% |

**The single largest, most specific, most actionable funnel finding in this report:** the drop between shipping-info entry and payment-info entry (38.65%) is meaningfully steeper than the drop between payment-info entry and final purchase (28.89%). **This means the primary checkout friction is being asked for payment information not the final submission step.** Plausible causes include unexpected costs revealed only at this stage (shipping fees, taxes), a limited set of accepted payment methods, forced account creation, or a technically clunky payment form but the *stage* is now identified with confidence; the *specific cause* requires a UX walkthrough, not further SQL.

**Cart abandonment by device** [Q6.5]: desktop shows the *highest* abandonment (81.73%), ahead of tablet (80.69%) and mobile (80.58%) inverting the common industry assumption that mobile checkout is the weaker experience. Cross-referenced with Section 11's device-revenue findings, this is a consistent, not contradictory, pattern: mobile is the more efficient device on this site, not the weaker one.

**Where should Product invest first?** In order of leverage: (1) homepage/discovery activation (Section 3, upstream of this entire funnel), (2) the payment-info checkout step specifically, (3) resolution of the 42.91% "no cart found" open item, since it represents a large, currently unexplained share of the purchase path that could contain either a valuable UX pattern worth doubling down on or a tracking gap worth fixing.

**Key SQL references:** Q6.1-Q6.6, Q3.1.

---

# 6. Revenue Analysis

**Revenue source of truth (R3):** All revenue figures in this section use transaction-level `ecommerce.purchase_revenue_in_usd`, never item-level or `user_ltv` sums, per the mandatory R3 rule established in Stage 3/4 and directly evidenced in Q11.2.

**Total revenue:** $362,165 across the 92-day window [Q5.1, confirmed independently in Q5.4 and Q11.2]. Weekly revenue peaked at $59,865 (week of Dec 7) and fell to a low of $5,546 (final week, Jan 25) [Q5.1] an **~11x swing** between peak and trough, considerably steeper than the traffic swing itself, indicating revenue is even more seasonally concentrated than session volume.

**AOV and transaction quality (R2 applied):** After excluding unattributable `(not set)` transaction IDs and deduplicating cross-user ID collisions (a confirmed data-quality issue affecting 767+ distinct users sharing a placeholder ID, per Stage 3 §4.2), the cleaned dataset shows 4,466 valid transactions totaling $308,830, for an **AOV of $69.15** but a **median order value of only $48** [Q5.2]. The $21 gap between mean and median is explained by confirmed bulk/corporate orders (up to 400 units in a single transaction, $1,530 single-order value [§6.5 Data Validation]); items per transaction show the same pattern (mean 4.37, median 2.0 [Q5.3]). **AOV should always be reported alongside the median for this business the mean is not representative of a typical purchase.**

**The cost of the transaction_id data-quality issue, quantified:** applying R2's cleaning rules removes $53,335 **14.7% of total revenue** from AOV and transaction-count scope (though this revenue remains correctly included in the $362,165 headline total). This is not a rounding footnote; it's a material, quantified consequence of a confirmed data-integrity issue that any Finance stakeholder relying on transaction-level detail should be aware of.

**Refund rate: resolved as a genuine "not tracked" finding, not a data gap left open.** Zero `refund` events exist anywhere in the dataset [Q5.4]. This should be reported to Finance/Operations as "refunds are not observable via a dedicated event in this GA4 implementation," not as "0% refund rate" the distinction matters, since the latter implies a measured, confirmed absence of refunds rather than an absence of tracking.

**Revenue per session by device** [Q5.5]: mobile ($146,768 revenue / 143,185 sessions = 1.025 RPS) modestly outperforms desktop ($208,815 / 208,942 = 0.999 RPS) and clearly outperforms tablet (0.823 RPS), despite carrying 40% of session volume against desktop's 58%. Desktop's larger *absolute* revenue contribution is a volume effect, not an efficiency advantage see Section 11 for the full device discussion.

**Revenue concentration (cross-referenced with Section 8):** the top 0.43% of users by measured value account for roughly 62% of tiered customer value [Q8.3] this concentration pattern extends into revenue as well as customer segmentation, and is discussed fully in Section 8 to avoid duplication.

**Two revenue-adjacent figures that must never be used as "revenue," per R3:** item-level revenue (aggregate total $362,110, reconciling within 0.015% of the transaction-level total in aggregate, but with a confirmed 28.56% of *individual* transactions showing a >$1 discrepancy [§4.3 Data Validation, Q11.2]) is safe for *relative* category-share analysis only; `user_ltv.revenue`, summing to $407,626 across users, **overstates real revenue by 12.6%** and must be restricted to relative customer-tier segmentation, never cited as a dollar figure.

**Key SQL references:** Q5.1-Q5.5, Q11.2, Q8.3 (cross-reference).

---

# 7. Retention Analysis

**Observation:** Day 1 retention is 4.63% (7,883 of a 170,412-user cohort), Day 7 is 0.71% (1,218 users), Day 30 is 0.13% (228 users) [Q7.1].
**Supporting Metrics:** Metric 7.1 (Day N Retention), cross-referenced with Metric 4.4 (82.47% single-session users, Q4.4).
**Business Interpretation:** This is not a "retention needs improvement" finding it is a near-total absence of return-visit behavior at any measured horizon. By 30 days, essentially no one from a given acquisition cohort is still coming back.
**Root Cause Hypothesis:** Consistent with a low-frequency-purchase product category (branded apparel/merchandise) rather than a habitual-use product the appropriate benchmark comparison is other occasional-purchase ecommerce sites, not subscription or daily-use app retention curves. This is a category characteristic as much as a fixable "problem," and framing it purely as a fixable metric would be a mistake this report does not want leadership to make.
**Confidence Level:** High (directly measured across the full window).
**Recommended Action:** Do not set retention targets benchmarked against habitual-use products. Do invest specifically in the warmer, already-purchasing segment (see below), where retention economics are demonstrably better.
**Priority:** High (as a strategic framing issue) / Medium (as a directly "fixable" metric, given the category constraint).
**Expected Business Outcome:** Realistic target-setting prevents a Growth team from being held to an unreachable retention benchmark, and redirects investment toward the segment where it will actually move the needle.

**The nuance that changes the recommendation:** Weekly Returning User Rate holds steady in a 7–10% band across the window, dipping to 7.3% during the low-traffic holiday lull and recovering to nearly 10% by the final week [Q7.2] a relatively stable, not collapsing, pattern. But **Repeat Purchase Rate among people who have already bought once is 11.85%** [Q7.3] meaningfully higher than the general population's near-zero return rate. **The clearest, most actionable retention insight in this report: retention is not uniformly hopeless it is concentrated almost entirely among people who have already converted once.** A generic "improve retention" initiative aimed at all visitors will fail; a **targeted remarketing/loyalty program aimed specifically at past purchasers** has real, evidenced grounding.

**Session-grain retention proxy** [Q7.4]: the share of sessions from returning visitors ranged from 19.4% to 34.2% across the window, generally higher during the low-traffic post-holiday weeks meaning the *sessions* that do occur later in the window skew more returning-visitor, even though *absolute* new-user counts also stayed high. This is a useful daily-monitoring proxy but should not replace the user-grain metrics above for strategic decisions.

**Key SQL references:** Q7.1-Q7.4, Q4.4 (cross-reference).

---

# 8. Customer Analysis

**LTV snapshot, correctly scoped:** `user_ltv.revenue` is fully populated (0 nulls across 270,154 users) but heavily zero-inflated the 25th, 50th, and 75th percentiles are all exactly 0 [Q8.1], since the field only carries a non-zero value for the 4,445 users (1.65% of the base) who have ever purchased. **The average LTV snapshot across all users ($1.51) is a near-meaningless headline figure** reported without qualification, it would understate the value of an actual customer by roughly 60x. **Any LTV figure presented to leadership must be scoped to purchasers, not the full user base.**

**Purchase frequency:** 3,713 purchasing users generated 4,451 valid transactions a frequency of 1.199 [Q8.2], only modestly above 1.0. Combined with Section 7's 11.85% repeat-purchase rate, this paints a consistent picture: **most purchasers buy once; a minority buy more than once, and that minority is the store's most valuable, least-served segment.**

**Value tier concentration** [Q8.3]:

| Tier | Users | % of Users | Tiered Value | % of Value |
|---|---|---|---|---|
| High value | 1,161 | 0.43% | $254,278 | ~62.4% |
| Medium value | 2,307 | 0.85% | $135,405 | ~33.2% |
| Low value | 977 | 0.36% | $17,943 | ~4.4% |
| Non-purchaser | 265,709 | 98.36% | $0 | 0% |

**Observation:** Fewer than half a percent of all users drive nearly two-thirds of measured customer value.
**Supporting SQL Query IDs:** Q8.1, Q8.3.
**Supporting Metrics:** Metric 8.1 (LTV Snapshot), Metric 8.3 (Value Tier Segmentation).
**Business Interpretation:** This store has a small, identifiable population of power customers whose behavior if understood and replicated likely has more leverage on total revenue than any acquisition-channel optimization in this report.
**Root Cause Hypothesis:** Not yet isolated by an executed query a natural next analysis is to profile the High-value tier by acquisition channel, device, and geography to see whether it concentrates in any of the segments already identified as strong (e.g., the `(data deleted)` channel, or the internal/self-domain referral channel, both of which showed unusually high engagement and conversion in Sections 2 and 10).
**Confidence Level:** High (the concentration figures are directly measured); Medium (the underlying "why" is a hypothesis pending further segmentation work).
**Recommended Action:** Commission a High-value-tier profiling analysis before designing any loyalty/VIP program, so the program targets the channels/behaviors that actually produce this segment rather than a generic "big spenders" definition.
**Priority:** High.
**Expected Business Outcome:** A correctly-targeted VIP/loyalty program could plausibly protect or grow the ~62% of value currently concentrated in under 1,200 people a far higher-leverage investment than a blanket retention campaign aimed at the full user base.

**Important caveat carried through this entire section, per R3:** the dollar figures above derive from `user_ltv.revenue`, which Section 6 confirmed overstates real transaction revenue by 12.6% in aggregate. The *tier structure and relative concentration* are trustworthy; the *absolute dollar figures* in the table above should be treated as directionally, not precisely, accurate.

**Key SQL references:** Q8.1-Q8.3, Q7.3 (cross-reference), Q11.2 (R3 caveat source).

---

# 9. Product Analysis

**A necessary methodology note (R7):** Query 9.4's first attempt at a category-performance rollup produced a materially misleading result a mega-category (`"Home/Apparel/Men's / Unisex/"`) showing 462,115 views, 118,125 carts, and **$0.00 revenue**, which would have been reported as a catastrophic 0%-converting category if taken at face value. Investigation (Q9.5) revealed the true cause: `item_category` and, as the failed Q9.5 fix confirmed, **`item_id` itself** is tagged inconsistently between browsing-stage events (a full breadcrumb path) and purchase-stage events (a clean label), and **`item_id` does not reliably join across event types in this dataset at all**. The corrected version (Q9.6), joining on `item_name` instead, resolved the issue: the same "Apparel" volume that appeared to convert at 0% actually converts at a plausible, in-line rate, with $168,987 in attributable revenue. **This is now a permanent rule (R7): any product-level analysis in this dataset must join browsing and purchase data via `item_name`, never `item_id`.** This finding matters beyond this report any future analyst or dashboard builder on this dataset who trusts `item_id` will silently misattribute revenue exactly as Query 9.4 initially did.

**Top viewed products** [Q9.1]: dominated by apparel (Google Zip Hoodie F/C, Google Navy Speckled Tee, Google Tee Yellow each 34,000+ views) and a cluster of Android-branded accessories/journals/framed art in the "Sale" grouping (28,000–32,000 views each). This concentration matches the landing-page finding in Section 2 (Apparel and YouTube-brand pages dominate entrances) **the products people view are consistent with the pages they land on**, suggesting the merchandising/navigation funnel is internally coherent, even if its overall conversion rate is limited by the upstream activation problem (Section 3).

**Product-level view-to-cart spread** [Q9.2]: individual products range from 16.4% (YouTube Icon Tee Grey) to 31.1% (Google Crewneck Sweatshirt Navy) against the ~19.7% session-scoped baseline (Q6.1) a meaningful spread. Products clustering at the low end of this range (YouTube Icon Tee Grey 16.4%, Google Heather Green Speckled Tee 17.8%, Google Sherpa Zip Hoodie Charcoal 17.5%) are candidates for a pricing/presentation review; products at the high end (Google Crewneck Sweatshirt Navy 31.1%, Google Badge Heavyweight Pullover Black 28.7%) are candidates for increased homepage/promotional visibility, since they convert well once seen but may not be getting proportional placement.

**Category revenue concentration, using the corrected R7 methodology** [Q9.6]: Apparel dominates at $168,987 (the largest single category by a wide margin), followed by New ($26,156), Bags ($23,924), Campus Collection ($20,134), and Accessories ($17,871). This store's revenue is heavily Apparel-dependent any Apparel-specific supply, pricing, or inventory risk carries outsized business risk relative to its category-diversification appearance in the original (flawed) category count.

**A legitimate, non-artifact finding from the corrected analysis:** 193,921 views and 50,627 carts remain genuinely unmapped to any purchase consistent with the confirmed ~1.65% purchasing-user rate, this represents products that were browsed and even added to cart but never converted within the window, a real "interested but didn't buy" signal rather than a data defect, and a natural target list for cart-abandonment remarketing creative.

**Key SQL references:** Q9.1, Q9.2, Q9.3 (superseded), Q9.4 (superseded/flawed), Q9.5 (diagnostic), Q9.6 (corrected, authoritative).

---

# 10. Marketing Analysis

**Organic vs. paid mix** [Q10.4]: Organic/Referral (51.75%), Direct (23.17%), an unclassified `<Other>` bucket (20.74%), and Paid (4.34%) this is an **organic-dependent acquisition model with minimal paid investment**, not a paid-acquisition-driven business.

**Channel performance, side by side** (volume, conversion, revenue share, new-user share):

| Channel | Session Share | Conversion Rate | Revenue Share | New User Share |
|---|---|---|---|---|
| organic/google | 31.29% | 1.109% | 26.85% | 34.9% |
| (none)/direct | 23.17% | 1.293% | 22.19% | 23.8% |
| (data deleted) | 6.17% | **3.139%** | **12.97%** | 0.33% |
| referral/self-domain | 8.01% | 2.028% | 12.71% | 5.1% |
| referral/other | 9.63% | 1.358% | 10.34% | 9.31% |
| cpc/google | 4.34% | **0.98%** | 2.54% | 5.43% |

*(Q2.4, Q10.1, Q10.2, Q10.3)*

**The two standout findings, already introduced in the Executive Summary, deserve full elaboration here:**

**`(data deleted)` is this store's best-performing segment by a wide margin, and it is nearly invisible in standard reporting.** At 3.139% conversion 2.8x organic google's 1.109% and 12.97% of revenue from only 6.17% of sessions, this segment is overrepresented in value by roughly 2.1x its traffic share. Combined with its near-total returning-session skew (96.16%, Q2.7) and exceptional engagement rate (96.01%, Q2.10), this reads as a **specific, identifiable, highly-engaged repeat-customer cohort whose traffic-source data has been redacted**, most plausibly for privacy-consent reasons. **This is an open investigation item, not a channel Marketing can currently act on directly** but understanding *why* this cohort converts so well (loyalty program members? a specific email list? logged-in accounts?) is very likely the single highest-value open question this report raises for the Marketing and Analytics teams jointly.

**`cpc/google` underperforms on every available quality dimension simultaneously.** It has the lowest engagement rate of any real channel (70.11%, Q2.10), the lowest conversion rate (0.98%, Q10.1) below even `organic/<Other>` (0.954%) and a revenue share (2.54%) below its already-small session share (4.34%). This is not "a small channel with room to grow" it is **a small channel currently performing worse than the store's free channels.** The correct recommendation is not "invest more in paid," but **"fix targeting/landing-page fit for paid search before scaling spend."**

**A strong, underleveraged channel:** the internal/self-domain referral channel (`referral/shop.googlemerchandisestore.com`) converts at 2.028% second only to `(data deleted)` with 53.8% returning-session share (Q2.7), suggesting cross-domain navigation (e.g., from a blog, account portal, or related Google property) reliably brings warm, high-intent traffic. This channel's mechanics (what specifically links here, and from where) merits documentation and potential deliberate expansion.

**Key SQL references:** Q2.4, Q2.7, Q2.10, Q10.1-Q10.4.

---

# 11. Device Analysis

| Metric | Desktop | Mobile | Tablet |
|---|---|---|---|
| Session share | 58.16% | 39.67% | 2.17% |
| Sessions [Q5.5] | 208,942 | 143,185 | 8,002 |
| Revenue [Q5.5] | $208,815 | $146,768 | $6,582 |
| Revenue per Session [Q5.5] | 0.999 | **1.025** | 0.823 |
| Cart Abandonment [Q6.5] | **81.73%** | 80.58% | 80.69% |

**Observation:** Mobile modestly outperforms desktop on both revenue efficiency (+2.6% RPS) and cart abandonment (1.15 points lower), despite carrying 18 fewer percentage points of session share.
**Supporting SQL Query IDs:** Q5.5, Q6.5, §3.3 Data Validation (device split).
**Business Interpretation:** This inverts the default industry assumption that a desktop-majority ecommerce site also converts best on desktop, or that mobile checkout is inherently the weaker experience. On this site, the opposite is measured.
**Root Cause Hypothesis:** Not isolated by an executed query. Plausible explanations include a simpler, more focused mobile checkout flow, a mobile-skewed audience with higher purchase intent (e.g., returning app-like habitual browsing), or a desktop-specific friction point (larger form surface inviting hesitation, more tabs/distractions). This requires a dedicated device-level session-recording or heatmap review to resolve with confidence SQL alone cannot answer "why."
**Confidence Level:** High (the directional finding itself); Low (any causal explanation, which is speculative at this stage).
**Recommended Action:** Before investing in a "mobile-first redesign" (a common default recommendation this report deliberately does not make), conduct a desktop-specific checkout UX audit the data suggests desktop, not mobile, is this store's weaker device experience.
**Priority:** Medium (real, evidenced, but a smaller absolute-dollar lever than the activation or checkout-payment-step findings above).
**Expected Business Outcome:** Closing even part of the 1.15-point abandonment gap on desktop, which carries the largest session share, would have a larger absolute revenue impact than a mobile-only initiative.

**Tablet** is confirmed as the weakest device on every metric (lowest RPS, smallest volume at 2.17% of sessions) but represents too small a share of the business to prioritize independently.

**Key SQL references:** Q5.5, Q6.5.

---

# 12. Geographic Analysis

**This section is intentionally short, per this report's standing rule against invented findings.** No query in `05_SQL_Query_Repository.md` executed a country- or region-level breakdown of revenue, conversion, or engagement. The only executed geographic validation is the confirmation that `geo.country` is well-populated (only 0.75% `(not set)`, per Stage 3 §6.2) and a Stage 2 methodology decision to treat `city`/`metro`-level cuts as descriptive-only due to small sample sizes at that granularity.

**What this report can state with confidence:** the data infrastructure to support geographic analysis is sound (low null rate, reasonable country-level granularity). **What this report cannot state:** which countries or regions over- or under-perform, whether any market shows disproportionate value concentration, or whether localization would be worthwhile. **This is a genuine, acknowledged gap, flagged here rather than filled with assumption.**

**Recommended follow-up (not yet executed):** a country-level cut of Total Revenue, Conversion Rate, and New User Rate (Metrics 5.1, 6.4, 2.3) would close this gap directly using the existing metrics framework no new metric definitions are required, only additional queries.

---

# 13. Cross-Functional Insights

The findings across every section connect into a single causal chain, and understanding that chain is more valuable to leadership than any single-team metric in isolation:

```
Marketing drives traffic (organic/direct-dependent, minimal paid)
        ↓
Product loses ~80% of new visitors before they ever see a product (Section 3)
        ↓
Of those who reach the funnel, checkout mechanics work reasonably well
    39.25% cart→checkout, 43.62% checkout completion (Section 5)
        ↓
But almost no one returns 82% single-session, Day 30 retention 0.13% (Section 7)
        ↓
So total revenue depends almost entirely on continuously acquiring NEW
    first-time visitors, forever, with no retention safety net (Sections 6, 7)
        ↓
Meanwhile, the store's two best-performing segments the (data deleted)
    channel and the top 0.43% value tier are both under-leveraged,
    representing the clearest available offset to the retention gap (Sections 8, 10)
```

**Direct cross-team implications:**

- **Marketing's channel-mix decisions (Section 10) are constrained by Product's activation problem (Section 3):** driving more traffic to a homepage that only converts 20% of visitors to a product view compounds the acquisition-dependency problem rather than solving it. Marketing should not be evaluated on traffic volume alone until Product's activation rate improves the two teams' roadmaps should be sequenced, not parallel.
- **Finance's revenue reporting depends on Analytics' methodology discipline (R1–R7):** the R6 and R7 findings in this report show that even "just report the numbers" work in this dataset carries real risk of materially wrong conclusions (a false engagement-doubling story, a false 0%-converting mega-category) without careful validation. Finance-facing dashboards built on this data should be reviewed against this report's rules before being trusted at face value.
- **Growth's retention mandate (Section 7) should be re-scoped around the warm segment, not the general population:** the 11.85% repeat-purchase rate among past buyers is a fundamentally different, more tractable target than the near-zero general retention rate Growth's roadmap should reflect this distinction explicitly rather than setting one retention target for the whole user base.

---

# 14. Root Cause Analysis - Top 10 Business Problems

| # | Problem | Evidence | Impact | Root Cause | Confidence | Suggested Owner | Difficulty | Potential ROI |
|---|---|---|---|---|---|---|---|---|
| 1 | Only 19.97% of new visitors ever view a product | Q3.1 | Highest-leverage leak in the entire journey; caps every downstream metric | Homepage/navigation friction (hypothesized, not yet isolated) | High (finding) / Medium (cause) | Product | Medium | Very High |
| 2 | Checkout's steepest drop is payment-info entry (38.6% loss), not final submission | Q6.3 | Direct, quantifiable revenue loss at a specific, fixable step | Payment UX/method friction (hypothesized) | High (finding) / Medium (cause) | Product/Eng | Low–Medium | High |
| 3 | 82.47% single-session users; Day 30 retention 0.13% | Q4.4, Q7.1 | Business has no organic retention floor; revenue fully acquisition-dependent | Low-frequency purchase category (structural, not purely fixable) | High | Growth/Leadership | High (strategic) | Medium (realistic ceiling) |
| 4 | 0.43% of users drive ~62% of value | Q8.3 | Extreme, unexploited concentration | No targeted high-value program currently evidenced | High | Marketing/CRM | Medium | Very High |
| 5 | `(data deleted)` channel converts 2.8x better than next-best, but is unactionable as reported | Q10.1, Q10.2 | Best-performing segment is invisible to standard channel management | Privacy/consent-driven source redaction | High (finding) / Low (cause) | Analytics/Legal | High | High (if resolved) |
| 6 | `cpc/google` underperforms on every quality metric | Q2.10, Q10.1, Q10.4 | Inefficient spend, not just small spend | Targeting/landing-page mismatch (hypothesized) | High (finding) / Medium (cause) | Marketing | Low | Medium |
| 7 | Engagement-rate trend has an undisclosed instrumentation break (Dec 28) | Q4.6 | Risk of false "engagement improved" narrative reaching leadership | GTM/tagging configuration change | High | Analytics/Eng | Low (fix reporting) | High (risk avoidance) |
| 8 | `item_id` unreliable as a cross-event join key | Q9.5, Q9.6 | Any product analysis built on item_id silently misattributes revenue | GA4/GTM implementation inconsistency between event types | High | Analytics/Eng | Medium | High (risk avoidance) |
| 9 | Homepage carries 46% of all entrances | Q2.5, Q2.9 | Single point of failure for total traffic quality | Concentrated top-of-funnel design, no diversified entry strategy | High | Marketing/SEO | Medium | Medium |
| 10 | 43% of checkouts show no prior add-to-cart event | Q6.6 | Large, unexplained share of the purchase path | Buy-Now UX flow or add_to_cart tracking gap (unresolved) | Medium | Product/Analytics | Medium | Medium (until diagnosed) |

---

# 15. Prioritized Recommendations

**Quick Wins (immediate, low effort, evidence already in hand):**

- Standardize all financial reporting on transaction-level revenue only (R3); explicitly ban `user_ltv` sums from any dollar-figure report. *Owner: Finance/Analytics. Dependency: none. Complexity: Low.*
- Normalize homepage URL variants in every dashboard/report (Q2.9's fix). *Owner: Analytics. Dependency: none. Complexity: Low.*
- Pause incremental `cpc/google` budget increases pending a targeting review. *Owner: Marketing. Dependency: none. Complexity: Low. Expected KPI impact: prevents further spend on a sub-1% converting channel.*
- File an engineering ticket to audit and either confirm or roll back the Dec 28 tagging change (R6) before any further trend reporting is built on engagement-time data. *Owner: Analytics/Engineering. Dependency: none. Complexity: Low.*

**30-Day Roadmap:**

- Launch a homepage/navigation A/B test targeting the 19.97% first-session product-view rate (Finding #1). *Owner: Product. Dependency: UX research capacity. Complexity: Medium. Expected KPI impact: Metric 3.1 (First-Session Product View Rate) as primary; Metric 6.4 (View-to-Purchase) as downstream check.*
- Conduct a payment-step checkout UX audit (payment methods, hidden costs, form friction) targeting the 38.65% shipping→payment drop. *Owner: Product/Engineering. Dependency: checkout system access. Complexity: Medium. Expected KPI impact: Metric 6.3 (Checkout-to-Purchase granular rate).*
- Investigate the 42.91% "no cart found" checkout population via session recordings or a UX walkthrough of actual entry points. *Owner: Product/Analytics. Dependency: session-recording tooling. Complexity: Medium.*
- Fix the confirmed `item_id`/`item_category` tagging inconsistency at the source (GTM/dataLayer implementation), not just in downstream SQL. *Owner: Engineering/Analytics. Dependency: GTM access. Complexity: Medium.*

**90-Day Roadmap:**

- Design and launch a targeted remarketing/loyalty program for past purchasers, grounded in the 11.85% repeat-purchase-rate finding (Finding #4/Section 7). *Owner: Marketing/CRM. Dependency: CRM/email infrastructure. Complexity: Medium. Expected KPI impact: Metric 7.3 (Repeat Purchase Rate), Metric 8.2 (Purchase Frequency).*
- Commission a High-value-tier profiling analysis (channel, device, geography) to inform a VIP program design (Finding #4/Section 8). *Owner: Analytics/Marketing. Dependency: none, uses existing metrics framework. Complexity: Low-Medium.*
- Open a compliant, legal-reviewed investigation into the `(data deleted)` segment's underlying characteristics (Finding #5). *Owner: Analytics/Legal/Marketing. Dependency: Legal/Privacy review. Complexity: High.*
- Run a desktop-specific checkout UX audit given its higher abandonment rate relative to mobile (Section 11). *Owner: Product/Engineering. Dependency: none. Complexity: Medium.*
- Rebuild the Product Performance / merchandising report using the corrected `item_name`-based methodology (R7) rather than the flawed `item_category`/`item_id` approach. *Owner: Merchandising/Analytics. Dependency: none. Complexity: Low.*

**Long-Term Strategy:**

- Build a permanent data-transformation layer (e.g., a dbt-style pipeline) that encodes R1–R7 as enforced rules rather than tribal SQL-repository knowledge, so future analysts cannot silently reintroduce the same errors this project caught. *Owner: Data Engineering/Analytics. Complexity: High.*
- Bring a strategic decision to Leadership: given confirmed near-zero organic retention (Section 7), explicitly decide whether this store's mandate is brand-engagement (optimize for reach and one-time conversion) or a repeat-revenue business (invest heavily in retention infrastructure) the current operating pattern is closer to the former, whether or not that was a deliberate choice. *Owner: CEO/VP Product.*
- Extend geographic analysis (Section 12's acknowledged gap) as part of any international growth consideration. *Owner: Analytics.*

---

# 16. Risks & Limitations

This report's conclusions are subject to the following, explicitly stated rather than hidden:

- **This is a public, obfuscated sample dataset.** Per Google's own documentation and this project's Stage 1 scope, absolute figures (session counts, revenue totals) should not be read as literal statements about the real Google Merchandise Store's actual business performance findings are directional and methodology-driven.
- **R1 - 85.74% of events (99.7%+ of sessions) carry `debug_mode = 1`.** This was not filtered out because doing so would gut the dataset to statistical uselessness, and Q2.8 confirmed the saturation is uniform across channels (so relative comparisons are unaffected). Absolute volumes should still be read as directional.
- **R2 - Transaction IDs required cleaning.** 767+ distinct users' purchases collided under a shared `(not set)` placeholder ID; a smaller number of real numeric IDs also collided across genuinely different users. All per-transaction metrics in this report exclude these; headline aggregate revenue includes them.
- **R3 - Three different "revenue" figures exist in this dataset and do not agree by design:** transaction-level ($362,165, the source of truth), item-level ($362,110 in aggregate, but with 28.56% of individual transactions diverging), and `user_ltv` ($407,626, a confirmed 12.6% overstatement). This report uses transaction-level revenue exclusively for dollar figures.
- **R4 - Product-level view analysis is scoped to the 62.65% of `view_item` events with populated item detail;** the remaining 37.35% are real events with missing product-level detail, not missing traffic.
- **R5 - A small number of engagement-time outliers (10–19+ hours on a single event, from background browser tabs) were capped at 1 hour before any averaging.**
- **R6 - Engagement-rate and engagement-time trend data cannot be compared before and after Dec 28, 2020** due to a confirmed instrumentation change; this report treats the two periods as methodologically distinct wherever engagement trends are discussed.
- **R7 - `item_id` cannot be used as a join key across event types in this dataset;** `item_name` was used instead for all product-level analysis in Section 9, with the acknowledged trade-off that `item_name` is not a guaranteed-unique identifier the way `item_id` nominally should be.
- **No refund events exist in this dataset** this is reported as an absence of tracking, not a confirmed 0% refund rate.
- **Geographic analysis (Section 12) could not be performed** no country/region-level query was executed in this project's scope; this is an acknowledged gap, not a finding of "no geographic variation."
- **Several root-cause hypotheses in this report (Sections 3, 5, 11, 14) are explicitly marked as Medium or Low confidence** they explain *what* the data shows but not definitively *why*, and require UX research, session recordings, or further targeted analysis (not more SQL alone) to confirm.

---

# 17. Executive Action Plan - One Quarter to Improve This Business

**Week 1–2:**
- Analytics/Engineering: resolve or formally document the R6 tagging discontinuity and the R7 item_id join failure; freeze any dashboard currently built on the flawed versions (raw engagement-rate trend, `item_category`-based product reports).
- Finance: adopt R3 as a standing reporting rule transaction-level revenue only for dollar figures.
- Marketing: pause incremental `cpc/google` spend pending review.

**Week 3–4:**
- Product: scope and launch the homepage/navigation activation A/B test (targeting Metric 3.1).
- Product/Engineering: begin the payment-step checkout UX audit.
- Analytics: deliver the High-value-tier profiling analysis (Section 8) to inform the upcoming loyalty program design.

**Month 2:**
- Marketing/CRM: design and begin building the past-purchaser remarketing/loyalty program.
- Product/Analytics: complete the "no cart found" checkout-origin investigation (Q6.6 open item) via session recordings.
- Analytics/Legal: initiate the `(data deleted)` segment investigation.
- Merchandising: rebuild the category performance report using the corrected R7 methodology and use it to inform Q2 assortment/placement decisions.

**Month 3:**
- Product: implement whichever homepage/navigation variant won the Week 3-4 A/B test; begin the desktop checkout UX audit.
- Marketing/CRM: launch the past-purchaser loyalty program.
- Leadership: hold the strategic "brand-engagement vs. repeat-revenue business" decision meeting (Section 15, Long-Term Strategy), using this quarter's early A/B and loyalty-program results as evidence.
- Data Engineering: begin scoping the permanent R1-R7 transformation-layer project for the following quarter.

---

# 18. Appendix

**Methodology rules referenced throughout this report:**
- R1 - debug_mode traffic included, not filtered (§6.4 Data Validation; confirmed uniform across channels, Q2.8)
- R2 - transaction_id cleaned: exclude `(not set)`, dedupe by (transaction_id, user_pseudo_id) (§4.2 Data Validation)
- R3 - transaction-level revenue is sole source of truth; item-level and user_ltv never reconciled to it (§4.3 Data Validation; quantified in Q11.2)
- R4 - view_item/begin_checkout product-level cuts scoped to items-populated rows only (§5.2 Data Validation)
- R5 - engagement_time_msec capped at 3,600,000ms before averaging (§6.5 Data Validation)
- R6 - engagement-time metrics not comparable pre/post Dec 28, 2020 (confirmed via Q4.6)
- R7 - item_id unreliable as cross-event-type join key; use item_name instead (confirmed via Q9.5/Q9.6)

**Key metrics referenced:** Metric 1 (Weekly Transacting Users, North Star), 2.1-2.5 (Sessions, New Users, New User Rate, Channel Mix, Landing Page Entrance Rate), 3.1-3.3 (Activation), 4.1-4.4 (Engagement), 5.1-5.5 (Commerce), 6.1–6.5 (Funnel), 7.1-7.4 (Retention), 8.1-8.3 (Customer), 9.1-9.4 (Product), 10.1-10.4 (Marketing) full definitions in `04_Metrics_Framework.md`.

**Key SQL queries referenced:** Q1.1-Q1.3 (North Star), Q2.1-Q2.10 (Acquisition), Q3.1-Q3.3 (Activation), Q4.1-Q4.6 (Engagement), Q5.1-Q5.5 (Commerce), Q6.1-Q6.6 (Funnel), Q7.1-Q7.4 (Retention), Q8.1-Q8.3 (Customer), Q9.1-Q9.6 (Product, including the R7 correction), Q10.1-Q10.4 (Marketing), Q11.1-Q11.3 (Executive/Cross-Validation) full queries and expected-vs-actual output in `05_SQL_Query_Repository.md`.

**Document lineage:** `01_Project_Overview.md` → `02_Data_Understanding.md` → `03_Data_Validation.md` → `04_Metrics_Framework.md` → `05_SQL_Query_Repository.md` → this document. Every finding above traces to a specific query ID and, where applicable, a specific methodology rule no figure in this report was estimated or invented.
