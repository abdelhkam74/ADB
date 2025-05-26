# Performance Improvements and Transaction Links

This document details the expected performance improvements for each proposed partitioning strategy and explicitly links them to specific transactions and queries in our Tourism Agency Database workload.

## 1. Range Partitioning for Bookings Table

### Expected Performance Improvements

| Metric | Without Partitioning | With Range Partitioning | Improvement |
|--------|---------------------|------------------------|-------------|
| Query execution time for date-filtered queries | Baseline | 40-60% reduction | High |
| I/O operations for date-filtered queries | Baseline | 50-70% reduction | High |
| Parallel query execution | Limited | Significantly improved | High |
| Maintenance operations time | Baseline | 30-50% reduction | Medium |
| Storage tiering benefits | None | Cost optimization for historical data | Medium |

### Links to Specific Transactions

#### Transaction 1: InsertPromotionsForTopPackages

From our Phase 9 workload analysis, this transaction includes:

```sql
WITH ActivePackages AS (
    SELECT
        b.PackageID,
        COUNT(DISTINCT b.BookingID) AS BookingCount,
        COUNT(DISTINCT p.PaymentID) AS PaymentCount,
        SUM(p.AmountPaid) AS TotalRevenue,
        RANK() OVER (ORDER BY SUM(p.AmountPaid) DESC) AS RevenueRank
    FROM Bookings b
    JOIN Payments p ON p.BookingID = b.BookingID
    WHERE b.BookingDate >= ADD_MONTHS(CURRENT_DATE, -6)
    GROUP BY b.PackageID
),
CustomerEngagement AS (
    SELECT
        b.PackageID,
        COUNT(DISTINCT c.CustomerID) AS UniqueCustomers,
        AVG(EXTRACT(YEAR FROM SYSDATE) - EXTRACT(YEAR FROM c.DateOfBirth)) AS AvgCustomerAge
    FROM Bookings b
    JOIN Customers c ON c.CustomerID = b.CustomerID
    WHERE b.BookingDate >= ADD_MONTHS(CURRENT_DATE, -6)
    GROUP BY b.PackageID
)
INSERT INTO Promotions (PackageID, DiscountPercentage, StartDate, EndDate)
SELECT
    a.PackageID,
    CASE
        WHEN a.RevenueRank <= 5 THEN 10.00
        WHEN ce.AvgCustomerAge < 30 THEN 20.00
        ELSE 15.00
    END AS DiscountPercentage,
    CURRENT_DATE,
    CURRENT_DATE + INTERVAL '30' DAY
FROM ActivePackages a
JOIN CustomerEngagement ce ON a.PackageID = ce.PackageID
JOIN TravelPackages tp ON tp.PackageID = a.PackageID
WHERE a.BookingCount > 50
  AND ce.UniqueCustomers > 20
  AND NOT EXISTS (
      SELECT 1
      FROM Promotions pr
      WHERE pr.PackageID = a.PackageID
        AND pr.EndDate >= CURRENT_DATE
  )
  AND EXISTS (
      SELECT 1
      FROM TourGuides tg
      WHERE tg.PackageID = a.PackageID
        AND tg.ExperienceYears >= 3
  );
```

**Performance Impact Analysis:**
- This transaction filters bookings by date range (`b.BookingDate >= ADD_MONTHS(CURRENT_DATE, -6)`), which will directly benefit from range partitioning on BookingDate.
- The query plan from Phase 9 shows a full table scan of the Bookings table with a cost of 1806 (4% CPU).
- With range partitioning, only the partitions containing the last 6 months of data will be scanned, significantly reducing I/O and CPU usage.
- Based on the query plan, we estimate a 45-55% reduction in execution time for this transaction.
- The transaction's execution time was 1.524 seconds without indexes; with range partitioning, we expect it to drop to approximately 0.7-0.8 seconds.

#### Transaction 5: UpdateDestinationStats

This transaction includes:

```sql
MERGE INTO Destinations d
USING (
  SELECT
      dp.DestinationID,
      COUNT(DISTINCT b.BookingID) AS Popularity,
      AVG(p.AmountPaid) AS AvgSpent
  FROM Bookings b
  JOIN Payments p ON p.BookingID = b.BookingID
  JOIN TravelPackages tp ON tp.PackageID = b.PackageID
  JOIN DestinationPackages dp ON dp.PackageID = tp.PackageID
  GROUP BY dp.DestinationID
) stats ON (d.DestinationID = stats.DestinationID)
WHEN MATCHED THEN
  UPDATE SET d.PopularityScore = stats.Popularity,
             d.AvgSpending = stats.AvgSpent;
```

**Performance Impact Analysis:**
- This transaction performs an analytical query across all bookings to calculate destination statistics.
- While there's no explicit date filter, range partitioning will still benefit this query through parallel execution.
- Each partition of the Bookings table can be processed in parallel, with results aggregated at the end.
- Based on similar workloads, we estimate a 25-35% reduction in execution time for this transaction.
- The transaction's execution time was 0.084 seconds in the baseline; with range partitioning and parallel execution, we expect it to drop to approximately 0.055-0.065 seconds.

## 2. List Partitioning for Customers Table

### Expected Performance Improvements

| Metric | Without Partitioning | With List Partitioning | Improvement |
|--------|---------------------|------------------------|-------------|
| Query execution time for tier-filtered queries | Baseline | 20-30% reduction | Medium |
| I/O operations for tier-specific operations | Baseline | 30-40% reduction | Medium |
| Parallel processing of customer segments | Limited | Significantly improved | High |
| Storage tiering benefits | None | Prioritized access for high-value customers | Medium |
| Security and data management | Baseline | Enhanced tier-based policies | Medium |

### Links to Specific Transactions

#### Transaction 2: TagHighValueCustomers

From our Phase 9 workload analysis, this transaction includes:

```sql
MERGE INTO Customers c
USING (
  SELECT
      b.CustomerID,
      SUM(p.AmountPaid) AS TotalSpent,
      COUNT(DISTINCT b.BookingID) AS TotalBookings
  FROM Bookings b
  JOIN Payments p ON p.BookingID = b.BookingID
  GROUP BY b.CustomerID
) spending ON (c.CustomerID = spending.CustomerID)
WHEN MATCHED THEN
  UPDATE SET c.CustomerTier = CASE
    WHEN spending.TotalSpent >= 10000 AND spending.TotalBookings >= 5 THEN 'Platinum'
    WHEN spending.TotalSpent >= 5000 THEN 'Gold'
    WHEN spending.TotalSpent >= 2000 THEN 'Silver'
    ELSE 'Standard'
  END;
```

**Performance Impact Analysis:**
- This transaction updates customer tiers based on spending patterns.
- After the initial update, subsequent runs will primarily modify customers who change tiers.
- With list partitioning by CustomerTier, updates will be localized to specific partitions.
- The query plan from Phase 9 shows a hash join with a cost of 21115 (2% CPU).
- List partitioning will reduce the scope of updates and improve concurrency by reducing lock contention.
- Based on the query plan and similar workloads, we estimate a 20-25% reduction in execution time.
- The transaction's execution time was 3.184 seconds without indexes; with list partitioning, we expect it to drop to approximately 2.4-2.5 seconds.

#### Customer Analytics Query (Common in Reporting Workload)

```sql
SELECT 
    c.CustomerTier,
    COUNT(*) AS CustomerCount,
    AVG(EXTRACT(YEAR FROM SYSDATE) - EXTRACT(YEAR FROM c.DateOfBirth)) AS AvgAge,
    SUM(p.AmountPaid) AS TotalRevenue,
    AVG(p.AmountPaid) AS AvgSpending
FROM Customers c
JOIN Bookings b ON c.CustomerID = b.CustomerID
JOIN Payments p ON b.BookingID = p.BookingID
WHERE c.CustomerTier IN ('Platinum', 'Gold')
GROUP BY c.CustomerTier
ORDER BY TotalRevenue DESC;
```

**Performance Impact Analysis:**
- This analytical query focuses on high-value customers (Platinum and Gold tiers).
- With list partitioning, only the relevant customer tier partitions will be scanned.
- Additionally, these partitions will be stored on premium storage for faster access.
- Based on similar workloads, we estimate a 30-40% reduction in execution time for this query.
- For a typical execution time of 2-3 seconds, this would result in a reduction to approximately 1.2-2.1 seconds.

## 3. Hash Partitioning for Payments Table

### Expected Performance Improvements

| Metric | Without Partitioning | With Hash Partitioning | Improvement |
|--------|---------------------|------------------------|-------------|
| Query execution time for full table scans | Baseline | 25-35% reduction | Medium |
| I/O distribution | Concentrated | Evenly distributed | High |
| Parallel query processing | Limited | Significantly improved | High |
| Scalability for growing data | Challenging | Easily accommodated | High |
| Maintenance operations time | Baseline | 20-30% reduction | Medium |

### Links to Specific Transactions

#### Transaction 1: InsertPromotionsForTopPackages (Payment Processing Component)

From our Phase 9 workload analysis, this transaction includes a join with the Payments table:

```sql
FROM Bookings b
JOIN Payments p ON p.BookingID = b.BookingID
WHERE b.BookingDate >= ADD_MONTHS(CURRENT_DATE, -6)
```

**Performance Impact Analysis:**
- This transaction joins the Bookings and Payments tables to calculate revenue metrics.
- The query plan from Phase 9 shows a hash join with the Payments table, with a cost of 1792 (2% CPU).
- Hash partitioning of the Payments table will distribute I/O across multiple storage devices.
- Each partition can be processed in parallel, improving join performance.
- Based on the query plan and similar workloads, we estimate a 25-30% reduction in the payment processing component of this transaction.
- This contributes to the overall performance improvement for Transaction 1, complementing the benefits from range partitioning of the Bookings table.

#### Financial Reporting Query (Common in Reporting Workload)

```sql
SELECT 
    TRUNC(p.PaymentDate, 'MM') AS Month,
    COUNT(*) AS PaymentCount,
    SUM(p.AmountPaid) AS TotalRevenue,
    AVG(p.AmountPaid) AS AvgPaymentAmount,
    MIN(p.AmountPaid) AS MinPayment,
    MAX(p.AmountPaid) AS MaxPayment
FROM Payments p
WHERE p.PaymentDate >= ADD_MONTHS(CURRENT_DATE, -12)
GROUP BY TRUNC(p.PaymentDate, 'MM')
ORDER BY Month;
```

**Performance Impact Analysis:**
- This analytical query scans a year's worth of payment data for financial reporting.
- With hash partitioning, the scan will be distributed across multiple partitions and storage devices.
- Each partition can be processed in parallel, significantly improving query performance.
- Based on similar workloads, we estimate a 30-40% reduction in execution time for this query.
- For a typical execution time of 5-7 seconds for this type of financial report, this would result in a reduction to approximately 3-4.9 seconds.

## Combined Impact on Complex Workload Queries

### Complex Query: Customer Spending Analysis by Destination and Time Period

```sql
SELECT 
    d.Name AS Destination,
    TO_CHAR(b.BookingDate, 'YYYY-Q') AS Quarter,
    c.CustomerTier,
    COUNT(DISTINCT c.CustomerID) AS UniqueCustomers,
    COUNT(DISTINCT b.BookingID) AS BookingCount,
    SUM(p.AmountPaid) AS TotalRevenue,
    AVG(p.AmountPaid) AS AvgBookingValue
FROM Destinations d
JOIN DestinationPackages dp ON d.DestinationID = dp.DestinationID
JOIN TravelPackages tp ON dp.PackageID = tp.PackageID
JOIN Bookings b ON tp.PackageID = b.PackageID
JOIN Customers c ON b.CustomerID = c.CustomerID
JOIN Payments p ON b.BookingID = p.BookingID
WHERE b.BookingDate BETWEEN TO_DATE('2022-01-01', 'YYYY-MM-DD') AND TO_DATE('2023-12-31', 'YYYY-MM-DD')
GROUP BY d.Name, TO_CHAR(b.BookingDate, 'YYYY-Q'), c.CustomerTier
ORDER BY Quarter, TotalRevenue DESC;
```

**Combined Performance Impact Analysis:**
- This complex analytical query involves all three tables we've proposed for partitioning.
- Range partitioning of Bookings will limit the scan to relevant date range partitions.
- List partitioning of Customers will optimize the processing of customer tier data.
- Hash partitioning of Payments will distribute I/O and enable parallel processing.
- The combined effect of these partitioning strategies will be multiplicative rather than merely additive.
- Based on our analysis, we estimate a 45-60% reduction in execution time for this complex query.
- For a typical execution time of 15-20 seconds for this type of complex analysis, this would result in a reduction to approximately 6-11 seconds.

## Summary of Performance Improvements by Transaction

| Transaction | Without Partitioning | With Partitioning | Improvement | Primary Partition Type Benefit |
|-------------|---------------------|------------------|-------------|--------------------------------|
| InsertPromotionsForTopPackages | 1.524s | 0.7-0.8s | 45-55% | Range (Bookings) + Hash (Payments) |
| TagHighValueCustomers | 3.184s | 2.4-2.5s | 20-25% | List (Customers) |
| DeleteExpiredLowPerformingPromotions | 28.890s | 20-22s | 25-30% | Range (Bookings) |
| QueryTopSpendingCustomers | 0.047s | 0.03-0.035s | 25-35% | List (Customers) + Hash (Payments) |
| UpdateDestinationStats | 0.084s | 0.055-0.065s | 25-35% | Range (Bookings) |
| Complex Analytical Queries | 15-20s | 6-11s | 45-60% | All partition types combined |

## Conclusion

The proposed partitioning strategies are directly linked to specific transactions and queries in our Tourism Agency Database workload. Each partition type addresses particular performance challenges:

1. **Range Partitioning of Bookings** optimizes date-filtered queries and enables efficient historical data management.
2. **List Partitioning of Customers** improves customer tier-based operations and enables prioritized access for high-value customers.
3. **Hash Partitioning of Payments** distributes I/O for this high-volume table and enables efficient parallel processing.

When combined, these partitioning strategies provide a comprehensive performance optimization solution for our database workload. The expected improvements range from 20% to 60%, depending on the specific transaction and query patterns.

These performance improvements will directly benefit business operations by:
- Reducing response times for customer-facing operations
- Enabling more complex analytical queries within acceptable timeframes
- Improving system scalability as data volumes grow
- Optimizing storage costs through tiered storage strategies
- Enhancing maintenance operations through partition-level management

The experimental comparisons outlined in the previous section will provide empirical validation of these expected improvements and guide any necessary refinements to the partitioning strategy.
