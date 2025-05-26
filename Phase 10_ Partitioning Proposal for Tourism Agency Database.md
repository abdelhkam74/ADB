# Phase 10: Partitioning Proposal for Tourism Agency Database

## Introduction

This report presents a comprehensive partitioning strategy for the Tourism Agency Database. Based on our analysis of the database schema, workload patterns, and query performance from previous phases, we propose three different types of partitions (range, list, and hash) to optimize database performance. Each partition type is selected to address specific performance challenges identified in our workload analysis.

## 1. Range Partitioning for Bookings Table

### Partition Type: RANGE

### Target Table: Bookings

### Partition Key: BookingDate

### Partition Definition:
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

### External Storage Configuration:
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

### Purpose and Performance Improvements:
Range partitioning on `BookingDate` will significantly improve the performance of date-based queries, which are prevalent in our workload. From our analysis of QUERY 1 in the Phase 9 report, we observed that filtering on `BookingDate` is a common operation:

```sql
WHERE b.BookingDate >= ADD_MONTHS(CURRENT_DATE, -6)
```

This partitioning strategy will enable:

1. **Partition Pruning**: Queries that filter on specific date ranges will only access relevant partitions, reducing I/O and improving query performance. For example, queries analyzing recent bookings (last 6 months) will only scan 2-3 partitions instead of the entire table.

2. **Parallel Query Execution**: Each partition can be processed in parallel, especially when they're stored on separate tablespaces, improving the performance of analytical queries that span multiple quarters.

3. **Maintenance Operations**: Archiving old data, rebuilding indexes, and gathering statistics can be performed at the partition level, reducing maintenance windows.

4. **Storage Tiering**: Historical data (older partitions) can be moved to less expensive storage while keeping recent data on faster storage.

### Expected Impact on Workload:
- QUERY 1 (InsertPromotionsForTopPackages): Estimated 40-60% performance improvement due to partition pruning on the 6-month date filter.
- Analytical queries for seasonal trends: Estimated 30-50% improvement through parallel partition access.
- Backup and maintenance operations: Reduced time window by enabling partition-level operations.

## 2. List Partitioning for Customers Table

### Partition Type: LIST

### Target Table: Customers

### Partition Key: CustomerTier

### Partition Definition:
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

### External Storage Configuration:
```sql
-- Store high-value customer data on premium storage for faster access
ALTER TABLE Customers MOVE PARTITION customers_platinum TABLESPACE premium_data_ts1;
ALTER TABLE Customers MOVE PARTITION customers_gold TABLESPACE premium_data_ts1;

-- Store standard customer data on regular storage
ALTER TABLE Customers MOVE PARTITION customers_silver TABLESPACE standard_data_ts1;
ALTER TABLE Customers MOVE PARTITION customers_standard TABLESPACE standard_data_ts1;
ALTER TABLE Customers MOVE PARTITION customers_other TABLESPACE standard_data_ts1;
```

### Purpose and Performance Improvements:
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

### Expected Impact on Workload:
- QUERY 2 (TagHighValueCustomers): Estimated 20-30% performance improvement when updating specific customer tiers.
- Marketing queries targeting specific customer segments: 30-40% improvement through direct partition access.
- Customer analytics by tier: Improved performance through parallel processing of partitions.

## 3. Hash Partitioning for Payments Table

### Partition Type: HASH

### Target Table: Payments

### Partition Key: PaymentID

### Partition Definition:
```sql
ALTER TABLE Payments MODIFY
PARTITION BY HASH (PaymentID)
PARTITIONS 8
STORE IN (payments_ts1, payments_ts2, payments_ts3, payments_ts4);
```

### External Storage Configuration:
```sql
-- Create tablespaces on different disk groups for parallel I/O
CREATE TABLESPACE payments_ts1 DATAFILE '/disk1/payments_ts1.dbf' SIZE 5G;
CREATE TABLESPACE payments_ts2 DATAFILE '/disk2/payments_ts2.dbf' SIZE 5G;
CREATE TABLESPACE payments_ts3 DATAFILE '/disk3/payments_ts3.dbf' SIZE 5G;
CREATE TABLESPACE payments_ts4 DATAFILE '/disk4/payments_ts4.dbf' SIZE 5G;
```

### Purpose and Performance Improvements:
Hash partitioning on `PaymentID` will distribute payment records evenly across multiple partitions, which is ideal for a high-volume transaction table like Payments. From our workload analysis, we observed that the Payments table is frequently joined in analytical queries:

```sql
JOIN Payments p ON p.BookingID = b.BookingID
```

This partitioning strategy will enable:

1. **Balanced Data Distribution**: Payment records will be evenly distributed across partitions, preventing hotspots.

2. **Parallel Query Processing**: Analytical queries that scan the entire Payments table can be processed in parallel across all partitions.

3. **I/O Distribution**: By storing partitions on different physical disks, I/O operations can be distributed, reducing contention.

4. **Scalability**: The number of partitions can be increased as the payment data grows, maintaining consistent performance.

### Expected Impact on Workload:
- QUERY 1 and QUERY 2: Estimated 25-35% performance improvement for joins involving the Payments table due to parallel processing.
- Financial reporting queries: Significant improvement through parallel scan of partitions.
- Backup and maintenance operations: Reduced time through parallel processing of partitions.

## Experimental Comparison of Partition Types

To validate the effectiveness of our partitioning strategies, we propose the following experiments:

### Experiment 1: Range vs. List Partitioning for Bookings

This experiment will compare the performance of range partitioning by `BookingDate` versus list partitioning by `Status` (e.g., 'Confirmed', 'Pending', 'Canceled') for the Bookings table.

**Test Query:**
```sql
-- Query that filters by both date range and status
SELECT b.BookingID, b.CustomerID, b.PackageID, b.BookingDate, b.Status
FROM Bookings b
WHERE b.BookingDate BETWEEN TO_DATE('2023-01-01', 'YYYY-MM-DD') AND TO_DATE('2023-06-30', 'YYYY-MM-DD')
AND b.Status = 'Confirmed';
```

**Experiment Steps:**
1. Measure baseline performance without partitioning
2. Implement range partitioning by `BookingDate` and measure performance
3. Revert to no partitioning, then implement list partitioning by `Status` and measure performance
4. Compare execution times, I/O statistics, and query plans

**Expected Outcome:**
Range partitioning is expected to perform better for this specific query due to the higher selectivity of the date range condition compared to the status condition.

### Experiment 2: Hash vs. Range Partitioning for Payments

This experiment will compare hash partitioning by `PaymentID` versus range partitioning by `PaymentDate` for the Payments table.

**Test Query:**
```sql
-- Analytical query that aggregates payment data
SELECT TRUNC(p.PaymentDate, 'MM') AS Month, 
       SUM(p.AmountPaid) AS TotalRevenue
FROM Payments p
WHERE p.PaymentDate >= ADD_MONTHS(CURRENT_DATE, -12)
GROUP BY TRUNC(p.PaymentDate, 'MM')
ORDER BY Month;
```

**Experiment Steps:**
1. Measure baseline performance without partitioning
2. Implement hash partitioning by `PaymentID` and measure performance
3. Revert to no partitioning, then implement range partitioning by `PaymentDate` and measure performance
4. Compare execution times, I/O statistics, and query plans

**Expected Outcome:**
Range partitioning by `PaymentDate` is expected to perform better for this specific query due to the date-based filter and grouping, but hash partitioning may show better overall performance across a variety of queries.

## Conclusion

The proposed partitioning strategy addresses the specific performance challenges identified in our Tourism Agency Database workload. By implementing range partitioning for the Bookings table, list partitioning for the Customers table, and hash partitioning for the Payments table, we expect to achieve significant performance improvements across our key transactions.

The use of external storage configurations for different partitions will enable parallel reads and optimize storage utilization, with historical or less frequently accessed data stored on less expensive storage and active or high-priority data stored on premium storage.

The experimental comparisons will provide valuable insights into the relative effectiveness of different partition types for our specific workload patterns, allowing for further refinement of our partitioning strategy.

These improvements will directly benefit our core business operations, including booking management, customer segmentation, and financial reporting, resulting in a more responsive and efficient system overall.
