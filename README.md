# Superstore-Data-Cleaning
SQL Data Cleaning &amp; Analysis Case Study of Superstore Sales Dataset

## The Question
Can the same Superstore dataset be validated for quality and analyzed for business insight entirely in SQL — covering data integrity, category/regional performance, customer rankings, and time trends — using only queries (aggregates, joins, and window functions)?

## The Data
Same Superstore Sales dataset as Project 1, loaded into a table called `SuperstoreSalesDataset` (Order_ID, Customer_ID, Product_ID, Order_Date, Ship_Date, Ship_Mode, Segment, City, State, Postal_Code, Region, Category, Sub_Category, Product_Name, Sales).

## The Process

### A. Data Cleaning & Validation

**Q: Are there duplicate order/customer/product combinations?**
```sql
SELECT Order_ID, Customer_ID, Product_ID, COUNT(*) AS duplicate_count
FROM SuperstoreSalesDataset
GROUP BY Order_ID, Customer_ID, Product_ID
HAVING COUNT(*) > 1;
```

**Q: Which rows have a missing value in any column?**
```sql
SELECT *
FROM SuperstoreSalesDataset
WHERE Row_ID IS NULL OR Order_ID IS NULL OR Order_Date IS NULL
   OR Ship_Date IS NULL OR Ship_Mode IS NULL OR Customer_ID IS NULL
   OR Customer_Name IS NULL OR Segment IS NULL OR Country IS NULL
   OR City IS NULL OR State IS NULL OR Postal_Code IS NULL
   OR Region IS NULL OR Product_ID IS NULL OR Category IS NULL
   OR Sub_Category IS NULL OR Product_Name IS NULL OR Sales IS NULL;
```

**Fix: filled missing postal codes with a known default**
```sql
UPDATE SuperstoreSalesDataset
SET postal_code = '05401'
WHERE postal_code IS NULL;
```

**Removed duplicates — two approaches compared**

Self-join method:
```sql
DELETE t1
FROM SuperstoreSalesDataset t1
JOIN SuperstoreSalesDataset t2
    ON t1.Order_ID = t2.Order_ID
    AND t1.Product_ID = t2.Product_ID
    AND t1.Row_ID > t2.Row_ID;
```

`ROW_NUMBER()` window function method (more flexible for larger datasets):
```sql
DELETE FROM SuperstoreSalesDataset
WHERE Row_ID IN (
    SELECT Row_ID
    FROM (
        SELECT Row_ID,
               ROW_NUMBER() OVER (
                   PARTITION BY Order_ID, Product_ID
                   ORDER BY Row_ID
               ) AS rn
        FROM SuperstoreSalesDataset
    ) t
    WHERE rn > 1
);
```

**Q: Are there invalid Sales values (zero or negative)?**
```sql
SELECT * FROM SuperstoreSalesDataset WHERE Sales <= 0;
```

**Standardized text fields by trimming whitespace**
```sql
UPDATE SuperstoreSalesDataset
SET ship_mode = TRIM(ship_mode),
    Segment = TRIM(Segment),
    City = TRIM(City),
    state = TRIM(State),
    product_name = TRIM(product_name);
```

Also validated categorical consistency with simple lookups, e.g.:
```sql
SELECT DISTINCT Category FROM SuperstoreSalesDataset;
```
(repeated for `state` and `ship_mode` to confirm no inconsistent or misspelled values)

### B. Sales Overview

**Q: What's the overall range and average sale value?**
```sql
SELECT MIN(Sales) AS Min_Sales, MAX(Sales) AS Max_Sales, AVG(Sales) AS Avg_Sales
FROM SuperstoreSalesDataset;
```

### C. Category & Sub-Category Performance

**Q: Which categories generate the most revenue, and how many orders does each have?**
```sql
SELECT Category,
    COUNT(*) AS Num_Orders,
    AVG(Sales) AS Avg_Sales,
    MIN(Sales) AS Min_Sales,
    MAX(Sales) AS Max_Sales,
    SUM(Sales) AS Total_Sales
FROM SuperstoreSalesDataset
GROUP BY Category
ORDER BY Total_Sales DESC;
```

**Q: How does performance break down at the sub-category level?**
```sql
SELECT Category, sub_category,
    ROUND(SUM(Sales), 2) AS Total_Sales,
    ROUND(AVG(Sales), 2) AS Avg_Sale_Value,
    COUNT(*) AS Total_Orders
FROM SuperstoreSalesDataset
GROUP BY Category, sub_category
ORDER BY Total_Sales DESC;
```

### D. Shipping & Customer Segments

**Q: Which shipping mode is associated with the highest average order value?**
```sql
SELECT ship_mode,
    COUNT(*) AS Order_Count,
    ROUND(SUM(Sales), 2) AS Total_Sales,
    ROUND(AVG(Sales), 2) AS Avg_Order_Value
FROM SuperstoreSalesDataset
GROUP BY ship_mode
ORDER BY Avg_Order_Value DESC;
```

**Q: How do Consumer, Corporate, and Home Office segments compare?**
```sql
SELECT Segment,
    COUNT(*) AS OrderCount,
    SUM(Sales) AS TotalSales,
    AVG(Sales) AS AvgSales
FROM SuperstoreSalesDataset
GROUP BY Segment;
```

### E. Geographic Analysis

**Q: Which states drive the most revenue?**
```sql
SELECT state,
    COUNT(*) AS OrderCount,
    SUM(Sales) AS TotalSales,
    AVG(Sales) AS AvgSales
FROM SuperstoreSalesDataset
GROUP BY state
ORDER BY TotalSales DESC;
```

**Q: What's the total sales per region, and what share does each category contribute within its region?**
```sql
SELECT Region, Category,
    ROUND(SUM(Sales), 2) AS Regional_Sales,
    ROUND(
        (SUM(Sales) * 100.0 / SUM(SUM(Sales)) OVER (PARTITION BY Region)),
        2
    ) AS Pct_of_Regional_Total
FROM SuperstoreSalesDataset
GROUP BY Region, Category
ORDER BY Region, Regional_Sales DESC;
```
*This uses a window function (`SUM() OVER PARTITION BY`) to calculate each category's percentage of its region's total — without a separate subquery.*

### F. Time-Based Analysis

**Q: How do sales trend by year and quarter, broken down by category?**
```sql
SELECT
    substr(Order_Date, 7, 4) || '-' || substr(Order_Date, 4, 2) || '-' || substr(Order_Date, 1, 2) AS Proper_Date,
    substr(Order_Date, 7, 4) AS Year,
    ((CAST(substr(Order_Date, 4, 2) AS INTEGER) - 1) / 3 + 1) AS Quarter,
    Category,
    SUM(Sales) AS Total_Sales,
    AVG(Sales) AS Avg_Sales
FROM SuperstoreSalesDataset
WHERE Order_Date IS NOT NULL
GROUP BY Year, Quarter, Category
ORDER BY Year, Total_Sales DESC;
```
*Parses the raw `DD/MM/YYYY` date string directly in SQL to derive year and calendar quarter.*

### G. Rankings & Outliers

**Q: Which 5 products have generated the least total revenue?**
```sql
SELECT Category, Sub_Category, Product_ID, SUM(Sales) AS Total_Sales
FROM SuperstoreSalesDataset
GROUP BY Category, Sub_Category, Product_ID
ORDER BY Total_Sales ASC
LIMIT 5;
```

**Q: Which individual sales fall in the top 10% by value?**
```sql
SELECT * FROM (
    SELECT product_name, sub_category, Sales,
        ROUND(PERCENT_RANK() OVER (ORDER BY Sales) * 100, 2) AS sales_rank_percent
    FROM SuperstoreSalesDataset
) t
WHERE sales_rank_percent > 90
ORDER BY Sales DESC;
```
*Uses `PERCENT_RANK()` to identify the top decile of individual transactions by sale amount.*

**Q: Who are the top 10 customers by total spend?**
```sql
SELECT customer_name,
    SUM(Sales) AS Total_Sales,
    COUNT(*) AS Total_Orders,
    ROUND(AVG(Sales), 2) AS Avg_Order_Value
FROM SuperstoreSalesDataset
GROUP BY customer_id
ORDER BY Total_Sales DESC
LIMIT 10;
```

## Tools
`SQL` (aggregate functions, `JOIN`, `ROW_NUMBER()`, `PERCENT_RANK()`, `SUM() OVER PARTITION BY`)
