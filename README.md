# Sales Insights: SQL + Power BI Analytics Project

An end-to-end sales analytics project: raw transactional data in **MySQL**, cleaned and enriched with profit fields, then explored with SQL and presented in a 3-page interactive **Power BI** dashboard.

## Business Problem

Sales leadership of a hardware retailer selling across India needed one place to answer:

- How much revenue and volume are we generating, and where does it come from?
- Which markets, customers and products matter most?
- Are we actually *profitable* in those markets, or just big?
- How does this year compare with last year?

## Dataset

| Table | Rows | Description |
|---|---|---|
| `transactions` | 148,395 (v2) | Orders: product, customer, market, date, quantity, sales amount, currency, profit margin %, profit margin, cost price |
| `customers` | 38 | 19 Brick & Mortar, 19 E-Commerce |
| `products` | 279 | Own Brand (191) and Distribution (88). Covers only `Prod001` to `Prod279`; transactions reference 338 distinct products (see Data Quality Notes) |
| `markets` | 17 | City, zone (North / South / Central); 14 have sales |
| `date` | 1,126 | Calendar table (date, year, month name, `cy_date`) |

**Coverage:** Oct 2017 to Jun 2020 · **Database:** MySQL 8.0 (`sales`)

### Two database versions

| File | Transactions | What it is |
|---|---|---|
| `db_dump.sql` | 150,283 | Raw load. Has a trailing carriage return in `currency` (`'INR\r'`), 1,611 rows with zero or negative sales amount, and 281 groups of duplicate rows. No profit columns. |
| `db_dump_version_2.sql` | 148,395 | Cleaned (no duplicates, no zero/negative sales, clean currency values) and enriched with `profit_margin_percentage`, `profit_margin`, `cost_price`. **This is the version the dashboard uses.** |

## Key Findings (v2 data)

- **Total revenue: about ₹984.8M** from 2.43M units sold.
- **Revenue is highly concentrated.** Delhi NCR alone is ~53% of revenue, with Mumbai (~15%) and Ahmedabad (~13%) next. A single customer, Electricalsara Stores, accounts for ~42%.
- **Overall profit margin is only ~2.5%** (₹24.7M). Big markets are not the most profitable: Delhi NCR earns 2.3% while Patna (4.1%) and Surat (4.9%) earn nearly double on far less revenue.
- **Loss-making markets:** Bengaluru (-20.8%) and Kanpur (-0.5%) over the full period; Lucknow is the loss-maker in 2020 (-2.7%).
- **Year-over-year:** 2018 was the peak (~₹413.7M), 2019 came in at ~₹336.0M, and 2020 reached ~₹142.2M through June.

## Dashboard (Power BI)

| Page | What it shows |
|---|---|
| **Key Insights** | Revenue and quantity KPIs, revenue and quantity by market, top 5 customers, top 5 products, revenue trend |
| **Profit Analysis** | Profit margin % and contribution % by market, revenue contribution %, top customers with profit metrics, total profit margin |
| **Performance Insights** | Revenue vs last year with profit margin % overlay, zone / market / customer / product drill-down, profit target parameter |

**Key Insights** (screenshot filtered to 2020)

![Key Insights](images/key-insights.png)

**Profit Analysis**

![Profit Analysis](images/profit-analysis.png)

**Performance Insights** (revenue vs last year, with a 2% profit target slicer; markets below target show in red)

![Performance Insights](images/performance-insights.png)

**2020 snapshot (Jan to Jun):** ₹142.2M revenue, 350K units, ₹2.1M profit (1.4% margin, below the overall 2.5%). Delhi NCR is 54.7% of revenue but earns only 0.6% margin; Lucknow is loss-making at -2.7%; Bhubaneshwar, Hyderabad and Chennai lead on margin (10.5%, 6.7%, 6.3%). Electricalsara Stores is 46.2% of 2020 revenue.

Interactive slicers for year and date on every page. Core measures: `Revenue`, `Sales Qty`, `Revenue LY`, `Revenue Contribution %`, `Profit Margin %`, `Profit Margin Contribution %`, `Total Profit Margin`.

## SQL Analysis

Run against `db_dump_version_2.sql`. For the raw v1 dump, wrap `currency` in `TRIM()` because of the trailing carriage return.

```sql
-- 1. All customers
SELECT * FROM customers;

-- 2. Total number of customers
SELECT COUNT(*) FROM customers;

-- 3. Transactions in Chennai (market code Mark001)
SELECT * FROM transactions WHERE market_code = 'Mark001';

-- 4. Distinct products sold in Chennai
SELECT DISTINCT product_code FROM transactions WHERE market_code = 'Mark001';

-- 5. Transactions in US dollars
SELECT * FROM transactions WHERE currency = 'USD';

-- 6. Transactions in 2020 (join to the date table)
SELECT t.*, d.*
FROM transactions t
INNER JOIN date d ON t.order_date = d.date
WHERE d.year = 2020;

-- 7. Total revenue in 2020
SELECT SUM(t.sales_amount) AS revenue_2020
FROM transactions t
INNER JOIN date d ON t.order_date = d.date
WHERE d.year = 2020
  AND t.currency IN ('INR', 'USD');

-- 8. Total revenue in January 2020
SELECT SUM(t.sales_amount) AS revenue_jan_2020
FROM transactions t
INNER JOIN date d ON t.order_date = d.date
WHERE d.year = 2020
  AND d.month_name = 'January'
  AND t.currency IN ('INR', 'USD');

-- 9. Total revenue in 2020 in Chennai
SELECT SUM(t.sales_amount) AS revenue_chennai_2020
FROM transactions t
INNER JOIN date d ON t.order_date = d.date
WHERE d.year = 2020
  AND t.market_code = 'Mark001';

-- 10. Revenue and profit margin % by market
SELECT m.markets_name,
       SUM(t.sales_amount) AS revenue,
       ROUND(100 * SUM(t.profit_margin) / SUM(t.sales_amount), 1) AS profit_margin_pct
FROM transactions t
JOIN markets m ON m.markets_code = t.market_code
GROUP BY m.markets_name
ORDER BY revenue DESC;
```

## Data Quality Notes

- **Product dimension is incomplete.** The `products` table stops at `Prod279`, but 54,599 transactions (about ₹469M, 48% of revenue) use `Prod280` to `Prod339`. In Power BI these show as **(Blank)**, which is why the "Top 5 Products" chart on the Key Insights page shows a single blank bar of ₹65M for 2020. Fix by adding the missing codes to `products` (or sourcing the visual from `transactions.product_code`).
- `currency` and `product_type` contain trailing `\r` characters from a Windows line-ending import. v2 fixes `currency`; `product_type` still has it, so use `TRIM()`.
- Two `USD` transactions are not converted to INR (₹750 total in the data, immaterial).
- `Bhopal` appears under two market codes (`Mark007`, `Mark013`), and `Mark097` (New York) and `Mark999` (Paris) have no sales.
- Column `custmer_name` in `customers` is misspelled in the source schema and kept as-is for compatibility.

## How to Run

1. Install MySQL 8.0+ and Power BI Desktop.
2. Load the data:
   ```bash
   mysql -u root -p < db_dump_version_2.sql
   ```
3. Open `dashboard.pbix` in Power BI Desktop, then **Home → Transform data → Data source settings** and point it at your MySQL server.
4. Refresh.

## Repository Structure

```
├── README.md
├── dashboard.pbix
├── db_dump.sql                 # raw data
├── db_dump_version_2.sql       # cleaned + profit columns
├── report/Sales_Insights_Report.docx
└── images/                     # dashboard screenshots
```

## Tools

MySQL 8 · SQL (joins, aggregations, filtering) · Power BI · DAX · Data modeling (star schema) · Data cleaning
