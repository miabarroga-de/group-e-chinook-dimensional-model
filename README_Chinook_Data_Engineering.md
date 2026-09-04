# 🎵 Chinook Data Engineering Project

## Project Overview

This project uses the **Chinook dataset** and builds the data workflow in Databricks using SQL.

The notebook is documented below **in the exact sequence of its cells**, from importing the source tables, creating Silver tables, validating the data, and running the final analytics queries.

### Technology
- Databricks
- Databricks SQL
- SQL
- Delta Tables

---

## Notebook Cell Sequence

## 1. Import Chinook tables

**Purpose:** Imports the Chinook CSV files into Databricks tables using `read_files()`. The source schema is explicitly defined for each table.

### Code

```sql
%sql
-- Import Chinook tables (Idempotent)
-- NOTE (M7): chinook_playlist and chinook_mediatype are loaded but not used downstream
-- Consider removing these to reduce compute/egress costs 
CREATE OR REPLACE TABLE chinook_artist AS
SELECT * FROM read_files(
  'r2://ftw-b12-dataengineering@6338489909d41c2f78a0a2345a684267.r2.cloudflarestorage.com/shared/week05/chinook_csv/Artist.csv',
  format => 'csv', header => true,
  schema => 'ArtistId INT, Name STRING'
);

CREATE OR REPLACE TABLE chinook_album AS
SELECT * FROM read_files(
  'r2://ftw-b12-dataengineering@6338489909d41c2f78a0a2345a684267.r2.cloudflarestorage.com/shared/week05/chinook_csv/Album.csv',
  format => 'csv', header => true,
  schema => 'AlbumId INT, Title STRING, ArtistId INT'
);

CREATE OR REPLACE TABLE chinook_genre AS
SELECT * FROM read_files(
  'r2://ftw-b12-dataengineering@6338489909d41c2f78a0a2345a684267.r2.cloudflarestorage.com/shared/week05/chinook_csv/Genre.csv',
  format => 'csv', header => true,
  schema => 'GenreId INT, Name STRING'
);

CREATE OR REPLACE TABLE chinook_mediatype AS
SELECT * FROM read_files(
  'r2://ftw-b12-dataengineering@6338489909d41c2f78a0a2345a684267.r2.cloudflarestorage.com/shared/week05/chinook_csv/MediaType.csv',
  format => 'csv', header => true,
  schema => 'MediaTypeId INT, Name STRING'
);

CREATE OR REPLACE TABLE chinook_track AS
SELECT * FROM read_files(
  'r2://ftw-b12-dataengineering@6338489909d41c2f78a0a2345a684267.r2.cloudflarestorage.com/shared/week05/chinook_csv/Track.csv',
  format => 'csv', header => true,
  schema => 'TrackId INT, Name STRING, AlbumId INT, MediaTypeId INT, GenreId INT, Composer STRING, Milliseconds INT, Bytes INT, UnitPrice DECIMAL(10,2)'
);

CREATE OR REPLACE TABLE chinook_playlist AS
SELECT * FROM read_files(
  'r2://ftw-b12-dataengineering@6338489909d41c2f78a0a2345a684267.r2.cloudflarestorage.com/shared/week05/chinook_csv/Playlist.csv',
  format => 'csv', header => true,
  schema => 'PlaylistId INT, Name STRING'
);

CREATE OR REPLACE TABLE chinook_playlisttrack AS
SELECT * FROM read_files(
  'r2://ftw-b12-dataengineering@6338489909d41c2f78a0a2345a684267.r2.cloudflarestorage.com/shared/week05/chinook_csv/PlaylistTrack.csv',
  format => 'csv', header => true,
  schema => 'PlaylistId INT, TrackId INT'
);

CREATE OR REPLACE TABLE chinook_employee AS
SELECT * FROM read_files(
  'r2://ftw-b12-dataengineering@6338489909d41c2f78a0a2345a684267.r2.cloudflarestorage.com/shared/week05/chinook_csv/Employee.csv',
  format => 'csv', header => true,
  schema => 'EmployeeId INT, LastName STRING, FirstName STRING, Title STRING, ReportsTo INT, BirthDate TIMESTAMP, HireDate TIMESTAMP, Address STRING, City STRING, State STRING, Country STRING, PostalCode STRING, Phone STRING, Fax STRING, Email STRING'
);

CREATE OR REPLACE TABLE chinook_customer AS
SELECT * FROM read_files(
  'r2://ftw-b12-dataengineering@6338489909d41c2f78a0a2345a684267.r2.cloudflarestorage.com/shared/week05/chinook_csv/Customer.csv',
  format => 'csv', header => true,
  schema => 'CustomerId INT, FirstName STRING, LastName STRING, Company STRING, Address STRING, City STRING, State STRING, Country STRING, PostalCode STRING, Phone STRING, Fax STRING, Email STRING, SupportRepId INT'
);

CREATE OR REPLACE TABLE chinook_invoice AS
SELECT * FROM read_files(
  'r2://ftw-b12-dataengineering@6338489909d41c2f78a0a2345a684267.r2.cloudflarestorage.com/shared/week05/chinook_csv/Invoice.csv',
  format => 'csv', header => true,
  schema => 'InvoiceId INT, CustomerId INT, InvoiceDate TIMESTAMP, BillingAddress STRING, BillingCity STRING, BillingState STRING, BillingCountry STRING, BillingPostalCode STRING, Total DECIMAL(10,2)'
);

CREATE OR REPLACE TABLE chinook_invoiceline AS
SELECT * FROM read_files(
  'r2://ftw-b12-dataengineering@6338489909d41c2f78a0a2345a684267.r2.cloudflarestorage.com/shared/week05/chinook_csv/InvoiceLine.csv',
  format => 'csv', header => true,
  schema => 'InvoiceLineId INT, InvoiceId INT, TrackId INT, UnitPrice DECIMAL(10,2), Quantity INT'
);
```

---

## 2. Silver Genre Table

**Purpose:** Creates the Silver Genre table, trims the genre name, and removes rows where the ID or name is NULL.

### Code

```sql
%sql
CREATE OR REPLACE TABLE silver_genre AS
SELECT
    GenreId,
    TRIM(Name) AS GenreName
FROM workspace.default.chinook_genre
WHERE GenreId IS NOT NULL
  AND Name IS NOT NULL;
```

---

## 3. Silver Artist Table

**Purpose:** Creates the Silver Artist table, trims artist names, and removes rows where the ID or name is NULL.

### Code

```sql
%sql
CREATE OR REPLACE TABLE silver_artist AS
SELECT
    ArtistId,
    TRIM(Name) AS ArtistName
FROM workspace.default.chinook_artist
WHERE ArtistId IS NOT NULL
  AND Name IS NOT NULL;
```

---

## 4. Silver Album Table

**Purpose:** Creates the Silver Album table, trims album titles, joins artist information, and uses `Unknown Artist` when the artist name is unavailable.

### Code

```sql
%sql
CREATE OR REPLACE TABLE silver_album AS
SELECT
    a.AlbumId,
    TRIM(a.Title) AS AlbumTitle,
    a.ArtistId,
    COALESCE(TRIM(ar.ArtistName), 'Unknown Artist') AS ArtistName
FROM workspace.default.chinook_album a
LEFT JOIN silver_artist ar
    ON a.ArtistId = ar.ArtistId
WHERE a.AlbumId IS NOT NULL
  AND a.Title IS NOT NULL;
```

---

## 5. Silver Track Table

**Purpose:** Creates the Silver Track table by joining album and genre information, cleaning text fields, deriving duration and file size, and excluding MediaTypeId 3.

### Code

```sql
%sql
CREATE OR REPLACE TABLE silver_track AS
SELECT
    t.TrackId,
    TRIM(t.Name) AS TrackName,
    t.AlbumId,
    COALESCE(al.AlbumTitle, 'Unknown Album') AS Title,
    COALESCE(al.ArtistId, -1) AS ArtistId,
    COALESCE(al.ArtistName, 'Unknown Artist') AS ArtistName,
    t.GenreId,
    COALESCE(g.GenreName, 'Unknown Genre') AS GenreName,
    t.MediaTypeId,
    TRIM(t.Composer) AS Composer,
    t.Milliseconds,
    ROUND(t.Milliseconds / 60000.0, 2) AS DurationMinutes,
    t.Bytes,
    ROUND(t.Bytes / 1048576.0, 4) AS FileSizeMb,
    t.UnitPrice
FROM workspace.default.chinook_track t
LEFT JOIN silver_album al
    ON t.AlbumId = al.AlbumId
LEFT JOIN silver_genre g
    ON t.GenreId = g.GenreId
WHERE t.TrackId IS NOT NULL
  AND t.Name IS NOT NULL
  AND t.MediaTypeId != 3;
  
  SELECT * FROM silver_track LIMIT 10
```

---

## 6. Silver Customer Table

**Purpose:** Creates the Silver Customer table, calculates customer spending, derives dynamic spending thresholds using percentiles, and assigns High, Medium, or Low spending tiers.

### Code

```sql
%sql
CREATE OR REPLACE TABLE silver_customer AS
WITH customer_spending AS (
    SELECT
        c.CustomerId,
        SUM(il.UnitPrice * il.Quantity) AS total_spending
    FROM workspace.default.chinook_customer c
    LEFT JOIN workspace.default.chinook_invoice i ON c.CustomerId = i.CustomerId
    LEFT JOIN workspace.default.chinook_invoiceline il ON i.InvoiceId = il.InvoiceId
    GROUP BY c.CustomerId
),
percentile_thresholds AS (
    SELECT
        PERCENTILE_CONT(0.80) WITHIN GROUP (ORDER BY total_spending) AS high_threshold,
        PERCENTILE_CONT(0.60) WITHIN GROUP (ORDER BY total_spending) AS medium_threshold
    FROM customer_spending
)
SELECT
    c.CustomerId,
    TRIM(c.FirstName) AS FirstName,
    TRIM(c.LastName) AS LastName,
    TRIM(c.Company) AS Company,
    TRIM(c.Country) AS Country,
    TRIM(c.Email) AS Email,
    c.SupportRepId,
    CASE
        WHEN COALESCE(cs.total_spending, 0) >= pt.high_threshold THEN 'High'
        WHEN COALESCE(cs.total_spending, 0) >= pt.medium_threshold THEN 'Medium'
        ELSE 'Low'
    END AS SpendingTier
FROM workspace.default.chinook_customer c
LEFT JOIN customer_spending cs ON c.CustomerId = cs.CustomerId
CROSS JOIN percentile_thresholds pt
WHERE c.CustomerId IS NOT NULL
  AND c.Country IS NOT NULL
QUALIFY ROW_NUMBER() OVER (PARTITION BY c.CustomerId ORDER BY c.LastName, c.FirstName) = 1;

SELECT * FROM silver_customer
```

---

## 7. Silver Employee Table

**Purpose:** Creates the Silver Employee table and trims the employee name and title fields.

### Code

```sql
%sql
CREATE OR REPLACE TABLE silver_employee AS
SELECT
    EmployeeId,
    TRIM(LastName) AS LastName,
    TRIM(FirstName) AS FirstName,
    TRIM(Title) AS Title
FROM workspace.default.chinook_employee;
```

---

## 8. Silver Invoice Table

**Purpose:** Creates the Silver Invoice table and derives Month, Quarter, Year, and YearQuarter from InvoiceDate.

### Code

```sql
%sql
CREATE OR REPLACE TABLE silver_invoice AS
SELECT
    i.InvoiceId,
    i.CustomerId,
    i.InvoiceDate,
    MONTH(i.InvoiceDate) AS Month,
    QUARTER(i.InvoiceDate) AS Quarter,
    YEAR(i.InvoiceDate) AS Year,
    CONCAT(YEAR(i.InvoiceDate), '-Q', QUARTER(i.InvoiceDate)) AS YearQuarter,
    i.Total
FROM workspace.default.chinook_invoice i
WHERE i.InvoiceId IS NOT NULL
  AND i.CustomerId IS NOT NULL
  AND i.InvoiceDate IS NOT NULL;
```

---

## 9. Silver InvoiceLine Table

**Purpose:** Creates the Silver InvoiceLine table by connecting invoice, customer, employee, and track information and calculating LineAmount.

### Code

```sql
%sql
CREATE OR REPLACE TABLE silver_invoiceline AS
SELECT
    il.InvoiceLineId,
    il.InvoiceId,
    c.CustomerId,
    e.EmployeeId,
    il.TrackId,
    il.UnitPrice,
    il.Quantity,
    ROUND(il.UnitPrice * il.Quantity, 2) AS LineAmount
FROM workspace.default.chinook_invoiceline il
LEFT JOIN silver_invoice i
    ON il.InvoiceId = i.InvoiceId
LEFT JOIN silver_customer c
    ON i.CustomerId = c.CustomerId
LEFT JOIN silver_employee e
    ON c.SupportRepId = e.EmployeeId
LEFT JOIN silver_track t
    ON il.TrackId = t.TrackId
WHERE il.InvoiceLineId IS NOT NULL
  AND il.InvoiceId IS NOT NULL
  AND i.CustomerId IS NOT NULL
  AND il.TrackId IS NOT NULL
  AND e.EmployeeId IS NOT NULL
  AND il.Quantity > 0
  AND il.UnitPrice >= 0;

SELECT * FROM silver_invoiceline LIMIT 10;
```

---

## 10. Invoice Total Reconciliation Check

**Purpose:** Performs a data quality check by comparing stored invoice totals with totals calculated from invoice line items.

### Code

```sql
%sql
-- Data Quality Check: Reconcile invoice totals against line item sums
-- Identifies invoices where stored Total != SUM(line items)

WITH line_item_totals AS (
    SELECT
        InvoiceId,
        ROUND(SUM(UnitPrice * Quantity), 2) AS calculated_total
    FROM silver_invoiceline
    GROUP BY InvoiceId
),
reconciliation AS (
    SELECT
        i.InvoiceId,
        i.Total AS stored_total,
        COALESCE(lit.calculated_total, 0.00) AS calculated_total,
        ROUND(i.Total - COALESCE(lit.calculated_total, 0.00), 2) AS difference
    FROM silver_invoice i
    LEFT JOIN line_item_totals lit
        ON i.InvoiceId = lit.InvoiceId
    WHERE ABS(i.Total - COALESCE(lit.calculated_total, 0.00)) > 0.01
)
SELECT
    COUNT(*) AS mismatched_invoices,
    SUM(ABS(difference)) AS total_discrepancy
FROM reconciliation;
```

---

## 11. 1. Top Revenue by Genre per Country

**Purpose:** Finds the highest-revenue music genre for each country using a window function to rank genres within each country.

### Code

```sql
%sql
-- 1. Top Revenue by Genre per Country
-- Which music genres generate the most revenue in each country?


WITH genre_revenue_by_country AS (
    SELECT
        c.Country,
        t.GenreName AS Genre,
        ROUND(SUM(il.UnitPrice * il.Quantity), 2) AS TotalRevenue,
        ROW_NUMBER() OVER (PARTITION BY c.Country ORDER BY SUM(il.UnitPrice * il.Quantity) DESC, t.GenreName ASC) AS rank
    FROM silver_invoiceline il
    INNER JOIN silver_customer c
        ON il.CustomerId = c.CustomerId
    INNER JOIN silver_track t  
        ON il.TrackId = t.TrackId
    GROUP BY
        c.Country,
        t.GenreName
)
SELECT
    Country,
    Genre,
    TotalRevenue,
    rank AS GenreRank
FROM genre_revenue_by_country
WHERE rank = 1
ORDER BY
    Country,
    GenreRank;
```

---

## 12. 2. Customer Segmentation by Spending Tier

**Purpose:** Creates a Gold summary of customers by spending tier using dynamic percentile thresholds.

### Code

```sql
%sql
-- 2. Customer Segmentation by Spending Tier
-- Using dynamic percentiles instead of fixed thresholds

CREATE OR REPLACE TABLE gold_customer_spending AS WITH customer_spending AS (
    SELECT
        c.CustomerId,
        c.FirstName,
        c.LastName,
        SUM(il.UnitPrice * il.Quantity) AS total_spending
    FROM silver_invoiceline il
    JOIN silver_invoice i ON il.InvoiceId = i.InvoiceId
    JOIN silver_customer c ON i.CustomerId = c.CustomerId
    GROUP BY c.CustomerId, c.FirstName, c.LastName
),
percentile_thresholds AS (
    SELECT
        PERCENTILE_CONT(0.80) WITHIN GROUP (ORDER BY total_spending) AS high_threshold,
        PERCENTILE_CONT(0.60) WITHIN GROUP (ORDER BY total_spending) AS medium_threshold
    FROM customer_spending
),
customer_tiers AS (
    SELECT
        cs.CustomerId,
        cs.FirstName,
        cs.LastName,
        cs.total_spending,
        CASE
            WHEN cs.total_spending >= pt.high_threshold THEN 'High'
            WHEN cs.total_spending >= pt.medium_threshold THEN 'Medium'
            ELSE 'Low'
        END AS spending_tier
    FROM customer_spending cs
    CROSS JOIN percentile_thresholds pt
)
SELECT
    spending_tier,
    COUNT(*) AS customer_count,
    MIN(total_spending) AS min_spending,
    MAX(total_spending) AS max_spending,
    ROUND(AVG(total_spending), 2) AS avg_spending
FROM customer_tiers
GROUP BY spending_tier
ORDER BY
    CASE spending_tier
        WHEN 'High' THEN 1
        WHEN 'Medium' THEN 2
        WHEN 'Low' THEN 3
    END;

SELECT * FROM gold_customer_spending;
```

---

## 13. 3. Monthly Sales Trends

**Purpose:** Creates a Gold monthly sales summary covering the latest 24-month period represented by the data.

### Code

```sql
%sql
-- 3. Monthly Sales Trends
-- How has revenue trended month-by-month over the last 2 years?

CREATE OR REPLACE TABLE gold_monthly_sales_trends AS 
SELECT
    DATE_FORMAT(i.InvoiceDate, 'yyyy-MM') AS Month,
    ROUND(SUM(il.UnitPrice * il.Quantity), 2) AS Revenue,
    COUNT(DISTINCT i.InvoiceId) AS NumberOfInvoices,
    SUM(il.Quantity) AS TotalUnitsSold
FROM silver_invoice AS i
INNER JOIN silver_invoiceline AS il
    ON i.InvoiceId = il.InvoiceId
WHERE i.InvoiceDate >= ADD_MONTHS(
    (SELECT MAX(InvoiceDate) FROM silver_invoice),
    -23
)
GROUP BY DATE_FORMAT(i.InvoiceDate, 'yyyy-MM')
ORDER BY Month;

SELECT * FROM gold_monthly_sales_trends ORDER BY Month;
```

---

## 14. 4. Employee Sales Performance

**Purpose:** Calculates employee revenue by year and quarter using the Silver invoice line, employee, and invoice tables.

### Code

```sql
%sql
SELECT
    CONCAT(e.FirstName, ' ', e.LastName) AS EmployeeName,
    SUM(fact.LineAmount) AS RevenueTotal,
    YEAR(i.InvoiceDate) AS InvoiceYear,
    QUARTER(i.InvoiceDate) AS InvoiceQuarter,
    CONCAT(YEAR(i.InvoiceDate),'-Q',QUARTER(i.InvoiceDate)) AS YearQuarter

FROM silver_invoiceline fact
LEFT JOIN silver_employee e
    ON fact.EmployeeId = e.EmployeeId
LEFT JOIN silver_invoice i
    ON fact.InvoiceId = i.InvoiceId

GROUP BY
    CONCAT(e.FirstName, ' ', e.LastName),
    YEAR(i.InvoiceDate),
    QUARTER(i.InvoiceDate),
    CONCAT(YEAR(i.InvoiceDate),'-Q',QUARTER(i.InvoiceDate))

HAVING RevenueTotal IS NOT NULL
ORDER BY RevenueTotal DESC;
```

---

## 15. 5. Top 20 Tracks by Quantity Sold

**Purpose:** Creates a Gold table containing the 20 tracks with the highest total units sold, including album, artist, and average unit price.

### Code

```sql
%sql
-- 5. Popular Tracks by Quantity Sold 
-- What are the top 20 tracks by total units sold, and which albums/artists do they belong to?
CREATE OR REPLACE TABLE gold_popular_tracks AS
SELECT 
    t.TrackName AS track_name,
    t.Title AS album_title,
    t.ArtistName AS artist_name,
    SUM(il.Quantity) AS total_units_sold,
    ROUND(AVG(il.UnitPrice), 2) AS avg_unit_price
FROM silver_invoiceline AS il
INNER JOIN silver_track AS t
    ON il.TrackId = t.TrackId
GROUP BY 
    t.TrackName,
    t.Title,
    t.ArtistName
ORDER BY total_units_sold DESC
LIMIT 20;

SELECT * FROM gold_popular_tracks ORDER BY total_units_sold DESC;
```

---

## 16. 6. Regional Pricing Insights

**Purpose:** Creates a Gold country-level summary containing invoice count, units sold, and average unit price.

### Code

```sql
%sql
-- 6. Regional Pricing Insights
-- Do average unit prices differ across countries or regions?
CREATE OR REPLACE TABLE gold_regional_pricing AS
SELECT 
    c.Country AS country,
    COUNT(DISTINCT inv.InvoiceId) AS number_of_invoices,
    SUM(il.Quantity) AS total_units_sold,
    ROUND(AVG(il.UnitPrice), 2) AS avg_unit_price
FROM silver_invoiceline AS il
INNER JOIN silver_invoice AS inv
    ON il.InvoiceId = inv.InvoiceId
INNER JOIN silver_customer AS c
    ON inv.CustomerId = c.CustomerId
GROUP BY c.Country
ORDER BY avg_unit_price DESC;

SELECT * FROM gold_regional_pricing ORDER BY avg_unit_price DESC;
```

---

## Data Flow

The notebook follows this implementation sequence:

```text
Chinook CSV files
      ↓
Import Chinook tables
      ↓
Silver Genre
Silver Artist
Silver Album
Silver Track
Silver Customer
Silver Employee
Silver Invoice
Silver InvoiceLine
      ↓
Invoice Total Reconciliation Check
      ↓
Analytics / Gold outputs
```

## Key SQL Techniques Used

- `read_files()` for CSV ingestion
- Explicit schemas during ingestion
- `CREATE OR REPLACE TABLE`
- `TRIM()` for text cleaning
- `COALESCE()` for missing lookup values
- `LEFT JOIN` and `INNER JOIN`
- `CASE` for spending-tier classification
- `PERCENTILE_CONT()` for dynamic thresholds
- `ROW_NUMBER()` for ranking
- `ROUND()` for calculated values
- Date functions such as `MONTH()`, `QUARTER()`, `YEAR()`, `DATE_FORMAT()`, and `ADD_MONTHS()`
- Aggregations using `SUM()`, `AVG()`, `COUNT()`, and `COUNT(DISTINCT ...)`
- Data reconciliation between invoice totals and invoice-line totals

## Project Outcome

The notebook takes the raw Chinook CSV data through table creation and Silver-layer transformations, validates invoice totals against line-item calculations, and produces SQL outputs for customer, sales, employee, track, genre, and regional pricing analysis.
