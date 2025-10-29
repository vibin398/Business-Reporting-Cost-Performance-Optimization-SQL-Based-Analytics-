# AWS Athena Query Optimization: Partitioning & Compression Strategy

## 📊 Project Summary

Optimized SQL queries on a 1GB+ e-commerce order dataset by implementing **Parquet columnar format** and **year-month partitioning strategy**. Achieved **51.8% faster query execution** and **99.5% data scan reduction**, resulting in approximately **170x cost savings per query**.

---

## 🎯 Problem Statement

### Original Challenges
- **Full Table Scans:** Every query scanned the entire 1GB CSV dataset, regardless of filters
- **Slow Performance:** Aggregation queries took 2.554 seconds
- **High Query Costs:** Each query scanned 1.03 GB at $5 per TB = ~$5.15 per query
- **No Partition Pruning:** Queries for specific months still processed entire dataset
- **Inefficient Storage:** CSV format has no compression; wasted storage and bandwidth

### Business Impact
- Slow analytics dashboards and reports
- Expensive query operations on large datasets
- Inability to efficiently analyze data by specific time periods
- Scalability concerns as data grows

---

## ✅ Solution Implemented

### Three-Tier Optimization Strategy

#### **Tier 1: Format Conversion (CSV → Parquet)**
- **Before:** Raw CSV format (no compression, row-based storage)
- **After:** Parquet format with SNAPPY compression (columnar storage)
- **Impact:** 70-80% file size reduction

#### **Tier 2: Partitioning Strategy**
- **Partitioning Scheme:** Year-Month (`order_year`, `order_month`)
- **Benefit:** Enables partition pruning (Athena skips irrelevant partitions)
- **Storage Structure:** Separate Parquet files for each year-month combinatio

---

## 📈 Performance Results

### Metrics Comparison

| Metric | Non-Optimized | Optimized | Improvement |
|--------|---------------|-----------|-------------|
| **Query Runtime** | 2.554 seconds | 1.234 seconds | **51.8% faster** ⚡ |
| **Data Scanned** | 1.03 GB | 5.24 MB | **99.5% reduction** 📉 |
| **Query Cost** | $5.15 | $0.03 | **~170x cheaper** 💰 |
| **Storage Format** | CSV (uncompressed) | Parquet (SNAPPY) | 70-80% compression |
| **Partitioning** | None | Year-Month | Partition pruning enabled |

### Query Execution Flow

```
NON-OPTIMIZED FLOW:
┌─────────────────────────────────────────────────────┐
│ Raw CSV (1 GB) in S3                                │
│ - No compression                                     │
│ - Row-based storage                                  │
└──────────────────┬──────────────────────────────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │  Full Table Scan     │
        │ (All rows processed) │
        └──────────────┬───────┘
                       │
                       ▼
        ┌──────────────────────┐
        │  Date Parsing        │
        │  Aggregation         │
        │  Filtering           │
        └──────────────┬───────┘
                       │
                       ▼
           ⏱️  2.554 seconds
          📊 1.03 GB scanned
          💵 ~$5.15 cost


OPTIMIZED FLOW:
┌────────────────────────────────────────────────────────┐
│ Partitioned Parquet Files in S3 (SNAPPY Compressed)    │
│ /order_year=2022/order_month=7/data.parquet            │
│ /order_year=2022/order_month=8/data.parquet            │
│ /order_year=2025/order_month=1/data.parquet            │
└──────────────────┬───────────────────────────────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │ PARTITION PRUNING    │
        │ (Only 2025 scanned)  │
        └──────────────┬───────┘
                       │
                       ▼
        ┌──────────────────────┐
        │ Aggregation          │
        │ (on 5.24 MB only)    │
        └──────────────┬───────┘
                       │
                       ▼
           ⏱️  1.234 seconds
          📊 5.24 MB scanned
          💵 ~$0.03 cost
```

---

## 🐛 Challenges & Troubleshooting

### Challenge 1: Database Does Not Exist

**Error Message:**
```
FAILED: SemanticException [Error 10072]: Database does not exist: investigation
```

**Root Cause:**
- Attempted to create table in a non-existent `investigation` database
- Athena only has `default` database by default

**Solution:**
```sql
CREATE DATABASE investigation;
```

**Learning:** Always create database before creating tables.

---

### Challenge 2: Table Reading Metadata Instead of CSV Data

**Symptom:**
- `SELECT * FROM orders_raw LIMIT 5` returned cryptic metadata
- Output showed `_col0`, `_col1`, timestamps like `20251029_081854_00039_9dfnv`
- Column values appeared as schema information instead of data

**Root Cause:**
- Table `LOCATION` pointed to S3 folder containing:
  - Parquet metadata files
  - Previous query result folders
  - "Unsaved" subfolder with mixed file types
- Athena was parsing metadata instead of CSV data

**Solution:**

1. **Identify actual data location:**
   ```
   s3://fba-investigation/raw/orders_part_001.csv
   ```

2. **Clean S3 folder structure:**
   - Removed "unsaved" subfolder
   - Ensured only CSV files in `raw/` folder

3. **Recreate table with correct LOCATION:**
   ```sql
   CREATE EXTERNAL TABLE investigation.orders_raw (
       order_id STRING,
       customer_id STRING,
       order_date STRING,
       status STRING,
       payment_method STRING,
       shipping_address STRING,
       billing_address STRING,
       discount_amount DOUBLE,
       tax_amount DOUBLE,
       shipping_cost DOUBLE,
       total_amount DOUBLE,
       currency STRING,
       created_at STRING,
       updated_at STRING,
       subtotal DOUBLE
   )
   ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
   WITH SERDEPROPERTIES (
      "separatorChar" = ",",
      "quoteChar"     = "\""
   )
   LOCATION 's3://fba-investigation/raw/'
   TBLPROPERTIES ('skip.header.line.count'='1');
   ```

**Learning:** Keep S3 organized with separate folders for raw data, Parquet output, and query results.

---

### Challenge 3: Timestamp Parsing Failed (Multiple Formats)

**Error 1: Fractional Seconds Present**
```
INVALID_CAST_ARGUMENT: Value cannot be cast to timestamp: 
2025-05-19T18:28:10.366427
```

**Root Cause:**
- Using `CAST(order_date AS timestamp)` doesn't support fractional seconds
- Some timestamps had format: `2025-05-19T18:28:10.366427`

**Error 2: Inconsistent Format**
```
INVALID_FUNCTION_ARGUMENT: Invalid format: "2024-07-20T08:35:10" is too short
```

**Root Cause:**
- Dataset had mixed timestamp formats
- Some with fractional seconds, some without
- Format string expected all timestamps with same precision

**Solution: Dual-Format Parsing with Fallback**

```sql
COALESCE(
    try(date_parse(order_date, '%Y-%m-%dT%H:%i:%s.%f')),
    try(date_parse(order_date, '%Y-%m-%dT%H:%i:%s'))
)
```

**How it works:**
1. `try()` attempts parsing without throwing error (returns NULL on failure)
2. First tries format WITH fractional seconds (`%f`)
3. If that fails (returns NULL), tries format WITHOUT fractional seconds
4. `COALESCE` returns first non-NULL value

**Learning:** Real-world data is messy. Always handle multiple formats and use `try()` with fallback logic.

---

### Challenge 4: GROUP BY Column Alias Error

**Error Message:**
```
COLUMN_NOT_FOUND: line 18:10: Column 'order_year' cannot be resolved
```

**Root Cause:**
- Attempted to reference column aliases in GROUP BY clause
- In SQL, you cannot use aliases in GROUP BY; must repeat full expression

**Incorrect Approach:**
```sql
SELECT 
    year(...) AS order_year,  -- Alias defined here
    month(...) AS order_month -- Alias defined here
FROM orders_raw
GROUP BY order_year, order_month  -- ❌ Can't use alias in GROUP BY
```

**Correct Approach:**
```sql
SELECT 
    year(...) AS order_year,
    month(...) AS order_month
FROM orders_raw
GROUP BY 
    year(...),  -- ✅ Repeat full expression
    month(...)
```

**Learning:** Never rely on column aliases in GROUP BY. Always repeat the full expression.

---

```
NON-OPTIMIZED QUERY:
├── Table: orders_raw (CSV)
├── Runtime: 2.554 seconds
├── Data Scanned: 1.03 GB
├── Query Type: Full table scan
└── Cost: ~$5.15

OPTIMIZED QUERY:
├── Table: orders_parquet (Parquet with partitions)
├── Runtime: 1.234 seconds
├── Data Scanned: 5.24 MB
├── Query Type: Partition pruning (2025 only)
└── Cost: ~$0.03

IMPROVEMENT:
├── Speed: 51.8% faster (1.32 seconds saved)
├── Data: 99.5% reduction (1.025 GB saved)
├── Cost: ~170x cheaper
└── Efficiency: Query now 52x more efficient
```

---

## 💡 Key Learnings

### 1. **Data Format Matters**
- Parquet columnar storage with compression is 70-80% more efficient than CSV
- Query engines can skip columns entirely in Parquet format
- CSV must load entire rows into memory

### 2. **Partitioning Strategy is Critical**
- Year-month partitioning enables partition pruning
- Queries can skip entire partitions without scanning
- Partition key selection depends on query patterns

### 3. **Real-World Data is Messy**
- Timestamp formats can vary within same dataset
- Always use `try()` with fallback parsing logic
- Test with diverse data samples before production

### 4. **Measurement is Essential**
- Always measure before and after optimization
- Small improvements compound at scale
- Document all metrics for visibility

### 5. **Cost Optimization is Data Optimization**
- Cheaper queries = optimized data architecture
- Partitioning reduces scanned data
- Pre-aggregation saves repeated calculations

---

## 🎓 Technical Skills Demonstrated

- **Cloud Platforms:** AWS Athena, Amazon S3
- **Data Formats:** Parquet, CSV, compression algorithms (SNAPPY)
- **SQL Optimization:** Query optimization, partition pruning, aggregation
- **Database Design:** Partitioning strategy, table design, indexing concepts
- **Performance Analysis:** Benchmarking, metrics tracking, cost optimization
- **Data Engineering:** ETL concepts, data transformation, data quality handling
- **Problem Solving:** Debugging complex issues, root cause analysis, solution design

---

## 🔗 Resources

- [AWS Athena Documentation](https://docs.aws.amazon.com/athena/)
- [Parquet Format Guide](https://parquet.apache.org/)
- [SQL Query Optimization Best Practices](https://docs.aws.amazon.com/athena/latest/ug/querying-supported-statements.html)
- [AWS Athena Partition Pruning](https://docs.aws.amazon.com/athena/latest/ug/partitions.html)

---

## 📝 Author

**Vibin krishna** |(https://www.linkedin.com/in/vibin-krishna-3a713518b)|(vibinkrishna574@gmail.com)

---
