# 01 — Project Overview

## Google Merchandise Store: End-to-End Product Analytics Engagement

**Author:** [Your Name]
**Role simulated:** Senior Product Analyst / Business Analyst
**Dataset:** `bigquery-public-data.ga4_obfuscated_sample_ecommerce`
**Engagement window:** November 1, 2020 – January 31, 2021 (92 days)

---

## 1. Business Problem

This project simulates a real analytics engagement for the **Google Merchandise Store**, an e-commerce site selling Google-branded apparel and accessories. Unlike a pure-profit retailer, this store's business model prioritizes **brand engagement and clean digital experience** over aggressive margin extraction — a framing that matters because it shapes what counts as a "good" recommendation later in this project (e.g., we would not recommend aggressive price discrimination the way we might for a margin-maximizing retailer).

Leadership (simulated stakeholders: Head of Analytics, VP Product, Growth Lead, Store PM) needs to understand:

- Where the product experience is losing users
- Whether acquisition channels are bringing back valuable, repeat traffic
- Which segments (device, geography, new vs. returning) disproportionately drive value
- Whether the store retains users at all, or is a one-and-done storefront
- Which products are over/under-performing relative to the traffic they receive
- Concrete, ROI-framed recommendations the Product and Growth teams can act on

This project answers those questions using the same event-level methodology a Product Analytics team at a top technology company would use — not a generic BI dashboard tutorial.

## 2. Dataset Overview

| Attribute | Detail |
|---|---|
| Source | Google Analytics 4 (GA4) BigQuery Export, public sample dataset |
| Grain | **One row = one event** (not one user, session, or purchase) |
| Date range | `events_20201101` → `events_20210131` (92 daily tables) |
| Total events | 4,295,584 |
| Distinct event types | 17 (page_view, view_item, add_to_cart, begin_checkout, purchase, etc.) |
| Platform | 100% Web (confirmed via `platform` and null `app_info`) |
| Identity model | Pseudonymous (`user_pseudo_id`); `user_id` almost entirely null (guest checkout site) |

**Known constraints (stated upfront, not discovered later):**
- This is a **sample, obfuscated** dataset. Google has stated it should not be read as the real store's actual business performance — findings in this project are **directional and methodology-driven**, not literal financial claims.
- No cost/ad-spend data exists in this schema — any "ROI" language in recommendations is qualitative/directional, not a real CAC calculation.
- No customer PII — segmentation relies on device, geography, traffic source, and behavior, not demographics.
- The engagement window spans Black Friday, Cyber Monday, and Christmas — any raw trend line will show a large seasonal spike that must be called out explicitly rather than presented as organic growth.

## 3. Project Objectives

This project is designed to demonstrate, with evidence (not just claims):

- Product Analytics & Funnel Analysis
- Business & Growth Analytics
- Customer Segmentation & Cohort/Retention Analysis
- Revenue Analytics
- Marketing Channel Analytics
- Executive-level Business Storytelling
- Experimentation thinking (A/B test design, not just descriptive stats)
- SQL engineering fluency on nested, semi-structured production-scale data (not toy flat tables)

## 4. Key Business Questions

These six questions are the spine of the entire project. Every KPI, query, and chart in this repository exists to answer one of them — if it doesn't, it was cut.

1. **Where are we losing the most users** in the path from landing to purchase?
2. **Which acquisition channels** bring users who convert and return — not just click?
3. **Which segments** (device, geography, new vs. returning) drive disproportionate revenue?
4. **Are users retaining** or is this a one-and-done storefront?
5. **Which products/categories** are over- or under-performing relative to the traffic they receive?
6. **What are 3–5 concrete, ROI-justified recommendations** for Product and Growth?

## 5. Project Roadmap

| Stage | Document | Deliverable |
|---|---|---|
| 1 | `01_Project_Overview.md` | Business framing, objectives, scope (this document) |
| 2 | `02_Data_Understanding.md` | Data architecture, dictionary, nested field deep-dive, ER-style relationships |
| 3 | `03_Data_Validation.md` | Data quality checks, PASS/WARNING/FAIL report |
| 4 | `04_Metrics_Framework.md` | North Star Metric, input/guardrail metrics, full KPI glossary |
| 5 | `05_SQL_Query_Repository.md` | 75–100 interview-level SQL queries, each with business purpose |
| 6 | `06_Analysis.md` | Funnel, journey, revenue, retention, cohort, device, geo, marketing, product analysis |
| 7 | `07_Business_Recommendations.md` | Prioritized, impact-scored recommendations |
| 8 | `08_Executive_Summary.md` | Leadership-ready summary + dashboard + interview talking points |
| 9 | `09_Analytics_Decision_Log.md` | Every assumption and methodology decision, with rationale |

**Status:** Stages 1–2 complete. Proceeding to Stage 3 (Data Validation) next.

## 6. Scope Boundaries

**In scope:** acquisition, engagement, funnel conversion, retention, revenue, segmentation, product performance, executive reporting, recommendations, experiment design.

**Explicitly out of scope** (named rather than silently ignored, per senior-analyst practice): customer support/service data, inventory and supply chain, real advertising spend/CAC data (not present in this schema — any cost-based framing is directional only).
