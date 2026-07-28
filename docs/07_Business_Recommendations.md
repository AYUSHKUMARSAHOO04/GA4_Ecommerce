# 07 : Strategic Business Recommendations Report

**Prepared for:** CEO, VP Product, CMO, Head of Growth, Finance, Product Leadership
**Prerequisite:** This document assumes `06_Analysis.md` has been read. It does not re-explain findings it converts them into decisions.
**The question this document answers:** What should the business do next, in what order, owned by whom, measured how?

Throughout this document, recommendations are referenced by code (**BR1-BR13**) so they can be traced consistently across every matrix below.

---

# 1. Executive Recommendation Summary

This business does not need a turnaround. It needs three specific interventions and one disciplined governance fix, applied in the right order.

**Top priorities, in sequence:**
1. **Lock down data integrity before acting on anything else (BR7).** Two confirmed measurement issues (a tagging break, an unreliable join key) mean some existing dashboards are currently wrong. Fixing this is a precondition for every other recommendation being trustworthy.
2. **Fix the homepage/navigation activation gap (BR1).** This is the single highest-leverage lever available four out of five new visitors never reach a product page, which caps the return on every other investment in this document.
3. **Fix the checkout payment step (BR2).** The most specific, cheapest-to-fix leak in the revenue path.
4. **Activate the two segments already proven to work disproportionately well (BR3, BR4, BR5)** past purchasers, the top value tier, and the unexplained high-converting traffic segment rather than continuing to spend acquisition budget at the current blended average efficiency.

**Business outlook:** This is a structurally acquisition-dependent business (near-zero organic retention) that is nonetheless leaving two proven, high-return pools of value under-activated. The near-term opportunity is not "grow traffic" it's "convert and retain more of what already arrives."

**Immediate actions this quarter:** Freeze reporting built on the two confirmed data-integrity issues; commission the homepage activation test; fix the checkout payment step; begin the high-value-tier and `(data deleted)` segment investigations in parallel, since both are analysis-only work that can start immediately without engineering dependencies.

**Strategic direction:** Shift the operating model from "acquire continuously and accept the funnel as-is" to "convert a larger share of existing traffic, then deliberately cultivate the two segments already shown to outperform the blended average by 2-3x." This is a sequencing and focus decision, not a request for new investment categories.

---

# 2. Executive Priority Matrix

| Priority | Recommendation | Problem | Evidence | Business Impact | Owner | Expected KPI |
|---|---|---|---|---|---|---|
| **Critical** | BR7 - Data & Reporting Integrity Lockdown | Two confirmed measurement defects (engagement-rate tagging break; item_id join failure) risk materially wrong conclusions reaching Finance/Merchandising | `06_Analysis.md` §4, §9 (R6, R7) | Prevents false narratives (a fake "engagement doubled," a fake "0%-converting mega-category") from driving real budget decisions | Analytics/Engineering | Reporting accuracy (no direct KPI; a governance precondition) |
| **Critical** | BR1 - Homepage & Navigation Activation Redesign | Only 19.97% of first-time visitors ever view a product | `06_Analysis.md` §3 | Highest-leverage lever in the business; caps every downstream funnel and revenue metric | Product | Metric 3.1 (First-Session Product View Rate) |
| **Critical** | BR2 - Checkout Payment-Step UX Fix | Steepest checkout drop is shipping→payment (38.65% loss), not final submission | `06_Analysis.md` §5 | Direct, specific, quantifiable revenue recovery at an identified stage | Product/Engineering | Metric 6.3 (Checkout-to-Purchase, granular) |
| **High** | BR3 - Past-Purchaser Remarketing/Loyalty Program | Repeat purchase rate among past buyers (11.85%) is far above general retention (~0%) | `06_Analysis.md` §7 | Converts an already-proven warm segment into recurring revenue | Marketing/CRM | Metric 7.3, Metric 8.2 |
| **High** | BR4 - High-Value Tier Profiling & VIP Program | 0.43% of users drive ~62% of measured customer value | `06_Analysis.md` §8 | Protects and potentially grows the store's most concentrated value pool | Analytics/Marketing | Metric 8.3 (tier composition and value share) |
| **High** | BR5 - `(data deleted)` Segment Investigation | Best-converting segment (3.14%, 2.8x next-best) is currently unactionable due to source redaction | `06_Analysis.md` §10 | Understanding this segment could unlock replicable, high-return targeting | Analytics/Legal/Marketing | Metric 10.1, Metric 10.2 |
| **High** | BR6 - Paid Search Targeting Overhaul | `cpc/google` underperforms on engagement, conversion, and revenue share simultaneously | `06_Analysis.md` §2, §10 | Redirects inefficient spend toward measurable return | Marketing | Metric 10.1, Metric 10.4 |
| **Medium** | BR8 - Landing Page Diversification | Homepage carries 45.96% of all entrances — a concentration risk | `06_Analysis.md` §2 | Reduces single-point-of-failure exposure on total traffic quality | Marketing/SEO | Metric 2.5 |
| **Medium** | BR9 - Checkout-Origin ("No Cart Found") Investigation | 42.91% of checkouts show no detectable prior add-to-cart | `06_Analysis.md` §5 | Resolves a large, currently unexplained share of the purchase path | Product/Analytics | Metric 6.4/6.6 (diagnostic, not a KPI target itself) |
| **Medium** | BR10 - Desktop Checkout UX Audit | Desktop shows higher cart abandonment (81.73%) than mobile (80.58%), inverting the default assumption | `06_Analysis.md` §11 | Desktop carries the largest session share, so even a small fix has outsized absolute impact | Product/Engineering | Metric 6.5 |
| **Medium** | BR13 - Corrected Merchandising Reporting Rollout | Category performance reporting was previously built on a flawed methodology (R7) | `06_Analysis.md` §9 | Ensures merchandising/assortment decisions use trustworthy category data | Merchandising/Analytics | Metric 9.3/9.4 (corrected basis) |
| **Strategic (see §5)** | BR12 - Business Model Framing Decision | Near-zero organic retention (Day 30: 0.13%) is a structural, not purely fixable, characteristic | `06_Analysis.md` §7 | Determines whether future investment optimizes for brand reach or repeat revenue | CEO/VP Product | Sets the frame for all retention-related KPI targets |

---

# 3. Quick Wins (0–30 Days)

### QW1 - Data & Reporting Integrity Lockdown
**Business Problem:** Engagement-rate trend data has an undisclosed break at Dec 28, 2020 (confirmed tagging change); product-category reporting built on `item_id` silently misattributes revenue.
**Supporting Evidence:** `06_Analysis.md` §4 (R6), §9 (R7).
**Affected KPIs:** All engagement-time-based metrics (4.1, 4.2); all product/category revenue metrics (9.3, 9.4).
**Expected Improvement:** Not a KPI lift a correctness fix. Prevents at least two materially wrong conclusions from reaching decision-makers.
**Implementation Steps:** (1) Freeze any dashboard showing a continuous pre/post-Dec-28 engagement trend line; (2) audit the GTM/tagging change directly with Engineering to confirm intent; (3) reissue all product/category reports using the `item_name`-based join (already built, Q9.6) in place of `item_id`.
**Business Owner:** Analytics, with Engineering support.
**Implementation Complexity:** Low.
**Risk Level:** Low (the fix is a reporting correction, not a system change).
**Priority:** Critical.
**Success Metrics:** Zero dashboards presenting the flawed pre/post engagement trend as continuous; zero category reports using `item_id`-based joins.

### QW2 - Pause Incremental Paid Search Spend
**Business Problem:** `cpc/google` converts below the store's free channels (0.98% vs. organic's 1.11%) and shows the lowest first-session engagement rate of any real channel (70.11%).
**Supporting Evidence:** `06_Analysis.md` §2, §10.
**Affected KPIs:** Metric 10.1 (Channel Conversion Rate), Metric 5.5 (Revenue per Session, blended).
**Expected Improvement:** Avoids further spend on the store's least-efficient channel while BR6's full review is scoped.
**Implementation Steps:** (1) Freeze any planned paid-search budget increase; (2) hold current spend flat, not cut to zero, pending the Month 2 targeting review (BR6) so the channel isn't starved before it's properly diagnosed.
**Business Owner:** Marketing.
**Implementation Complexity:** Low.
**Risk Level:** Low.
**Priority:** Critical.
**Success Metrics:** No incremental `cpc/google` spend approved until BR6's review concludes.

### QW3 - Landing Page Reporting Normalization
**Business Problem:** Homepage traffic is fragmented across three URL variants in raw reporting, understating true homepage dependency (45.96% combined vs. any single row).
**Supporting Evidence:** `06_Analysis.md` §2.
**Affected KPIs:** Metric 2.5 (Landing Page Entrance Rate).
**Expected Improvement:** Accurate visibility into true homepage concentration, informing the urgency of BR8.
**Implementation Steps:** Apply the existing normalization logic to every landing-page report immediately; no new analysis required.
**Business Owner:** Analytics.
**Implementation Complexity:** Low.
**Risk Level:** Low.
**Priority:** High.
**Success Metrics:** All landing-page dashboards show a single, consolidated "Homepage" row.

### QW4 - High-Value Tier Profiling Analysis
**Business Problem:** The 0.43% of users driving ~62% of value have not been profiled by channel, device, or geography any VIP program built without this would be guessing at the segment's defining characteristics.
**Supporting Evidence:** `06_Analysis.md` §8.
**Affected KPIs:** Metric 8.3 (Value Tier Segmentation), informing future Metric 7.3/8.2 targets.
**Expected Improvement:** A concrete customer profile ready to inform BR4's Month 2 program design no new data collection required, only additional segmentation of existing data.
**Implementation Steps:** Cut the existing High-value tier by acquisition channel, device category, and geography using the metrics framework already defined in `04_Metrics_Framework.md`.
**Business Owner:** Analytics.
**Implementation Complexity:** Low.
**Risk Level:** Low.
**Priority:** High.
**Success Metrics:** A documented High-value customer profile delivered to Marketing before Month 2 begins.

### QW5 - Corrected Merchandising Report Rollout
**Business Problem:** The prior category-performance view showed a mega-category at 0% conversion an artifact, not a real finding which could have driven a wrong assortment decision.
**Supporting Evidence:** `06_Analysis.md` §9.
**Affected KPIs:** Metric 9.3/9.4 (Product Revenue Share, Category Performance Index).
**Expected Improvement:** Merchandising decisions this quarter are based on the corrected view (Apparel $168,987, the confirmed largest category, and accurate category-level view-to-cart rates).
**Implementation Steps:** Distribute the corrected, `item_name`-based category report (already built) directly to Merchandising; retire the flawed version.
**Business Owner:** Merchandising/Analytics.
**Implementation Complexity:** Low.
**Risk Level:** Low.
**Priority:** Medium.
**Success Metrics:** Merchandising's next assortment/placement review cites the corrected figures.

---

# 4. Medium-Term Roadmap (30–90 Days)

### MT1 - Homepage & Navigation Activation Redesign (BR1)
**Objective:** Increase the share of first-time visitors who reach a product page, from a confirmed baseline of 19.97%.
**Evidence:** `06_Analysis.md` §3.
**Business Value:** This is the single highest-leverage initiative in this document every downstream funnel, revenue, and retention metric is capped by how many people ever see a product.
**Dependencies:** UX research capacity; A/B testing infrastructure (flagged as a capability gap in §12 below if not already in place).
**Risks:** A redesign that increases product views without addressing the underlying navigation problem could inflate Metric 3.1 without improving Metric 6.4 (View-to-Purchase) success criteria must include the downstream funnel rate, not just the activation rate in isolation.
**Timeline:** Design and launch A/B test by Month 2; read results by Month 3.
**KPIs:** Metric 3.1 (primary), Metric 6.4 (secondary/validation).
**Expected Outcome:** A measurable lift in first-session product-view rate that also shows through in overall view-to-purchase conversion, confirming the fix addressed the real bottleneck rather than a symptom of it.

### MT2 - Checkout Payment-Step UX Overhaul (BR2)
**Objective:** Reduce the 38.65% session loss between shipping-info entry and payment-info entry.
**Evidence:** `06_Analysis.md` §5.
**Business Value:** A direct, specific revenue-recovery lever at an already-identified stage the cheapest fix in this document relative to its potential return, since the problem is localized to one step, not the whole funnel.
**Dependencies:** Checkout system/payment-provider access; UX audit capacity.
**Risks:** Root cause is currently a hypothesis (payment method limitations, hidden costs, form friction) implementation should begin with a diagnostic UX walkthrough, not a blind redesign, or effort may be spent fixing the wrong sub-cause.
**Timeline:** Diagnostic audit in Month 1–2; fix implementation and A/B test in Month 2-3.
**KPIs:** Metric 6.3 (granular checkout completion rate).
**Expected Outcome:** A narrower gap between the shipping-info and payment-info stages, closing part of the 38.65% loss.

### MT3 - Paid Search Targeting & Landing Page Overhaul (BR6)
**Objective:** Bring `cpc/google`'s conversion and engagement rates at least to parity with organic search before considering any spend increase.
**Evidence:** `06_Analysis.md` §2, §10.
**Business Value:** Converts an underperforming spend category into an efficient one, rather than simply cutting it preserves the acquisition volume it provides (5.43% of new users) while fixing its economics.
**Dependencies:** Marketing analytics/campaign-management access; landing-page design resources (likely shared with MT1).
**Risks:** Cutting paid search entirely (rather than fixing it) would reduce new-user volume without addressing whether a well-targeted paid program could work the recommendation is a targeting fix, not elimination.
**Timeline:** Campaign/audience review in Month 1–2; landing-page-specific test in Month 2–3.
**KPIs:** Metric 10.1 (Channel Conversion Rate), Metric 2.10 (First-Session Engagement Rate by channel).
**Expected Outcome:** `cpc/google` conversion rate closing the gap with organic google (currently 0.98% vs. 1.11%).

### MT4 - Checkout-Origin ("No Cart Found") Investigation (BR9)
**Objective:** Determine whether the 42.91% of checkouts with no detectable prior add-to-cart reflects a "Buy Now" UX pattern or a tracking gap.
**Evidence:** `06_Analysis.md` §5.
**Business Value:** Resolves genuine ambiguity in how a large share of purchases actually happen either confirms a UX pattern worth reinforcing, or surfaces a measurement gap affecting every funnel metric that assumes cart-add precedes checkout.
**Dependencies:** Session-recording or heatmap tooling (flagged as a capability gap in §12 if not currently available).
**Risks:** If this reveals a tracking gap rather than a UX pattern, funnel metrics reported throughout `06_Analysis.md` (Metrics 6.1, 6.2) may need to be re-scoped once resolved.
**Timeline:** Investigation in Month 2; findings and any resulting instrumentation fix by Month 3.
**KPIs:** Diagnostic informs Metric 6.4/6.6 interpretation rather than targeting a specific KPI movement itself.
**Expected Outcome:** A confirmed explanation for this population, with either a validated "Buy Now" flow to formally support or a tracking fix scoped for Engineering.

### MT5 - Desktop Checkout UX Audit (BR10)
**Objective:** Understand and reduce desktop's higher cart-abandonment rate (81.73%) relative to mobile (80.58%).
**Evidence:** `06_Analysis.md` §11.
**Business Value:** Desktop carries the largest session share (58.16%), so even a modest improvement here has a larger absolute-dollar impact than a mobile-only initiative would.
**Dependencies:** Session-recording/heatmap tooling (shared dependency with MT4).
**Risks:** Root cause is currently unconfirmed (form friction vs. audience composition vs. distraction) avoid defaulting to a "mobile-first redesign," which the data does not support as the priority.
**Timeline:** Audit in Month 2; fix scoping by Month 3.
**KPIs:** Metric 6.5 (Cart Abandonment Rate by device).
**Expected Outcome:** Desktop abandonment rate moving toward, or below, mobile's.

### MT6 - Past-Purchaser Remarketing/Loyalty Program v1 (BR3)
**Objective:** Convert the proven 11.85% repeat-purchase-rate signal into an active program rather than an organic-only pattern.
**Evidence:** `06_Analysis.md` §7.
**Business Value:** Targets the one segment already shown to behave meaningfully better than the general population a far more tractable target than improving general-population retention (which Section 5's strategic framing question addresses separately).
**Dependencies:** CRM/email infrastructure; creative development.
**Risks:** A generic "come back" campaign aimed at all past purchasers, rather than segmented by value tier (BR4), risks diluting the return this program should be sequenced after or alongside QW4's profiling work.
**Timeline:** Program design Month 2; launch Month 3.
**KPIs:** Metric 7.3 (Repeat Purchase Rate), Metric 8.2 (Purchase Frequency).
**Expected Outcome:** A measurable increase in repeat-purchase rate among the targeted cohort versus a held-out control group.

### MT7 — `(data deleted)` Segment Investigation (BR5)
**Objective:** Identify the underlying characteristics of the store's best-converting traffic segment, currently unactionable due to source redaction.
**Evidence:** `06_Analysis.md` §10.
**Business Value:** If this segment's defining trait (e.g., a loyalty list, logged-in status, a specific email campaign) can be identified through compliant means, it represents the highest-ROI targeting opportunity identified in this entire engagement 2.8x the conversion rate of the next-best channel.
**Dependencies:** Legal/Privacy review before any investigative work touching consent-redacted data.
**Risks:** This is the highest-risk item in this roadmap from a compliance standpoint must not proceed without Legal sign-off, and any finding must be actioned only through consent-compliant channels.
**Timeline:** Legal scoping Month 1; investigation Month 2-3, contingent on approval.
**KPIs:** Metric 10.1/10.2, if and when a compliant activation path is identified.
**Expected Outcome:** Either a compliant path to understanding and partially replicating this segment's behavior, or a documented conclusion that it cannot be further investigated both are valid, informative outcomes.

---

# 5. Long-Term Strategy (3–12 Months)

**Retention - Strategic Business Model Framing Decision (BR12).** Day 30 retention (0.13%) reflects a low-frequency-purchase product category as much as a fixable defect. Leadership should make an explicit choice: continue operating as a brand-engagement vehicle (optimize acquisition and one-time conversion, accept low retention as a category norm) or invest deliberately in becoming a repeat-revenue business (loyalty infrastructure, subscription/replenishment mechanics, CRM maturity). This decision should be made **after** seeing early results from MT6's loyalty pilot, not before the pilot is the evidence base for this decision, not a substitute for it.

**Customer Value - Scaled VIP Program (extension of BR4).** Once QW4's profiling and MT6's pilot are complete, formalize a durable high-value-customer program (tiered benefits, dedicated service, early access) targeting the 0.43% segment shown to drive ~62% of value this is a 6-12 month build, not a 30-day one, but should begin design immediately after QW4 delivers its profile.

**Product Growth - Personalization Roadmap.** Product-level conversion-rate spread (16.4%–31.1% across similar apparel items, per `06_Analysis.md` §9) suggests real opportunity in matching the right product to the right visitor a homepage/PDP personalization layer building on MT1's activation fix and the corrected product data from QW5.

**Data Platform - Permanent R1-R7 Enforcement Layer.** Convert the seven methodology rules discovered during this engagement (debug-traffic handling, transaction cleaning, revenue source-of-truth, item coverage scoping, outlier capping, the R6 tagging-break caveat, the R7 join-key fix) from documented tribal knowledge into an enforced data-transformation layer (e.g., a dbt project), so future analysts cannot silently reintroduce the same errors this engagement caught and corrected.

**Analytics Maturity - Formal Experimentation Program.** MT1, MT2, MT3, and MT6 all depend on properly measured A/B tests. If this capability doesn't already exist at production quality, building it is a prerequisite investment underlying nearly every other recommendation in this document.

**Marketing Maturity - Channel Portfolio Strategy.** Once MT3 resolves whether paid search can be made efficient, build a deliberate channel-investment framework (which channels to grow, hold, or fix) rather than the current default of organic-dependent acquisition with a small, currently underperforming paid layer.

**Executive Decision-Making - Institutionalize the R1–R7 Discipline.** The most transferable lesson from this engagement is procedural: multiple findings in `06_Analysis.md` (the false engagement-doubling trend, the false 0%-converting category) were only caught because of deliberate validation before reporting. Leadership should require this standard validate before trusting a metric as a permanent expectation of the Analytics function, not a one-time engagement artifact.

---

# 6. Department-Wise Action Plan

| Department | Objectives | Responsibilities | KPIs | Deliverables |
|---|---|---|---|---|
| **Product** | Fix activation and checkout leaks; resolve device/checkout-origin ambiguity | Own MT1, MT2, MT4, MT5 | Metric 3.1, 6.3, 6.4, 6.5 | Homepage A/B test, checkout UX fix, device/checkout-origin diagnostic reports |
| **Marketing** | Fix paid search efficiency; reduce landing-page concentration risk; activate warm segments | Own QW2, QW3, BR8, MT3, co-own MT6/MT7 | Metric 10.1, 10.4, 2.5, 7.3 | Paid search targeting review, landing-page diversification plan, loyalty program creative |
| **Growth** | Convert past-purchaser signal into a program; own the strategic retention framing input | Co-own MT6; feed evidence into BR12 | Metric 7.3, 8.2 | Loyalty program v1 launch and results readout |
| **Analytics** | Lock down data integrity; deliver segmentation profiling; maintain the metrics/query repository | Own QW1, QW4; support MT4, MT5, MT7; own long-term BR7 data-platform build | Reporting accuracy; Metric 8.3 profile delivery | Integrity lockdown confirmation, High-value profile, permanent transformation-layer scoping doc |
| **Engineering** | Support checkout/payment fixes; resolve any confirmed tracking gaps; support the data-platform build | Support MT2, MT4 (if tracking gap confirmed); support long-term BR7 platform work | Metric 6.3, 6.4/6.6 (post-fix) | Checkout fix implementation, any instrumentation corrections |
| **Finance** | Adopt corrected revenue reporting standards; monitor AOV/transaction-quality metrics | Adopt R3 as standing policy (transaction-level revenue only); monitor the 14.7%-of-revenue transaction-cleaning impact | Metric 5.1, 5.2 | Updated financial reporting template reflecting R2/R3 rules |
| **Leadership** | Make the strategic business-model framing decision; sequence and fund the roadmap | Own BR12 decision; approve MT7's Legal review; sponsor the quarterly roadmap | North Star Metric (Weekly Transacting Users) | Q2 strategic framing decision, approved roadmap and budget |

---

# 7. KPI Impact Matrix

| Recommendation | Affected KPI | Expected Direction | Confidence | Expected Business Impact |
|---|---|---|---|---|
| BR1 - Homepage Activation Redesign | Metric 3.1 (First-Session Product View Rate) | ↑ | High | Highest-leverage lift in the roadmap; compounds through every downstream metric |
| BR1 - Homepage Activation Redesign | Metric 6.4 (View-to-Purchase) | ↑ (secondary) | Medium | Validates whether the fix addressed the real bottleneck |
| BR2 - Checkout Payment-Step Fix | Metric 6.3 (Checkout-to-Purchase, granular) | ↑ | Medium-High | Direct, localized revenue recovery |
| BR3 - Loyalty Program | Metric 7.3 (Repeat Purchase Rate) | ↑ | Medium | Converts a proven warm-segment signal into recurring revenue |
| BR3 - Loyalty Program | Metric 8.2 (Purchase Frequency) | ↑ | Medium | Increases value per existing customer |
| BR4 - VIP Program | Metric 8.3 (Value tier composition/share) | Value share retained or grown | Medium | Protects the concentrated ~62% value pool from churn |
| BR5 - `(data deleted)` Investigation | Metric 10.1/10.2 (if activation path found) | ↑ (conditional) | Low (contingent on Legal approval and findings) | Highest potential ROI in this document, but least certain to be actionable |
| BR6 - Paid Search Overhaul | Metric 10.1 (Channel Conversion Rate, cpc/google) | ↑ | Medium | Converts inefficient spend into efficient spend |
| BR8 - Landing Page Diversification | Metric 2.5 (Landing Page Entrance Rate distribution) | Distribution broadens | Medium | Reduces single-point-of-failure risk, not a direct revenue KPI |
| BR9 - Checkout-Origin Investigation | Metric 6.4/6.6 interpretation | Clarified, not directional | High (as a diagnostic) | Resolves ambiguity affecting multiple funnel metrics |
| BR10 - Desktop Checkout Audit | Metric 6.5 (Cart Abandonment, desktop) | ↓ | Medium | Largest session-share device, so outsized absolute impact per point improved |
| BR13 - Corrected Merchandising Reporting | Metric 9.3/9.4 (accuracy) | N/A (correctness, not direction) | High | Prevents a wrong assortment decision from a flawed prior report |
| BR7 - Data Integrity Lockdown | All engagement-time and product-category metrics | N/A (correctness, not direction) | High | Precondition for trusting every other KPI movement in this matrix |

---

# 8. ROI vs. Effort Matrix

**High ROI / Low Effort:**
- **QW1 (Data Integrity Lockdown)** : No new investment required, prevents costly wrong decisions.
- **QW3 (Landing Page Normalization)** : A reporting fix, immediately informs BR8's urgency.
- **QW4 (High-Value Tier Profiling)** : Uses existing data and the existing metrics framework; directly informs BR4's design.
- **QW5 (Corrected Merchandising Report)** : The query already exists (Q9.6); this is a distribution fix, not new analysis.
- **BR2 (Checkout Payment-Step Fix)** : A single, localized UX fix with a specific, quantified problem already identified.

**High ROI / High Effort:**
- **BR1 (Homepage Activation Redesign)** : The single highest-leverage recommendation in this document, but a full navigation/homepage redesign is a genuine, multi-week Product/Design/Engineering effort.
- **BR3 (Loyalty Program)** : High expected value given the proven repeat-purchase signal, but requires CRM infrastructure and creative development to build properly.
- **BR4 (VIP Program, full build)** : High potential given the 62% value concentration, but a durable tiered program is a substantial, multi-month build.
- **BR7 (Permanent Data Platform, long-term)** : High value in preventing future errors, but a real engineering investment (dbt-style transformation layer).

**Low ROI / Low Effort (do, but don't over-invest):**
- **QW2 (Pause Paid Search Spend)**, easy to do, but a pause alone doesn't create value; it only stops further loss until BR6 is resolved.
- **BR13's ongoing maintenance**, once QW5 is delivered, keeping it current is low-effort, low-marginal-value upkeep.

**Low ROI / High Effort (deprioritize relative to the above):**
- **BR5 (`(data deleted)` Investigation) if Legal does not approve a path forward** this item carries genuine upside (see §7) but also genuine risk of high effort (Legal review, technical investigation) for zero actionable outcome if compliance constraints block any activation path. This is why BR5 is sequenced as an investigation with an explicit "may conclude as non-actionable" outcome, not a guaranteed program.
- **A full geographic-expansion analysis before the acknowledged data gap (see §12) is closed** would require new query development with no existing groundwork, for a currently unvalidated opportunity.

---

# 9. Risk Assessment

**Business Risks:**
- Continuing to fund acquisition at the current blended efficiency while activation (BR1) remains unfixed wastes a portion of every incremental marketing dollar new visitors added on top of a 19.97% product-view rate mostly churn immediately, regardless of channel quality.
- Over-indexing on the strategic framing decision (BR12) without first piloting BR3 risks a leadership decision made on hypothesis rather than evidence.

**Technical Risks:**
- MT2 and MT4/MT5 depend on session-recording or heatmap tooling; if this capability doesn't currently exist, timelines in Section 4 will need to shift to account for tooling procurement.
- Any instrumentation fix arising from MT4 (if a genuine tracking gap is confirmed) will require re-validating Metrics 6.1/6.2/6.4, since they were built on the current, potentially incomplete, add-to-cart tracking.

**Data Risks:**
- BR7's lockdown is necessary specifically because R6 and R7 are *confirmed, evidenced* issues, not hypothetical ones proceeding with any further analysis before this lockdown risks compounding the same errors into new reports.
- The `(not set)`/cross-user transaction ID collision issue (R2) continues to remove 14.7% of revenue from per-transaction visibility; this should be monitored, not just cleaned around, in case its rate changes over time.

**Measurement Risks:**
- Refund tracking does not exist as a dedicated event in this dataset (confirmed absence, not a 0% rate) any future claim about product quality or satisfaction cannot rely on refund data and must use an alternative signal.
- Geographic analysis (flagged in `06_Analysis.md` §12 and again in §12 below) has no existing query coverage any geographic claim made before this gap is closed is unsupported.

**Operational Risks:**
- BR5's Legal/Privacy dependency means this initiative's timeline is the least controllable in this roadmap Marketing and Analytics should plan MT7 as a parallel, not blocking, workstream relative to the rest of Section 4.
- Running MT1, MT2, MT3, MT4, and MT5 simultaneously (as Section 4 suggests) requires real UX/Product/Engineering bandwidth if resourcing is constrained, MT1 and MT2 should be prioritized over MT4/MT5, consistent with the Critical/High priority tiers in Section 2.

**Mitigation Plan:** Sequence Critical-tier items (BR7, BR1, BR2) before High-tier items; treat BR5 as a parallel, non-blocking investigation; require every A/B test in Section 4 to report both its primary KPI and at least one downstream validation metric (as specified in MT1's risk note), so a superficial win isn't mistaken for a real fix.

---

# 10. Success Measurement Framework

| Recommendation | Primary KPI | Secondary KPI | Measurement Frequency | Target Direction | Review Cadence |
|---|---|---|---|---|---|
| BR1 - Homepage Activation | Metric 3.1 | Metric 6.4 | Weekly during test | ↑ | Bi-weekly during test; monthly post-launch |
| BR2 - Checkout Payment Fix | Metric 6.3 (granular) | Metric 5.2 (AOV, watch for unintended shifts) | Weekly | ↑ | Bi-weekly |
| BR3 - Loyalty Program | Metric 7.3 | Metric 8.2 | Monthly | ↑ | Monthly |
| BR4 - VIP Program | Metric 8.3 (value share retained) | Metric 8.1 (LTV snapshot, purchaser-scoped) | Quarterly | Value share stable or growing | Quarterly |
| BR5 - `(data deleted)` Investigation | Metric 10.1/10.2 (contingent) | N/A until a path is confirmed | Ad hoc, dependent on Legal timeline | ↑ (if actionable) | At each Legal/investigation milestone |
| BR6 - Paid Search Overhaul | Metric 10.1 (cpc/google) | Metric 2.10 (engagement by channel) | Weekly during test | ↑ toward organic parity | Bi-weekly |
| BR8 - Landing Page Diversification | Metric 2.5 (distribution) | Metric 2.1 (session resilience) | Monthly | Reduced homepage concentration | Monthly |
| BR9 - Checkout-Origin Investigation | Diagnostic resolution (not a KPI) | Metric 6.4/6.6 (re-baselined if needed) | One-time, at investigation close | Ambiguity resolved | At investigation close |
| BR10 - Desktop Checkout Audit | Metric 6.5 (desktop) | Metric 5.5 (desktop RPS) | Weekly during test | ↓ (abandonment) | Bi-weekly |
| BR7 - Data Integrity Lockdown | Reporting accuracy (pass/fail, not a KPI) | N/A | Immediate, then ongoing spot-checks | 100% compliance | Monthly spot-check |

---

# 11. Executive Quarterly Roadmap

**Month 1:**
- **Owner:** Analytics/Engineering - **Deliverable:** QW1 Data Integrity Lockdown complete. **KPIs:** Reporting accuracy. **Dependencies:** None. **Expected Result:** All dashboards corrected or flagged.
- **Owner:** Marketing - **Deliverable:** QW2 paid-search spend freeze; QW3 landing-page reporting normalized. **KPIs:** Metric 10.1, 2.5. **Dependencies:** None. **Expected Result:** No further inefficient spend; accurate homepage-concentration visibility.
- **Owner:** Analytics - **Deliverable:** QW4 High-value tier profile; QW5 corrected merchandising report distributed. **KPIs:** Metric 8.3, 9.3/9.4. **Dependencies:** None. **Expected Result:** Both feed directly into Month 2 program design.
- **Owner:** Product - **Deliverable:** MT1 and MT2 design/scoping begins. **Dependencies:** UX research capacity.
- **Owner:** Leadership - **Deliverable:** Approve MT7's Legal review request. **Dependencies:** Legal availability.

**Month 2:**
- **Owner:** Product - **Deliverable:** MT1 homepage A/B test launched; MT2 checkout UX audit complete, fix in development. **KPIs:** Metric 3.1, 6.3. **Dependencies:** Month 1 scoping. **Expected Result:** Live test data on activation; a scoped payment-step fix.
- **Owner:** Marketing - **Deliverable:** MT3 paid-search targeting review complete. **KPIs:** Metric 10.1. **Dependencies:** QW2's freeze holding. **Expected Result:** A revised targeting/landing-page plan for `cpc/google`.
- **Owner:** Product/Analytics - **Deliverable:** MT4 and MT5 diagnostics underway (session recordings). **Dependencies:** Tooling availability (flagged risk, §9). **Expected Result:** Preliminary findings on checkout-origin and desktop abandonment causes.
- **Owner:** Marketing/CRM - **Deliverable:** MT6 loyalty program design, informed by QW4. **Dependencies:** QW4 delivered. **Expected Result:** Program ready for Month 3 launch.
- **Owner:** Analytics/Legal - **Deliverable:** MT7 investigation underway, contingent on Month 1 approval. **Expected Result:** Either a compliant path forward or a documented non-actionable conclusion.

**Month 3:**
- **Owner:** Product - **Deliverable:** MT1 test results read and winning variant implemented; MT2 fix launched and measured. **KPIs:** Metric 3.1, 6.3, 6.4 (validation). **Expected Result:** Confirmed lift in activation and/or checkout completion.
- **Owner:** Marketing - **Deliverable:** MT3 landing-page-specific test live. **KPIs:** Metric 10.1. **Expected Result:** Early read on `cpc/google` conversion-rate movement.
- **Owner:** Product/Engineering - **Deliverable:** MT4/MT5 findings finalized; any confirmed instrumentation fix scoped for Engineering. **Expected Result:** Resolved ambiguity on checkout origin and desktop abandonment cause.
- **Owner:** Marketing/CRM - **Deliverable:** MT6 loyalty program v1 launched. **KPIs:** Metric 7.3, 8.2. **Expected Result:** Initial repeat-purchase-rate movement in the targeted cohort vs. control.
- **Owner:** Leadership - **Deliverable:** BR12 strategic framing decision made, informed by MT6's early results. **Expected Result:** A committed direction (brand-engagement vs. repeat-revenue investment) for the following quarters.
- **Owner:** Data Engineering - **Deliverable:** Long-term data-platform project (Section 5) scoped for next quarter.

---

# 12. Future Analytics Opportunities

The following are explicitly **out of this engagement's scope** and are not supported by any query executed in `05_SQL_Query_Repository.md`. They are flagged here as future work, not presented as findings:

- **Marketing Attribution Modeling.** This engagement used last-touch, session-scoped channel data only (per `04_Metrics_Framework.md`'s stated attribution methodology). A multi-touch attribution model would give a more accurate picture of channel contribution, particularly relevant to resolving BR6's paid-search question with more confidence.
- **Formal A/B Testing Infrastructure and Results Analysis.** Multiple Section 4 recommendations (MT1, MT2, MT3, MT6) assume a testing capability; a dedicated analysis of the org's current experimentation maturity was not part of this engagement.
- **Pricing and Price-Elasticity Analysis.** No query in this engagement examined price sensitivity, discount/coupon usage (the `coupon` field on `items` was documented in `02_Data_Understanding.md` but never queried), or elasticity by category.
- **Full Customer Lifetime Value Cohort Modeling.** This engagement used a within-window LTV *snapshot* (Metric 8.1), explicitly caveated as not a true lifetime figure. A proper CLV cohort model, projecting value beyond the 92-day window, was out of scope.
- **Geographic Analysis.** Explicitly flagged as an acknowledged gap in `06_Analysis.md` §12 no country/region-level revenue or conversion query was executed. This remains the clearest, lowest-effort next analysis to commission, since the underlying data quality was confirmed sound (§6.2 Data Validation) and the metrics framework already supports the necessary cuts.
- **Demand and Revenue Forecasting.** This engagement is descriptive and diagnostic, not predictive no forecasting model was built or implied by any query in the repository.
- **Market Basket Analysis / Cross-Sell Affinity.** The `items` array's structure would support a co-purchase analysis (which products are frequently bought together), directly relevant to BR1's future personalization roadmap (Section 5) this was not executed in this engagement and should be scoped as a distinct follow-up.

---
