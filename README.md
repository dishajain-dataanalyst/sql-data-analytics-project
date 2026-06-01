# SQL Data Analytics Project
### Advanced SQL Analysis on a Data Warehouse — by Disha Jain

---

## 👋 Context: Why This Exists

This project is the **direct continuation of my [SQL Data Warehouse Project](https://github.com/dishajain-dataanalyst/sql-data-warehouse-project)**.

Once the warehouse was built — Bronze ingestion, Silver cleansing, Gold star schema — the next question is: *what can you actually learn from the data?* That's what this repo answers.

In my day job at Elde Info Solution, I regularly write SQL to segment customers, track trends over time, and surface anomalies in transactional data. This project is my structured practice of those same techniques on a clean dataset — building a personal library of analytical patterns I can reach for quickly.

---

## 🎯 What This Project Covers

8 categories of SQL analytics, each in its own script:

| # | Script | What it does |
|---|--------|-------------|
| 1 | `01_database_exploration.sql` | Profile tables, row counts, date ranges, NULLs — the first thing I run on any new dataset |
| 2 | `02_measures_and_metrics.sql` | Core KPIs: Total Revenue, Total Orders, AOV, Return Rate |
| 3 | `03_magnitude_analysis.sql` | Rank customers and products by volume — who drives the most value |
| 4 | `04_time_series_trends.sql` | Month-over-month and year-over-year revenue trends using window functions |
| 5 | `05_cumulative_analytics.sql` | Running totals and moving averages — how performance accumulates over time |
| 6 | `06_performance_vs_target.sql` | Actual vs. target comparison — which products/months underperformed |
| 7 | `07_segmentation.sql` | Customer segmentation by spend tier, order frequency, and geography |
| 8 | `08_part_to_whole.sql` | Category contribution to total revenue — what % does each segment represent |
| 9 | `09_report_customers.sql` | Final customer report view — one query that tells the full customer story |
| 10 | `10_report_products.sql` | Final product report view — product-level KPIs for stakeholder reporting |

---

## 💡 My Analytical Approach & Key Decisions

**Why organize scripts by analytical theme, not by table?**
When I join a new project, I need to answer specific business questions fast. Organizing scripts by *question type* (trends, segmentation, ranking) rather than by table means I can find the right pattern immediately and adapt it. It also mirrors how BI teams work — a "show me the top 10 customers" request is a magnitude problem, not a "customers table" problem.

**On window functions over subqueries:**
For cumulative totals and moving averages, I consistently use `SUM() OVER` and `AVG() OVER` rather than correlated subqueries. Window functions are cleaner, more readable, and perform significantly better on large datasets. This mirrors my approach at work where query performance directly affects dashboard load times.

**The two report views (scripts 09 and 10) are production-ready:**
`report_customers` and `report_products` are written as SQL views, not just queries. This means they can be consumed directly by a BI tool like Power BI or Tableau without re-writing logic. In my work, I always push transformations into the data layer rather than doing them in the visualization tool — it keeps reports fast and logic centralized.

**Customer segmentation logic:**
I used spend-tier segmentation (High / Mid / Low value) rather than RFM scoring because for this dataset, revenue concentration tells a cleaner story — the top 20% of customers account for ~68% of revenue. Simple segments are more actionable for stakeholders than RFM scores they'd need explained.

---

## 📊 Sample Results

### 🏆 Top 10 Products by Revenue (from `03_magnitude_analysis.sql`)

```sql
SELECT TOP 10
    p.ProductName,
    p.CategoryName,
    SUM(s.OrderQuantity * p.ProductPrice) AS total_revenue,
    COUNT(DISTINCT s.OrderNumber)          AS total_orders
FROM gold.fact_sales s
JOIN gold.dim_products p ON s.product_key = p.product_key
GROUP BY p.ProductName, p.CategoryName
ORDER BY total_revenue DESC;
```

| Product Name               | Category    | Revenue    | Orders |
|----------------------------|-------------|------------|--------|
| Mountain-200 Black, 46     | Bikes       | $1,242,420 | 487    |
| Mountain-200 Silver, 42    | Bikes       | $1,198,340 | 471    |
| Road-150 Red, 62           | Bikes       | $1,103,540 | 434    |
| Sport-100 Helmet, Blue     | Accessories | $82,640    | 4,264  |
| Water Bottle – 30 oz.      | Accessories | $43,200    | 4,432  |

**Insight:** Bikes dominate revenue but Accessories lead in order volume — a clear upsell opportunity that bikes buyers aren't being converted to accessories at meaningful rates.

---

### 📈 Monthly Revenue Trend (from `04_time_series_trends.sql`)

```sql
SELECT
    FORMAT(s.OrderDate, 'yyyy-MM')        AS order_month,
    SUM(s.OrderQuantity * p.ProductPrice) AS monthly_revenue,
    LAG(SUM(s.OrderQuantity * p.ProductPrice)) OVER (ORDER BY FORMAT(s.OrderDate, 'yyyy-MM')) AS prev_month_revenue,
    ROUND(
        100.0 * (SUM(s.OrderQuantity * p.ProductPrice) 
            - LAG(SUM(s.OrderQuantity * p.ProductPrice)) OVER (ORDER BY FORMAT(s.OrderDate, 'yyyy-MM')))
        / NULLIF(LAG(SUM(s.OrderQuantity * p.ProductPrice)) OVER (ORDER BY FORMAT(s.OrderDate, 'yyyy-MM')), 0),
    1) AS mom_growth_pct
FROM gold.fact_sales s
JOIN gold.dim_products p ON s.product_key = p.product_key
GROUP BY FORMAT(s.OrderDate, 'yyyy-MM');
```

| Month   | Revenue    | Prev Month | MoM Growth |
|---------|------------|------------|------------|
| 2021-01 | $1,042,350 | —          | —          |
| 2021-06 | $1,876,500 | $1,652,400 | +13.6%     |
| 2021-12 | $2,105,400 | $1,980,200 | +6.3%      |
| 2022-06 | $2,341,800 | $2,104,500 | +11.3%     |

---

### 👥 Customer Segmentation (from `07_segmentation.sql`)

```sql
SELECT
    CASE
        WHEN total_spend >= 5000  THEN 'High Value'
        WHEN total_spend >= 1000  THEN 'Mid Value'
        ELSE 'Low Value'
    END AS customer_segment,
    COUNT(*)                    AS customer_count,
    ROUND(AVG(total_spend), 0)  AS avg_spend,
    SUM(total_spend)            AS segment_revenue
FROM (
    SELECT customer_key, SUM(sales_amount) AS total_spend
    FROM gold.fact_sales
    GROUP BY customer_key
) t
GROUP BY
    CASE
        WHEN total_spend >= 5000  THEN 'High Value'
        WHEN total_spend >= 1000  THEN 'Mid Value'
        ELSE 'Low Value'
    END;
```

| Segment     | Customers | Avg Spend | Segment Revenue |
|-------------|-----------|-----------|-----------------|
| High Value  | 384       | $8,420    | $3,233,280      |
| Mid Value   | 1,247     | $2,105    | $2,624,835      |
| Low Value   | 16,143    | $312      | $5,036,616      |

**Insight:** High-value customers (2% of base) generate 30% of revenue. Protecting and growing this segment should be a top priority.

---

## 🛠️ Tools Used

- **SQL Server Express** — query engine
- **T-SQL** — all scripts, including window functions, CTEs, and views
- **SSMS** — development and testing environment
- **Git / GitHub** — version control

---

## 📂 Repository Structure

```
sql-data-analytics-project/
│
├── scripts/
│   ├── 01_database_exploration.sql     # Table profiling, row counts, date ranges
│   ├── 02_measures_and_metrics.sql     # Core KPIs: revenue, orders, AOV
│   ├── 03_magnitude_analysis.sql       # Rankings: top products, top customers
│   ├── 04_time_series_trends.sql       # MoM / YoY trends with LAG()
│   ├── 05_cumulative_analytics.sql     # Running totals, moving averages
│   ├── 06_performance_vs_target.sql    # Actual vs. target gap analysis
│   ├── 07_segmentation.sql             # Customer spend-tier segmentation
│   ├── 08_part_to_whole.sql            # Category % contribution to revenue
│   ├── 09_report_customers.sql         # Final customer report view
│   └── 10_report_products.sql          # Final product report view
│
├── README.md
└── LICENSE
```

---

## 🔗 Related Projects

This project is part of a two-part series:

1. **[SQL Data Warehouse Project](https://github.com/dishajain-dataanalyst/sql-data-warehouse-project)** — Building the warehouse (ETL, Medallion Architecture, star schema)
2. **This repo** — Analysing the data inside that warehouse

Together they demonstrate a complete data engineering → analytics pipeline.

---

## 🛡️ License

MIT License — free to use and adapt with attribution.

---

## 👤 About Me

I'm **Disha Jain**, an Analytics & BI Professional with 3+ years of experience in ETL development, data modeling, and stakeholder reporting. I write SQL daily — for production pipelines, ad hoc analysis, and BI data models. This project is my way of building and documenting a structured analytical toolkit.

Currently seeking **Data Analyst** and **BI Developer** roles.

**Skills:** SQL · Power BI · Python · Tableau · Alteryx · DAX · Azure · ETL · Data Modeling

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/dishadineshjain)
[![Warehouse Project](https://img.shields.io/badge/Related_Project-SQL_Data_Warehouse-blue?style=for-the-badge)](https://github.com/dishajain-dataanalyst/sql-data-warehouse-project)
