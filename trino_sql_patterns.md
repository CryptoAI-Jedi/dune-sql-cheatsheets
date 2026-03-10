# Trino SQL Patterns

**50 production-ready Dune SQL queries showcasing Trino-specific syntax**

**Topics covered:** UNNEST, CROSS JOIN UNNEST, TRY_CAST, APPROX_DISTINCT, VARBINARY handling, hex conversions, array/map operations, JSON functions

---

## Array Operations

### 1. UNNEST array column to rows
```sql
SELECT 
    tx.hash,
    tx.block_time,
    topic AS log_topic
FROM ethereum.transactions tx
CROSS JOIN UNNEST(tx.access_list) AS t(topic)
WHERE tx.block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100
```

### 2. UNNEST with ordinality (get array index)
```sql
SELECT 
    block_number,
    idx AS position,
    account_key
FROM solana.transactions
CROSS JOIN UNNEST(account_keys) WITH ORDINALITY AS t(account_key, idx)
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100
```

### 3. ARRAY_AGG to collect values into array
```sql
SELECT 
    "from" AS wallet,
    ARRAY_AGG(DISTINCT "to") AS unique_recipients,
    COUNT(*) AS tx_count
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
HAVING COUNT(*) >= 5
LIMIT 50
```

### 4. ARRAY_JOIN to concatenate array elements
```sql
SELECT 
    tx_hash,
    ARRAY_JOIN(ARRAY_AGG(DISTINCT token_symbol), ', ') AS tokens_traded
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
GROUP BY 1
LIMIT 50
```

### 5. CARDINALITY to get array length
```sql
SELECT 
    signature,
    CARDINALITY(account_keys) AS num_accounts,
    CARDINALITY(instructions) AS num_instructions
FROM solana.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY CARDINALITY(instructions) DESC
LIMIT 50
```

### 6. CONTAINS to check array membership
```sql
SELECT 
    signature,
    block_time,
    fee / 1e9 AS fee_sol
FROM solana.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND CONTAINS(account_keys, 'JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4')
LIMIT 100
```

### 7. ARRAY_DISTINCT to remove duplicates
```sql
SELECT 
    tx_hash,
    ARRAY_DISTINCT(ARRAY_AGG(token_bought_symbol)) AS unique_tokens
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
GROUP BY 1
LIMIT 50
```

### 8. SLICE for array subsetting
```sql
SELECT 
    signature,
    SLICE(account_keys, 1, 5) AS first_5_accounts
FROM solana.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 50
```

### 9. ARRAY constructor with SELECT
```sql
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    ARRAY_AGG(hash ORDER BY gas_used DESC)[1] AS highest_gas_tx
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '6' HOUR
GROUP BY 1
ORDER BY 1 DESC
```

### 10. Filter array elements with TRANSFORM
```sql
SELECT 
    block_number,
    TRANSFORM(
        FILTER(topics, x -> x IS NOT NULL),
        x -> CAST(x AS VARCHAR)
    ) AS topic_strings
FROM ethereum.logs
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 50
```

---

## VARBINARY and Hex Conversions

### 11. Convert address to lowercase hex string
```sql
SELECT 
    LOWER(CAST("from" AS VARCHAR)) AS from_address,
    LOWER(CAST("to" AS VARCHAR)) AS to_address,
    value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100
```

### 12. Convert hex string to varbinary for comparison
```sql
SELECT 
    hash,
    block_time,
    value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE "to" = FROM_HEX('0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48')
    AND block_time >= NOW() - INTERVAL '24' HOUR
LIMIT 100
```

### 13. Extract bytes from varbinary
```sql
SELECT 
    tx_hash,
    SUBSTR(data, 1, 4) AS function_selector,
    SUBSTR(data, 5, 32) AS first_param
FROM ethereum.logs
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND BYTELENGTH(data) >= 36
LIMIT 50
```

### 14. BYTELENGTH for data size
```sql
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    AVG(BYTELENGTH(data)) AS avg_calldata_bytes,
    MAX(BYTELENGTH(data)) AS max_calldata_bytes
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 1 DESC
```

### 15. TO_HEX for display
```sql
SELECT 
    TO_HEX(SUBSTR(data, 1, 4)) AS function_sig,
    COUNT(*) AS call_count
FROM ethereum.logs
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND BYTELENGTH(data) >= 4
GROUP BY 1
ORDER BY 2 DESC
LIMIT 20
```

### 16. Decode uint256 from varbinary
```sql
SELECT 
    tx_hash,
    CAST(CONV(TO_HEX(SUBSTR(data, 5, 32)), 16, 10) AS DECIMAL(38,0)) / 1e18 AS amount
FROM ethereum.logs
WHERE topic0 = 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
    AND block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 50
```

### 17. Compare partial address bytes
```sql
SELECT 
    hash,
    "to",
    value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE SUBSTR(CAST("to" AS VARBINARY), 1, 4) = FROM_HEX('0xA0b86991')
    AND block_time >= NOW() - INTERVAL '24' HOUR
LIMIT 50
```

### 18. Concatenate varbinary values
```sql
SELECT 
    tx_hash,
    CONCAT(topic0, topic1) AS combined_topics
FROM ethereum.logs
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND topic1 IS NOT NULL
LIMIT 50
```

### 19. XOR operation on addresses
```sql
SELECT 
    hash,
    BITWISE_XOR(
        CAST("from" AS BIGINT),
        CAST("to" AS BIGINT)
    ) AS address_xor
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND "to" IS NOT NULL
LIMIT 50
```

### 20. Keccak256 signature matching
```sql
SELECT 
    tx_hash,
    block_time,
    contract_address
FROM ethereum.logs
WHERE topic0 = 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef -- Transfer(address,address,uint256)
    AND block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100
```

---

## TRY_CAST and Safe Type Conversions

### 21. TRY_CAST for safe numeric conversion
```sql
SELECT 
    tx_hash,
    TRY_CAST(CONV(TO_HEX(SUBSTR(data, 1, 32)), 16, 10) AS DECIMAL(38,0)) AS parsed_value
FROM ethereum.logs
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND BYTELENGTH(data) >= 32
LIMIT 100
```

### 22. COALESCE with TRY_CAST for defaults
```sql
SELECT 
    tx_hash,
    COALESCE(
        TRY_CAST(amount_usd AS DOUBLE),
        0
    ) AS safe_amount_usd
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
LIMIT 100
```

### 23. TRY for expression error handling
```sql
SELECT 
    tx_hash,
    TRY(value / NULLIF(gas_used, 0)) AS value_per_gas
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100
```

### 24. NULLIF to avoid division by zero
```sql
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    SUM(amount_usd) / NULLIF(COUNT(*), 0) AS avg_trade_size
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
GROUP BY 1
ORDER BY 1 DESC
```

### 25. CAST timestamp to date
```sql
SELECT 
    CAST(block_time AS DATE) AS trade_date,
    COUNT(*) AS trades,
    SUM(amount_usd) AS volume_usd
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '7' DAY
    AND blockchain = 'ethereum'
GROUP BY 1
ORDER BY 1 DESC
```

### 26. CAST between numeric types
```sql
SELECT 
    block_number,
    CAST(gas_used AS DOUBLE) / CAST(gas_limit AS DOUBLE) AS gas_utilization
FROM ethereum.blocks
WHERE time >= NOW() - INTERVAL '1' HOUR
ORDER BY time DESC
```

### 27. TRY_CAST for timestamp parsing
```sql
SELECT 
    tx_hash,
    TRY_CAST(FROM_UNIXTIME(
        CAST(CONV(TO_HEX(SUBSTR(data, 1, 32)), 16, 10) AS BIGINT)
    ) AS TIMESTAMP) AS decoded_timestamp
FROM ethereum.logs
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND BYTELENGTH(data) >= 32
LIMIT 50
```

### 28. Safe JSON parsing with TRY
```sql
SELECT 
    tx_hash,
    TRY(JSON_EXTRACT_SCALAR(metadata, '$.name')) AS nft_name
FROM nft.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
LIMIT 100
```

### 29. Converting between address formats
```sql
SELECT 
    '0x' || LOWER(TO_HEX("from")) AS from_address_string,
    '0x' || LOWER(TO_HEX("to")) AS to_address_string,
    value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100
```

### 30. Format numbers with precision
```sql
SELECT 
    tx_hash,
    CAST(amount_usd AS DECIMAL(20,2)) AS formatted_amount,
    ROUND(amount_usd, 2) AS rounded_amount
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
    AND amount_usd IS NOT NULL
LIMIT 100
```

---

## Approximate Functions

### 31. APPROX_DISTINCT for fast unique counts
```sql
SELECT 
    DATE_TRUNC('day', block_time) AS day,
    APPROX_DISTINCT("from") AS approx_unique_senders,
    COUNT(*) AS total_txs
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 1 DESC
```

### 32. APPROX_PERCENTILE for fast percentiles
```sql
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    APPROX_PERCENTILE(gas_price / 1e9, 0.5) AS median_gas_gwei,
    APPROX_PERCENTILE(gas_price / 1e9, 0.95) AS p95_gas_gwei
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 1 DESC
```

### 33. APPROX_MOST_FREQUENT for top values
```sql
SELECT 
    APPROX_MOST_FREQUENT(5, "to") AS top_5_recipients
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
```

### 34. Compare APPROX_DISTINCT vs COUNT(DISTINCT)
```sql
SELECT 
    APPROX_DISTINCT(taker) AS approx_unique_takers,
    COUNT(DISTINCT taker) AS exact_unique_takers,
    ABS(APPROX_DISTINCT(taker) - COUNT(DISTINCT taker)) AS difference
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '7' DAY
    AND blockchain = 'ethereum'
```

### 35. APPROX_PERCENTILE array for multiple percentiles
```sql
SELECT 
    APPROX_PERCENTILE(amount_usd, ARRAY[0.25, 0.5, 0.75, 0.9, 0.99]) AS trade_size_distribution
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
    AND amount_usd > 0
```

---

## Map Operations

### 36. MAP_AGG to create key-value pairs
```sql
SELECT 
    taker,
    MAP_AGG(token_bought_symbol, SUM(amount_usd)) AS token_volumes
FROM (
    SELECT taker, token_bought_symbol, SUM(amount_usd) AS amount_usd
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '24' HOUR
        AND blockchain = 'ethereum'
    GROUP BY 1, 2
)
GROUP BY 1
LIMIT 50
```

### 37. MAP_KEYS and MAP_VALUES extraction
```sql
WITH token_map AS (
    SELECT 
        taker,
        MAP_FROM_ENTRIES(ARRAY[
            ('ETH', eth_volume),
            ('USDC', usdc_volume)
        ]) AS volumes
    FROM (
        SELECT 
            taker,
            SUM(CASE WHEN token_bought_symbol = 'WETH' THEN amount_usd ELSE 0 END) AS eth_volume,
            SUM(CASE WHEN token_bought_symbol = 'USDC' THEN amount_usd ELSE 0 END) AS usdc_volume
        FROM dex.trades
        WHERE block_time >= NOW() - INTERVAL '24' HOUR
            AND blockchain = 'ethereum'
        GROUP BY 1
    )
)
SELECT 
    taker,
    MAP_KEYS(volumes) AS tokens,
    MAP_VALUES(volumes) AS volume_values
FROM token_map
LIMIT 50
```

### 38. ELEMENT_AT for map access
```sql
WITH protocol_volumes AS (
    SELECT 
        project,
        MAP_FROM_ENTRIES(ARRAY[
            ('volume', SUM(amount_usd)),
            ('trades', CAST(COUNT(*) AS DOUBLE))
        ]) AS metrics
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '24' HOUR
        AND blockchain = 'ethereum'
    GROUP BY 1
)
SELECT 
    project,
    ELEMENT_AT(metrics, 'volume') AS total_volume,
    ELEMENT_AT(metrics, 'trades') AS trade_count
FROM protocol_volumes
ORDER BY 2 DESC
LIMIT 20
```

### 39. MAP_FILTER for conditional filtering
```sql
WITH daily_volumes AS (
    SELECT 
        taker,
        MAP_FROM_ENTRIES(
            ARRAY_AGG(ROW(token_bought_symbol, vol))
        ) AS token_volumes
    FROM (
        SELECT taker, token_bought_symbol, SUM(amount_usd) AS vol
        FROM dex.trades
        WHERE block_time >= NOW() - INTERVAL '24' HOUR
            AND blockchain = 'ethereum'
        GROUP BY 1, 2
    )
    GROUP BY 1
)
SELECT 
    taker,
    MAP_FILTER(token_volumes, (k, v) -> v > 1000) AS significant_volumes
FROM daily_volumes
LIMIT 50
```

### 40. TRANSFORM_VALUES on maps
```sql
WITH raw_volumes AS (
    SELECT 
        project,
        MAP_FROM_ENTRIES(ARRAY[
            ('eth_volume', SUM(CASE WHEN token_bought_symbol = 'WETH' THEN amount_usd ELSE 0 END)),
            ('usdc_volume', SUM(CASE WHEN token_bought_symbol = 'USDC' THEN amount_usd ELSE 0 END))
        ]) AS volumes
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '24' HOUR
        AND blockchain = 'ethereum'
    GROUP BY 1
)
SELECT 
    project,
    TRANSFORM_VALUES(volumes, v -> ROUND(v, 2)) AS rounded_volumes
FROM raw_volumes
ORDER BY ELEMENT_AT(volumes, 'eth_volume') DESC
LIMIT 20
```

---

## JSON Functions

### 41. JSON_EXTRACT_SCALAR for string values
```sql
SELECT 
    tx_hash,
    JSON_EXTRACT_SCALAR(metadata, '$.name') AS nft_name,
    JSON_EXTRACT_SCALAR(metadata, '$.collection') AS collection
FROM nft.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
    AND metadata IS NOT NULL
LIMIT 100
```

### 42. JSON_EXTRACT for nested objects
```sql
SELECT 
    tx_hash,
    JSON_EXTRACT(metadata, '$.attributes') AS attributes_json
FROM nft.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
    AND metadata IS NOT NULL
LIMIT 50
```

### 43. JSON_ARRAY_LENGTH for array counting
```sql
SELECT 
    tx_hash,
    JSON_ARRAY_LENGTH(JSON_EXTRACT(metadata, '$.attributes')) AS num_attributes
FROM nft.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
    AND metadata IS NOT NULL
LIMIT 50
```

### 44. JSON_FORMAT for creating JSON strings
```sql
SELECT 
    project,
    JSON_FORMAT(
        CAST(MAP_FROM_ENTRIES(ARRAY[
            ('volume', SUM(amount_usd)),
            ('trades', COUNT(*)),
            ('avg_size', AVG(amount_usd))
        ]) AS JSON)
    ) AS metrics_json
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
GROUP BY 1
ORDER BY SUM(amount_usd) DESC
LIMIT 20
```

### 45. JSON_PARSE for parsing JSON strings
```sql
SELECT 
    tx_hash,
    JSON_EXTRACT_SCALAR(
        JSON_PARSE('{"chain": "ethereum", "version": "v3"}'),
        '$.chain'
    ) AS chain_name
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND blockchain = 'ethereum'
LIMIT 10
```

---

## Advanced Trino Patterns

### 46. REDUCE for array aggregation
```sql
SELECT 
    signature,
    REDUCE(
        SLICE(account_keys, 1, 5),
        '',
        (s, x) -> s || SUBSTR(x, 1, 8) || '...',
        s -> s
    ) AS account_preview
FROM solana.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 50
```

### 47. ROW type construction
```sql
SELECT 
    ROW(project, SUM(amount_usd), COUNT(*)) AS protocol_summary
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
GROUP BY project
ORDER BY SUM(amount_usd) DESC
LIMIT 20
```

### 48. SEQUENCE for generating series
```sql
SELECT 
    d AS date,
    COALESCE(volume, 0) AS volume
FROM UNNEST(SEQUENCE(
    CAST(NOW() - INTERVAL '7' DAY AS DATE),
    CAST(NOW() AS DATE),
    INTERVAL '1' DAY
)) AS t(d)
LEFT JOIN (
    SELECT CAST(block_time AS DATE) AS dt, SUM(amount_usd) AS volume
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '7' DAY
        AND blockchain = 'ethereum'
    GROUP BY 1
) v ON t.d = v.dt
ORDER BY 1
```

### 49. REGEXP_EXTRACT for pattern matching
```sql
SELECT 
    REGEXP_EXTRACT(CAST(data AS VARCHAR), '0x[a-fA-F0-9]{8}', 0) AS function_sig
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND BYTELENGTH(data) >= 4
LIMIT 50
```

### 50. WITH RECURSIVE for iterative queries
```sql
WITH RECURSIVE blocks AS (
    SELECT 
        number,
        time,
        1 AS depth
    FROM ethereum.blocks
    WHERE number = (SELECT MAX(number) FROM ethereum.blocks)
    
    UNION ALL
    
    SELECT 
        b.number,
        b.time,
        r.depth + 1
    FROM ethereum.blocks b
    JOIN blocks r ON b.number = r.number - 1
    WHERE r.depth < 10
)
SELECT number, time, depth
FROM blocks
ORDER BY number DESC
```

---

*Last updated: March 2026*
