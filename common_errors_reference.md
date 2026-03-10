# Common Errors Reference

**50 production-ready examples for debugging Dune SQL errors**

**Topics covered:** Trino error messages decoded, Dune-specific errors, data freshness issues, indexing delays, type mismatches, syntax errors with solutions

---

## Syntax Errors

### 1. ERROR: Column "address" does not exist
```sql
-- WRONG: Using unquoted reserved word or wrong column name
SELECT address, value FROM ethereum.transactions LIMIT 10;

-- CORRECT: Use proper column names with quotes for reserved words
SELECT "from", "to", value FROM ethereum.transactions LIMIT 10;
-- Note: In Ethereum tables, addresses are in "from" and "to" columns
```

### 2. ERROR: Table "ethereum.transaction" does not exist
```sql
-- WRONG: Incorrect table name (singular)
SELECT * FROM ethereum.transaction LIMIT 10;

-- CORRECT: Use plural table names
SELECT * FROM ethereum.transactions LIMIT 10;

-- COMMON TABLE NAMES:
-- ethereum.transactions (not transaction)
-- ethereum.logs (not log)
-- ethereum.traces (not trace)
-- ethereum.blocks (not block)
```

### 3. ERROR: Function "now" not found
```sql
-- WRONG: Missing parentheses
SELECT * FROM ethereum.transactions WHERE block_time >= now - INTERVAL '1' DAY;

-- CORRECT: NOW() is a function - needs parentheses
SELECT * FROM ethereum.transactions WHERE block_time >= NOW() - INTERVAL '1' DAY LIMIT 100;
```

### 4. ERROR: line 1:8: mismatched input 'FROM'
```sql
-- WRONG: Missing columns after SELECT
SELECT FROM ethereum.transactions LIMIT 10;

-- CORRECT: Specify columns or use *
SELECT * FROM ethereum.transactions LIMIT 10;
-- OR
SELECT hash, "from", "to" FROM ethereum.transactions LIMIT 10;
```

### 5. ERROR: Invalid literal interval
```sql
-- WRONG: Incorrect interval syntax
SELECT * FROM ethereum.transactions WHERE block_time >= NOW() - INTERVAL 1 DAY;

-- CORRECT: Quote the number and unit separately
SELECT * FROM ethereum.transactions 
WHERE block_time >= NOW() - INTERVAL '1' DAY
LIMIT 100;

-- ALSO CORRECT:
-- INTERVAL '24' HOUR
-- INTERVAL '7' DAY
-- INTERVAL '30' MINUTE
```

### 6. ERROR: Column must appear in GROUP BY clause
```sql
-- WRONG: Selecting non-aggregated column without GROUP BY
SELECT "from", block_time, SUM(value) 
FROM ethereum.transactions 
WHERE block_time >= NOW() - INTERVAL '1' HOUR;

-- CORRECT: Include all non-aggregated columns in GROUP BY
SELECT "from", DATE_TRUNC('hour', block_time) AS hour, SUM(value) 
FROM ethereum.transactions 
WHERE block_time >= NOW() - INTERVAL '1' HOUR
GROUP BY 1, 2;
```

### 7. ERROR: Unexpected token: ORDER
```sql
-- WRONG: ORDER BY before GROUP BY
SELECT "from", COUNT(*) 
FROM ethereum.transactions 
WHERE block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY 2 DESC
GROUP BY 1;

-- CORRECT: GROUP BY comes before ORDER BY
SELECT "from", COUNT(*) 
FROM ethereum.transactions 
WHERE block_time >= NOW() - INTERVAL '1' HOUR
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10;
```

### 8. ERROR: Expression not in GROUP BY clause
```sql
-- WRONG: Using column alias in GROUP BY (not supported in Trino)
SELECT DATE_TRUNC('day', block_time) AS day, COUNT(*) 
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '7' DAY
GROUP BY day;

-- CORRECT: Repeat the expression or use position
SELECT DATE_TRUNC('day', block_time) AS day, COUNT(*) 
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '7' DAY
GROUP BY DATE_TRUNC('day', block_time);
-- OR use position: GROUP BY 1
```

### 9. ERROR: LIMIT must not be negative
```sql
-- WRONG: Negative or invalid LIMIT
SELECT * FROM ethereum.transactions LIMIT -10;

-- CORRECT: Use positive integer
SELECT * FROM ethereum.transactions LIMIT 10;
```

### 10. ERROR: Missing FROM clause
```sql
-- WRONG: Just a function call without context
SELECT NOW() - INTERVAL '1' DAY;

-- CORRECT: This works in Trino (no FROM needed for constants)
SELECT NOW() - INTERVAL '1' DAY AS one_day_ago;
-- Some databases require FROM DUAL, Trino doesn't
```

---

## Type Mismatch Errors

### 11. ERROR: Cannot compare varbinary and varchar
```sql
-- WRONG: Comparing address with string directly
SELECT * FROM ethereum.transactions 
WHERE "to" = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48'
LIMIT 10;

-- CORRECT: Addresses in Dune are varbinary - use 0x prefix without quotes
SELECT * FROM ethereum.transactions 
WHERE "to" = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
    AND block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 10;
```

### 12. ERROR: Cannot apply operator / to types varbinary and integer
```sql
-- WRONG: Dividing raw value (might be interpreted wrong)
SELECT topic1 / 1e18 FROM ethereum.logs LIMIT 10;

-- CORRECT: Cast to appropriate type first
SELECT CAST(topic1 AS DECIMAL(38,0)) / 1e18 
FROM ethereum.logs 
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 10;
```

### 13. ERROR: Value cannot be cast to bigint
```sql
-- WRONG: Overflow when casting large numbers
SELECT CAST(value AS BIGINT) FROM ethereum.transactions LIMIT 10;

-- CORRECT: Use DECIMAL for large numbers
SELECT CAST(value AS DECIMAL(38,0)) / 1e18 AS eth_value
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 10;
```

### 14. ERROR: Cannot cast varchar to timestamp
```sql
-- WRONG: String isn't auto-cast to timestamp
SELECT * FROM dex.trades WHERE block_time = '2024-01-01';

-- CORRECT: Explicit cast
SELECT * FROM dex.trades 
WHERE block_time >= CAST('2024-01-01' AS TIMESTAMP)
    AND block_time < CAST('2024-01-02' AS TIMESTAMP)
    AND blockchain = 'ethereum'
LIMIT 100;
```

### 15. ERROR: Unexpected parameters for function DATE_DIFF
```sql
-- WRONG: Wrong order of parameters
SELECT DATE_DIFF(NOW(), block_time, 'day') 
FROM ethereum.transactions LIMIT 10;

-- CORRECT: DATE_DIFF(unit, start, end)
SELECT DATE_DIFF('day', block_time, NOW()) AS days_ago
FROM ethereum.transactions 
WHERE block_time >= NOW() - INTERVAL '30' DAY
LIMIT 10;
```

### 16. ERROR: Type mismatch in CASE expression
```sql
-- WRONG: Mixing types in CASE branches
SELECT 
    CASE 
        WHEN value > 0 THEN 'positive'
        ELSE 0  -- number vs string
    END
FROM ethereum.transactions LIMIT 10;

-- CORRECT: All branches return same type
SELECT 
    CASE 
        WHEN value > 0 THEN 'positive'
        ELSE 'zero or negative'
    END AS value_type
FROM ethereum.transactions 
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 10;
```

### 17. ERROR: Cannot cast boolean to varchar
```sql
-- WRONG: Implicit boolean to string
SELECT success || ' transaction' FROM ethereum.transactions LIMIT 10;

-- CORRECT: Explicit cast
SELECT CAST(success AS VARCHAR) || ' transaction' AS status
FROM ethereum.transactions 
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 10;
```

### 18. ERROR: Division by zero
```sql
-- WRONG: Direct division that might be zero
SELECT value / gas_used FROM ethereum.transactions LIMIT 10;

-- CORRECT: Use NULLIF to prevent division by zero
SELECT value / NULLIF(gas_used, 0) AS value_per_gas
FROM ethereum.transactions 
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 10;
```

### 19. ERROR: Invalid interval start
```sql
-- WRONG: Negative interval
SELECT * FROM ethereum.transactions 
WHERE block_time >= NOW() + INTERVAL '-1' DAY LIMIT 10;

-- CORRECT: Use subtraction
SELECT * FROM ethereum.transactions 
WHERE block_time >= NOW() - INTERVAL '1' DAY
LIMIT 10;
```

### 20. ERROR: Numeric value out of range
```sql
-- WRONG: Calculation exceeds numeric limits
SELECT value * 1e18 FROM ethereum.transactions LIMIT 10;

-- CORRECT: Use appropriate decimal precision
SELECT CAST(value AS DECIMAL(38,0)) * CAST(1e18 AS DECIMAL(38,0))
FROM ethereum.transactions 
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 10;
-- Or just divide properly: value / 1e18
```

---

## Query Performance Errors

### 21. ERROR: Query exceeded resource limits
```sql
-- WRONG: No time filter (scans entire history)
SELECT COUNT(DISTINCT "from") FROM ethereum.transactions;

-- CORRECT: Always add time filter
SELECT COUNT(DISTINCT "from") 
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '7' DAY;
```

### 22. ERROR: Query timeout after 30 minutes
```sql
-- WRONG: Too much data without optimization
SELECT * FROM ethereum.transactions WHERE value > 0;

-- CORRECT: Add filters and limit columns
SELECT hash, "from", "to", value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND value > 0
ORDER BY value DESC
LIMIT 100;
```

### 23. ERROR: Exceeded memory limit
```sql
-- WRONG: Large GROUP BY without pre-filtering
SELECT "from", "to", COUNT(*) 
FROM ethereum.transactions 
GROUP BY 1, 2;

-- CORRECT: Pre-filter and aggregate
SELECT "from", "to", COUNT(*) AS tx_count
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1, 2
HAVING COUNT(*) > 10
ORDER BY 3 DESC
LIMIT 100;
```

### 24. ERROR: Too many partitions
```sql
-- WRONG: Window function over entire dataset
SELECT ROW_NUMBER() OVER() FROM ethereum.transactions;

-- CORRECT: Add partition or filter
SELECT 
    ROW_NUMBER() OVER(PARTITION BY DATE_TRUNC('hour', block_time) ORDER BY block_time) AS row_num
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100;
```

### 25. Fix: Use APPROX_DISTINCT for large cardinality
```sql
-- SLOW: Exact distinct count on large data
-- SELECT COUNT(DISTINCT "from") FROM ethereum.transactions WHERE block_time >= NOW() - INTERVAL '30' DAY;

-- FAST: Approximate (< 2% error)
SELECT APPROX_DISTINCT("from") AS unique_senders
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '30' DAY;
```

### 26. Fix: Pre-aggregate before JOIN
```sql
-- SLOW: Join then aggregate
-- SELECT t.symbol, COUNT(*) FROM ethereum.transactions tx
-- JOIN tokens.erc20 t ON tx."to" = t.contract_address
-- GROUP BY 1;

-- FAST: Aggregate first, then join
WITH tx_counts AS (
    SELECT "to" AS contract, COUNT(*) AS tx_count
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '24' HOUR
    GROUP BY 1
)
SELECT t.symbol, tc.tx_count
FROM tx_counts tc
JOIN tokens.erc20 t 
    ON tc.contract = t.contract_address 
    AND t.blockchain = 'ethereum'
ORDER BY tx_count DESC
LIMIT 50;
```

### 27. Fix: Use index-friendly date predicates
```sql
-- SLOW: Function on indexed column
-- SELECT * FROM ethereum.transactions WHERE DATE(block_time) = '2024-01-01';

-- FAST: Range comparison
SELECT * FROM ethereum.transactions
WHERE block_time >= TIMESTAMP '2024-01-01 00:00:00'
    AND block_time < TIMESTAMP '2024-01-02 00:00:00'
LIMIT 100;
```

### 28. Fix: Avoid CROSS JOIN
```sql
-- WRONG: Accidental cross join
-- SELECT * FROM ethereum.transactions t1, ethereum.transactions t2 LIMIT 100;

-- CORRECT: Explicit JOIN with condition
SELECT t1.hash, t2.hash
FROM ethereum.transactions t1
JOIN ethereum.transactions t2 
    ON t1."from" = t2."to"
    AND t2.block_time >= NOW() - INTERVAL '1' HOUR
WHERE t1.block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100;
```

### 29. Fix: Sample data for exploration
```sql
-- Instead of: SELECT * FROM ethereum.transactions LIMIT 1000000;

-- Use sampling
SELECT *
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND block_number % 100 = 0  -- ~1% sample
LIMIT 1000;
```

### 30. Fix: Break complex queries into CTEs
```sql
-- Instead of deep nested subqueries
WITH step1 AS (
    SELECT "from", SUM(value) AS total_value
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '24' HOUR
    GROUP BY 1
),
step2 AS (
    SELECT "from", total_value
    FROM step1
    WHERE total_value > 1e18
)
SELECT * FROM step2
ORDER BY total_value DESC
LIMIT 100;
```

---

## Data Quality Issues

### 31. NULL values in aggregation
```sql
-- Problem: NULL values are ignored in aggregations
SELECT 
    COUNT(*) AS total_trades,
    COUNT(amount_usd) AS trades_with_price,
    AVG(amount_usd) AS avg_trade_size  -- This ignores NULLs
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum';

-- Solution: Be aware that AVG ignores NULLs
-- Use COALESCE if you want to treat NULL as 0
SELECT 
    AVG(COALESCE(amount_usd, 0)) AS avg_with_nulls_as_zero,
    AVG(amount_usd) AS avg_excluding_nulls
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum';
```

### 32. Missing decimal handling
```sql
-- WRONG: Raw value without decimals
SELECT value FROM erc20_ethereum.evt_Transfer LIMIT 10;

-- CORRECT: Join with token metadata for proper decimals
SELECT 
    e.evt_tx_hash,
    t.symbol,
    e.value / POWER(10, t.decimals) AS amount
FROM erc20_ethereum.evt_Transfer e
JOIN tokens.erc20 t 
    ON e.contract_address = t.contract_address 
    AND t.blockchain = 'ethereum'
WHERE e.evt_block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100;
```

### 33. Duplicate rows in results
```sql
-- Check for duplicates
SELECT 
    tx_hash,
    COUNT(*) AS occurrences
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
GROUP BY 1
HAVING COUNT(*) > 1
LIMIT 20;

-- Solution: Use DISTINCT or dedupe in your query
SELECT DISTINCT tx_hash, block_time, amount_usd
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
LIMIT 100;
```

### 34. Inconsistent address casing
```sql
-- Addresses in Dune are varbinary, but comparison can fail if converted wrong
-- WRONG:
-- SELECT * FROM ethereum.transactions WHERE "to" = UPPER('0xabc...');

-- CORRECT: Use raw hex format
SELECT * FROM ethereum.transactions
WHERE "to" = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
    AND block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 10;

-- For string comparison, convert properly:
SELECT * FROM ethereum.transactions
WHERE LOWER(CAST("to" AS VARCHAR)) = LOWER('0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48')
    AND block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 10;
```

### 35. Token with wrong/missing price data
```sql
-- Debug: Check if token has price data
SELECT 
    minute,
    price,
    symbol
FROM prices.usd
WHERE contract_address = 0xYourTokenAddress
    AND blockchain = 'ethereum'
    AND minute >= NOW() - INTERVAL '24' HOUR
ORDER BY minute DESC
LIMIT 50;

-- If empty, token may not be tracked in prices.usd
-- Alternative: Use DEX trades to estimate price
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    AVG(amount_usd / NULLIF(token_bought_amount, 0)) AS estimated_price
FROM dex.trades
WHERE token_bought_address = 0xYourTokenAddress
    AND block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
    AND token_bought_amount > 0
GROUP BY 1
ORDER BY 1 DESC;
```

---

## Dune-Specific Issues

### 36. Query parameter not working
```sql
-- Make sure parameters are defined in Dune UI
-- Syntax: {{parameter_name}}

-- WRONG:
-- SELECT * FROM ethereum.transactions WHERE "from" = $wallet_address;

-- CORRECT: Use double curly braces
SELECT * FROM ethereum.transactions
WHERE "from" = {{wallet_address}}
    AND block_time >= NOW() - INTERVAL '{{days_back}}' DAY
LIMIT 100;

-- Parameter types:
-- text: {{my_address}}
-- number: {{days_back}}
-- date: {{start_date}}
```

### 37. Spellbook table not found
```sql
-- WRONG: Using old table names
-- SELECT * FROM uniswap.trades;

-- CORRECT: Use current Spellbook naming
SELECT * FROM dex.trades
WHERE project = 'uniswap'
    AND blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '24' HOUR
LIMIT 100;

-- Check available tables:
-- dex.trades (not protocol_name.trades)
-- nft.trades
-- prices.usd
-- tokens.erc20
```

### 38. Decoded table not available for contract
```sql
-- If your contract isn't decoded, you need to:
-- 1. Submit it for decoding via Dune UI
-- 2. Use raw tables in the meantime

-- Query raw logs for the contract
SELECT 
    tx_hash,
    block_time,
    topic0 AS event_signature,
    data
FROM ethereum.logs
WHERE contract_address = 0xYourContractAddress
    AND block_time >= NOW() - INTERVAL '24' HOUR
LIMIT 100;

-- Common event signatures:
-- Transfer: 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
-- Approval: 0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925
```

### 39. Data freshness lag
```sql
-- Check when data was last updated
SELECT 
    'ethereum.transactions' AS table_name,
    MAX(block_time) AS latest_data,
    DATE_DIFF('minute', MAX(block_time), NOW()) AS minutes_behind
FROM ethereum.transactions

UNION ALL

SELECT 
    'dex.trades',
    MAX(block_time),
    DATE_DIFF('minute', MAX(block_time), NOW())
FROM dex.trades
WHERE blockchain = 'ethereum'

UNION ALL

SELECT 
    'ethereum.blocks',
    MAX(time),
    DATE_DIFF('minute', MAX(time), NOW())
FROM ethereum.blocks;
```

### 40. Cross-database query not supported
```sql
-- WRONG: Trying to join across different data sources
-- SELECT * FROM ethereum.transactions e JOIN postgres.my_table p ON ...

-- Dune only supports its own tables
-- Solution: Import your data to Dune or export Dune data to your system

SELECT * FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100;
-- Then process with external data in your application
```

---

## Advanced Error Fixes

### 41. UNNEST errors
```sql
-- WRONG: UNNEST without CROSS JOIN
-- SELECT *, element FROM ethereum.transactions, UNNEST(access_list);

-- CORRECT: Use CROSS JOIN UNNEST
SELECT 
    hash,
    element
FROM ethereum.transactions
CROSS JOIN UNNEST(access_list) AS t(element)
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100;
```

### 42. Window function ordering
```sql
-- WRONG: Using alias in window ORDER BY
SELECT 
    "from",
    value / 1e18 AS eth_value,
    ROW_NUMBER() OVER(ORDER BY eth_value DESC) AS rank
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR;

-- CORRECT: Use the original expression
SELECT 
    "from",
    value / 1e18 AS eth_value,
    ROW_NUMBER() OVER(ORDER BY value DESC) AS rank
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100;
```

### 43. Subquery correlation error
```sql
-- WRONG: Correlated subquery referencing outer table incorrectly
-- SELECT * FROM ethereum.transactions t
-- WHERE value > (SELECT AVG(value) FROM ethereum.transactions WHERE "from" = t."from");

-- CORRECT: Use window function instead
SELECT *
FROM (
    SELECT 
        *,
        AVG(value) OVER(PARTITION BY "from") AS avg_value_by_sender
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '1' HOUR
) sub
WHERE value > avg_value_by_sender
LIMIT 100;
```

### 44. HAVING vs WHERE confusion
```sql
-- WRONG: Using HAVING without GROUP BY
-- SELECT * FROM ethereum.transactions HAVING value > 1e18;

-- WRONG: Using WHERE with aggregates
-- SELECT "from", SUM(value) FROM ethereum.transactions WHERE SUM(value) > 1e18 GROUP BY 1;

-- CORRECT: WHERE filters rows, HAVING filters groups
SELECT 
    "from",
    SUM(value) / 1e18 AS total_eth
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR  -- Row filter
    AND value > 0
GROUP BY 1
HAVING SUM(value) > 1e18  -- Group filter
ORDER BY 2 DESC
LIMIT 50;
```

### 45. DISTINCT with ORDER BY
```sql
-- WRONG: ORDER BY column not in SELECT DISTINCT
-- SELECT DISTINCT "from" FROM ethereum.transactions ORDER BY value DESC;

-- CORRECT: Include order column or use different approach
SELECT DISTINCT "from"
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
LIMIT 100;

-- Or use GROUP BY with aggregation
SELECT "from", MAX(value) AS max_value
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 2 DESC
LIMIT 100;
```

### 46. UNION type mismatch
```sql
-- WRONG: Different column types in UNION
-- SELECT "from", value FROM ethereum.transactions
-- UNION ALL
-- SELECT "from", 'string' FROM ethereum.transactions;

-- CORRECT: Ensure same types
SELECT "from", CAST(value AS DECIMAL(38,0)) AS amount
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 50

UNION ALL

SELECT "from", CAST(0 AS DECIMAL(38,0)) AS amount
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND value = 0
LIMIT 50;
```

### 47. CTE recursive limit
```sql
-- WRONG: Recursive CTE without termination
-- WITH RECURSIVE bad AS (SELECT 1 AS n UNION ALL SELECT n+1 FROM bad)
-- SELECT * FROM bad;

-- CORRECT: Add termination condition
WITH RECURSIVE good AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM good WHERE n < 10  -- Termination condition
)
SELECT * FROM good;
```

### 48. Ambiguous column reference
```sql
-- WRONG: Column name exists in multiple tables
-- SELECT block_time FROM ethereum.transactions t
-- JOIN ethereum.logs l ON t.hash = l.tx_hash;

-- CORRECT: Qualify with table alias
SELECT 
    t.hash,
    t.block_time AS tx_time,
    l.block_time AS log_time
FROM ethereum.transactions t
JOIN ethereum.logs l 
    ON t.hash = l.tx_hash 
    AND l.block_time >= NOW() - INTERVAL '1' HOUR
WHERE t.block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100;
```

### 49. JSON extraction errors
```sql
-- WRONG: Invalid JSON path or NULL JSON
-- SELECT JSON_EXTRACT_SCALAR(metadata, 'name') FROM nft.trades;

-- CORRECT: Handle NULL and use proper path
SELECT 
    tx_hash,
    TRY(JSON_EXTRACT_SCALAR(metadata, '$.name')) AS nft_name,
    COALESCE(
        TRY(JSON_EXTRACT_SCALAR(metadata, '$.name')),
        'Unknown'
    ) AS safe_name
FROM nft.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
LIMIT 100;
```

### 50. Hex string conversion
```sql
-- WRONG: Using wrong hex conversion function
-- SELECT CONV('0xabc', 16, 10);

-- CORRECT: Use appropriate functions
SELECT 
    -- Hex string to decimal
    FROM_BASE('abc', 16) AS decimal_value,
    
    -- Address to hex string
    '0x' || LOWER(TO_HEX(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48)) AS address_string,
    
    -- Hex string to varbinary
    FROM_HEX('A0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48') AS address_bytes
;
```

---

*Last updated: March 2026*
