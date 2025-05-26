# Tourism Agency Database Management System
# Phase 10: Partitioning Strategy

## Table of Contents
1. [Introduction](#introduction)
2. [Partition Proposals](#partition-proposals)
   - [Range Partitioning for Bookings Table](#range-partitioning-for-bookings-table)
   - [List Partitioning for Customers Table](#list-partitioning-for-customers-table)
   - [Hash Partitioning for Payments Table](#hash-partitioning-for-payments-table)
3. [External Memory Storage and Parallel Reads](#external-memory-storage-and-parallel-reads)
4. [Experimental Comparison of Partition Types](#experimental-comparison-of-partition-types)
5. [Performance Improvements and Transaction Links](#performance-improvements-and-transaction-links)
6. [Conclusion](#conclusion)

## Introduction

This report presents a comprehensive partitioning strategy for the Tourism Agency Database. Based on our analysis of the database schema, workload patterns, and query performance from previous phases, we propose three different types of partitions (range, list, and hash) to optimize database performance. Each partition type is selected to address specific performance challenges identified in our workload analysis.

## Partition Proposals

### Range Partitioning for Bookings Table

#### Partition Type: RANGE

#### Target Table: Bookings

#### Partition Key: BookingDate

#### Partition Definition:
```sql
ALTER TABLE Bookings MODIFY
PARTITION BY RANGE (BookingDate) (
    PARTITION bookings_2022_q1 VALUES LESS THAN (TO_DATE('2022-04-01', 'YYYY-MM-DD')),
    PARTITION bookings_2022_q2 VALUES LESS THAN (TO_DATE('2022-07-01', 'YYYY-MM-DD')),
    PARTITION bookings_2022_q3 VALUES LESS THAN (TO_DATE('2022-10-01', 'YYYY-MM-DD')),
    PARTITION bookings_2022_q4 VALUES LESS THAN (TO_DATE('2023-01-01', 'YYYY-MM-DD')),
    PARTITION bookings_2023_q1 VALUES LESS THAN (TO_DATE('2023-04-01', 'YYYY-MM-DD')),
    PARTITION bookings_2023_q2 VALUES LESS THAN (TO_DATE('2023-07-01', 'YYYY-MM-DD')),
    PARTITION bookings_2023_q3 VALUES LESS THAN (TO_DATE('2023-10-01', 'YYYY-MM-DD')),
    PARTITION bookings_2023_q4 VALUES LESS THAN (TO_DATE('2024-01-01', 'YYYY-MM-DD')),
    PARTITION bookings_2024_q1 VALUES LESS THAN (TO_DATE('2024-04-01', 'YYYY-MM-DD')),
    PARTITION bookings_2024_q2 VALUES LESS THAN (TO_DATE('2024-07-01', 'YYYY-MM-DD')),
    PARTITION bookings_future VALUES LESS THAN (MAXVALUE)
);
```

#### External Storage Configuration:
```sql
-- Move older partitions to separate tablespace for historical data
ALTER TABLE Bookings MOVE PARTITION bookings_2022_q1 TABLESPACE hist_data_ts1;
ALTER TABLE Bookings MOVE PARTITION bookings_2022_q2 TABLESPACE hist_data_ts1;
ALTER TABLE Bookings MOVE PARTITION bookings_2022_q3 TABLESPACE hist_data_ts1;
ALTER TABLE Bookings MOVE PARTITION bookings_2022_q4 TABLESPACE hist_data_ts1;

-- Keep recent partitions on faster storage
ALTER TABLE Bookings MOVE PARTITION bookings_2023_q3 TABLESPACE active_data_ts1;
ALTER TABLE Bookings MOVE PARTITION bookings_2023_q4 TABLESPACE active_data_ts1;
ALTER TABLE Bookings MOVE PARTITION bookings_2024_q1 TABLESPACE active_data_ts1;
ALTER TABLE Bookings MOVE PARTITION bookings_2024_q2 TABLESPACE active_data_ts1;
ALTER TABLE Bookings MOVE PARTITION bookings_future TABLESPACE active_data_ts1;
```

#### Purpose and Performance Improvements:
Range partitioning on `BookingDate` will significantly improve the performance of date-based queries, which are prevalent in our workload. From our analysis of QUERY 1 in the Phase 9 report, we observed that filtering on `BookingDate` is a common operation:

```sql
WHERE b.BookingDate >= ADD_MONTHS(CURRENT_DATE, -6)
```

This partitioning strategy will enable:

1. **Partition Pruning**: Queries that filter on specific date ranges will only access relevant partitions, reducing I/O and improving query performance. For example, queries analyzing recent bookings (last 6 months) will only scan 2-3 partitions instead of the entire table.

2. **Parallel Query Execution**: Each partition can be processed in parallel, especially when they're stored on separate tablespaces, improving the performance of analytical queries that span multiple quarters.

3. **Maintenance Operations**: Archiving old data, rebuilding indexes, and gathering statistics can be performed at the partition level, reducing maintenance windows.

4. **Storage Tiering**: Historical data (older partitions) can be moved to less expensive storage while keeping recent data on faster storage.

### List Partitioning for Customers Table

#### Partition Type: LIST

#### Target Table: Customers

#### Partition Key: CustomerTier

#### Partition Definition:
```sql
ALTER TABLE Customers MODIFY
PARTITION BY LIST (CustomerTier) (
    PARTITION customers_platinum VALUES ('Platinum'),
    PARTITION customers_gold VALUES ('Gold'),
    PARTITION customers_silver VALUES ('Silver'),
    PARTITION customers_standard VALUES ('Standard'),
    PARTITION customers_other VALUES (DEFAULT)
);
```

#### External Storage Configuration:
```sql
-- Store high-value customer data on premium storage for faster access
ALTER TABLE Customers MOVE PARTITION customers_platinum TABLESPACE premium_data_ts1;
ALTER TABLE Customers MOVE PARTITION customers_gold TABLESPACE premium_data_ts1;

-- Store standard customer data on regular storage
ALTER TABLE Customers MOVE PARTITION customers_silver TABLESPACE standard_data_ts1;
ALTER TABLE Customers MOVE PARTITION customers_standard TABLESPACE standard_data_ts1;
ALTER TABLE Customers MOVE PARTITION customers_other TABLESPACE standard_data_ts1;
```

#### Purpose and Performance Improvements:
List partitioning on `CustomerTier` will optimize operations that target specific customer segments, which is a common pattern in our marketing and analytics queries. From QUERY 2 in the Phase 9 report, we see operations that update customer information based on their spending tier:

```sql
UPDATE Customers c
SET Address = 
    CASE
        WHEN ft.Tag IS NOT NULL AND Address NOT LIKE '%' || ft.Tag THEN Address || ft.Tag
        ELSE Address
    END
FROM FinalTagging ft
WHERE c.CustomerID = ft.CustomerID
  AND ft.Tag IS NOT NULL;
```

This partitioning strategy will enable:

1. **Targeted Marketing Operations**: Queries that target specific customer tiers (e.g., promotions for Platinum customers) will only access the relevant partition.

2. **Prioritized Access**: High-value customer data (Platinum, Gold) can be stored on faster storage for improved response times.

3. **Parallel Processing**: Operations across different customer tiers can be executed in parallel.

4. **Improved Data Management**: Security policies, backup schedules, and retention policies can be implemented at the partition level based on customer value.

### Hash Partitioning for Payments Table

#### Partition Type: HASH

#### Target Table: Payments

#### Partition Key: PaymentID

#### Partition Definition:
```sql
ALTER TABLE Payments MODIFY
PARTITION BY HASH (PaymentID)
PARTITIONS 8
STORE IN (payments_ts1, payments_ts2, payments_ts3, payments_ts4);
```

#### External Storage Configuration:
```sql
-- Create tablespaces on different disk groups for parallel I/O
CREATE TABLESPACE payments_ts1 DATAFILE '/disk1/payments_ts1.dbf' SIZE 5G;
CREATE TABLESPACE payments_ts2 DATAFILE '/disk2/payments_ts2.dbf' SIZE 5G;
CREATE TABLESPACE payments_ts3 DATAFILE '/disk3/payments_ts3.dbf' SIZE 5G;
CREATE TABLESPACE payments_ts4 DATAFILE '/disk4/payments_ts4.dbf' SIZE 5G;
```

#### Purpose and Performance Improvements:
Hash partitioning on `PaymentID` will distribute payment records evenly across multiple partitions, which is ideal for a high-volume transaction table like Payments. From our workload analysis, we observed that the Payments table is frequently joined in analytical queries:

```sql
JOIN Payments p ON p.BookingID = b.BookingID
```

This partitioning strategy will enable:

1. **Balanced Data Distribution**: Payment records will be evenly distributed across partitions, preventing hotspots.

2. **Parallel Query Processing**: Analytical queries that scan the entire Payments table can be processed in parallel across all partitions.

3. **I/O Distribution**: By storing partitions on different physical disks, I/O operations can be distributed, reducing contention.

4. **Scalability**: The number of partitions can be increased as the payment data grows, maintaining consistent performance.

## External Memory Storage and Parallel Reads

### Overview of External Storage Strategy

To maximize the benefits of our partitioning scheme, we propose implementing a tiered storage strategy that places different partitions on separate tablespaces, which can be configured on different physical storage devices. This approach enables:

1. **Parallel Read Operations**: When queries access multiple partitions, the database can read from different storage devices simultaneously, significantly improving I/O throughput.

2. **Storage Tiering**: Different classes of data can be stored on appropriate storage tiers based on access patterns and importance.

3. **Resource Isolation**: Critical business operations can be isolated from analytical workloads by placing their data on separate storage resources.

### Detailed External Storage Configuration

#### For Range-Partitioned Bookings Table

```sql
-- Create tablespaces on different storage tiers
CREATE TABLESPACE hist_data_ts1 
  DATAFILE '/disk_array1/hist_data_ts1.dbf' 
  SIZE 10G 
  EXTENT MANAGEMENT LOCAL 
  SEGMENT SPACE MANAGEMENT AUTO;

CREATE TABLESPACE active_data_ts1 
  DATAFILE '/ssd_array1/active_data_ts1.dbf' 
  SIZE 5G 
  EXTENT MANAGEMENT LOCAL 
  SEGMENT SPACE MANAGEMENT AUTO;

-- Configure parallel degree for tablespaces
ALTER TABLESPACE hist_data_ts1 DEFAULT PARALLEL 4;
ALTER TABLESPACE active_data_ts1 DEFAULT PARALLEL 8;

-- Move partitions to appropriate tablespaces
ALTER TABLE Bookings MOVE PARTITION bookings_2022_q1 TABLESPACE hist_data_ts1;
ALTER TABLE Bookings MOVE PARTITION bookings_2022_q2 TABLESPACE hist_data_ts1;
ALTER TABLE Bookings MOVE PARTITION bookings_2022_q3 TABLESPACE hist_data_ts1;
ALTER TABLE Bookings MOVE PARTITION bookings_2022_q4 TABLESPACE hist_data_ts1;

ALTER TABLE Bookings MOVE PARTITION bookings_2023_q3 TABLESPACE active_data_ts1;
ALTER TABLE Bookings MOVE PARTITION bookings_2023_q4 TABLESPACE active_data_ts1;
ALTER TABLE Bookings MOVE PARTITION bookings_2024_q1 TABLESPACE active_data_ts1;
ALTER TABLE Bookings MOVE PARTITION bookings_2024_q2 TABLESPACE active_data_ts1;
ALTER TABLE Bookings MOVE PARTITION bookings_future TABLESPACE active_data_ts1;
```

**Rationale**: 
- Historical data (2022) is placed on cost-effective storage (`/disk_array1/`) with moderate parallel degree (4)
- Recent and current data (2023-2024) is placed on high-performance SSD storage (`/ssd_array1/`) with higher parallel degree (8)
- This configuration enables parallel reads across different storage devices while optimizing storage costs

#### For List-Partitioned Customers Table

```sql
-- Create tablespaces for different customer tiers
CREATE TABLESPACE premium_data_ts1 
  DATAFILE '/ssd_array2/premium_data_ts1.dbf' 
  SIZE 2G 
  EXTENT MANAGEMENT LOCAL 
  SEGMENT SPACE MANAGEMENT AUTO;

CREATE TABLESPACE standard_data_ts1 
  DATAFILE '/disk_array2/standard_data_ts1.dbf' 
  SIZE 8G 
  EXTENT MANAGEMENT LOCAL 
  SEGMENT SPACE MANAGEMENT AUTO;

-- Configure parallel degree for tablespaces
ALTER TABLESPACE premium_data_ts1 DEFAULT PARALLEL 8;
ALTER TABLESPACE standard_data_ts1 DEFAULT PARALLEL 4;

-- Move partitions to appropriate tablespaces
ALTER TABLE Customers MOVE PARTITION customers_platinum TABLESPACE premium_data_ts1;
ALTER TABLE Customers MOVE PARTITION customers_gold TABLESPACE premium_data_ts1;

ALTER TABLE Customers MOVE PARTITION customers_silver TABLESPACE standard_data_ts1;
ALTER TABLE Customers MOVE PARTITION customers_standard TABLESPACE standard_data_ts1;
ALTER TABLE Customers MOVE PARTITION customers_other TABLESPACE standard_data_ts1;
```

**Rationale**:
- High-value customers (Platinum, Gold) are placed on premium SSD storage (`/ssd_array2/`) with high parallel degree (8)
- Standard customers are placed on regular storage (`/disk_array2/`) with moderate parallel degree (4)
- This prioritizes performance for operations involving high-value customers while maintaining cost efficiency

#### For Hash-Partitioned Payments Table

```sql
-- Create tablespaces on different physical disks for I/O distribution
CREATE TABLESPACE payments_ts1 
  DATAFILE '/disk1/payments_ts1.dbf' 
  SIZE 5G 
  EXTENT MANAGEMENT LOCAL 
  SEGMENT SPACE MANAGEMENT AUTO;

CREATE TABLESPACE payments_ts2 
  DATAFILE '/disk2/payments_ts2.dbf' 
  SIZE 5G 
  EXTENT MANAGEMENT LOCAL 
  SEGMENT SPACE MANAGEMENT AUTO;

CREATE TABLESPACE payments_ts3 
  DATAFILE '/disk3/payments_ts3.dbf' 
  SIZE 5G 
  EXTENT MANAGEMENT LOCAL 
  SEGMENT SPACE MANAGEMENT AUTO;

CREATE TABLESPACE payments_ts4 
  DATAFILE '/disk4/payments_ts4.dbf' 
  SIZE 5G 
  EXTENT MANAGEMENT LOCAL 
  SEGMENT SPACE MANAGEMENT AUTO;

-- Configure parallel degree for tablespaces
ALTER TABLESPACE payments_ts1 DEFAULT PARALLEL 4;
ALTER TABLESPACE payments_ts2 DEFAULT PARALLEL 4;
ALTER TABLESPACE payments_ts3 DEFAULT PARALLEL 4;
ALTER TABLESPACE payments_ts4 DEFAULT PARALLEL 4;

-- Hash partitions will be automatically distributed across these tablespaces
ALTER TABLE Payments MODIFY
PARTITION BY HASH (PaymentID)
PARTITIONS 8
STORE IN (payments_ts1, payments_ts2, payments_ts3, payments_ts4);
```

**Rationale**:
- Hash partitions are distributed across four separate physical disks to maximize I/O parallelism
- Each tablespace has a moderate parallel degree (4) to balance resource utilization
- This configuration enables highly parallel reads and writes for the high-volume Payments table

### Parallel Read Optimization

To fully leverage the external storage configuration, we recommend the following parallel read optimizations:

#### Database Parameter Settings

```sql
-- Enable parallel DML operations
ALTER SYSTEM SET PARALLEL_DML = TRUE;

-- Set appropriate parallel degree policy
ALTER SYSTEM SET PARALLEL_DEGREE_POLICY = 'AUTO';

-- Configure parallel adaptive multi-user to balance resources
ALTER SYSTEM SET PARALLEL_ADAPTIVE_MULTI_USER = TRUE;

-- Set parallel execution message size for efficient communication
ALTER SYSTEM SET PARALLEL_EXECUTION_MESSAGE_SIZE = 16384;
```

#### Table-Level Parallel Hints

For critical queries that benefit from parallel execution:

```sql
-- Example: Parallel hint for analytical query on Bookings
SELECT /*+ PARALLEL(b 8) */ 
  TRUNC(b.BookingDate, 'MM') AS BookingMonth,
  COUNT(*) AS BookingCount,
  SUM(p.AmountPaid) AS TotalRevenue
FROM Bookings b
JOIN Payments p ON p.BookingID = b.BookingID
WHERE b.BookingDate >= ADD_MONTHS(CURRENT_DATE, -12)
GROUP BY TRUNC(b.BookingDate, 'MM')
ORDER BY BookingMonth;
```

## Experimental Comparison of Partition Types

To validate the effectiveness of our partitioning strategies, we propose the following experiments:

### Experiment 1: Comparing Range vs. List vs. Hash Partitioning for Bookings Table

#### Objective
Determine which partitioning strategy provides the best performance improvement for the Bookings table across different query patterns.

#### Test Queries

##### Query 1A: Date Range Filter (favors Range Partitioning)
```sql
-- Query that filters by date range
SELECT b.BookingID, b.CustomerID, b.PackageID, b.BookingDate, b.Status, p.AmountPaid
FROM Bookings b
JOIN Payments p ON p.BookingID = b.BookingID
WHERE b.BookingDate BETWEEN TO_DATE('2023-01-01', 'YYYY-MM-DD') AND TO_DATE('2023-06-30', 'YYYY-MM-DD')
ORDER BY b.BookingDate;
```

##### Query 1B: Status Filter (favors List Partitioning)
```sql
-- Query that filters by booking status
SELECT b.BookingID, b.CustomerID, b.PackageID, b.BookingDate, b.Status, p.AmountPaid
FROM Bookings b
JOIN Payments p ON p.BookingID = b.BookingID
WHERE b.Status = 'Confirmed'
ORDER BY b.BookingDate;
```

##### Query 1C: Specific BookingID Lookup (favors Hash Partitioning)
```sql
-- Query that looks up specific BookingIDs (simulating multiple concurrent sessions)
SELECT b.BookingID, b.CustomerID, b.PackageID, b.BookingDate, b.Status, p.AmountPaid
FROM Bookings b
JOIN Payments p ON p.BookingID = b.BookingID
WHERE b.BookingID IN (
    SELECT BookingID FROM (
        SELECT BookingID FROM Bookings
        ORDER BY DBMS_RANDOM.VALUE
    ) WHERE ROWNUM <= 1000
);
```

#### Experiment Setup

1. **Baseline Measurement**:
   - Execute each test query 5 times on the unpartitioned Bookings table
   - Record execution time, I/O statistics, and query plan for each run
   - Calculate average execution time and I/O metrics

2. **Range Partitioning Implementation**:
   ```sql
   ALTER TABLE Bookings MODIFY
   PARTITION BY RANGE (BookingDate) (
       PARTITION bookings_2022_q1 VALUES LESS THAN (TO_DATE('2022-04-01', 'YYYY-MM-DD')),
       PARTITION bookings_2022_q2 VALUES LESS THAN (TO_DATE('2022-07-01', 'YYYY-MM-DD')),
       PARTITION bookings_2022_q3 VALUES LESS THAN (TO_DATE('2022-10-01', 'YYYY-MM-DD')),
       PARTITION bookings_2022_q4 VALUES LESS THAN (TO_DATE('2023-01-01', 'YYYY-MM-DD')),
       PARTITION bookings_2023_q1 VALUES LESS THAN (TO_DATE('2023-04-01', 'YYYY-MM-DD')),
       PARTITION bookings_2023_q2 VALUES LESS THAN (TO_DATE('2023-07-01', 'YYYY-MM-DD')),
       PARTITION bookings_2023_q3 VALUES LESS THAN (TO_DATE('2023-10-01', 'YYYY-MM-DD')),
       PARTITION bookings_2023_q4 VALUES LESS THAN (TO_DATE('2024-01-01', 'YYYY-MM-DD')),
       PARTITION bookings_future VALUES LESS THAN (MAXVALUE)
   );
   ```
   - Execute each test query 5 times
   - Record execution time, I/O statistics, and query plan for each run
   - Calculate average execution time and I/O metrics

3. **List Partitioning Implementation**:
   ```sql
   ALTER TABLE Bookings MODIFY
   PARTITION BY LIST (Status) (
       PARTITION bookings_confirmed VALUES ('Confirmed'),
       PARTITION bookings_pending VALUES ('Pending'),
       PARTITION bookings_canceled VALUES ('Canceled'),
       PARTITION bookings_completed VALUES ('Completed'),
       PARTITION bookings_other VALUES (DEFAULT)
   );
   ```
   - Execute each test query 5 times
   - Record execution time, I/O statistics, and query plan for each run
   - Calculate average execution time and I/O metrics

4. **Hash Partitioning Implementation**:
   ```sql
   ALTER TABLE Bookings MODIFY
   PARTITION BY HASH (BookingID)
   PARTITIONS 8;
   ```
   - Execute each test query 5 times
   - Record execution time, I/O statistics, and query plan for each run
   - Calculate average execution time and I/O metrics

#### Expected Outcomes

- Query 1A (date range filter) is expected to perform best with Range partitioning due to partition pruning on the date range
- Query 1B (status filter) is expected to perform best with List partitioning due to direct access to the relevant status partition
- Query 1C (BookingID lookup) is expected to perform best with Hash partitioning due to even distribution of BookingIDs

### Experiment 2: Comparing Different Range Partition Granularities

#### Objective
Determine the optimal granularity for range partitioning of the Bookings table (quarterly vs. monthly vs. yearly).

#### Test Queries

##### Query 2A: Quarterly Analysis
```sql
-- Query that analyzes bookings by quarter
SELECT 
    TO_CHAR(b.BookingDate, 'YYYY-Q') AS BookingQuarter,
    COUNT(*) AS BookingCount,
    SUM(p.AmountPaid) AS TotalRevenue,
    AVG(p.AmountPaid) AS AvgBookingValue
FROM Bookings b
JOIN Payments p ON p.BookingID = b.BookingID
WHERE b.BookingDate BETWEEN TO_DATE('2022-01-01', 'YYYY-MM-DD') AND TO_DATE('2023-12-31', 'YYYY-MM-DD')
GROUP BY TO_CHAR(b.BookingDate, 'YYYY-Q')
ORDER BY BookingQuarter;
```

#### Expected Outcomes

- Query 2A (quarterly analysis) is expected to perform best with quarterly partitioning due to alignment with the analysis granularity

### Experiment 3: Comparing Bitmap vs. B-Tree Indexes on Partitioned Tables

#### Objective
Determine whether bitmap or B-tree indexes provide better performance for queries on partitioned tables, particularly for the Customers table with list partitioning.

#### Test Queries

##### Query 3A: Customer Tier Analysis
```sql
-- Query that analyzes customers by tier
SELECT 
    c.CustomerTier,
    COUNT(*) AS CustomerCount,
    AVG(EXTRACT(YEAR FROM SYSDATE) - EXTRACT(YEAR FROM c.DateOfBirth)) AS AvgAge,
    COUNT(DISTINCT b.BookingID) AS TotalBookings,
    SUM(p.AmountPaid) AS TotalSpent,
    AVG(p.AmountPaid) AS AvgBookingValue
FROM Customers c
JOIN Bookings b ON c.CustomerID = b.CustomerID
JOIN Payments p ON b.BookingID = p.BookingID
GROUP BY c.CustomerTier
ORDER BY TotalSpent DESC;
```

#### Expected Outcomes

- For Query 3A (customer tier analysis), bitmap indexes are expected to perform better due to the low cardinality of the CustomerTier column and the aggregation nature of the query

## Performance Improvements and Transaction Links

### Expected Performance Improvements for Range Partitioning of Bookings Table

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

### Summary of Performance Improvements by Transaction

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

The experimental comparisons outlined in this report will provide empirical validation of these expected improvements and guide any necessary refinements to the partitioning strategy.
