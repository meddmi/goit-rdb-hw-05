# Olist customer feature store — Homework 5

This repository contains a PostgreSQL notebook that builds buyer-level features for repeat-purchase analysis. It loads the Olist e-commerce data into constrained tables, derives features through a named CTE chain, explores a synthetic referral graph, caches the features in a materialized view, and compares execution plans.

The work is in [hw_5.ipynb](hw_5.ipynb). Each feature row represents one `customer_unique_id` (one buyer). Olist assigns a separate `customer_id` to each order-specific customer record, so one buyer can have several `customer_id` values.

## Run the notebook

1. Open a terminal in the repository root so the relative `data/olist/` paths resolve correctly.
2. If a file is missing in `data/olist/`, the notebook downloads the Olist archive from Kaggle and extracts it. The first cell installs `pgserver`, `psycopg2-binary`, `sqlalchemy`, and `pandas`, so the first run needs network access if dependencies or CSVs are missing.
3. Open `hw_5.ipynb` in a Jupyter-compatible environment and use **Restart & Run All**. The notebook starts a local PostgreSQL server, prints its version, verifies the CSV hashes, and runs Sections 1–5 in order.

Run All rebuilds the typed Olist tables, the synthetic `hw5_referrals` table, and the materialized view in the notebook's local PostgreSQL database. The notebook shows raw and typed row counts, sample feature rows, the referral-depth result, MV checks, and both `EXPLAIN (ANALYZE, BUFFERS)` plans.

## Dataset and reproducibility

Source: [Olist Brazilian E-Commerce Public Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce). The notebook uses the customers, orders, order items, products, and sellers CSVs. It creates a raw staging table for each file, then loads typed `olist_` tables with primary keys, foreign keys, checks, and normalized column types. `hw5_` is reserved for the homework's synthetic referral table.

## Feature pipeline

The business question is which previous buyers should receive a category-specific win-back offer, using delivered-purchase recency and value, recent order size, cancellations, cohort, and category preference. The pipeline has seven named stages, each with a `-- Stage N: ...` comment and one main transformation.

| Stage | CTE | Transformation |
| --- | --- | --- |
| 0 | `reference_date` | Set a reproducible reference date to one day after the latest order timestamp in the dataset. |
| 1 | `stage1_order_totals` | Aggregate item price and freight to one row per order, with its buyer and status. |
| 2 | `stage2_customer_summary` | Aggregate each buyer's orders into RFM, cohort quarter, cancellation rate, and a representative `customer_id`. |
| 3 | `stage3_rolling_aov` | Calculate a rolling 30-day average over delivered order totals. |
| 4 | `stage4_latest_aov` | Keep the rolling average at each buyer's latest delivered order. |
| 5 | `stage5_category_revenue` | Sum delivered item prices by buyer and known product category. |
| 6 | `stage6_preferred_category` | Keep the highest-revenue category per buyer, with a deterministic tie break. |

`recency_days` is based on the dataset's reference date rather than `CURRENT_DATE`. RFM, rolling AOV, and category revenue use delivered orders; `cancellation_rate` uses all orders. `order_total` includes price and freight, while category revenue uses item price only. The reported `avg_order_value_30d` is the window ending at each buyer's **latest delivered order**, not the 30 days before the dataset reference date. Buyers with orders but no deliveries have NULL recency, cohort, AOV, and preferred category; their frequency and monetary value are zero.

## Recursive CTE, materialized view, and plans

Olist has no referral data. Section 3 creates `hw5_referrals` as a small synthetic chain. The recursive CTE starts at customers with no incoming referral (`NOT EXISTS`), records depth and path, rejects a customer ID already in the path, and stops at depth 50.

Section 4 materializes the **full** buyer feature pipeline as `mv_olist_customer_features`. A unique index on `customer_id` supports the assignment's concurrent-refresh requirement. The MV uses `MIN(customer_id)` as one representative record per buyer; another unique index on `customer_unique_id` supports real buyer lookups. To look up features using any Olist `customer_id`, first map it through `olist_customers` to `customer_unique_id`, then join to the MV. The notebook checks this mapping and runs `REFRESH MATERIALIZED VIEW CONCURRENTLY` after creating both indexes.

Section 5 updates PostgreSQL planner statistics with `ANALYZE`, then saves `EXPLAIN (ANALYZE, BUFFERS)` output for a full pipeline computation and a representative `customer_id` lookup from the MV. These queries return different numbers of rows, so their elapsed times are not a direct speedup ratio. The Reflection discusses the costly plan operations, caching choice, refresh cycle, recursive termination, and freshness-versus-latency trade-off.
