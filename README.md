# PowerBI-Project

> A professional end-to-end Power BI project covering data modelling, DAX measures, report design, and publishing — built on a retail sales dataset spanning 2022–2023 across multiple US regions, product categories, and customer segments.

[![Power BI](https://img.shields.io/badge/Power%20BI-Desktop%202.x-yellow?logo=powerbi&logoColor=white)](https://powerbi.microsoft.com/)
[![Dataset](https://img.shields.io/badge/Dataset-Retail%20Sales-blue)](data/sales_data.csv)

---

## Project Overview

This project demonstrates a complete Power BI analytics solution built on a fictional **US retail company's sales data** (2022–2023). It covers the full workflow — from raw CSV ingestion through to a published, interactive report.

### Business Questions Answered

| # | Question | Visual Type |
|---|---|---|
| 1 | What is total revenue, profit, and order volume YTD vs prior year? | KPI cards + line chart |
| 2 | Which product categories and sub-categories drive the most profit? | Treemap + bar chart |
| 3 | How does sales performance vary by US region and state? | Filled map + matrix |
| 4 | Which customer segments have the highest average order value? | Donut chart + table |
| 5 | Who are the top-performing sales reps? | Bar chart with rank |
| 6 | What is the trend of discount rate vs profit margin over time? | Dual-axis line chart |
| 7 | What is the month-over-month revenue growth rate? | Line chart + smart narrative |

---

## Dataset

The dataset is in [`data/sales_data.csv`](data/sales_data.csv). It contains **102 sales orders** placed between January 2022 and December 2023, covering 20 customers, 12 products, 4 US regions, and 5 sales representatives.

### Schema

| Column | Type | Description |
|---|---|---|
| `order_id` | Text | Unique order identifier (format: `ORD-YYYY-NNNN`) |
| `order_date` | Date | Date the order was placed |
| `ship_date` | Date | Date the order shipped |
| `customer_id` | Text | Unique customer identifier |
| `customer_name` | Text | Customer full name |
| `customer_segment` | Text | `Consumer` / `Corporate` / `Home Office` |
| `region` | Text | US sales region (`East`, `West`, `South`, `Central`) |
| `country` | Text | Country of sale |
| `state` | Text | US state |
| `city` | Text | City |
| `product_id` | Text | Unique product identifier |
| `product_name` | Text | Product display name |
| `category` | Text | `Technology` / `Furniture` / `Office Supplies` |
| `sub_category` | Text | Product sub-category (e.g., `Computers`, `Chairs`) |
| `quantity` | Integer | Units ordered |
| `unit_price` | Decimal | Listed unit price (USD) |
| `discount` | Decimal | Discount fraction applied (0.00 – 1.00) |
| `sales` | Decimal | Actual revenue after discount (USD) |
| `profit` | Decimal | Gross profit (USD) |
| `profit_margin` | Decimal | Profit margin (%) |
| `sales_rep` | Text | Name of the responsible sales representative |

> A full data dictionary is available at [`data/data_dictionary.csv`](data/data_dictionary.csv).

### Sample Records

```
order_id,     order_date, customer_name,   category,   sub_category, quantity, sales,   profit, profit_margin
ORD-2022-0001,2022-01-03, James Harrison,  Technology, Computers,    2,        1619.98, 324.00, 20.0
ORD-2022-0037,2022-06-05, Jessica Martinez,Furniture,  Chairs,       4,        4416.60, 441.66, 10.0
ORD-2023-0026,2023-10-02, Linda Taylor,    Technology, Computers,    10,       8099.91, 1620.00,20.0
```

---


## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/powerbi-project.git
cd powerbi-project
```

### 2. Open Power BI Desktop

Launch **Power BI Desktop** and select **Get Data → Text/CSV**.

### 3. Load the dataset

Navigate to `data/sales_data.csv` and click **Load**.

In the **Power Query Editor**:

```
1. Select the order_date and ship_date columns
   → Transform → Data Type → Date

2. Select the discount column
   → Transform → Data Type → Decimal Number

3. Select sales, profit, unit_price columns
   → Transform → Data Type → Fixed Decimal Number (Currency)

4. Click Close & Apply
```

### 4. Apply the custom theme (optional)

```
View → Themes → Browse for themes → select assets/theme.json
```

### 5. Build the data model

Follow the [Data Model](#data-model) section below to create the Date table and relationships.

---

## Data Model

This project uses a **star schema** design — a central fact table surrounded by dimension tables extracted from the flat CSV.


### Create a Date Table (DAX)

In the **Model** view, create a calculated table named `DimDate`:

```dax
DimDate =
VAR MinDate = MIN(FactSales[order_date])
VAR MaxDate = MAX(FactSales[order_date])
RETURN
ADDCOLUMNS(
    CALENDAR(MinDate, MaxDate),
    "Year",         YEAR([Date]),
    "Quarter",      "Q" & QUARTER([Date]),
    "Quarter No",   QUARTER([Date]),
    "Month No",     MONTH([Date]),
    "Month Name",   FORMAT([Date], "MMMM"),
    "Month Short",  FORMAT([Date], "MMM"),
    "Week No",      WEEKNUM([Date]),
    "Day of Week",  FORMAT([Date], "dddd"),
    "Day No",       WEEKDAY([Date], 2),
    "Is Weekend",   IF(WEEKDAY([Date], 2) >= 6, TRUE, FALSE),
    "Year-Month",   FORMAT([Date], "YYYY-MM")
)
```

Mark `DimDate` as a **Date Table** (Table Tools → Mark as date table → Date column).

---

## DAX Measures Reference

Create a dedicated `_Measures` table to keep all measures organised.

### Core Metrics

```dax
-- Total Sales
Total Sales =
SUM(FactSales[sales])

-- Total Profit
Total Profit =
SUM(FactSales[profit])

-- Profit Margin %
Profit Margin % =
DIVIDE([Total Profit], [Total Sales], 0)

-- Total Orders
Total Orders =
DISTINCTCOUNT(FactSales[order_id])

-- Average Order Value
Avg Order Value =
DIVIDE([Total Sales], [Total Orders], 0)

-- Total Units Sold
Total Units Sold =
SUM(FactSales[quantity])
```

### Time Intelligence

```dax
-- Sales Prior Year
Sales PY =
CALCULATE([Total Sales], SAMEPERIODLASTYEAR(DimDate[Date]))

-- Year-over-Year Growth
YoY Growth % =
DIVIDE([Total Sales] - [Sales PY], [Sales PY], 0)

-- Sales YTD
Sales YTD =
TOTALYTD([Total Sales], DimDate[Date])

-- Sales MTD
Sales MTD =
TOTALMTD([Total Sales], DimDate[Date])

-- Rolling 3-Month Sales
Sales Rolling 3M =
CALCULATE(
    [Total Sales],
    DATESINPERIOD(DimDate[Date], LASTDATE(DimDate[Date]), -3, MONTH)
)
```

### Ranking

```dax
-- Product Rank by Sales
Product Rank =
RANKX(
    ALLSELECTED(DimProduct[product_name]),
    [Total Sales],
    ,
    DESC,
    DENSE
)

-- Sales Rep Rank
Sales Rep Rank =
RANKX(
    ALLSELECTED(FactSales[sales_rep]),
    [Total Sales],
    ,
    DESC,
    DENSE
)
```

### Conditional Metrics

```dax
-- High-Value Orders (> $1,000)
High Value Orders =
CALCULATE(
    [Total Orders],
    FactSales[sales] > 1000
)

-- Low Margin Flag
Low Margin Flag =
IF([Profit Margin %] < 0.15, "⚠ Low Margin", "✓ Healthy")
```

---

## Report Pages

Build your report across the following pages for a comprehensive executive dashboard:

### Page 1 — Executive Summary
- **KPI Cards:** Total Sales | Total Profit | Total Orders | Profit Margin %
- **Line Chart:** Monthly Sales vs Prior Year (dual series)
- **Bar Chart:** Sales by Region (sorted descending)
- **Slicer:** Year | Customer Segment

### Page 2 — Product Performance
- **Treemap:** Sales by Category → Sub-Category
- **Table:** Product Name | Total Sales | Profit | Profit Margin % | Rank
- **Bar Chart:** Top 5 Sub-Categories by Profit
- **Slicer:** Category

### Page 3 — Regional Analysis
- **Filled Map:** Sales by State (bubble size = profit)
- **Matrix:** Region → State | Total Sales | YoY Growth %
- **Bar Chart:** Sales by City (Top 10)
- **Slicer:** Region | Year

### Page 4 — Customer & Sales Rep Analysis
- **Donut Chart:** Sales split by Customer Segment
- **Table:** Customer Name | Segment | Total Sales | Total Orders | Avg Order Value
- **Bar Chart:** Sales Rep performance (with target line)
- **Scatter Plot:** Orders vs Sales per Customer (bubble = profit)

### Page 5 — Discount & Margin Analysis
- **Dual-Axis Line Chart:** Average Discount % vs Profit Margin % over time
- **Bar Chart:** Profit Margin % by Sub-Category
- **Table:** Orders with Discount > 10% and Profit Margin < 15%
- **Slicer:** Category | Sales Rep

---

## Visualisation Best Practices

### Design Principles
- Use a **consistent colour palette** across all pages — assign one colour per category and stick to it.
- Apply **conditional formatting** to tables: red for negative margins, green for above-target metrics.
- Keep each page to **one business question** — avoid cramming too many visuals onto a single canvas.
- Always provide a **clear page title** and subtitle explaining the audience and time period.
- Use **bookmarks** to create toggle states (e.g., show/hide a detail table).

### Accessibility
- Set **alt text** on every visual (Format → General → Alt text).
- Ensure colour contrast meets WCAG AA standards — do not rely on colour alone to convey meaning.
- Use **tooltips** to surface additional context without cluttering the canvas.

### Slicers
- Place slicers consistently on the left or top of every page.
- Use a **sync slicers** panel (View → Sync slicers) so year/segment filters apply across all pages.
- Prefer **dropdown slicers** over list slicers to save canvas space.

---

## Publishing to Power BI Service

### Step-by-step

```
1. Sign in to Power BI Desktop with your Microsoft account.

2. File → Publish → Publish to Power BI

3. Select a workspace (use a dedicated workspace per project,
   not "My Workspace", for team collaboration).

4. Open Power BI Service (app.powerbi.com) → navigate to your workspace.

5. Click the report → Share → Manage permissions to control access.
```

### Scheduled Refresh (for live data)

```
Workspace → Dataset → Settings → Scheduled Refresh
→ Configure gateway (for on-prem sources) or use cloud connectors
→ Set frequency: Daily / Hourly
→ Add data source credentials
```

### Row-Level Security (RLS)

Define RLS roles to restrict data by sales rep or region:

```dax
-- In Manage Roles → Create role: "SalesRep"
-- Table: FactSales | Filter expression:
[sales_rep] = USERPRINCIPALNAME()
```

Assign users to roles in the Power BI Service under **Dataset → Security**.

---

## Performance Tips

| Issue | Root Cause | Fix |
|---|---|---|
| Slow report load | Too many visuals on one page | Split into multiple focused pages |
| Slow DAX measures | CALCULATE with large row context | Use variables (`VAR`) to pre-compute values |
| Large .pbix file | Images or uncompressed data embedded | Import data in DirectQuery or use external images via URL |
| Slow time intelligence | No date table or wrong relationships | Always use a dedicated `DimDate` table marked as a date table |
| Cross-filter lag | Bidirectional relationships | Use unidirectional relationships; use `CROSSFILTER()` in DAX when needed |
| High memory usage | Importing unnecessary columns | Remove unused columns in Power Query before loading |

### Import vs DirectQuery vs Live Connection

| Mode | Best for | Limitation |
|---|---|---|
| **Import** | Small–medium datasets (< 1 GB) | Data refreshes on schedule only |
| **DirectQuery** | Large or real-time data | Every visual triggers a live query — can be slow |
| **Live Connection** | SSAS / Power BI datasets | No Power Query transformations allowed |

---


