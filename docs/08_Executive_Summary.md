# 08 : Executive Summary & Board Briefing

## Cover

**Project Title:** Google Merchandise Store - Product Analytics Engagement
**Prepared By:** Ayush Kumar Sahoo, Product Analytics
**Date:** July 2026
**Project Duration:** Full-funnel analysis of a 92-day operating window (Nov 1, 2020 - Jan 31, 2021)
**Dataset:** GA4 BigQuery Export, `bigquery-public-data.ga4_obfuscated_sample_ecommerce` (public sample dataset; findings are directional and methodology-driven, per the scope note in `01_Project_Overview.md`)
**Objective:** Determine where this ecommerce store loses customers, which segments and channels actually drive value, whether the business retains anyone, and what leadership should do about it in the next quarter.

---

## 1. Executive Snapshot

**Business Health:** Operationally sound, strategically under-leveraged. Traffic acquisition, checkout mechanics, and revenue processing all function normally. The gap is in converting interest into engagement and in retaining or activating the customers already proven to be valuable.

**Top Opportunity:** Two segments already outperform the blended average by 2-3x past purchasers (11.85% repeat-purchase rate vs. near-zero general retention) and the top 0.43% of users (driving ~62% of measured value) and neither has a dedicated program built around it yet.

**Largest Risk:** Two confirmed data-integrity issues (a tagging change that fakes an engagement-rate "improvement," and a broken product-ID join that fakes a "0%-converting" category) mean some existing reporting is currently wrong. Acting on unvalidated numbers is the single biggest near-term risk to good decision-making.

**Biggest Win:** A fully validated, production-grade analytics layer - 54 SQL queries, 7 documented methodology rules, and a complete metrics framework that makes every finding in this engagement traceable and defensible, not anecdotal.

**Highest Priority:** Fix the reporting issues first (this quarter, low effort), then fix the homepage/navigation activation gap only 19.97% of first-time visitors ever view a product, the single largest constraint on the entire business.

**Overall Assessment:** This is a fixable, well-understood business problem, not an ambiguous one. Every recommendation in this engagement is backed by an executed query and a specific number the path forward is a sequencing and focus decision, not a research problem.

---

## 2. Executive KPI Dashboard

| KPI | Current State | Business Health | Priority | Owner |
|---|---|---|---|---|
| North Star Weekly Transacting Users | Peaked at 711 (week of Dec 7); fell to 99 (final week) | Moderate tracks seasonality, no independent growth signal yet | High | VP Product |
| First-Session Product View Rate | 19.97% | **Critical** | Critical | Product |
| Overall View-to-Purchase Conversion | 6.1% | Moderate | High | VP Product |
| Checkout Completion Rate (post-checkout-start) | 43.62% | Good | Medium | Product/Eng |
| Checkout Payment-Step Loss | 38.65% (steepest single stage) | Poor | Critical | Product/Eng |
| Total Revenue (92-day window) | $362,165 | Good (functioning) | Medium | Finance |
| Average Order Value | $69.15 (median $48) | Good | Low | Finance |
| Day 30 Retention | 0.13% | **Critical (structural)** | High (strategic) | Growth/Leadership |
| Repeat Purchase Rate (past buyers) | 11.85% | Good, under-activated | High | Marketing/CRM |
| Value Concentration (top tier) | 0.43% of users = ~62% of value | Good signal, unexploited | High | Analytics/Marketing |
| `(data deleted)` Channel Conversion | 3.14% (best in dataset) | Excellent, but unactionable | High | Analytics/Legal |
| Paid Search (`cpc/google`) Conversion | 0.98% (below organic) | Poor | High | Marketing |
| Data Reporting Integrity (R6/R7) | 2 confirmed defects, both diagnosed and fixed in this engagement | Was Poor, now corrected | Critical | Analytics/Eng |

---

## 3. Top 10 Executive Findings

| # | Finding | Evidence | Business Impact | Confidence |
|---|---|---|---|---|
| 1 | Only 19.97% of first-time visitors ever view a product | Metric 3.1, Query 3.1 | Caps every downstream funnel and revenue number | High |
| 2 | 82.47% of users are single-session; Day 30 retention is 0.13% | Metric 4.4/7.1, Queries 4.4/7.1 | Confirms a brand-engagement, not repeat-revenue, business model | High |
| 3 | Checkout's steepest drop is payment-info entry (38.65% loss), not final submission | Metric 6.3, Query 6.3 | A specific, fixable, high-value checkout fix | High |
| 4 | 0.43% of users drive ~62% of measured value | Metric 8.3, Query 8.3 | A concentrated, currently unexploited value pool | High |
| 5 | `(data deleted)` traffic converts 2.8x better than the next-best channel | Metric 10.1/10.2, Queries 10.1/10.2 | The store's best segment is invisible to standard reporting | High (finding) / Low (actionability) |
| 6 | Paid search underperforms every free channel on conversion and engagement | Metric 10.1/2.10, Queries 10.1/2.10 | Current paid spend is inefficient, not just small | High |
| 7 | A confirmed tagging change (Dec 28) fakes an engagement-rate "improvement" | Query 4.6, Rule R6 | Risk of a false narrative reaching leadership, now caught and documented | High |
| 8 | Product ID cannot be trusted as a join key in this dataset | Queries 9.5/9.6, Rule R7 | Would have produced a false "0%-converting" mega-category if unfixed | High |
| 9 | Mobile modestly outperforms desktop on revenue-per-session and cart abandonment | Metric 5.5/6.5, Queries 5.5/6.5 | Inverts the default "fix mobile first" assumption for this store | High |
| 10 | 43% of checkouts show no prior add-to-cart event | Query 6.6 | A large, still-unexplained share of the purchase path | Medium |

---

## 4. Biggest Business Risks

**Data-integrity risk (Critical, immediate).** Two confirmed measurement defects (R6, R7) mean specific dashboards were producing false conclusions until this engagement caught and corrected them. Any team still using the uncorrected versions is at risk of acting on wrong numbers today.

**Structural retention risk (High, strategic).** Near-zero organic retention means this business has no safety net if acquisition slows revenue is fully dependent on continuous new-visitor volume. This is a business-model characteristic, not a quick fix, and needs an explicit leadership decision (see `07_Business_Recommendations.md`, BR12).

**Concentration risk (Medium, ongoing).** Both the homepage (46% of all entrances) and the customer base (0.43% of users driving 62% of value) represent single points of concentration a homepage issue or the loss of a handful of top customers would have outsized impact.

**Compliance risk (Medium, contained).** Investigating the `(data deleted)` segment (Finding #5) touches privacy-redacted data and must proceed only with Legal approval this is flagged, sequenced, and contained in the recommendations, not being pursued unilaterally.

**Marketing efficiency risk (Medium, active).** Continued paid-search spend at current targeting quality is actively inefficient this is a live, ongoing cost until `07_Business_Recommendations.md`'s BR6 is resolved.

---

## 5. Executive Recommendations

| Recommendation | Owner | Priority | Expected KPI | Timeline |
|---|---|---|---|---|
| Data & Reporting Integrity Lockdown (BR7) | Analytics/Engineering | Critical | Reporting accuracy (precondition for all else) | Month 1 |
| Homepage & Navigation Activation Redesign (BR1) | Product | Critical | First-Session Product View Rate (Metric 3.1) | Month 2-3 |
| Checkout Payment-Step UX Fix (BR2) | Product/Engineering | Critical | Checkout-to-Purchase, granular (Metric 6.3) | Month 2-3 |
| Past-Purchaser Loyalty Program (BR3) | Marketing/CRM | High | Repeat Purchase Rate (Metric 7.3) | Month 2-3 |
| High-Value Tier Profiling & VIP Program (BR4) | Analytics/Marketing | High | Value Tier composition (Metric 8.3) | Month 1 (profile), 6-12mo (program) |
| `(data deleted)` Segment Investigation (BR5) | Analytics/Legal/Marketing | High | Channel Conversion, if actionable (Metric 10.1/10.2) | Month 1-3 |
| Paid Search Targeting Overhaul (BR6) | Marketing | High | Channel Conversion Rate (Metric 10.1) | Month 1-3 |

*(Full recommendation set, including Medium-priority items and department-level ownership, is in `07_Business_Recommendations.md`.)*

---

## 6. 90-Day Roadmap

| **Month 1 - Stabilize & Validate** | **Month 2 - Optimize & Experiment** | **Month 3 - Scale & Execute** |
|------------------------------------|-------------------------------------|-------------------------------|
| Data integrity lockdown | Homepage A/B test launched | Homepage improvements implemented |
| Paid search spend freeze | Checkout payment UX improvements | Checkout improvements deployed |
| Landing page reporting normalized | Paid search targeting review completed | Loyalty Program v1 launched |
| High-value customer profiling | Session-recording diagnostics underway | Desktop checkout audit completed |
| Merchandise reporting corrected | Loyalty program designed | Findings validated & finalized |
| Executive report distributed | `(data deleted)` investigation (Legal-gated) | Strategic retention decision (BR12) completed |
| - | - | Data platform roadmap prepared for next quarter |

**Key deliverables:** integrity-corrected reporting (M1), a live activation test and a scoped checkout fix (M2), a launched loyalty program and a leadership decision on retention strategy (M3).
**Expected outcomes:** measurable movement on Metric 3.1 (activation) and Metric 6.3 (checkout), an initial read on loyalty-driven repeat purchase rate, and a resolved (or explicitly closed) investigation into the `(data deleted)` segment.

---

## 7. Executive KPI Scorecard

| KPI | Status | Business Impact | Priority |
|---|---|---|---|
| First-Session Product View Rate (19.97%) | **Critical** | Caps entire business | Critical |
| Day 30 Retention (0.13%) | **Critical** | Structural, defines the business model | High |
| Checkout Payment-Step Loss (38.65%) | Poor | Specific, fixable revenue leak | Critical |
| Paid Search Conversion (0.98%) | Poor | Inefficient spend | High |
| Data Reporting Integrity (pre-fix) | Poor → now corrected | Risk of false conclusions | Critical |
| Checkout Completion Rate (43.62%) | Good | Functioning normally | Medium |
| Total Revenue Processing ($362,165) | Good | Reliable, reconciled 3 ways | Low |
| Mobile Device Performance | Good | Outperforms desktop | Medium |
| Repeat Purchase Rate, past buyers (11.85%) | Good, under-activated | Untapped upside | High |
| Value Concentration (0.43% = ~62% value) | Moderate | Real, but unexploited | High |
| `(data deleted)` Segment | Excellent (performance) / Unactionable (status) | Highest potential ROI, gated by Legal | High |
| Desktop Cart Abandonment (81.73%) | Moderate | Inverts default assumption, needs audit | Medium |
| Geographic Performance | **Not Assessed** | Acknowledged scope gap, not a finding | Future work |

---

## 8. Portfolio Highlights

This engagement produced a complete, production-grade Product Analytics deliverable set:

- **54 production SQL queries**, each mapped to a specific metric definition, executed against real BigQuery data, and validated against actual (not assumed) output.
- **7 documented methodology rules (R1–R7)**, each discovered through evidence, not assumed upfront including a confirmed instrumentation break and a confirmed broken join key, both caught before they could produce wrong conclusions.
- **A complete metrics framework** spanning North Star, Acquisition, Activation, Engagement, Commerce, Funnel, Retention, Customer, Product, and Marketing metrics each with formula, grain, dimension compatibility, and common-mistake guidance.
- **A full executive analysis** (18 sections) connecting findings across every function Marketing, Product, Growth, Finance into a single causal narrative.
- **A strategic recommendations report** with 13 prioritized, ROI/effort-classified, KPI-linked initiatives and a full quarterly roadmap.
- **Built entirely on GA4 BigQuery export data**, using the same nested-schema SQL patterns (`UNNEST`, correlated subqueries on `event_params`, session-grain reconstruction) required in real production GA4 analytics work.
- **Production-ready documentation**, structured as a navigable, GitHub-publishable repository rather than a single monolithic notebook.

---

## 9. Project Deliverables

| Document | Purpose |
|---|---|
| `01_Project_Overview.md` | Frames the business problem, dataset, objectives, and key questions before any SQL is written. |
| `02_Data_Understanding.md` | Documents the dataset's architecture, event-driven data model, full field dictionary, and core analytics assumptions. |
| `03_Data_Validation.md` | Executes and scores 23 data-quality checks (PASS/WARNING/FAIL), establishing the evidence base for every methodology rule. |
| `04_Metrics_Framework.md` | Defines 26 metrics across 10 categories, each with formula, grain, and common-mistake guidance, built on the validation findings. |
| `05_SQL_Query_Repository.md` | 54 production SQL queries implementing every metric, executed and cross-checked against real output, discovering rules R6 and R7 along the way. |
| `06_Analysis.md` | The full executive analysis 18 sections connecting every finding into one business narrative, with root causes, confidence levels, and cross-functional implications. |
| `07_Business_Recommendations.md` | Converts findings into 13 prioritized, owned, KPI-linked recommendations with a full quarterly roadmap and ROI/effort classification. |
| `08_Executive_Summary.md` | This document a board-ready briefing and interview/portfolio reference. |
| `09_Analytics_Decision_Log.md` | *(Next deliverable)* a durable record of every assumption, validation result, and methodology decision made across the engagement, for future analysts inheriting this dataset. |

---

## 10. Final Conclusion

This engagement set out to determine whether a real business story could be extracted from a public GA4 sample dataset, using the same rigor a Product Analytics team at a top technology company would apply to a live production decision. It succeeded on both counts: a coherent, evidence-based business narrative emerged, and the process of finding it surfaced genuine data-quality issues that would have compromised a less careful analysis.

**Business outcome:** The Google Merchandise Store is an operationally healthy but strategically under-leveraged business. Its core constraint is not acquisition volume or checkout mechanics both function adequately but activation (only 19.97% of new visitors ever view a product) and retention (a near-total absence of return visits, consistent with a low-frequency-purchase category rather than a fixable defect). The clearest near-term opportunity is not more traffic, but better conversion of the traffic already arriving, and deliberate investment in the two segments past purchasers and the top value tier already proven to outperform the blended average by 2-3x.

**Analytics maturity demonstrated:** This engagement did not stop at "here are the numbers." Every stage built on and stress-tested the one before it: data validation exposed real defects before any metric was trusted; the metrics framework encoded those defects as enforceable rules rather than footnotes; the SQL repository tested those rules against real query output and discovered two *additional* defects (R6, R7) that hadn't been visible during validation alone; the executive analysis synthesized findings into a single causal narrative rather than a list of disconnected charts; and the recommendations report translated that narrative into owned, sequenced, risk-assessed decisions. This is the difference between reporting metrics and building an analytics function's institutional judgment.

**Key lessons:**
- **Grain discipline is not optional.** The single most common error opportunity in this dataset event vs. session vs. user grain was directly responsible for at least two major findings in this engagement (the cart-to-checkout naive-ratio gap, the view-to-purchase divergence). Every non-trivial query in this project states its grain explicitly for exactly this reason.
- **An anomaly is a question, not a footnote.** The Dec 28 engagement-rate jump and the `item_id` join failure could both have been reported as-is with a passing caveat. Instead, each was diagnosed to a specific, evidenced root cause (R6, R7) before being trusted this is what separates a validated finding from a plausible-looking one.
- **The most valuable finding is sometimes the one that isn't actionable yet.** The `(data deleted)` segment is this engagement's best-performing discovery and its most honestly-caveated one flagged as high-potential and Legal-gated, not oversold as a guaranteed win.

**Why this demonstrates real Product Analytics capability:** This project treats SQL as a means, not the deliverable the actual output is a defensible business narrative, a prioritized set of decisions, and a documented, reusable methodology that a real team could hand to a new analyst on day one. That combination technical rigor, business judgment, and honest handling of uncertainty is what separates a portfolio piece built to demonstrate SQL fluency from one built to demonstrate Product Analytics judgment.
