# 03 : Data Validation

All 20 validation queries from the Stage 3 suite were executed against
`bigquery-public-data.ga4_obfuscated_sample_ecommerce`. This document records actual results,
verdicts, and critically the methodology decisions those results forced us to make before
any KPI or funnel is built in later stages.

---

## Section 1: Dataset Integrity

### 1.1 Date Range Validation
| min_date | max_date | distinct_days | expected_days |
|---|---|---|---|
| 2020-11-01 | 2021-01-31 | 92 | 92 |


### 1.2 Total Events Validation
| total_events |
|---|
| 4,295,584 |



### 1.3 Event Distribution Validation
17 distinct event types confirmed, ranking and proportions match prior discovery exactly (page_view 31.44% → view_item_list 0.00%).


---

## Section 2: User & Session Validation

### 2.1 Unique Users
| unique_users | null_user_pseudo_id | empty_user_pseudo_id |
|---|---|---|
| 270,154 | 0 | 0 |


### 2.2 Unique Sessions
| unique_sessions | events_missing_session_id |
|---|---|
| 360,129 | 0 |

Session key (`user_pseudo_id + ga_session_id`) is 100% reconstructable. Confirms ~1.33 sessions per user and ~11.9 events per session on average across the window plausible for an ecommerce browsing site.

### 2.3 user_id Coverage
| total_rows | rows_with_user_id | pct_with_user_id |
|---|---|---|
| 4,295,584 | 0 | 0.0% |

Confirms guest-checkout assumption from `02_Data_Understanding.md`. `user_id` is **fully unusable** for this dataset (not partially zero rows), stronger than originally assumed. Any user-level segmentation must use `user_pseudo_id` exclusively.

### 2.4 user_pseudo_id Validation
Top user_pseudo_id by row count: 1,309 events over 92 days, 0.03% of total events. No concentration risk.

**⚠️ WARNING (non-blocking)** on the concentration check itself (no bot/QA account dominating), but flagging a genuine format anomaly: values like `56110190.2206479626` are float-formatted, not the typical opaque hashed-string client ID seen in most GA4 exports. This is almost certainly an artifact of this dataset's obfuscation process. It doesn't affect usability as a join/group-by key, but **cast explicitly to STRING before using in any `CONCAT()` or key-generation logic** implicit float formatting can introduce trailing-decimal inconsistencies across queries if left as FLOAT64/NUMERIC.

### 2.5 Session Reconstruction Validation
Query returned the full 100-row limit, virtually all showing `distinct_session_numbers = 1` (good) but `total_entrances = 2` (violation expected ≤1).

**⚠️ WARNING** this is **not a broken session key**; session_number consistency (the more serious failure mode) is intact. The doubled-entrance pattern is consistent with duplicate event firing, and correlates directly with the debug-traffic finding in §6.4 below (a page loading twice under `debug_mode` will log two `entrances=1` events in the same session). **Do not exclude sessions on this basis alone** instead, treat `entrances` as a signal to dedupe on `(session_key, event_name='page_view', page_location)` when precise landing-page analysis is needed in Stage 6.

---

## Section 3: Platform Validation

### 3.1 Platform Distribution
WEB = 100.0% of 4,295,584 events.

### 3.2 app_info Usage
`rows_with_app_id = 0`. **Confirmed unused, safe to exclude from all analysis.

### 3.3 Device Coverage
| category | pct |
|---|---|
| desktop | 58.16% |
| mobile | 39.67% |
| tablet | 2.17% |

No null/unknown category rows. Also a useful early finding: this store skews **desktop-majority**, contrary to the "mobile-first ecommerce" assumption many analysts default to. Worth carrying into Stage 6 Device Analysis and Stage 7 Recommendations.

---

## Section 4: Revenue Validation

### 4.1 Purchase Event Validation
| total_purchase_events | purchases_missing_revenue | purchases_missing_transaction_id |
|---|---|---|
| 5,692 | 0 | 23 (0.40%) |

Revenue itself is fully populated (great), but 23 purchases (0.40%) lack a transaction_id, which will need explicit handling wherever transaction-level joins are used (see §4.2).

### 4.2 transaction_id Uniqueness
Two distinct problems surfaced, both more serious than a simple double-fire:

1. **`(not set)` placeholder collisions:** 883 purchase events share the literal transaction_id `"(not set)"`, spanning **767 distinct users** these are 767+ separate real transactions that all lost their true transaction_id and collapsed into one fake "duplicate" group. This is a missing-ID problem, not a duplication problem.
2. **Real ID collisions across distinct users:** Beyond the placeholder issue, several genuine numeric transaction_ids (e.g., `43463`, `140428`, `333223`, `372598`, `240060`, `221950`, `310742`, `898666`, `594908`, `885245`, `952677`, `972372`) appear exactly twice **each with 2 distinct users** meaning the same transaction_id was independently generated for two different people's orders. This fails the strict "no cross-user collision" bar.
3. Most other duplicate transaction_ids (the majority of ~300+ groups) show `event_count=2, distinct_users=1` consistent with benign duplicate-firing (same user, same order, logged twice), most plausibly another symptom of the debug-traffic issue in §6.4.

Combined excess/affected rows are roughly 20%+ of the 5,692 purchase events once the `(not set)` group is included.

**❌ FAIL** : `transaction_id` **cannot be used as a bare uniqueness key** for revenue analysis without a cleaning layer. **Required methodology fix, adopted for Stage 4 onward:**
- Exclude `transaction_id = '(not set)'` rows from any *per-transaction* metric (AOV, transaction counts, product-level revenue joins) but still include their revenue in *headline* aggregate revenue totals, with a footnote on affected volume.
- Deduplicate real numeric transaction_ids by `(transaction_id, user_pseudo_id)` rather than `transaction_id` alone, since a handful of IDs are shared across genuinely different users.

### 4.3 Revenue Reconciliation
| total_transactions_compared | transactions_with_gap | pct_transactions_with_gap |
|---|---|---|
| 5,669 | 1,619 | 28.56% |

**❌ FAIL** : 28.56% exceeds the 15% FAIL threshold. This is higher than shipping/tax alone would typically explain. **Methodology decision:** transaction-level `ecommerce.purchase_revenue_in_usd` remains the **source of truth for headline revenue** (Section 4.1 confirms it's fully populated and internally consistent). Item-level `items.item_revenue_in_usd` will be used only for **relative** product/category revenue *share*, never presented as reconciling exactly to transaction totals. This distinction will be stated explicitly in `04_Metrics_Framework.md`.

### 4.4 event_value_in_usd Validation
| event_name | event_count | distinct_values | min | max | median |
|---|---|---|---|---|---|
| purchase | 5,242 | 342 | 1.0 | 1,530.0 | 48.0 |

| purchase_events | matches_real_revenue |
|---|---|
| 5,692 | 5,242 (92.1%) |

**Stage 2 assumption formally REVISED.** Contrary to the original assumption in `02_Data_Understanding.md` (based on the constant `1026454.43` value seen on non-purchase events in the initial 10-row sample), `event_value_in_usd` **does reconcile with real transaction revenue in 92.1% of purchase events**. This is a genuine finding the original assumption was wrong and is being corrected rather than defended.

**Revised rule for Stage 4 onward:** `event_value_in_usd` is **reliable specifically on `purchase` events** (92.1% match) and can serve as a secondary/cross-check revenue field there. It remains **excluded from analysis on all non-purchase events**, where it appears to carry an unrelated, likely non-revenue value (the constant seen in early sampling) this narrower exclusion is correct; the blanket exclusion was not.

---

## Section 5: Nested Field Validation

### 5.1 event_params Completeness
`ga_session_id`, `page_location`, `ga_session_number` are present on **100% of rows for all 17 event types**.

The strongest result in the entire suite. Session-grain and path-grain analysis can proceed with full confidence.

### 5.2 items Completeness
| event_name | total_events | pct_populated |
|---|---|---|
| view_item | 386,068 | 62.65% |
| add_to_cart | 58,543 | 100.00% |
| begin_checkout | 38,757 | 82.54% |
| select_item | 31,007 | 100.00% |
| purchase | 5,692 | 99.96% |
| view_item_list | 71 | 60.56% |

**❌ FAIL** `view_item` (62.65%), `begin_checkout` (82.54%), and `view_item_list` (60.56%) all fall below the 90% floor. **This is a real limitation, not a query bug** 144,192 `view_item` events (37% of all product views) carry no product detail at all.

**Methodology impact for Stage 6 (Product Performance Analysis):** item-level product analysis (Stage 20) is still viable 241,876 populated `view_item` rows is a large sample but any "% of product views that convert to cart" calculation must be scoped to **item-identifiable views only**, with the coverage gap stated as an explicit caveat rather than silently treating missing items as "zero interest." `add_to_cart`, `select_item`, and `purchase` (the highest-stakes events) are all ≥99.96% populated, so the funnel's *commercial* end is solid the gap sits earlier, at browsing-stage instrumentation.

### 5.3 ecommerce Completeness
`has_purchase_revenue` is non-zero **only** on `purchase` events (5,692), zero everywhere else.

No schema misuse; ecommerce struct is correctly scoped.

---

## Section 6: Data Quality

### 6.1 Null-Rate Checks
All core fields (`event_date`, `event_timestamp`, `event_name`, `user_pseudo_id`, `device.category`, `geo.country`, `traffic_source.medium/source`, `platform`) show **0.0% null**.



### 6.2 Missing Values in Key Dimensions
| field | value | count | % of total |
|---|---|---|---|
| geo.country | (not set) | 32,208 | 0.75% |
| traffic_source.medium | (none) | 989,684 | 23.04% |

**⚠️ WARNING (reclassified, not a defect)** `geo.country = (not set)` at 0.75% is a clean PASS on its own. `traffic_source.medium = (none)` at 23.04% technically crosses the mechanical 15% FAIL line, but **this is a misapplication of that rule** `(none)` is GA4's legitimate label for **direct traffic** (typed URL / bookmark), not a missing-data defect. 23% direct traffic is realistic for an ecommerce site and will be reported as its own channel segment in Stage 6, not treated as a data quality gap. This is the kind of blind-rule-application mistake a junior analyst makes and a senior one catches.

### 6.3 Duplicate Events
0 duplicate `(user_pseudo_id, event_timestamp, event_name)` groups found.

Genuinely clean; no row-level duplication at the raw grain, despite the transaction- and entrance-level duplication seen elsewhere (those are logical/business-key duplicates, not raw-row duplicates an important distinction).

### 6.4 Debug/Test Traffic
| debug_events | total_events | pct_debug |
|---|---|---|
| 3,682,991 | 4,295,584 | **85.74%** |

**❌ FAIL the headline finding of this validation pass.** This is not a "1-2% test traffic" nuisance the overwhelming majority of this entire public sample dataset carries `debug_mode = 1`.

**Methodology decision (documented here, not deferred):** Excluding 85.74% of events would gut the dataset to ~614,000 events destroying statistical power for retention, cohort, and geographic analysis. Rather than mechanically applying "exclude debug traffic," we're treating this as a **known characteristic of this specific public sample export**, not literal internal QA/test activity from the Merchandise Store's engineering team (the volume and universality make that interpretation implausible real QA traffic doesn't run at 6:1 over production). **Decision: `debug_mode` will NOT be used as a row-exclusion filter in this project.** It will instead be documented prominently as a labeled limitation of the dataset in every relevant document (`02`, `04`, `09`), and called out anywhere it plausibly explains a secondary anomaly (as it does in §2.5 and partially in §4.2). This decision itself will be logged with full rationale in `09_Analytics_Decision_Log.md` exactly the kind of judgment call an interviewer wants to see defended, not hidden.

### 6.5 Outlier Detection
- **Revenue outliers:** top purchase values ($1,530, $1,200, $1,170...) are plausible bulk/corporate merch orders, not data errors. Several again show the `(not set)` transaction_id issue from §4.2.
- **Engagement time outliers:** several `user_engagement` events show 36M–70M ms (10–19+ **hours**) of single-event engagement time implausible for genuine active engagement, consistent with a tab left open in the background continuing to ping.
- **Quantity outliers:** transactions up to 400 units in one order plausible for corporate bulk orders, but one transaction_id (`285846`) appears twice with matching quantities, another confirmation of the §4.2 collision issue.

**⚠️ WARNING** No evidence of a units/scaling bug (which would be an automatic FAIL); these are edge-case, explainable outliers. **Methodology fix:** cap/winsorize `engagement_time_msec` at a defensible ceiling (e.g., 3,600,000 ms = 1 hour) before computing any average engagement-time KPI in Stage 4, so a handful of background-tab sessions don't distort the metric.

---

## Data Quality Scorecard

**23 checks total. 61% clean PASS, 22% WARNING (manageable with documented handling), 17% FAIL (requiring methodology adjustments, all addressed above none are project-ending).**

---

## Go/No-Go Decision: **GO, with four binding methodology rules carried into Stage 4**

None of the 4 FAILs invalidate the project all four are addressable with explicit, documented handling rather than silent workarounds. That handling is now locked in:

1. **Debug traffic (85.74%) is not filtered out.** Treated as a labeled dataset limitation, not excluded exclusion would destroy sample size. Documented in every relevant stage.
2. **`transaction_id = '(not set)'` and cross-user ID collisions are excluded from per-transaction metrics** (AOV, transaction counts) but included in headline aggregate revenue, with footnoted affected volume.
3. **Item-level revenue is never presented as reconciling to transaction-level revenue.** Transaction-level `ecommerce.purchase_revenue_in_usd` is the sole source of truth for headline revenue KPIs; item-level data is used only for relative category/product revenue share.
4. **`view_item` product-level analysis is scoped to the 62.65% of rows with populated `items`**, with the coverage gap stated as an explicit caveat, not silently ignored.

Additionally: `event_value_in_usd` is now a validated secondary signal on `purchase` events (assumption reversed from Stage 2); `engagement_time_msec` will be capped at 1 hour before averaging; device split is confirmed desktop-majority (58%/40%/2%), which should inform Stage 6/7 findings.
