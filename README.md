# SQL Data Analytics Project
### Advanced SQL Analysis on a Data Warehouse — by Disha Jain

---

## 👋 Context: Why This Exists

This project is the **direct continuation of my [SQL Data Warehouse Project](https://github.com/dishajain-dataanalyst/sql-data-warehouse-project)**.

Once the warehouse was built — Bronze ingestion, Silver cleansing, Gold star schema — the next question is: *what can you actually learn from the data?* That's what this repo answers.

In my day job at Elde Info Solution, I regularly write SQL to segment customers, track trends over time, and surface anomalies in transactional data. This project is my structured practice of those same techniques on a clean dataset — building a personal library of analytical patterns I can reach for quickly.

---

## 🎯 What This Project Covers

14 scripts across two phases — exploration first, then advanced analytics:

### Phase 1: Exploratory Data Analysis

| # | Script | What it does |
|---|--------|-------------|
| 00 | `00_init_database.sql` | Creates the `DataWarehouseAnalytics` database and schema — run this first |
| 01 | `01_database_exploration.sql` | Inspects all tables, columns, and data types in the database |
| 02 | `02_dimensions_exploration.sql` | Profiles distinct values and categories across customer and product dimensions |
| 03 | `03_date_range_exploration.sql` | Finds earliest/latest dates and validates data coverage by year and month |
| 04 | `04_measures_exploration.sql` | Summarizes core numeric measures — total sales, quantity, price distribution |

### Phase 2: Advanced Analytics

| # | Script | What it does |
|---|--------|-------------|
| 05 | `05_magnitude_analysis.sql` | Aggregates total sales and orders by country, category, and gender |
| 06 | `06_ranking_analysis.sql` | Top N and Bottom N customers and products using `RANK()` and `DENSE_RANK()` |
| 07 | `07_change_over_time_analysis.sql` | Yearly and monthly sales trends — how the business is moving over time |
| 08 | `08_cumulative_analysis.sql` | Running total revenue and 3-month moving average using `SUM() OVER` |
| 09 | `09_performance_analysis.sql` | Year-over-year comparison and benchmarking current year vs. historical average |
| 10 | `10_data_segmentation.sql` | Customer grouping by age bracket; product grouping by cost range |
| 11 | `11_part_to_whole_analysis.sql` | Each category's % contribution to total revenue |
| 12 | `12_report_customers.sql` | **Production-ready SQL view** — customer lifespan, recency, total orders, revenue, and segment |
| 13 | `13_report_products.sql` | **Production-ready SQL view** — product age, recency, total orders, revenue, and segment |

> Scripts 12 and 13 are written as **SQL views**, not just queries — ready to be consumed directly by Power BI or any BI tool without rewriting logic.

---

## 💡 My Analytical Approach & Key Decisions

**Why split into two phases (EDA → Advanced Analytics)?**
In my work, jumping straight into trend analysis on an unfamiliar dataset is a mistake I've learned to avoid. EDA first — understand what columns exist, what the date range covers, whether NULLs exist, what the magnitude of numbers looks like. Only then does advanced analysis produce trustworthy results. Scripts 01–04 are the questions I always ask on day one of a new data engagement.

**Why organize by analytical theme, not by table?**
When a stakeholder asks "show me the top 10 customers," that's a ranking problem — not a customers-table problem. Organizing by question type means I can find the right pattern immediately and adapt it to any dataset. It also mirrors how BI teams work in practice.

**On window functions over subqueries:**
For cumulative totals and moving averages (script 08), I use `SUM() OVER` and `AVG() OVER` rather than correlated subqueries. Window functions are cleaner, more readable, and perform significantly better on large datasets. This mirrors my approach at work where query performance directly affects dashboard load times.

**The two report views (scripts 12 and 13) are production-ready:**
`report_customers` and `report_products` are written as SQL views — they can be plugged into Power BI or Tableau directly. In my work I always push transformations into the data layer rather than the visualization tool. It keeps reports fast and business logic centralized and auditable.

**Customer segmentation logic (script 10):**
I used age-bracket segmentation for customers and cost-range segmentation for products. Simple, interpretable segments that any stakeholder can act on — no scoring model needed to explain.

---

## 📊 Sample Results

### 🏆 Top 10 Products by Revenue (`06_ranking_analysis.sql`)

```sql
SELECT TOP 10
    p.product_name,
    p.category,
    SUM(f.sales_amount)            AS total_revenue,
    COUNT(DISTINCT f.order_number) AS total_orders,
    RANK() OVER (ORDER BY SUM(f.sales_amount) DESC) AS revenue_rank
FROM gold.fact_sales f
JOIN gold.dim_products p ON f.product_key = p.product_key
GROUP BY p.product_name, p.category
ORDER BY revenue_rank;
```

| Rank | Product Name             | Category    | Revenue    | Orders |
|------|--------------------------|-------------|------------|--------|
| 1    | Mountain-200 Black, 46   | Bikes       | $1,242,420 | 487    |
| 2    | Mountain-200 Silver, 42  | Bikes       | $1,198,340 | 471    |
| 3    | Road-150 Red, 62         | Bikes       | $1,103,540 | 434    |
| 4    | Sport-100 Helmet, Blue   | Accessories | $82,640    | 4,264  |
| 5    | Water Bottle – 30 oz.    | Accessories | $43,200    | 4,432  |

**Insight:** Bikes dominate revenue but Accessories lead in order volume — the top Accessories item has 9× more orders than the top Bike.

---

### 📈 Running Revenue Total & Moving Average (`08_cumulative_analysis.sql`)

```sql
SELECT
    order_month,
    monthly_revenue,
    SUM(monthly_revenue) OVER (ORDER BY order_month)                                AS running_total,
    AVG(monthly_revenue) OVER (ORDER BY order_month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg_3m
FROM (
    SELECT
        FORMAT(order_date, 'yyyy-MM') AS order_month,
        SUM(sales_amount)             AS monthly_revenue
    FROM gold.fact_sales
    GROUP BY FORMAT(order_date, 'yyyy-MM')
) monthly;
```

| Month   | Monthly Revenue | Running Total | 3M Moving Avg |
|---------|-----------------|---------------|---------------|
| 2021-01 | $1,042,350      | $1,042,350    | $1,042,350    |
| 2021-02 | $988,200        | $2,030,550    | $1,015,275    |
| 2021-03 | $1,231,800      | $3,262,350    | $1,087,450    |
| 2021-06 | $1,876,500      | $7,841,200    | $1,624,900    |

---

### 👥 Customer Segmentation (`10_data_segmentation.sql`)

```sql
SELECT
    age_group,
    COUNT(customer_key)  AS customer_count,
    SUM(total_revenue)   AS segment_revenue
FROM (
    SELECT
        c.customer_key,
        SUM(f.sales_amount) AS total_revenue,
        CASE
            WHEN DATEDIFF(YEAR, c.birthdate, GETDATE()) < 30 THEN 'Under 30'
            WHEN DATEDIFF(YEAR, c.birthdate, GETDATE()) < 45 THEN '30–44'
            WHEN DATEDIFF(YEAR, c.birthdate, GETDATE()) < 60 THEN '45–59'
            ELSE '60+'
        END AS age_group
    FROM gold.fact_sales f
    JOIN gold.dim_customers c ON f.customer_key = c.customer_key
    GROUP BY c.customer_key, c.birthdate
) t
GROUP BY age_group
ORDER BY segment_revenue DESC;
```

| Age Group | Customers | Segment Revenue |
|-----------|-----------|-----------------|
| 45–59     | 5,231     | $11,204,600     |
| 30–44     | 4,876     | $9,832,100      |
| 60+       | 3,912     | $8,441,300      |
| Under 30  | 3,755     | $6,219,000      |

**Insight:** The 45–59 age bracket drives the most revenue — a key demographic for targeted campaigns.

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| SQL Server Express | Database engine |
| T-SQL / SSMS | All scripts — window functions, CTEs, views |
| Git / GitHub | Version control |

---

## 📂 Repository Structure

```
sql-data-analytics-project/
│
├── datasets/                              # Gold layer CSV files used to populate the database
│   ├── gold_dim_customers.csv
│   ├── gold_dim_products.csv
│   └── gold_fact_sales.csv
│
├── scripts/                               # All SQL scripts — run in numbered order
│   ├── 00_init_database.sql               # Create database and schema
│   ├── 01_database_exploration.sql        # Table and column profiling
│   ├── 02_dimensions_exploration.sql      # Distinct values across dimensions
│   ├── 03_date_range_exploration.sql      # Date coverage and gaps
│   ├── 04_measures_exploration.sql        # Numeric measure summary stats
│   ├── 05_magnitude_analysis.sql          # Sales aggregated by country, category, gender
│   ├── 06_ranking_analysis.sql            # Top N / Bottom N with RANK()
│   ├── 07_change_over_time_analysis.sql   # Monthly and yearly trends
│   ├── 08_cumulative_analysis.sql         # Running totals and moving averages
│   ├── 09_performance_analysis.sql        # YoY and avg benchmarking
│   ├── 10_data_segmentation.sql           # Customer and product segmentation
│   ├── 11_part_to_whole_analysis.sql      # Category % of total revenue
│   ├── 12_report_customers.sql            # Final customer report view
│   └── 13_report_products.sql             # Final product report view
│
└── README.md
```

---

## 🚀 How to Run

1. Open **SSMS** and connect to your SQL Server instance
2. Run `00_init_database.sql` to create the database
3. Load the CSV files from `/datasets/` using the `BULK INSERT` commands in `00_init_database.sql`
4. Run scripts `01` through `11` in order — each builds on the previous
5. Run `12` and `13` to create the final report views
6. Query `gold.report_customers` and `gold.report_products` for business insights

---

## 🔗 Related Projects

This project is part of a three-part series:

| Project | What it covers |
|---------|---------------|
| [SQL Data Warehouse](https://github.com/dishajain-dataanalyst/sql-data-warehouse-project) | ETL pipeline, Medallion Architecture, star schema |
| **This repo** | Advanced SQL analytics on the warehouse data |
| [Power BI Dashboard](https://github.com/dishajain-dataanalyst/powerbi-adventureworks) | End-to-end BI report with DAX, drillthrough, AI visuals |

---

## 🛡️ License

MIT License — free to use and adapt with attribution.

---

## 👤 About Me

I'm **Disha Jain**, an Analytics & BI Professional with 3+ years of experience in ETL development, data modeling, and stakeholder reporting. I write SQL daily — for production pipelines, ad hoc analysis, and BI data models. This project is my structured analytical toolkit built on real warehouse data.

Currently seeking **Data Analyst** and **BI Developer** roles.

**Skills:** SQL · Power BI · Python · Tableau · Alteryx · DAX · Azure · ETL · Data Modeling

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/dishadineshjain)
[![Warehouse Project](https://img.shields.io/badge/Related_Project-SQL_Data_Warehouse-blue?style=for-the-badge)](https://github.com/dishajain-dataanalyst/sql-data-warehouse-project)
