# Experimental Comparison of Partition Types

To validate the effectiveness of our partitioning strategies and provide empirical evidence of their performance benefits, we propose the following detailed experiments. These experiments are designed to compare different partition types for the same purpose, as specifically requested in the project requirements.

## Experiment 1: Comparing Range vs. List vs. Hash Partitioning for Bookings Table

### Objective
Determine which partitioning strategy provides the best performance improvement for the Bookings table across different query patterns.

### Test Queries

#### Query 1A: Date Range Filter (favors Range Partitioning)
```sql
-- Query that filters by date range
SELECT b.BookingID, b.CustomerID, b.PackageID, b.BookingDate, b.Status, p.AmountPaid
FROM Bookings b
JOIN Payments p ON p.BookingID = b.BookingID
WHERE b.BookingDate BETWEEN TO_DATE('2023-01-01', 'YYYY-MM-DD') AND TO_DATE('2023-06-30', 'YYYY-MM-DD')
ORDER BY b.BookingDate;
```

#### Query 1B: Status Filter (favors List Partitioning)
```sql
-- Query that filters by booking status
SELECT b.BookingID, b.CustomerID, b.PackageID, b.BookingDate, b.Status, p.AmountPaid
FROM Bookings b
JOIN Payments p ON p.BookingID = b.BookingID
WHERE b.Status = 'Confirmed'
ORDER BY b.BookingDate;
```

#### Query 1C: Specific BookingID Lookup (favors Hash Partitioning)
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

### Experiment Setup

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

### Data Collection and Analysis

For each partitioning strategy and test query, collect:
- Execution time (min, max, avg)
- Physical reads
- Logical reads
- CPU time
- Query plan (especially partition pruning information)

Create a comparison table:

| Query | Metric | No Partition | Range Partition | List Partition | Hash Partition | Best Strategy |
|-------|--------|--------------|----------------|---------------|---------------|--------------|
| 1A    | Avg Time |            |                |               |               |              |
| 1A    | Phys Reads |          |                |               |               |              |
| 1A    | CPU Time |            |                |               |               |              |
| 1B    | Avg Time |            |                |               |               |              |
| 1B    | Phys Reads |          |                |               |               |              |
| 1B    | CPU Time |            |                |               |               |              |
| 1C    | Avg Time |            |                |               |               |              |
| 1C    | Phys Reads |          |                |               |               |              |
| 1C    | CPU Time |            |                |               |               |              |

### Expected Outcomes

- Query 1A (date range filter) is expected to perform best with Range partitioning due to partition pruning on the date range
- Query 1B (status filter) is expected to perform best with List partitioning due to direct access to the relevant status partition
- Query 1C (BookingID lookup) is expected to perform best with Hash partitioning due to even distribution of BookingIDs

## Experiment 2: Comparing Different Range Partition Granularities

### Objective
Determine the optimal granularity for range partitioning of the Bookings table (quarterly vs. monthly vs. yearly).

### Test Queries

#### Query 2A: Quarterly Analysis
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

#### Query 2B: Monthly Analysis
```sql
-- Query that analyzes bookings by month
SELECT 
    TO_CHAR(b.BookingDate, 'YYYY-MM') AS BookingMonth,
    COUNT(*) AS BookingCount,
    SUM(p.AmountPaid) AS TotalRevenue,
    AVG(p.AmountPaid) AS AvgBookingValue
FROM Bookings b
JOIN Payments p ON p.BookingID = b.BookingID
WHERE b.BookingDate BETWEEN TO_DATE('2022-01-01', 'YYYY-MM-DD') AND TO_DATE('2023-12-31', 'YYYY-MM-DD')
GROUP BY TO_CHAR(b.BookingDate, 'YYYY-MM')
ORDER BY BookingMonth;
```

#### Query 2C: Yearly Analysis with Monthly Breakdown
```sql
-- Query that analyzes bookings by year with monthly breakdown
SELECT 
    EXTRACT(YEAR FROM b.BookingDate) AS BookingYear,
    EXTRACT(MONTH FROM b.BookingDate) AS BookingMonth,
    COUNT(*) AS BookingCount,
    SUM(p.AmountPaid) AS TotalRevenue
FROM Bookings b
JOIN Payments p ON p.BookingID = b.BookingID
WHERE b.BookingDate BETWEEN TO_DATE('2022-01-01', 'YYYY-MM-DD') AND TO_DATE('2023-12-31', 'YYYY-MM-DD')
GROUP BY EXTRACT(YEAR FROM b.BookingDate), EXTRACT(MONTH FROM b.BookingDate)
ORDER BY BookingYear, BookingMonth;
```

### Experiment Setup

1. **Quarterly Partitioning (as in Experiment 1)**:
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

2. **Monthly Partitioning**:
   ```sql
   ALTER TABLE Bookings MODIFY
   PARTITION BY RANGE (BookingDate) (
       PARTITION bookings_2022_01 VALUES LESS THAN (TO_DATE('2022-02-01', 'YYYY-MM-DD')),
       PARTITION bookings_2022_02 VALUES LESS THAN (TO_DATE('2022-03-01', 'YYYY-MM-DD')),
       -- Additional monthly partitions for 2022
       PARTITION bookings_2022_12 VALUES LESS THAN (TO_DATE('2023-01-01', 'YYYY-MM-DD')),
       PARTITION bookings_2023_01 VALUES LESS THAN (TO_DATE('2023-02-01', 'YYYY-MM-DD')),
       -- Additional monthly partitions for 2023
       PARTITION bookings_2023_12 VALUES LESS THAN (TO_DATE('2024-01-01', 'YYYY-MM-DD')),
       PARTITION bookings_future VALUES LESS THAN (MAXVALUE)
   );
   ```

3. **Yearly Partitioning**:
   ```sql
   ALTER TABLE Bookings MODIFY
   PARTITION BY RANGE (BookingDate) (
       PARTITION bookings_2022 VALUES LESS THAN (TO_DATE('2023-01-01', 'YYYY-MM-DD')),
       PARTITION bookings_2023 VALUES LESS THAN (TO_DATE('2024-01-01', 'YYYY-MM-DD')),
       PARTITION bookings_2024 VALUES LESS THAN (TO_DATE('2025-01-01', 'YYYY-MM-DD')),
       PARTITION bookings_future VALUES LESS THAN (MAXVALUE)
   );
   ```

For each partitioning granularity:
- Execute each test query 5 times
- Record execution time, I/O statistics, and query plan for each run
- Calculate average execution time and I/O metrics

### Data Collection and Analysis

Create a comparison table:

| Query | Metric | Quarterly Partitioning | Monthly Partitioning | Yearly Partitioning | Best Granularity |
|-------|--------|------------------------|---------------------|---------------------|-----------------|
| 2A    | Avg Time |                      |                     |                     |                 |
| 2A    | Phys Reads |                    |                     |                     |                 |
| 2A    | CPU Time |                      |                     |                     |                 |
| 2B    | Avg Time |                      |                     |                     |                 |
| 2B    | Phys Reads |                    |                     |                     |                 |
| 2B    | CPU Time |                      |                     |                     |                 |
| 2C    | Avg Time |                      |                     |                     |                 |
| 2C    | Phys Reads |                    |                     |                     |                 |
| 2C    | CPU Time |                      |                     |                     |                 |

### Expected Outcomes

- Query 2A (quarterly analysis) is expected to perform best with quarterly partitioning due to alignment with the analysis granularity
- Query 2B (monthly analysis) is expected to perform best with monthly partitioning for similar reasons
- Query 2C (yearly with monthly breakdown) may show mixed results, but monthly partitioning is likely to perform well due to the monthly grouping

## Experiment 3: Comparing Bitmap vs. B-Tree Indexes on Partitioned Tables

### Objective
Determine whether bitmap or B-tree indexes provide better performance for queries on partitioned tables, particularly for the Customers table with list partitioning.

### Test Queries

#### Query 3A: Customer Tier Analysis
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

#### Query 3B: Customer Demographics by Tier
```sql
-- Query that analyzes customer demographics by tier
SELECT 
    c.CustomerTier,
    CASE 
        WHEN EXTRACT(YEAR FROM SYSDATE) - EXTRACT(YEAR FROM c.DateOfBirth) < 30 THEN 'Under 30'
        WHEN EXTRACT(YEAR FROM SYSDATE) - EXTRACT(YEAR FROM c.DateOfBirth) BETWEEN 30 AND 45 THEN '30-45'
        WHEN EXTRACT(YEAR FROM SYSDATE) - EXTRACT(YEAR FROM c.DateOfBirth) BETWEEN 46 AND 60 THEN '46-60'
        ELSE 'Over 60'
    END AS AgeGroup,
    COUNT(*) AS CustomerCount,
    SUM(p.AmountPaid) AS TotalSpent
FROM Customers c
JOIN Bookings b ON c.CustomerID = b.CustomerID
JOIN Payments p ON b.BookingID = p.BookingID
GROUP BY 
    c.CustomerTier,
    CASE 
        WHEN EXTRACT(YEAR FROM SYSDATE) - EXTRACT(YEAR FROM c.DateOfBirth) < 30 THEN 'Under 30'
        WHEN EXTRACT(YEAR FROM SYSDATE) - EXTRACT(YEAR FROM c.DateOfBirth) BETWEEN 30 AND 45 THEN '30-45'
        WHEN EXTRACT(YEAR FROM SYSDATE) - EXTRACT(YEAR FROM c.DateOfBirth) BETWEEN 46 AND 60 THEN '46-60'
        ELSE 'Over 60'
    END
ORDER BY c.CustomerTier, AgeGroup;
```

### Experiment Setup

1. **List Partitioning with No Additional Indexes**:
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

2. **List Partitioning with B-Tree Indexes**:
   ```sql
   -- Create B-tree index on DateOfBirth
   CREATE INDEX idx_customers_dob ON Customers(DateOfBirth)
   LOCAL;
   
   -- Create B-tree index on CustomerID
   CREATE INDEX idx_customers_id ON Customers(CustomerID)
   LOCAL;
   ```

3. **List Partitioning with Bitmap Indexes**:
   ```sql
   -- Drop B-tree indexes first
   DROP INDEX idx_customers_dob;
   DROP INDEX idx_customers_id;
   
   -- Create bitmap index on DateOfBirth
   CREATE BITMAP INDEX bmp_idx_customers_dob ON Customers(DateOfBirth)
   LOCAL;
   
   -- Create bitmap index on CustomerID
   CREATE BITMAP INDEX bmp_idx_customers_id ON Customers(CustomerID)
   LOCAL;
   ```

For each configuration:
- Execute each test query 5 times
- Record execution time, I/O statistics, and query plan for each run
- Calculate average execution time and I/O metrics

### Data Collection and Analysis

Create a comparison table:

| Query | Metric | No Indexes | B-Tree Indexes | Bitmap Indexes | Best Indexing Strategy |
|-------|--------|------------|---------------|----------------|------------------------|
| 3A    | Avg Time |          |               |                |                        |
| 3A    | Phys Reads |        |               |                |                        |
| 3A    | CPU Time |          |               |                |                        |
| 3B    | Avg Time |          |               |                |                        |
| 3B    | Phys Reads |        |               |                |                        |
| 3B    | CPU Time |          |               |                |                        |

### Expected Outcomes

- For Query 3A (customer tier analysis), bitmap indexes are expected to perform better due to the low cardinality of the CustomerTier column and the aggregation nature of the query
- For Query 3B (customer demographics by tier), the results may be mixed, but bitmap indexes might still have an advantage due to the grouping by discrete age ranges

## Conclusion

These experiments will provide empirical evidence of the performance benefits of different partitioning strategies for the Tourism Agency Database. By comparing range, list, and hash partitioning, as well as different partition granularities and indexing strategies, we can identify the optimal configuration for our specific workload patterns.

The results will guide our final partitioning implementation decisions and provide valuable insights for future database optimization efforts. Additionally, these experiments fulfill the project requirement to compare different types of partitions for the same purpose, demonstrating a thorough understanding of partitioning concepts and their practical applications.
