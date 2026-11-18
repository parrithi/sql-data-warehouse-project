# End-to-End Data Warehouse in SQL Server: From Messy CRM and ERP Files to Business-Ready Analytics

**Six raw CSV files. Two source systems that disagree with each other. One clean star schema that answers a 29.4M revenue question in a single query.**

---

## The Story

A bicycle retailer keeps its customers and sales in a CRM, and its product categories, customer birthdates and locations in an ERP. Each system makes sense on its own. Together, they do not.

Customer IDs carry hidden prefixes. Country names appear as `DE`, `US` and `USA` in the same column. Order dates are stored as integers, some of them zeros. Birthdates sit in the future. Sales totals do not equal quantity times price. A single customer appears twice with different details. Nobody can trust a report built on that.

This project fixes all of it. It ingests the raw files, cleans and standardizes them, integrates both systems into one model, proves the result with automated quality checks, and then uses it to answer real business questions.

**In short:** 6 files, 3 layers, 6 bronze tables, 6 silver tables, 3 gold views, 2 stored procedures, 2 test suites, and a data catalog.

---

## Results at a Glance

All figures below come from the final Gold layer.

| Metric | Value |
|---|---|
| Source records processed | 18,494 CRM customers, 397 product records, 60,398 sales lines, plus 3 ERP tables |
| Customers in the final dimension | 18,484 |
| Active products in the final dimension | 295 |
| Sales transactions in the fact table | 60,398 lines across 27,659 orders |
| Total revenue | 29.36M |
| Average order value | about 1,061 |
| Sales rows with no matching customer or product | 0 |
| Order date range | 29 Dec 2010 to 28 Jan 2014 |

---

## Architecture

The warehouse follows the **Medallion Architecture**, so every layer has one clear job.

```
  SOURCES                        DATA WAREHOUSE (SQL Server)
 ┌────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
 │  CRM (3)   │     │   BRONZE    │     │   SILVER    │     │    GOLD     │
 │  CSV files │ ──▶ │  Raw data   │ ──▶ │  Cleaned &  │ ──▶ │  Business-  │
 │            │     │  as-is      │     │ standardized│     │  ready data │
 │  ERP (3)   │     │  (Tables)   │     │  (Tables)   │     │  (Views)    │
 │  CSV files │     │             │     │             │     │ Star schema │
 └────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                     Truncate+Insert     Truncate+Insert      No load needed
                     No transforms       Cleansing, rules     Integration,
                                                              business logic
```

| Layer | Purpose | Objects | Load | What happens |
|---|---|---|---|---|
| **Bronze** | Keep an untouched copy of the source | Tables | Batch, full load, truncate and insert | Nothing is changed |
| **Silver** | Fix quality problems once | Tables | Batch, full load, truncate and insert | Cleansing, standardization, normalization, derived columns, enrichment |
| **Gold** | Serve analysts and reports | Views | None (views read from Silver) | Integration, aggregation, business logic, star schema |

**Why this matters:** because Bronze is never altered, any number in a report can be traced back to its original source row. Because Silver fixes problems in one place, no analyst ever has to re-clean the data.

---

## Source Data

| System | File | Content |
|---|---|---|
| CRM | `cust_info.csv` | Customer profiles |
| CRM | `prd_info.csv` | Product records with start and end dates |
| CRM | `sales_details.csv` | Order line transactions |
| ERP | `CUST_AZ12.csv` | Customer birthdates and gender |
| ERP | `LOC_A101.csv` | Customer country |
| ERP | `PX_CAT_G1V2.csv` | Product category, subcategory and maintenance flag |

---

## The Data Problems I Found and How the Pipeline Solves Them

This is the heart of the project. Every row below was found in the raw data and handled in `proc_load_silver.sql`.

### Customers (CRM)
| Problem found | Fix applied |
|---|---|
| 4 rows with no customer ID | Excluded |
| 6 customers duplicated across multiple records | Kept only the latest record using `ROW_NUMBER()` ordered by create date |
| 39 first and last names with stray spaces | Trimmed |
| Marital status stored as `S` and `M` | Converted to `Single` and `Married` |
| Gender stored as `F` and `M`, missing for 4,578 raw rows | Converted to `Female` and `Male`, unknowns set to `n/a` |

### Products (CRM)
| Problem found | Fix applied |
|---|---|
| Category ID and product key packed into one string | Split into `cat_id` and `prd_key` |
| 200 records where the end date was before the start date | Recalculated each end date as one day before the next version's start date using `LEAD()` |
| 2 products with missing cost | Replaced with 0 |
| Product line stored as `M`, `R`, `S`, `T` | Mapped to Mountain, Road, Other Sales, Touring |

### Sales (CRM)
| Problem found | Fix applied |
|---|---|
| Dates stored as 8-digit integers, 19 order dates invalid or zero | Converted valid ones to `DATE`, set invalid ones to `NULL` |
| 30 rows where sales did not equal quantity times price | Recalculated sales from quantity and absolute price |
| 12 rows with missing or non-positive price | Derived price from sales divided by quantity |

### Customers and Locations (ERP)
| Problem found | Fix applied |
|---|---|
| 11,042 customer IDs with an `NAS` prefix | Removed, so they match CRM keys |
| Location IDs with dashes (`AW-00011000`) | Dashes removed to match CRM keys |
| 16 birthdates in the future | Set to `NULL` |
| Gender with mixed formats, blanks and trailing spaces | Standardized to `Female`, `Male` or `n/a` |
| Country coded as `DE`, `US`, `USA` alongside full names | Standardized to `Germany` and `United States` |
| 337 blank or missing countries | Set to `n/a` |

**A good example of integration logic:** about 4,500 customers have no gender in the CRM. In `gold.dim_customers`, the CRM stays the primary source and the ERP is used as a fallback. After this rule, only 15 customers remain unknown.

---

## Data Model (Star Schema)

```
                        ┌──────────────────────┐
                        │   gold.dim_customers │
                        │──────────────────────│
                        │ customer_key (PK)    │
                        │ customer_id          │
                        │ customer_number      │
                        │ first_name           │
                        │ last_name            │
                        │ country              │
                        │ marital_status       │
                        │ gender               │
                        │ birthdate            │
                        │ create_date          │
                        └──────────┬───────────┘
                                   │
┌──────────────────────┐   ┌───────┴──────────────┐
│   gold.dim_products  │   │   gold.fact_sales    │
│──────────────────────│   │──────────────────────│
│ product_key (PK)     ├───┤ order_number         │
│ product_id           │   │ product_key (FK)     │
│ product_number       │   │ customer_key (FK)    │
│ product_name         │   │ order_date           │
│ category_id          │   │ shipping_date        │
│ category             │   │ due_date             │
│ subcategory          │   │ sales_amount         │
│ maintenance          │   │ quantity             │
│ cost                 │   │ price                │
│ product_line         │   └──────────────────────┘
│ start_date           │
└──────────────────────┘
```

Design choices worth noting:
- **Surrogate keys** (`customer_key`, `product_key`) are generated with `ROW_NUMBER()`, so the model does not depend on source-system IDs.
- **Only current products** are kept in `dim_products` (`prd_end_dt IS NULL`), since the scope is the latest data and historization is not required.
- **Integration happens once**, in the Gold views. Analysts only ever write simple joins.

---

## What the Data Says

Findings from querying the Gold layer.

**Revenue is overwhelmingly driven by bikes.**
Bikes generate 96.5% of revenue (28.3M). Accessories add 0.70M and Clothing 0.34M.

**Road bikes lead, mountain bikes follow.**
Road Bikes bring in 14.5M, Mountain Bikes 10.0M and Touring Bikes 3.8M. By product line, Road accounts for about 50% of revenue, Mountain 35% and Touring 13%.

**Sales grew sharply in 2013.**
Revenue was 7.1M in 2011, dipped to 5.8M in 2012, then jumped to 16.3M in 2013, up roughly 180% year over year. The strongest single month was December 2013 at 1.87M. (2010 and 2014 contain only a few weeks of data.)

**Two markets generate over 60% of revenue.**
United States (31.2%) and Australia (30.9%) lead, followed by the United Kingdom (11.6%), Germany (9.9%), France (9.0%) and Canada (6.7%).

**The best-selling products are Mountain-200 bikes.**
The top three products by revenue are all Mountain-200 variants, each earning around 1.3M to 1.4M.

**Every customer in the dimension has made a purchase.**
All 18,484 customers appear in the sales data.

### Example queries on the Gold layer

```sql
-- Revenue by product category
SELECT p.category,
       SUM(f.sales_amount) AS total_sales
FROM gold.fact_sales f
LEFT JOIN gold.dim_products p ON p.product_key = f.product_key
GROUP BY p.category
ORDER BY total_sales DESC;

-- Revenue and orders by year
SELECT YEAR(order_date)            AS order_year,
       SUM(sales_amount)           AS total_sales,
       COUNT(DISTINCT order_number) AS total_orders
FROM gold.fact_sales
WHERE order_date IS NOT NULL
GROUP BY YEAR(order_date)
ORDER BY order_year;

-- Revenue by customer country
SELECT c.country,
       SUM(f.sales_amount) AS total_sales
FROM gold.fact_sales f
LEFT JOIN gold.dim_customers c ON c.customer_key = f.customer_key
GROUP BY c.country
ORDER BY total_sales DESC;
```

---

## Data Quality Testing

Quality is verified, not assumed. Two test scripts are included.

**Silver layer (`quality_checks_silver.sql`)**
- Null or duplicate primary keys
- Unwanted spaces in text fields
- Standardization of categorical values (marital status, product line, gender, country, maintenance)
- Invalid date ranges and wrong date order (start after end, order after ship or due date)
- Sales consistency: sales must equal quantity times price, with no nulls, zeros or negatives
- Out-of-range birthdates

**Gold layer (`quality_checks_gold.sql`)**
- Uniqueness of surrogate keys in both dimensions
- Referential integrity between the fact table and both dimensions (no orphan sales rows)

---

## Repository Structure

```
├── datasets/
│   ├── source_crm/
│   │   ├── cust_info.csv
│   │   ├── prd_info.csv
│   │   └── sales_details.csv
│   └── source_erp/
│       ├── CUST_AZ12.csv
│       ├── LOC_A101.csv
│       └── PX_CAT_G1V2.csv
│
├── docs/
│   ├── data_catalog.md          Column-level documentation of the Gold layer
│   └── naming_conventions.md    Rules for schemas, tables, columns and procedures
│
├── scripts/
│   ├── init_database.sql        Creates the DataWarehouse database and 3 schemas
│   ├── bronze/
│   │   ├── ddl_bronze.sql       Bronze table definitions
│   │   └── proc_load_bronze.sql Stored procedure: CSV to Bronze
│   ├── silver/
│   │   ├── ddl_silver.sql       Silver table definitions
│   │   └── proc_load_silver.sql Stored procedure: Bronze to Silver
│   └── gold/
│       └── ddl_gold.sql         Dimension and fact views
│
└── tests/
    ├── quality_checks_silver.sql
    └── quality_checks_gold.sql
```

---

## Engineering Practices

- **Naming conventions** documented and followed: snake_case, source-system prefixes in Bronze and Silver (`crm_`, `erp_`), `dim_` and `fact_` in Gold, `_key` for surrogate keys, `dwh_` for technical columns, and `load_<layer>` for procedures.
- **Technical metadata:** every Silver table carries a `dwh_create_date` audit column.
- **Re-runnable scripts:** every DDL script drops and recreates its objects, and every load procedure truncates before inserting, so the whole pipeline can be rebuilt at any time.
- **Observability:** load procedures print the duration of each table and the total batch, and use `TRY...CATCH` to report the error message, number and state.
- **Documentation:** a data catalog describes every Gold column with its type and meaning, so business users and analysts can work without reading the code.

---

## How to Run

1. Install SQL Server and SQL Server Management Studio (SSMS).
2. Copy the project and place the `datasets` folder at `C:\sql\dwh_project\datasets\`, or update the six file paths in `proc_load_bronze.sql`.
3. Run the scripts in this order:

| Step | Script | Result |
|---|---|---|
| 1 | `init_database.sql` | Creates the database and the bronze, silver and gold schemas |
| 2 | `ddl_bronze.sql` | Creates the 6 Bronze tables |
| 3 | `proc_load_bronze.sql`, then `EXEC bronze.load_bronze;` | Loads the raw CSV files |
| 4 | `ddl_silver.sql` | Creates the 6 Silver tables |
| 5 | `proc_load_silver.sql`, then `EXEC silver.load_silver;` | Cleans and loads Silver |
| 6 | `ddl_gold.sql` | Creates the 3 Gold views |
| 7 | `quality_checks_silver.sql` and `quality_checks_gold.sql` | Validates the result |

Note: `init_database.sql` drops and recreates the `DataWarehouse` database, so do not run it on a database you want to keep.

---

## Skills Demonstrated

| Area | Details |
|---|---|
| **T-SQL** | Stored procedures, views, CTE-style subqueries, window functions (`ROW_NUMBER`, `LEAD`), `CASE`, `BULK INSERT`, `TRY...CATCH`, string and date functions |
| **ETL design** | Multi-layer pipeline, full-load pattern, truncate and insert, load logging |
| **Data cleansing** | Deduplication, standardization, null handling, date repair, derived and recalculated fields |
| **Data modeling** | Star schema, surrogate keys, fact and dimension design, multi-source integration |
| **Data quality** | Rule-based validation, referential integrity checks |
| **Documentation** | Data catalog, naming standards, architecture documentation |
| **Business analysis** | Revenue by category, product line, country and time |

---

## Possible Next Steps

- Build a Power BI dashboard on top of the Gold views
- Add incremental loading in place of full loads
- Schedule the pipeline with SQL Server Agent
- Add history tracking (slowly changing dimensions) for customers and products

---

## About the Author

**Parrithi Ela** is an MBA graduate in Business Analytics and Finance from Chennai, India, with hands-on experience in SQL, Power BI, DAX, Power Query and Excel, plus a background in equity research and Indian capital markets. This project reflects how I like to work with data: understand the business problem, clean the data until it can be trusted, model it clearly, and turn it into answers that people can act on.
