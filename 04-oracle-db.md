# 04 — Oracle Database (Indexing, Execution Plans, Partitioning, Query Tuning, HikariCP)

> **Priority**: 🟡 High  
> **Estimated Study Time**: 1 day  
> Markers: 🔥 = Frequently Asked | 💎 = Differentiator

---

## Section 1: Indexing

### 🔥 Q1. Explain types of indexes in Oracle. When would you use each?

**Answer:**

| Index Type | Structure | Use Case | Example |
|-----------|-----------|----------|---------|
| **B-Tree** (default) | Balanced tree | High cardinality columns (unique/near-unique) | `payment_id`, `txn_id`, `email` |
| **Bitmap** | Bit arrays | Low cardinality columns (few distinct values) | `status`, `gender`, `country` |
| **Composite** | Multi-column B-Tree | Queries filtering on multiple columns | `(merchant_id, created_at)` |
| **Function-based** | Index on expression | Queries using functions | `UPPER(email)`, `TRUNC(created_at)` |
| **Unique** | B-Tree with uniqueness | Primary keys, unique constraints | `txn_id`, `idempotency_key` |
| **Partial/Filtered** | Conditional index | Index subset of rows | `WHERE status = 'PENDING'` |
| **Reverse Key** | Reversed bytes B-Tree | Avoid hot blocks in sequences | `payment_id` (sequence-generated) |

```sql
-- B-Tree Index (most common)
CREATE INDEX idx_payment_txn_id ON payments(txn_id);

-- Composite Index (leftmost prefix rule applies)
CREATE INDEX idx_payment_merchant_date ON payments(merchant_id, created_at DESC);
-- ✅ Works for: WHERE merchant_id = 'M001'
-- ✅ Works for: WHERE merchant_id = 'M001' AND created_at > DATE '2024-01-01'
-- ❌ Does NOT work for: WHERE created_at > DATE '2024-01-01' (skips first column)

-- Function-based Index
CREATE INDEX idx_payment_date_trunc ON payments(TRUNC(created_at));
-- Now this query uses the index:
-- SELECT * FROM payments WHERE TRUNC(created_at) = DATE '2024-01-15';

-- Bitmap Index (OLAP/warehouse, NOT for OLTP!)
CREATE BITMAP INDEX idx_payment_status ON payments(status);
-- ⚠️ Never use bitmap indexes on tables with frequent DML — causes massive lock contention

-- Reverse Key Index (prevents hot blocks for sequential inserts)
CREATE INDEX idx_payment_id_rev ON payments(payment_id) REVERSE;
```

**Index Selection Guidelines:**
- **High cardinality** (many distinct values) → B-Tree
- **Low cardinality** (few distinct values) + read-heavy → Bitmap
- **Composite**: Put most selective column first, or the column used in equality conditions
- **Don't over-index**: Each index slows down INSERT/UPDATE/DELETE

---

### 🔥 Q2. What is the leftmost prefix rule in composite indexes?

**Answer:**

A composite index on `(A, B, C)` can be used for queries that filter on:
- `A` alone ✅
- `A, B` ✅
- `A, B, C` ✅
- `A, C` ✅ (uses A, skip-scans C in some cases)
- `B` alone ❌ (index not used, unless Index Skip Scan)
- `B, C` ❌
- `C` alone ❌

```sql
-- Index: CREATE INDEX idx_payments ON payments(merchant_id, status, created_at);

-- ✅ Uses index (full)
SELECT * FROM payments 
WHERE merchant_id = 'M001' AND status = 'COMPLETED' AND created_at > SYSDATE - 7;

-- ✅ Uses index (partial — first 2 columns)
SELECT * FROM payments WHERE merchant_id = 'M001' AND status = 'COMPLETED';

-- ✅ Uses index (first column only)
SELECT * FROM payments WHERE merchant_id = 'M001';

-- ❌ Does NOT use index efficiently
SELECT * FROM payments WHERE status = 'COMPLETED'; -- Skips merchant_id

-- ❌ Does NOT use index
SELECT * FROM payments WHERE created_at > SYSDATE - 7; -- Skips first 2 columns
```

**Design Tip**: Order columns in composite index by:
1. Equality conditions first (`=`)
2. Range conditions last (`>`, `<`, `BETWEEN`)
3. Most selective (highest cardinality) first among equals

---

## Section 2: Execution Plans

### 🔥 Q3. How do you read and analyze an Oracle execution plan?

**Answer:**

```sql
-- Generate execution plan
EXPLAIN PLAN FOR
SELECT p.*, m.name 
FROM payments p 
JOIN merchants m ON p.merchant_id = m.id
WHERE p.status = 'COMPLETED' 
AND p.created_at > SYSDATE - 30;

-- View the plan
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(format => 'ALL'));
```

**Reading the Plan (inside-out, top-to-bottom):**

```
---------------------------------------------------------------------------
| Id | Operation                    | Name                  | Rows | Cost |
---------------------------------------------------------------------------
|  0 | SELECT STATEMENT             |                       |  500 |  125 |
|  1 |  NESTED LOOPS                |                       |  500 |  125 |
|  2 |   TABLE ACCESS BY INDEX ROWID| PAYMENTS              |  500 |  100 |
|* 3 |    INDEX RANGE SCAN          | IDX_PAY_STATUS_DATE   |  500 |   10 |
|  4 |   TABLE ACCESS BY INDEX ROWID| MERCHANTS             |    1 |    1 |
|* 5 |    INDEX UNIQUE SCAN         | PK_MERCHANTS          |    1 |    0 |
---------------------------------------------------------------------------

Predicate Information:
  3 - access("P"."STATUS"='COMPLETED' AND "P"."CREATED_AT">SYSDATE-30)
  5 - access("P"."MERCHANT_ID"="M"."ID")
```

**Key Operations to Know:**

| Operation | Meaning | Good/Bad |
|-----------|---------|----------|
| `TABLE ACCESS FULL` | Full table scan | ❌ Bad for large tables |
| `INDEX RANGE SCAN` | Scans range of index | ✅ Good |
| `INDEX UNIQUE SCAN` | Finds exactly one row | ✅ Best |
| `INDEX FULL SCAN` | Scans entire index (ordered) | 🟡 Depends |
| `INDEX FAST FULL SCAN` | Scans entire index (unordered, multi-block) | 🟡 Depends |
| `NESTED LOOPS` | For each row in outer, scan inner | ✅ Good for small outer set |
| `HASH JOIN` | Build hash table, probe | ✅ Good for large joins |
| `SORT MERGE JOIN` | Sort both, merge | 🟡 When both are large and sorted |
| `TABLE ACCESS BY INDEX ROWID` | Fetch row from table using index | ✅ Normal |

**Red Flags in Execution Plans:**
1. `TABLE ACCESS FULL` on large tables → Missing index
2. High `COST` relative to rows → Inefficient plan
3. `SORT ORDER BY` with large rows → Missing index for ORDER BY
4. Cardinality estimates way off → Stale statistics

```sql
-- Gather fresh statistics
EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA_NAME', 'PAYMENTS', CASCADE => TRUE);
```

---

## Section 3: Query Tuning

### 🔥 Q4. How do you tune a slow Oracle query? Walk through your approach.

**Answer:**

**Step-by-step approach:**

**1. Identify the slow query:**
```sql
-- Find top SQL by elapsed time
SELECT sql_id, elapsed_time/1000000 as elapsed_secs, executions, 
       elapsed_time/GREATEST(executions,1)/1000 as avg_ms,
       sql_text
FROM v$sql 
WHERE elapsed_time/GREATEST(executions,1) > 1000000  -- > 1 second avg
ORDER BY elapsed_time DESC
FETCH FIRST 10 ROWS ONLY;
```

**2. Get the execution plan:**
```sql
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('sql_id_here', NULL, 'ALL'));
```

**3. Check for common issues:**

| Issue | Symptom | Fix |
|-------|---------|-----|
| Missing index | Full table scan | Create appropriate index |
| Stale statistics | Wrong cardinality estimates | `DBMS_STATS.GATHER_TABLE_STATS` |
| Implicit conversion | Index not used | Fix data types (`TO_DATE`, `TO_NUMBER`) |
| Function on indexed column | Index not used | Function-based index or rewrite |
| SELECT * | Fetching unnecessary columns | Select only needed columns |
| Correlated subquery | Executes per row | Rewrite as JOIN |
| Missing bind variables | Hard parsing each time | Use bind variables |

**4. Common Rewrites:**

```sql
-- ❌ Slow: Function on indexed column prevents index use
SELECT * FROM payments WHERE TRUNC(created_at) = DATE '2024-01-15';

-- ✅ Fast: Range scan on indexed column
SELECT * FROM payments 
WHERE created_at >= DATE '2024-01-15' 
AND created_at < DATE '2024-01-16';

-- ❌ Slow: Implicit conversion (varchar compared to number)
SELECT * FROM payments WHERE txn_id = 12345;  -- txn_id is VARCHAR2

-- ✅ Fast: Correct type
SELECT * FROM payments WHERE txn_id = '12345';

-- ❌ Slow: Correlated subquery
SELECT * FROM payments p 
WHERE p.amount > (SELECT AVG(amount) FROM payments WHERE merchant_id = p.merchant_id);

-- ✅ Fast: Rewrite with JOIN
SELECT p.* FROM payments p
JOIN (SELECT merchant_id, AVG(amount) as avg_amount FROM payments GROUP BY merchant_id) avg
ON p.merchant_id = avg.merchant_id AND p.amount > avg.avg_amount;

-- ❌ Slow: NOT IN with NULLs
SELECT * FROM payments WHERE merchant_id NOT IN (SELECT id FROM blocked_merchants);

-- ✅ Fast: NOT EXISTS
SELECT * FROM payments p 
WHERE NOT EXISTS (SELECT 1 FROM blocked_merchants b WHERE b.id = p.merchant_id);
```

**5. Use Hints (last resort):**
```sql
-- Force index usage
SELECT /*+ INDEX(p IDX_PAYMENT_MERCHANT_DATE) */ * 
FROM payments p WHERE p.merchant_id = 'M001';

-- Force parallel execution
SELECT /*+ PARALLEL(p, 4) */ * FROM payments p WHERE created_at > SYSDATE - 365;

-- Force hash join
SELECT /*+ USE_HASH(p m) */ p.*, m.name 
FROM payments p JOIN merchants m ON p.merchant_id = m.id;
```

---

## Section 4: Partitioning

### 🔥 Q5. Explain Oracle table partitioning. When and how would you use it?

**Answer:**

Partitioning divides a large table into smaller, manageable pieces while appearing as a single table.

**Types:**

| Type | Partition By | Use Case |
|------|-------------|----------|
| **Range** | Value ranges | Date-based (most common for payments) |
| **List** | Discrete values | Status, region, country |
| **Hash** | Hash of column | Even distribution, no natural range |
| **Composite** | Range-List, Range-Hash | Large tables with multiple access patterns |
| **Interval** | Auto-created range | Auto-extending date partitions |

```sql
-- Range Partitioning (most common for payment tables)
CREATE TABLE payments (
    payment_id    NUMBER GENERATED ALWAYS AS IDENTITY,
    txn_id        VARCHAR2(64) NOT NULL,
    merchant_id   VARCHAR2(32) NOT NULL,
    amount        NUMBER(15,2) NOT NULL,
    status        VARCHAR2(20) NOT NULL,
    created_at    TIMESTAMP NOT NULL,
    CONSTRAINT pk_payments PRIMARY KEY (payment_id, created_at)
) PARTITION BY RANGE (created_at) (
    PARTITION p_2024_q1 VALUES LESS THAN (TIMESTAMP '2024-04-01 00:00:00'),
    PARTITION p_2024_q2 VALUES LESS THAN (TIMESTAMP '2024-07-01 00:00:00'),
    PARTITION p_2024_q3 VALUES LESS THAN (TIMESTAMP '2024-10-01 00:00:00'),
    PARTITION p_2024_q4 VALUES LESS THAN (TIMESTAMP '2025-01-01 00:00:00'),
    PARTITION p_future   VALUES LESS THAN (MAXVALUE)
);

-- Interval Partitioning (auto-creates monthly partitions)
CREATE TABLE payment_events (
    event_id      NUMBER GENERATED ALWAYS AS IDENTITY,
    payment_id    NUMBER NOT NULL,
    event_type    VARCHAR2(50),
    event_data    CLOB,
    created_at    TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at)
  INTERVAL (NUMTOYMINTERVAL(1, 'MONTH')) (
    PARTITION p_initial VALUES LESS THAN (TIMESTAMP '2024-01-01 00:00:00')
);

-- List Partitioning
CREATE TABLE payments_by_region (
    payment_id NUMBER,
    region     VARCHAR2(10),
    amount     NUMBER(15,2)
) PARTITION BY LIST (region) (
    PARTITION p_india VALUES ('IN'),
    PARTITION p_us    VALUES ('US'),
    PARTITION p_eu    VALUES ('UK', 'DE', 'FR'),
    PARTITION p_other VALUES (DEFAULT)
);
```

**Benefits:**
- **Partition Pruning**: Queries only scan relevant partitions
- **Partition-wise Joins**: Parallel joins on partitioned tables
- **Easy Maintenance**: Drop old partitions instead of DELETE (instant, no undo)
- **Independent Backups**: Backup/restore individual partitions

```sql
-- Partition pruning in action
SELECT * FROM payments WHERE created_at BETWEEN DATE '2024-07-01' AND DATE '2024-09-30';
-- Only scans p_2024_q3 partition!

-- Drop old data (instant, no undo logs)
ALTER TABLE payments DROP PARTITION p_2024_q1;

-- Move partition to cheaper storage
ALTER TABLE payments MOVE PARTITION p_2024_q1 TABLESPACE archive_ts;
```

---

## Section 5: Materialized Views

### 💎 Q6. What are Materialized Views? How do you use them for reporting?

**Answer:**

A Materialized View (MV) is a **pre-computed, stored result set** of a query. Unlike regular views, MVs store data physically.

```sql
-- Materialized View for daily payment summary (reporting)
CREATE MATERIALIZED VIEW mv_daily_payment_summary
BUILD IMMEDIATE           -- Populate now
REFRESH FAST              -- Incremental refresh (not full rebuild)
ON DEMAND                 -- Manual refresh (or ON COMMIT for auto)
ENABLE QUERY REWRITE      -- Optimizer can use MV transparently
AS
SELECT 
    TRUNC(created_at) as payment_date,
    merchant_id,
    status,
    COUNT(*) as txn_count,
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount,
    MIN(amount) as min_amount,
    MAX(amount) as max_amount
FROM payments
GROUP BY TRUNC(created_at), merchant_id, status;

-- Create materialized view log (required for FAST refresh)
CREATE MATERIALIZED VIEW LOG ON payments
WITH ROWID, SEQUENCE (merchant_id, status, amount, created_at)
INCLUDING NEW VALUES;

-- Refresh the MV
EXEC DBMS_MVIEW.REFRESH('MV_DAILY_PAYMENT_SUMMARY', 'F'); -- F = Fast refresh

-- Schedule automatic refresh
BEGIN
    DBMS_SCHEDULER.CREATE_JOB(
        job_name   => 'REFRESH_PAYMENT_SUMMARY',
        job_type   => 'PLSQL_BLOCK',
        job_action => 'BEGIN DBMS_MVIEW.REFRESH(''MV_DAILY_PAYMENT_SUMMARY'', ''F''); END;',
        start_date => SYSTIMESTAMP,
        repeat_interval => 'FREQ=HOURLY;INTERVAL=1',
        enabled    => TRUE
    );
END;
/
```

**Query Rewrite**: Oracle optimizer automatically rewrites queries to use MV:
```sql
-- User writes this:
SELECT merchant_id, SUM(amount) FROM payments 
WHERE TRUNC(created_at) = DATE '2024-01-15' GROUP BY merchant_id;

-- Oracle rewrites to:
SELECT merchant_id, total_amount FROM mv_daily_payment_summary 
WHERE payment_date = DATE '2024-01-15';
-- Much faster! Reads pre-aggregated data
```

---

## Section 6: Deadlocks

### 🔥 Q7. How do deadlocks occur in Oracle? How do you prevent them?

**Answer:**

A deadlock occurs when two sessions each hold a lock that the other needs.

```
Session 1                          Session 2
─────────                          ─────────
UPDATE accounts SET balance=900    
WHERE id=1;  (locks row 1)        
                                   UPDATE accounts SET balance=1100
                                   WHERE id=2;  (locks row 2)
                                   
UPDATE accounts SET balance=1100   
WHERE id=2;  (WAITS for row 2)    
                                   UPDATE accounts SET balance=900
                                   WHERE id=1;  (WAITS for row 1)
                                   
         *** DEADLOCK! ***
Oracle detects and rolls back one session (ORA-00060)
```

**Prevention Strategies:**

1. **Consistent Lock Ordering**: Always lock resources in the same order
```java
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    // Always lock lower ID first
    Long firstId = Math.min(fromId, toId);
    Long secondId = Math.max(fromId, toId);
    
    Account first = accountRepo.findByIdForUpdate(firstId);
    Account second = accountRepo.findByIdForUpdate(secondId);
    // No deadlock possible!
}
```

2. **Use SELECT FOR UPDATE with NOWAIT or WAIT timeout**:
```sql
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
-- Fails immediately if locked (ORA-00054)

SELECT * FROM accounts WHERE id = 1 FOR UPDATE WAIT 5;
-- Waits up to 5 seconds, then fails
```

3. **Keep transactions short**: Minimize time between acquiring locks and committing

4. **Use optimistic locking** where possible (no DB locks at all)

---

## Section 7: Connection Pooling (HikariCP)

### 🔥 Q8. How does HikariCP work? What are the key configuration parameters?

**Answer:**

HikariCP is the **default connection pool** in Spring Boot. It's the fastest Java connection pool.

```yaml
# application.yml — Production HikariCP configuration
spring:
  datasource:
    url: jdbc:oracle:thin:@//prod-db:1521/PAYMENTDB
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      pool-name: PaymentServicePool
      
      # Pool sizing
      minimum-idle: 10          # Min connections to maintain
      maximum-pool-size: 20     # Max connections (CRITICAL!)
      
      # Timeouts
      connection-timeout: 5000   # 5s — max wait for connection from pool
      idle-timeout: 300000       # 5min — idle connection lifetime
      max-lifetime: 1800000      # 30min — max connection lifetime (< DB timeout)
      
      # Validation
      validation-timeout: 3000   # 3s — max wait for connection validation
      
      # Leak detection
      leak-detection-threshold: 30000  # 30s — log warning if connection held > 30s
      
      # Oracle-specific
      data-source-properties:
        oracle.jdbc.defaultNCharColumnType: true
        oracle.net.CONNECT_TIMEOUT: 5000
```

**Pool Sizing Formula:**
```
connections = ((core_count * 2) + effective_spindle_count)

For a payment service on 4-core machine with SSD:
connections = (4 * 2) + 1 = 9 ≈ 10

Rule of thumb: Start with 10, load test, adjust.
Maximum: 20-30 (more connections ≠ more throughput due to context switching)
```

**Common Issues:**

| Issue | Symptom | Fix |
|-------|---------|-----|
| Connection leak | Pool exhausted, app hangs | Enable `leak-detection-threshold` |
| Pool too small | `ConnectionTimeoutException` | Increase `maximum-pool-size` |
| Pool too large | DB CPU spike, context switching | Reduce pool size |
| Stale connections | Random SQL errors | Set `max-lifetime` < DB idle timeout |
| Slow validation | High latency spikes | Use `connection-test-query: SELECT 1 FROM DUAL` |

```java
// Monitoring HikariCP metrics with Micrometer
@Configuration
public class HikariMetricsConfig {
    
    @Bean
    public HikariDataSource dataSource(DataSourceProperties properties, MeterRegistry registry) {
        HikariDataSource ds = properties.initializeDataSourceBuilder()
            .type(HikariDataSource.class).build();
        ds.setMetricRegistry(registry);
        return ds;
    }
}

// Key metrics to monitor:
// hikaricp_connections_active — currently in use
// hikaricp_connections_idle — available in pool
// hikaricp_connections_pending — threads waiting for connection
// hikaricp_connections_timeout_total — connection timeout count
```

---

## Section 8: PL/SQL Essentials

### 💎 Q9. When would you use PL/SQL in a Spring Boot application?

**Answer:**

Use PL/SQL for operations that are **better done in the database**:

| Use Case | Why PL/SQL | Why Not Java |
|----------|-----------|-------------|
| Bulk data processing | No network roundtrips | N+1 problem, slow |
| Complex business rules on data | Close to data | Too many DB calls |
| Reconciliation | Set-based operations | Row-by-row is slow |
| Data migration | Direct table access | ORM overhead |
| Scheduled DB maintenance | DB scheduler | App may be down |

```sql
-- Stored Procedure: Daily payment reconciliation
CREATE OR REPLACE PROCEDURE reconcile_payments(
    p_date IN DATE,
    p_merchant_id IN VARCHAR2,
    p_result OUT SYS_REFCURSOR
) AS
    v_count NUMBER := 0;
    v_mismatch NUMBER := 0;
BEGIN
    -- Compare internal records with gateway records
    FOR rec IN (
        SELECT p.txn_id, p.amount as internal_amount, 
               g.amount as gateway_amount, p.status
        FROM payments p
        LEFT JOIN gateway_settlements g ON p.txn_id = g.txn_id
        WHERE TRUNC(p.created_at) = p_date
        AND p.merchant_id = p_merchant_id
    ) LOOP
        v_count := v_count + 1;
        
        IF rec.internal_amount != rec.gateway_amount OR rec.gateway_amount IS NULL THEN
            v_mismatch := v_mismatch + 1;
            INSERT INTO reconciliation_exceptions (txn_id, internal_amount, 
                gateway_amount, exception_type, created_at)
            VALUES (rec.txn_id, rec.internal_amount, rec.gateway_amount,
                CASE WHEN rec.gateway_amount IS NULL THEN 'MISSING_AT_GATEWAY'
                     ELSE 'AMOUNT_MISMATCH' END, SYSTIMESTAMP);
        END IF;
    END LOOP;
    
    COMMIT;
    
    OPEN p_result FOR
        SELECT v_count as total_records, v_mismatch as mismatches,
               v_count - v_mismatch as matched;
END;
/
```

**Calling from Spring Boot:**
```java
@Repository
public class ReconciliationRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public ReconciliationResult reconcile(LocalDate date, String merchantId) {
        return jdbcTemplate.execute((ConnectionCallback<ReconciliationResult>) conn -> {
            try (CallableStatement cs = conn.prepareCall("{call reconcile_payments(?, ?, ?)}")) {
                cs.setDate(1, java.sql.Date.valueOf(date));
                cs.setString(2, merchantId);
                cs.registerOutParameter(3, OracleTypes.CURSOR);
                cs.execute();
                
                try (ResultSet rs = (ResultSet) cs.getObject(3)) {
                    if (rs.next()) {
                        return new ReconciliationResult(
                            rs.getInt("total_records"),
                            rs.getInt("mismatches"),
                            rs.getInt("matched")
                        );
                    }
                }
            }
            return null;
        });
    }
}
```

---

## Quick Revision Checklist

- [ ] B-Tree Index: High cardinality, default choice
- [ ] Bitmap Index: Low cardinality, OLAP only (never OLTP)
- [ ] Composite Index: Leftmost prefix rule, equality first then range
- [ ] Function-based Index: When queries use functions on columns
- [ ] Execution Plan: EXPLAIN PLAN → DBMS_XPLAN.DISPLAY, read inside-out
- [ ] Red Flags: Full table scan, high cost, stale statistics
- [ ] Query Tuning: Check plan → fix index → fix query → hints (last resort)
- [ ] Partitioning: Range (dates), List (status), Interval (auto-extend)
- [ ] Partition Pruning: Only scans relevant partitions
- [ ] Materialized Views: Pre-computed aggregates, FAST refresh, query rewrite
- [ ] Deadlocks: Consistent lock ordering, short transactions, NOWAIT
- [ ] HikariCP: Pool size = (cores × 2) + spindles, leak detection, max-lifetime
- [ ] PL/SQL: Bulk operations, reconciliation, close-to-data processing