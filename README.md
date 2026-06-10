# 📦 Circle Stock Analysis

BigQuery SQL project analyzing inventory and sales data for **Circle**, a French sportswear brand.

---

## 🎯 Objective

Circle's stock team needed visibility into their inventory health. This project answers:

- Which products are in stock and which have run out?
- Which product categories have the highest shortage rates?
- How much stock value is held per category?
- Which top-selling products are at risk of running out — and how soon?

---

## 🗂️ Dataset

Two tables from BigQuery (`course15` dataset):

| Table | Rows | Description |
|---|---|---|
| `circle_stock` | 468 | Product catalog with stock levels, prices, and attributes |
| `circle_sales` | 20,679 | Daily sales records per product |

**Key columns in `circle_stock`:**

| Column | Description |
|---|---|
| `model_id` | Product model code |
| `model_name` | Product name (French) |
| `color` / `color_name` | Color code and description |
| `size` | Size (null for accessories) |
| `new` | New product flag (0/1) |
| `forecast_stock` | Forecasted stock level |
| `stock` | Current stock level |
| `price` | Unit price (€) |

**Key columns in `circle_sales`:**

| Column | Description |
|---|---|
| `date_date` | Sale date |
| `product_id` | Product identifier (joins to `circle_stock`) |
| `qty` | Quantity sold |

---

## 🪜 Project Steps

### 1. Data Exploration
Previewed both tables to understand structure, volume, and column types.

### 2. Primary Key Validation
- `circle_sales`: composite key `date_date + product_id` confirmed unique ✅
- `circle_stock`: `model_id` alone is NOT unique — composite key is `model_id + color + size` ✅

### 3. product_id Engineering
Derived a unique product identifier by concatenating the composite key:
```sql
CONCAT(model_id, "_", color, "_", IFNULL(size, "no-size")) AS product_id
```
> 💡 **Bug found & fixed:** 6 accessories had `size = NULL`, causing `CONCAT` to return NULL.  
> Fix: `IFNULL(size, "no-size")` — accessories get a placeholder instead of NULL.

### 4. Readable Table (`circle_stock_name`)
Renamed columns for clarity and added two derived fields:
- `product_id` — unique join key
- `product_name` — human-readable full product name for reporting

### 5. Category Labeling (`circle_stock_cat`)
No category column existed in the raw data. Used `REGEXP_CONTAINS + CASE WHEN` to classify products by reading their names:

| Category | Keywords matched |
|---|---|
| Aksesuar | tour de cou, tapis, gourde |
| T-shirt | t-shirt |
| Crop-top | brassiere, crop-top |
| Legging | legging |
| Şort | short |
| Üst | debardeur, haut |
| Other | everything else |

### 6. KPI Columns (`circle_stock_kpi`)
Added two business metrics per product variant:

| Column | Formula | Description |
|---|---|---|
| `in_stock` | `CASE WHEN stock = 0 THEN 0 ELSE 1` | Binary stock availability flag |
| `stock_value` | `price × stock` | Total inventory value in € |

### 7. Stock Reports
Built three levels of aggregation:
- **Global** — overall totals and averages across all 468 variants
- **By category** — shortage rate and stock value per product type
- **By model** — product-level stock dashboard (36 unique models)

### 8. Sales Analysis + Top Seller Flag
- Aggregated `circle_sales` to find top 10 best-selling products by total quantity
- Added `top_products` flag column (default 0) to `circle_stock_kpi`
- Used `UPDATE ... WHERE IN (subquery)` to set `top_products = 1` for the 10 best sellers

### 9. Days-of-Stock Calculation
Computed last 91 days of sales per product:
```sql
ROUND(SUM(qty) / 91, 2) AS avg_daily_qty_91
```
Then joined with stock data to calculate how many days of stock remain:
```sql
ROUND(forecast_stock / avg_daily_qty_91, 1) AS nb_days
```
Final alert query filtered for: `top_products = 1 AND stock < 50`

---

## 📊 Key Findings

### Overall Stock Health
| Metric | Value |
|---|---|
| Total product variants | 468 |
| Variants in stock | 405 |
| Out of stock | 63 |
| Shortage rate | **13.5%** |
| Total stock value | **€526,487** |
| Total units in stock | 22,474 |

### Stock by Category
| Category | Variants | In Stock | Shortage Rate | Stock Value |
|---|---|---|---|---|
| T-shirt | 109 | 103 | 5.5% | €206,762 |
| Other | 92 | 84 | 8.7% | €107,098 |
| Şort | 101 | 90 | 10.9% | €101,634 |
| Legging | 65 | 60 | 7.7% | €58,145 |
| Üst | 54 | 41 | **24.1%** | €39,910 |
| Crop-top | 36 | 21 | **41.7%** | €11,536 |
| Aksesuar | 11 | 6 | **45.5%** | €1,402 |

→ T-shirt is the largest and best-managed category (5.5% shortage).  
→ Crop-top and Aksesuar have critical shortage rates above 40%.

### 🚨 Critical Stock Alert — Top Sellers Running Out
| Product | Stock | Daily Sales | Days Left |
|---|---|---|---|
| T-shirt MAAM White - M | 5 | 18.71 | **0.3 days** |
| T-shirt MAAM White - XS | 35 | 45.78 | **0.8 days** |
| T-shirt MAAM Black - M | 48 | 32.52 | **1.5 days** |
| T-shirt MAAM White - L | 10 | 30.74 | **6.5 days** |

→ All four are top-selling variants of the same model.  
→ Immediate restocking required.

---

## 🛠️ SQL Techniques Used

| Technique | Purpose |
|---|---|
| `CONCAT` | Deriving `product_id` and `product_name` |
| `IFNULL` | Handling NULL size values for accessories |
| `IF` | Conditional logic in `product_name` construction |
| `REGEXP_CONTAINS` + `LOWER` | Text-based category labeling |
| `CASE WHEN` | Category assignment, `in_stock` flag |
| `UPDATE ... WHERE IN (subquery)` | Flagging top seller products |
| `JOIN ... USING` | Joining stock and sales tables |
| `DATE_SUB` | Filtering last 91 days of sales |
| `SUM` / `AVG` / `ROUND` | Aggregations and rounding |
| `COUNT` / `GROUP BY` / `HAVING` | PK validation and aggregation |
| `SAFE_DIVIDE` | Division without zero-division errors |
| `CREATE OR REPLACE TABLE` | Saving intermediate and final tables |
| `ORDER BY` | Sorting results |

---

## 🗃️ Output Tables

```
course15/
├── circle_stock_name       ← Stock table with product_id, product_name, renamed columns
├── circle_stock_cat        ← Adds model_type (category label)
├── circle_stock_kpi        ← Adds in_stock flag and stock_value
├── circle_stock_kpi_top    ← Adds top_products flag (1 = top seller)
├── top_products            ← Top 10 best-selling products by total qty
├── circle_sales_daily      ← Last 91 days: total qty and avg daily qty per product
```

---

## 🔧 Tools

- **Google BigQuery** — SQL engine and data warehouse
- **SQL** — All analysis done in standard SQL with BigQuery-specific functions

---

