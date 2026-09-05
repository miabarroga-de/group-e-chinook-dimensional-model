# Chinook Data Engineering Project

## Project Overview

This project, created by Group E, builds an end-to-end data engineering and analytics workflow using the **Chinook dataset** in Databricks. 

The notebook follows this sequence:

**Raw CSV Files → Bronze/Imported Tables → Silver Transformations → Data Quality Check → Analytics / Gold Outputs**

The main goals are to:

- Import the Chinook CSV files into Databricks tables
- Apply cleaning and standardization in the Silver layer
- Join related tables to create analytics-ready datasets
- Create customer spending tiers using dynamic percentiles
- Create invoice and invoice-line data for revenue analysis
- Validate invoice totals against invoice-line calculations
- Answer business questions using SQL analytics

---

## Architecture

```text
                         RAW CSV FILES
                              │
                              ▼
                    ┌──────────────────┐
                    │ IMPORTED TABLES  │
                    │   Chinook Raw    │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │  SILVER LAYER    │
                    │ Cleaned tables   │
                    └────────┬─────────┘
                             │
                             ▼
                 ┌─────────────────────────┐
                 │ DATA QUALITY CHECK      │
                 │ Invoice reconciliation  │
                 └────────────┬────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ GOLD / ANALYTICS │
                    │ SQL outputs      │
                    └────────┬─────────┘
                             │
             ┌───────────────┼────────────────┐
             ▼               ▼                ▼
       Revenue by       Customer         Sales / Track /
       Genre/Country    Spending Tier    Employee / Region
```

---

# 1. Dataset

The project uses the Chinook music store dataset.

The notebook imports the following source tables:

| Source Table | Purpose |
|---|---|
| `Artist` | Artist information |
| `Album` | Album information and artist relationships |
| `Genre` | Music genre lookup |
| `MediaType` | Media type lookup |
| `Track` | Track information, pricing, album and genre relationships |
| `Playlist` | Playlist information |
| `PlaylistTrack` | Playlist-to-track relationships |
| `Employee` | Employee and support representative information |
| `Customer` | Customer information and support representative |
| `Invoice` | Customer invoices and invoice totals |
| `InvoiceLine` | Individual tracks purchased within invoices |

The notebook uses Databricks SQL `read_files()` with explicitly defined schemas to import the CSV files.

---

# 2. Bronze / Imported Tables

The first notebook cell imports the Chinook CSV files into Databricks tables.

The imported tables are:

```text
workspace.default
├── chinook_artist
├── chinook_album
├── chinook_genre
├── chinook_mediatype
├── chinook_track
├── chinook_playlist
├── chinook_playlisttrack
├── chinook_employee
├── chinook_customer
├── chinook_invoice
└── chinook_invoiceline
```

The import is implemented using:

```sql
CREATE OR REPLACE TABLE ...
SELECT * FROM read_files(...)
```

Explicit schemas are provided during ingestion so that fields are loaded with the intended data types.

### Example

```sql
CREATE OR REPLACE TABLE chinook_artist AS
SELECT * FROM read_files(
  '.../Artist.csv',
  format => 'csv',
  header => true,
  schema => 'ArtistId INT, Name STRING'
);
```

The notebook also notes that `chinook_playlist` and `chinook_mediatype` are loaded but are not used downstream.

---

# 3. Silver Layer

The Silver layer contains cleaned and transformed versions of the imported Chinook tables.

The notebook creates the Silver tables in this order:

```text
silver_genre
silver_artist
silver_album
silver_track
silver_customer
silver_employee
silver_invoice
silver_invoiceline
```

---

## 3.1 Silver Genre

Table:

```text
silver_genre
```

Transformations:

- Keeps `GenreId`
- Renames `Name` to `GenreName`
- Uses `TRIM()` to clean the genre name
- Removes records where `GenreId` or `Name` is NULL

```sql
SELECT
    GenreId,
    TRIM(Name) AS GenreName
FROM workspace.default.chinook_genre
WHERE GenreId IS NOT NULL
  AND Name IS NOT NULL;
```

---

## 3.2 Silver Artist

Table:

```text
silver_artist
```

Transformations:

- Keeps `ArtistId`
- Renames `Name` to `ArtistName`
- Trims artist names
- Removes records where `ArtistId` or `Name` is NULL

---

## 3.3 Silver Album

Table:

```text
silver_album
```

The Album table is enriched with artist information from `silver_artist`.

Transformations:

- Trims album titles
- Joins Album to Artist
- Adds `ArtistName`
- Uses `COALESCE()` to provide `Unknown Artist` when artist information is unavailable
- Removes records where `AlbumId` or `Title` is NULL

Relationship:

```text
silver_artist
      │
      │ 1 : N
      ▼
silver_album
```

---

## 3.4 Silver Track

Table:

```text
silver_track
```

The Track table is enriched with album, artist, and genre information.

Transformations include:

- Trimming track names and composer values
- Joining to `silver_album`
- Joining to `silver_genre`
- Adding album title
- Adding artist information
- Adding genre name
- Replacing missing descriptive values with `Unknown`
- Calculating track duration in minutes
- Calculating file size in MB
- Excluding records where `MediaTypeId = 3`

### Derived fields

```text
DurationMinutes
FileSizeMb
```

For example:

```sql
ROUND(t.Milliseconds / 60000.0, 2) AS DurationMinutes
```

and:

```sql
ROUND(t.Bytes / 1048576.0, 4) AS FileSizeMb
```

---

## 3.5 Silver Customer

Table:

```text
silver_customer
```

The Customer table is transformed to include a **Spending Tier**.

First, total customer spending is calculated from:

```text
Customer
   ↓
Invoice
   ↓
InvoiceLine
```

Customer spending is calculated as:

```text
UnitPrice × Quantity
```

The notebook then calculates dynamic percentile thresholds:

- 80th percentile → High threshold
- 60th percentile → Medium threshold

Customers are assigned:

| Spending Tier | Rule |
|---|---|
| `High` | Spending ≥ 80th percentile |
| `Medium` | Spending ≥ 60th percentile |
| `Low` | Below the Medium threshold |

Other transformations include:

- Trimming customer names, company, country, and email
- Removing customers without a valid `CustomerId`
- Removing customers without a country
- Using `ROW_NUMBER()` with `QUALIFY` to maintain one record per customer

---

## 3.6 Silver Employee

Table:

```text
silver_employee
```

The Employee table is simplified to the fields needed downstream:

- `EmployeeId`
- `LastName`
- `FirstName`
- `Title`

Text fields are standardized using `TRIM()`.

---

## 3.7 Silver Invoice

Table:

```text
silver_invoice
```

The Invoice table is enhanced with time attributes.

### Derived fields

```text
Month
Quarter
Year
YearQuarter
```

Example:

```sql
MONTH(i.InvoiceDate) AS Month,
QUARTER(i.InvoiceDate) AS Quarter,
YEAR(i.InvoiceDate) AS Year,
CONCAT(YEAR(i.InvoiceDate), '-Q', QUARTER(i.InvoiceDate)) AS YearQuarter
```

Records with missing:

- `InvoiceId`
- `CustomerId`
- `InvoiceDate`

are excluded.

---

## 3.8 Silver InvoiceLine

Table:

```text
silver_invoiceline
```

This table becomes the main transaction-level dataset for the revenue analytics.

It connects:

```text
InvoiceLine
    │
    ├── Invoice
    │     └── Customer
    │           └── Employee
    │
    └── Track
```

### Main fields

- `InvoiceLineId`
- `InvoiceId`
- `CustomerId`
- `EmployeeId`
- `TrackId`
- `UnitPrice`
- `Quantity`
- `LineAmount`

### Derived measure

```sql
ROUND(il.UnitPrice * il.Quantity, 2) AS LineAmount
```

Validation rules include:

- Valid `InvoiceLineId`
- Valid `InvoiceId`
- Valid customer relationship
- Valid track relationship
- Valid employee relationship
- `Quantity > 0`
- `UnitPrice >= 0`

---

# 4. Data Quality Check

The notebook performs an invoice reconciliation check after creating the Silver Invoice and InvoiceLine tables.

The purpose is to compare:

```text
Invoice.Total
        vs
SUM(InvoiceLine.UnitPrice × Quantity)
```

The calculated invoice total is compared against the stored invoice total.

A difference greater than `0.01` is treated as a mismatch.

### Result

```text
Mismatched invoices: 0
```

This confirms that the invoice totals reconcile with the calculated line-item totals in the notebook output.

---

# 5. Analytics / Gold Outputs

After the Silver layer and reconciliation check, the notebook moves into analytics.

The notebook contains six analytics questions.

---

# 5.1 Top Revenue by Genre per Country

### Business Question

> Which music genres generate the most revenue in each country?

The query:

- Groups revenue by country and genre
- Calculates total revenue using `UnitPrice × Quantity`
- Uses `ROW_NUMBER()` to rank genres within each country
- Returns the highest-revenue genre for each country

### Main fields

```text
Country
Genre
TotalRevenue
GenreRank
```

The ranking logic is:

```sql
ROW_NUMBER() OVER (
    PARTITION BY c.Country
    ORDER BY SUM(il.UnitPrice * il.Quantity) DESC,
             t.GenreName ASC
)
```

Only:

```text
GenreRank = 1
```

is returned.

---

# 5.2 Customer Segmentation by Spending Tier

### Business Question

> How are customers distributed across spending tiers?

The notebook creates:

```text
gold_customer_spending
```

Customer spending is calculated from invoice-line purchases.

Dynamic thresholds are again calculated using:

```text
80th percentile → High
60th percentile → Medium
Below 60th percentile → Low
```

### Output fields

```text
spending_tier
customer_count
min_spending
max_spending
avg_spending
```

The result provides a summary of customer spending behavior by tier.

---

# 5.3 Monthly Sales Trends

### Business Question

> How has revenue trended month-by-month over the last 2 years?

The notebook creates:

```text
gold_monthly_sales_trends
```

The query uses the latest invoice date and looks back 23 months, producing a 24-month period.

### Metrics

- Revenue
- Number of invoices
- Total units sold

### Output fields

```text
Month
Revenue
NumberOfInvoices
TotalUnitsSold
```

Revenue is calculated from:

```text
UnitPrice × Quantity
```

---

# 5.4 Employee Sales Performance

### Business Question

> How much revenue is associated with each employee across years and quarters?

The query joins:

```text
silver_invoiceline
        ↓
silver_employee
        ↓
silver_invoice
```

### Output fields

```text
EmployeeName
RevenueTotal
InvoiceYear
InvoiceQuarter
YearQuarter
```

Revenue is calculated using:

```sql
SUM(fact.LineAmount)
```

The results are ordered by revenue in descending order.

---

# 5.5 Top 20 Tracks by Quantity Sold

### Business Question

> What are the top 20 tracks by total units sold, and which albums and artists do they belong to?

The notebook creates:

```text
gold_popular_tracks
```

The query joins `silver_invoiceline` with `silver_track`.

### Metrics

- Total units sold
- Average unit price

### Output fields

```text
track_name
album_title
artist_name
total_units_sold
avg_unit_price
```

The result is limited to the top 20 tracks by total quantity sold.

---

# 5.6 Regional Pricing Insights

### Business Question

> Do average unit prices differ across countries or regions?

The notebook creates:

```text
gold_regional_pricing
```

The query joins:

```text
silver_invoiceline
        ↓
silver_invoice
        ↓
silver_customer
```

### Metrics

- Number of invoices
- Total units sold
- Average unit price

### Output fields

```text
country
number_of_invoices
total_units_sold
avg_unit_price
```

Results are ordered by average unit price in descending order.

---

# 6. Data Model

The main transaction relationship used by the analytics is:

```text
Customer
    │
    │ 1 : N
    ▼
Invoice
    │
    │ 1 : N
    ▼
InvoiceLine
    │
    │ N : 1
    ▼
Track
```

Additional relationships used during transformation include:

```text
Artist
   │
   │ 1 : N
   ▼
Album
   │
   │ 1 : N
   ▼
Track

Genre
   │
   │ 1 : N
   ▼
Track

Employee
   │
   │ 1 : N
   ▼
Customer
```

---

# 7. Key Transformations

The project demonstrates several SQL data engineering techniques.

### Data ingestion

```sql
read_files()
```

Used to load CSV files with explicit schemas.

### Data cleaning

```sql
TRIM()
```

Used to standardize text fields.

### Missing-value handling

```sql
COALESCE()
```

Used for missing descriptive values such as artist, album, and genre names.

### Table joins

```sql
INNER JOIN
LEFT JOIN
```

Used to combine related Chinook entities.

### Derived metrics

```sql
UnitPrice * Quantity
```

Used to calculate line-level revenue.

### Dynamic segmentation

```sql
PERCENTILE_CONT()
```

Used to create spending tiers based on the distribution of customer spending.

### Ranking

```sql
ROW_NUMBER()
```

Used to rank genres within each country.

### Deduplication / uniqueness

```sql
ROW_NUMBER() ... QUALIFY
```

Used in the customer transformation to maintain one record per customer.

### Date transformations

```sql
MONTH()
QUARTER()
YEAR()
DATE_FORMAT()
ADD_MONTHS()
```

Used for time-based sales analysis.

---

# 8. Project Workflow

```text
1. Import Chinook CSV files
          ↓
2. Create imported Chinook tables
          ↓
3. Create Silver Genre
          ↓
4. Create Silver Artist
          ↓
5. Create Silver Album
          ↓
6. Create Silver Track
          ↓
7. Create Silver Customer
          ↓
8. Create Silver Employee
          ↓
9. Create Silver Invoice
          ↓
10. Create Silver InvoiceLine
          ↓
11. Reconcile Invoice totals
          ↓
12. Analyze revenue by genre and country
          ↓
13. Segment customers by spending
          ↓
14. Analyze monthly sales trends
          ↓
15. Analyze employee sales performance
          ↓
16. Identify popular tracks
          ↓
17. Analyze regional pricing
```

---

# 9. Gold Tables Created

The notebook creates the following Gold tables:

| Gold Table | Purpose |
|---|---|
| `gold_customer_spending` | Customer spending tier summary |
| `gold_monthly_sales_trends` | Monthly revenue and sales trends |
| `gold_popular_tracks` | Top tracks by quantity sold |
| `gold_regional_pricing` | Country-level pricing analysis |

The genre-by-country and employee sales queries are executed as analytical SQL outputs rather than being persisted as Gold tables in the notebook.

---

# 10. Technology Stack

- **Databricks**
- **Databricks SQL**
- **SQL**
- `read_files()`
- CSV
- `CREATE OR REPLACE TABLE`
- CTEs
- `JOIN`
- `CASE`
- `COALESCE`
- `PERCENTILE_CONT`
- `ROW_NUMBER`
- `QUALIFY`
- Aggregate functions
- Date functions
- Window functions

---

# 11. Key Takeaways

This project demonstrates an end-to-end SQL-based data engineering workflow using the Chinook dataset.

### Data Engineering

- CSV ingestion
- Explicit schema definition
- Data cleaning
- Missing-value handling
- Table joins
- Derived columns
- Data validation
- Reconciliation checks

### Data Modeling

- Understanding relationships between Chinook entities
- Connecting customers, invoices, invoice lines, tracks, artists, albums, genres, and employees
- Creating a transaction-level InvoiceLine dataset
- Creating analytics-ready Silver and Gold tables

### Analytics

- Revenue by genre and country
- Customer spending segmentation
- Monthly sales trends
- Employee sales performance
- Top tracks by quantity sold
- Regional pricing insights

The final result is a cleaned and structured Chinook dataset that can be used for SQL-based analysis and BI reporting.

---

# Repository Structure

A recommended GitHub repository structure is:

```text
chinook-data-engineering/
│
├── README.md
│
└── src/
    └── sql/
        ├── 00_setup/
        ├── 01_bronze/
        ├── 02_silver/
        ├── 03_gold/
        └── 04_analytics/
```

---

## Project Status

**Completed:** Chinook data ingestion, Silver transformations, customer spending segmentation, invoice-line preparation, invoice reconciliation, and six analytics outputs.
