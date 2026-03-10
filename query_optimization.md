# Query Optimization

**50 production-ready Dune SQL optimization patterns and timeout fixes**

**Covers:** Timeout fixes, `DATE_TRUNC` filtering, avoiding `SELECT *`, `LIMIT` strategies, CTEs vs subqueries, indexing patterns

---

## Time Filtering (Critical for Performance)

### 1. Always filter by block_time first (indexed column)
```sql
-- ✅ GOOD: Filter on indexed column first
SELECT hash, "from", "to", value
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
LIMIT 100

-- ❌ BAD: Filter on non-indexed column first causes full scan
-- SELECT * FROM ethereum.transactions WHERE "from" = 0x... 
```

### 2. Use DATE_TRUNC for aggregations
```sql
-- ✅ GOOD: DATE_TRUNC is optimized
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    COUNT(*) AS tx_count
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 1 DESC
```

### 3. Avoid functions on indexed columns in WHERE
```sql
-- ✅ GOOD: Direct comparison
SELECT *
FROM ethereum.transactions
WHERE block_time >= DATE '2024-01-01'
    AND block_time < DATE '2024-01-02'
LIMIT 100

-- ❌ BAD: Function on indexed column prevents index use
-- WHERE DATE(block_time) = DATE '2024-01-01'
```

### 4. Use block_number for recent queries
```sql
-- ✅ GOOD: block_number is also indexed
SELECT *
FROM ethereum.transactions
WHERE block_number >= (SELECT MAX(block_number) - 1000 FROM ethereum.transactions)
LIMIT 100
```

### 5. Progressive time windows for debugging
```sql
-- Start with smallest window, expand if needed
-- Step 1: 10 minutes
SELECT COUNT(*) FROM dex.trades WHERE block_time >= NOW() - INTERVAL '10' MINUTE;

-- Step 2: 1 hour
SELECT COUNT(*) FROM dex.trades WHERE block_time >= NOW() - INTERVAL '1' HOUR;

-- Step 3: 24 hours
SELECT COUNT(*) FROM dex.trades WHERE block_time >= NOW() - INTERVAL '24' HOUR;
```

---

## Avoiding SELECT *

### 6. Select only needed columns
```sql
-- ✅ GOOD: Explicit column selection
SELECT 
    hash,
    block_time,
    "from",
    "to",
    value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100

-- ❌ BAD: SELECT * transfers unnecessary data
-- SELECT * FROM ethereum.transactions
```

### 7. Use column aliases for clarity
```sql
SELECT 
    evt_tx_hash AS tx_hash,
    evt_block_time AS block_time,
    "from" AS sender,
    "to" AS recipient,
    value / 1e6 AS usdc_amount
FROM erc20_ethereum.evt_Transfer
WHERE contract_address = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
    AND evt_block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100
```

### 8. Avoid selecting large binary columns unless needed
```sql
-- ✅ GOOD: Skip data column if not needed
SELECT 
    hash,
    block_time,
    "from",
    "to",
    gas_used
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100

-- ❌ BAD: 'data' column can be very large
-- SELECT hash, "from", "to", data FROM ethereum.transactions
```

---

## LIMIT Strategies

### 9. Always use LIMIT during development
```sql
-- ✅ GOOD: Limit results while building query
SELECT 
    block_time,
    "from",
    "to",
    value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY value DESC
LIMIT 100
```

### 10. Use TOP N pattern instead of full aggregation
```sql
-- ✅ GOOD: Limit before aggregating large datasets
WITH top_traders AS (
    SELECT taker
    FROM dex.trades
    WHERE blockchain = 'ethereum'
        AND block_time >= NOW() - INTERVAL '24' HOUR
    GROUP BY 1
    ORDER BY COUNT(*) DESC
    LIMIT 100
)
SELECT 
    t.taker,
    COUNT(*) AS trade_count,
    SUM(t.amount_usd) AS volume
FROM dex.trades t
JOIN top_traders tt ON t.taker = tt.taker
WHERE t.block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 3 DESC
```

### 11. OFFSET for pagination (use carefully)
```sql
-- Page through results (costly for large offsets)
SELECT 
    hash,
    block_time,
    "from",
    value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100 OFFSET 0  -- Page 1
-- LIMIT 100 OFFSET 100  -- Page 2
```

### 12. Use cursor-based pagination for large datasets
```sql
-- ✅ GOOD: Cursor pagination (use last seen value)
SELECT 
    hash,
    block_number,
    block_time
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND block_number < 19000000  -- cursor: last seen block
ORDER BY block_number DESC
LIMIT 100
```

---

## CTEs vs Subqueries

### 13. CTEs for readability and reuse
```sql
-- ✅ GOOD: CTEs make complex queries readable
WITH daily_volume AS (
    SELECT 
        DATE_TRUNC('day', block_time) AS day,
        SUM(amount_usd) AS volume
    FROM dex.trades
    WHERE blockchain = 'ethereum'
        AND block_time >= NOW() - INTERVAL '30' DAY
    GROUP BY 1
),
volume_stats AS (
    SELECT 
        AVG(volume) AS avg_daily_volume,
        MAX(volume) AS max_daily_volume
    FROM daily_volume
)
SELECT 
    d.*,
    s.avg_daily_volume,
    CASE WHEN d.volume > s.avg_daily_volume THEN 'Above Avg' ELSE 'Below Avg' END AS status
FROM daily_volume d
CROSS JOIN volume_stats s
ORDER BY d.day DESC
```

### 14. Materialize heavy CTEs early
```sql
-- ✅ GOOD: Filter early in CTE
WITH filtered_trades AS (
    SELECT 
        block_time,
        taker,
        amount_usd
    FROM dex.trades
    WHERE blockchain = 'ethereum'
        AND block_time >= NOW() - INTERVAL '7' DAY
        AND amount_usd > 1000  -- Filter early
)
SELECT 
    DATE_TRUNC('day', block_time) AS day,
    taker,
    SUM(amount_usd) AS volume
FROM filtered_trades
GROUP BY 1, 2
ORDER BY 3 DESC
LIMIT 100
```

### 15. Avoid correlated subqueries
```sql
-- ✅ GOOD: JOIN instead of correlated subquery
SELECT 
    t.hash,
    t.block_time,
    l.name AS wallet_label
FROM ethereum.transactions t
LEFT JOIN labels.all l ON t."from" = l.address AND l.blockchain = 'ethereum'
WHERE t.block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100

-- ❌ BAD: Correlated subquery runs for each row
-- SELECT hash, (SELECT name FROM labels.all WHERE address = t."from") FROM ethereum.transactions t
```

### 16. Use EXISTS instead of IN for large lists
```sql
-- ✅ GOOD: EXISTS is often faster than IN
SELECT hash, block_time, "from"
FROM ethereum.transactions t
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND EXISTS (
        SELECT 1 FROM labels.all l 
        WHERE l.address = t."from" 
            AND l.blockchain = 'ethereum'
            AND l.category = 'cex'
    )
LIMIT 100
```

---

## Join Optimization

### 17. Join on indexed columns
```sql
-- ✅ GOOD: Join on tx_hash (indexed)
SELECT 
    t.hash,
    t.block_time,
    l.contract_address,
    l.topic0
FROM ethereum.transactions t
JOIN ethereum.logs l ON t.hash = l.tx_hash
WHERE t.block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100
```

### 18. Filter before joining
```sql
-- ✅ GOOD: Filter in CTEs before joining
WITH recent_txs AS (
    SELECT hash, block_time, "from", "to"
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '1' HOUR
),
recent_logs AS (
    SELECT tx_hash, contract_address, topic0
    FROM ethereum.logs
    WHERE block_time >= NOW() - INTERVAL '1' HOUR
)
SELECT 
    t.hash,
    t.block_time,
    l.contract_address
FROM recent_txs t
JOIN recent_logs l ON t.hash = l.tx_hash
LIMIT 100
```

### 19. Use appropriate join types
```sql
-- LEFT JOIN when right table may not match
SELECT 
    t.hash,
    t.block_time,
    COALESCE(l.name, 'Unknown') AS sender_label
FROM ethereum.transactions t
LEFT JOIN labels.all l ON t."from" = l.address AND l.blockchain = 'ethereum'
WHERE t.block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100
```

### 20. Reduce join cardinality with aggregation first
```sql
-- ✅ GOOD: Aggregate before joining
WITH trade_summary AS (
    SELECT 
        taker,
        SUM(amount_usd) AS total_volume
    FROM dex.trades
    WHERE blockchain = 'ethereum'
        AND block_time >= NOW() - INTERVAL '24' HOUR
    GROUP BY 1
)
SELECT 
    ts.taker,
    ts.total_volume,
    l.name AS label
FROM trade_summary ts
LEFT JOIN labels.all l ON ts.taker = l.address AND l.blockchain = 'ethereum'
ORDER BY ts.total_volume DESC
LIMIT 50
```

---

## Aggregation Optimization

### 21. Use APPROX functions for estimates
```sql
-- ✅ GOOD: APPROX_DISTINCT is faster than COUNT(DISTINCT)
SELECT 
    DATE_TRUNC('day', block_time) AS day,
    APPROX_DISTINCT(taker) AS approx_unique_traders,
    SUM(amount_usd) AS volume
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 1
```

### 22. Use APPROX_PERCENTILE instead of exact percentile
```sql
-- ✅ GOOD: Approximate percentile is faster
SELECT 
    project,
    APPROX_PERCENTILE(amount_usd, 0.5) AS median_trade,
    APPROX_PERCENTILE(amount_usd, 0.95) AS p95_trade
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 2 DESC
```

### 23. Pre-filter before GROUP BY
```sql
-- ✅ GOOD: Filter with WHERE before aggregating
SELECT 
    project,
    COUNT(*) AS trade_count,
    SUM(amount_usd) AS volume
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '24' HOUR
    AND amount_usd > 100  -- Filter before GROUP BY
GROUP BY 1
ORDER BY 3 DESC
```

### 24. Use HAVING for post-aggregation filters
```sql
-- ✅ GOOD: HAVING filters aggregated results
SELECT 
    taker,
    COUNT(*) AS trade_count,
    SUM(amount_usd) AS volume
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
HAVING COUNT(*) >= 10  -- Only active traders
ORDER BY 3 DESC
LIMIT 100
```

### 25. Combine multiple aggregations efficiently
```sql
-- ✅ GOOD: Single scan with multiple aggregations
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    COUNT(*) AS total_trades,
    SUM(CASE WHEN project = 'uniswap' THEN 1 ELSE 0 END) AS uniswap_trades,
    SUM(CASE WHEN project = 'sushiswap' THEN 1 ELSE 0 END) AS sushi_trades,
    SUM(amount_usd) AS total_volume,
    AVG(amount_usd) AS avg_trade_size
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 1 DESC
```

---

## Common Timeout Fixes

### 26. Timeout fix: Add time filter
```sql
-- If query times out, first add time filter
-- Before (times out):
-- SELECT COUNT(*) FROM ethereum.transactions WHERE "from" = 0x...

-- After (works):
SELECT COUNT(*) 
FROM ethereum.transactions 
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
```

### 27. Timeout fix: Sample data first
```sql
-- Sample to understand data before full query
SELECT *
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
TABLESAMPLE BERNOULLI(1)  -- 1% sample
LIMIT 100
```

### 28. Timeout fix: Break into smaller queries
```sql
-- Query 1: Get distinct addresses
WITH addresses AS (
    SELECT DISTINCT "from" AS address
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '1' HOUR
    LIMIT 1000
)
SELECT * FROM addresses;

-- Query 2: Use addresses for detailed analysis
SELECT 
    t."from",
    COUNT(*) AS tx_count
FROM ethereum.transactions t
WHERE t.block_time >= NOW() - INTERVAL '1' HOUR
    AND t."from" IN (SELECT address FROM addresses)
GROUP BY 1
```

### 29. Timeout fix: Use materialized views concept
```sql
-- Pre-aggregate heavy data in a saved query
-- Step 1: Create summary (save as query)
SELECT 
    DATE_TRUNC('day', block_time) AS day,
    project,
    SUM(amount_usd) AS volume
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '90' DAY
GROUP BY 1, 2;

-- Step 2: Query the summary (query_id reference)
-- SELECT * FROM query_123456 WHERE day >= NOW() - INTERVAL '30' DAY
```

### 30. Timeout fix: Avoid CROSS JOIN
```sql
-- ❌ BAD: CROSS JOIN explodes rows
-- SELECT a.*, b.* FROM tableA a CROSS JOIN tableB b

-- ✅ GOOD: Use proper join conditions
SELECT 
    t.hash,
    l.name
FROM ethereum.transactions t
JOIN labels.all l ON t."from" = l.address
WHERE t.block_time >= NOW() - INTERVAL '1' HOUR
    AND l.blockchain = 'ethereum'
LIMIT 100
```

---

## Window Functions Optimization

### 31. Optimize window function partition size
```sql
-- ✅ GOOD: Partition on high-cardinality column with filter
SELECT 
    "from",
    block_time,
    value,
    ROW_NUMBER() OVER (PARTITION BY "from" ORDER BY block_time DESC) AS tx_rank
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
QUALIFY tx_rank <= 5  -- Top 5 per address
```

### 32. Use window functions instead of self-joins
```sql
-- ✅ GOOD: LAG/LEAD instead of self-join
SELECT 
    block_time,
    amount_usd,
    LAG(amount_usd) OVER (ORDER BY block_time) AS prev_trade,
    amount_usd - LAG(amount_usd) OVER (ORDER BY block_time) AS change
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND taker = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
    AND block_time >= NOW() - INTERVAL '7' DAY
ORDER BY block_time DESC
LIMIT 100
```

### 33. Running totals with window functions
```sql
SELECT 
    DATE_TRUNC('day', block_time) AS day,
    SUM(amount_usd) AS daily_volume,
    SUM(SUM(amount_usd)) OVER (ORDER BY DATE_TRUNC('day', block_time)) AS cumulative_volume
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 1
```

### 34. Moving averages
```sql
SELECT 
    day,
    daily_volume,
    AVG(daily_volume) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma_7d
FROM (
    SELECT 
        DATE_TRUNC('day', block_time) AS day,
        SUM(amount_usd) AS daily_volume
    FROM dex.trades
    WHERE blockchain = 'ethereum'
        AND block_time >= NOW() - INTERVAL '30' DAY
    GROUP BY 1
)
ORDER BY day
```

### 35. Rank within groups efficiently
```sql
SELECT 
    project,
    taker,
    volume,
    rank
FROM (
    SELECT 
        project,
        taker,
        SUM(amount_usd) AS volume,
        RANK() OVER (PARTITION BY project ORDER BY SUM(amount_usd) DESC) AS rank
    FROM dex.trades
    WHERE blockchain = 'ethereum'
        AND block_time >= NOW() - INTERVAL '24' HOUR
    GROUP BY 1, 2
)
WHERE rank <= 10
ORDER BY project, rank
```

---

## Data Type Optimization

### 36. Cast to appropriate types
```sql
-- ✅ GOOD: Cast addresses properly
SELECT 
    CAST("from" AS VARCHAR) AS from_address,
    value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100
```

### 37. Handle NULL values efficiently
```sql
-- ✅ GOOD: COALESCE for NULL handling
SELECT 
    hash,
    COALESCE("to", 0x0000000000000000000000000000000000000000) AS to_address,
    CASE WHEN "to" IS NULL THEN 'Contract Creation' ELSE 'Transfer' END AS tx_type
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100
```

### 38. Efficient string operations
```sql
-- ✅ GOOD: Avoid LIKE with leading wildcard
SELECT hash, "to"
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND BYTEARRAY_SUBSTRING(data, 1, 4) = 0xa9059cbb  -- transfer method

-- ❌ BAD: Leading wildcard prevents index use
-- WHERE CAST(data AS VARCHAR) LIKE '%a9059cbb%'
```

### 39. Binary comparison for addresses
```sql
-- ✅ GOOD: Direct binary comparison
SELECT *
FROM ethereum.logs
WHERE contract_address = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
    AND block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100

-- ❌ BAD: String comparison
-- WHERE LOWER(CAST(contract_address AS VARCHAR)) = '0xa0b86991...'
```

### 40. Efficient decimal handling
```sql
-- ✅ GOOD: Divide after filtering
SELECT 
    hash,
    value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND value > 1e18  -- Filter on raw value
LIMIT 100
```

---

## Query Structure Best Practices

### 41. Build queries incrementally
```sql
-- Step 1: Validate base data
SELECT COUNT(*) 
FROM ethereum.transactions 
WHERE block_time >= NOW() - INTERVAL '1' HOUR;

-- Step 2: Add filters
SELECT COUNT(*) 
FROM ethereum.transactions 
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND value > 0;

-- Step 3: Add aggregations
SELECT 
    DATE_TRUNC('minute', block_time) AS minute,
    COUNT(*) AS tx_count
FROM ethereum.transactions 
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND value > 0
GROUP BY 1
ORDER BY 1 DESC;
```

### 42. Use UNION ALL instead of UNION
```sql
-- ✅ GOOD: UNION ALL doesn't deduplicate (faster)
SELECT hash, 'ethereum' AS chain FROM ethereum.transactions WHERE block_time >= NOW() - INTERVAL '1' HOUR LIMIT 50
UNION ALL
SELECT id AS hash, 'solana' AS chain FROM solana.transactions WHERE block_time >= NOW() - INTERVAL '1' HOUR LIMIT 50

-- ❌ BAD: UNION deduplicates (slower)
-- ... UNION ... (requires sorting and comparison)
```

### 43. Efficient CASE expressions
```sql
-- ✅ GOOD: CASE with common conditions first
SELECT 
    hash,
    CASE 
        WHEN value = 0 THEN 'Contract Call'
        WHEN value < 1e18 THEN 'Small Transfer'
        WHEN value < 10e18 THEN 'Medium Transfer'
        ELSE 'Large Transfer'
    END AS tx_category
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100
```

### 44. Avoid redundant calculations
```sql
-- ✅ GOOD: Calculate once, use multiple times
WITH base_data AS (
    SELECT 
        hash,
        value / 1e18 AS eth_value,
        gas_used * gas_price / 1e18 AS fee_eth
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '1' HOUR
)
SELECT 
    hash,
    eth_value,
    fee_eth,
    eth_value + fee_eth AS total_eth
FROM base_data
WHERE eth_value > 1
LIMIT 100
```

### 45. Use DISTINCT sparingly
```sql
-- ✅ GOOD: DISTINCT only when necessary
SELECT DISTINCT taker
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100

-- Better alternative using GROUP BY
SELECT taker
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '1' HOUR
GROUP BY 1
LIMIT 100
```

---

## Advanced Optimization

### 46. Partition pruning for blockchain tables
```sql
-- ✅ GOOD: Filter on partition column (blockchain)
SELECT *
FROM dex.trades
WHERE blockchain = 'ethereum'  -- Partition pruning
    AND block_time >= NOW() - INTERVAL '24' HOUR
LIMIT 100
```

### 47. Efficient array operations
```sql
-- ✅ GOOD: ARRAY_AGG with limit
SELECT 
    "from",
    SLICE(ARRAY_AGG(hash ORDER BY block_time DESC), 1, 5) AS recent_txs
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
LIMIT 100
```

### 48. Optimize ARRAY contains checks
```sql
-- ✅ GOOD: Direct element check
SELECT *
FROM solana.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND CONTAINS(account_keys, 'JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4')
LIMIT 100
```

### 49. Explain query plans (debugging)
```sql
-- Use EXPLAIN to understand query execution
EXPLAIN
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    COUNT(*) AS tx_count
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 1 DESC
```

### 50. Production query template
```sql
/*
Production Query Template
- Time-bounded with indexed column filter
- Explicit column selection
- Proper aggregation with limits
- Clear documentation
*/

WITH 
-- Step 1: Filter data early
filtered_data AS (
    SELECT 
        block_time,
        "from",
        "to",
        value / 1e18 AS eth_value,
        gas_used * gas_price / 1e18 AS fee_eth
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '24' HOUR  -- Time filter
        AND value > 0  -- Early filter
),

-- Step 2: Aggregate
aggregated AS (
    SELECT 
        DATE_TRUNC('hour', block_time) AS hour,
        COUNT(*) AS tx_count,
        SUM(eth_value) AS total_eth,
        AVG(fee_eth) AS avg_fee
    FROM filtered_data
    GROUP BY 1
)

-- Step 3: Final output with limits
SELECT 
    hour,
    tx_count,
    ROUND(total_eth, 4) AS total_eth,
    ROUND(avg_fee, 6) AS avg_fee
FROM aggregated
ORDER BY hour DESC
LIMIT 24  -- Last 24 hours
```

---

**Footer**  
Built by [CryptoAI-Jedi](https://github.com/CryptoAI-Jedi) | [Dune SQL Cheatsheet](https://github.com/CryptoAI-Jedi/dune-sql-cheatsheet)
