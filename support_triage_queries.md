# Support Triage Queries

**50 production-ready Dune SQL queries for troubleshooting common user issues**

**Topics covered:** Wrong values (decimal handling), query timeouts, finding tokens/contracts, cross-chain gotchas, missing data, indexing delays

---

## Decimal and Value Issues

### 1. Verify token decimals for correct value display
```sql
-- Use this to check if user is dividing by correct decimal
SELECT 
    contract_address,
    symbol,
    decimals,
    POWER(10, decimals) AS divisor
FROM tokens.erc20
WHERE symbol IN ('USDC', 'USDT', 'WETH', 'DAI', 'WBTC')
    AND blockchain = 'ethereum'
```

### 2. Compare raw vs properly decimalized transfer amounts
```sql
-- Show difference between raw and proper decimal handling
SELECT 
    evt_tx_hash,
    contract_address,
    value AS raw_value,
    value / 1e18 AS assuming_18_decimals,
    value / 1e6 AS assuming_6_decimals,
    -- Proper way using token metadata
    value / POWER(10, COALESCE(t.decimals, 18)) AS proper_amount
FROM erc20_ethereum.evt_Transfer e
LEFT JOIN tokens.erc20 t 
    ON e.contract_address = t.contract_address 
    AND t.blockchain = 'ethereum'
WHERE e.evt_block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 20
```

### 3. Debug: ETH vs Wei conversion issues
```sql
-- Common mistake: forgetting to divide by 1e18
SELECT 
    hash,
    value AS value_wei,
    value / 1e18 AS value_eth,
    gas_used * gas_price AS fee_wei,
    gas_used * gas_price / 1e18 AS fee_eth
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND value > 0
LIMIT 20
```

### 4. Verify USDC/USDT 6 decimal handling
```sql
-- USDC and USDT use 6 decimals, NOT 18
SELECT 
    evt_tx_hash,
    contract_address,
    CASE contract_address
        WHEN 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 THEN 'USDC'
        WHEN 0xdAC17F958D2ee523a2206206994597C13D831ec7 THEN 'USDT'
    END AS token,
    value AS raw_amount,
    value / 1e6 AS correct_amount,
    value / 1e18 AS wrong_amount_if_18_decimals
FROM erc20_ethereum.evt_Transfer
WHERE contract_address IN (
    0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606994597C13D831ec7, -- USDT
    0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48  -- USDC
)
    AND evt_block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 20
```

### 5. Debug SOL lamport conversion
```sql
-- SOL uses 9 decimals (lamports)
SELECT 
    signature,
    fee AS fee_lamports,
    fee / 1e9 AS fee_sol,
    -- Common mistake
    fee / 1e18 AS wrong_if_18_decimals
FROM solana.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 20
```

### 6. Handle NULL amounts in aggregations
```sql
-- NULL values can cause unexpected results
SELECT 
    SUM(amount_usd) AS total_with_nulls,
    SUM(COALESCE(amount_usd, 0)) AS total_null_as_zero,
    COUNT(*) AS total_rows,
    COUNT(amount_usd) AS rows_with_amount,
    COUNT(*) - COUNT(amount_usd) AS rows_missing_amount
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
```

### 7. Debug percentage calculations
```sql
-- Common issue: integer division
SELECT 
    project,
    COUNT(*) AS trades,
    SUM(amount_usd) AS volume,
    -- Wrong: integer division
    COUNT(*) / (SELECT COUNT(*) FROM dex.trades WHERE block_time >= NOW() - INTERVAL '24' HOUR AND blockchain = 'ethereum') AS wrong_pct,
    -- Correct: cast to float
    100.0 * COUNT(*) / (SELECT COUNT(*) FROM dex.trades WHERE block_time >= NOW() - INTERVAL '24' HOUR AND blockchain = 'ethereum') AS correct_pct
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10
```

### 8. Check for overflow in large numbers
```sql
-- Large values can overflow - use DECIMAL type
SELECT 
    tx_hash,
    TRY_CAST(value AS DECIMAL(38,0)) AS safe_large_value,
    TRY_CAST(value AS DECIMAL(38,0)) / POWER(CAST(10 AS DECIMAL(38,0)), 18) AS safe_eth_value
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND value > POWER(CAST(10 AS DECIMAL(38,0)), 25)
LIMIT 10
```

### 9. Debug WBTC 8 decimal handling
```sql
-- WBTC uses 8 decimals
SELECT 
    evt_tx_hash,
    value AS raw_amount,
    value / 1e8 AS correct_btc_amount,
    value / 1e18 AS wrong_if_18_decimals
FROM erc20_ethereum.evt_Transfer
WHERE contract_address = 0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599 -- WBTC
    AND evt_block_time >= NOW() - INTERVAL '24' HOUR
LIMIT 20
```

### 10. Validate price data reasonableness
```sql
-- Check for suspicious prices
SELECT 
    symbol,
    minute,
    price,
    CASE 
        WHEN price <= 0 THEN 'INVALID: Zero or negative'
        WHEN price > 1e10 THEN 'SUSPICIOUS: Extremely high'
        WHEN price < 1e-10 THEN 'SUSPICIOUS: Extremely low'
        ELSE 'OK'
    END AS price_check
FROM prices.usd
WHERE blockchain = 'ethereum'
    AND minute >= NOW() - INTERVAL '24' HOUR
    AND symbol IN ('WETH', 'USDC', 'USDT')
ORDER BY minute DESC
LIMIT 50
```

---

## Query Timeout Fixes

### 11. Add time filter to prevent full table scan
```sql
-- WRONG: Full table scan (will timeout)
-- SELECT COUNT(*) FROM ethereum.transactions;

-- CORRECT: Always filter by time
SELECT COUNT(*) 
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
```

### 12. Use LIMIT during development
```sql
-- Always use LIMIT when exploring data
SELECT 
    hash,
    "from",
    "to",
    value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
LIMIT 100  -- Remove this only when query is optimized
```

### 13. Avoid SELECT * on large tables
```sql
-- WRONG: SELECT * fetches all columns including large binaries
-- SELECT * FROM ethereum.transactions LIMIT 100;

-- CORRECT: Select only needed columns
SELECT 
    hash,
    block_time,
    "from",
    "to",
    value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100
```

### 14. Use APPROX_DISTINCT instead of COUNT(DISTINCT)
```sql
-- SLOW: Exact distinct count
-- SELECT COUNT(DISTINCT "from") FROM ethereum.transactions WHERE block_time >= NOW() - INTERVAL '7' DAY;

-- FAST: Approximate distinct count (< 2% error)
SELECT APPROX_DISTINCT("from") AS unique_senders
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '7' DAY
```

### 15. Pre-filter before JOIN
```sql
-- WRONG: Join then filter
-- SELECT * FROM ethereum.transactions t JOIN tokens.erc20 e ON t."to" = e.contract_address WHERE t.block_time >= NOW() - INTERVAL '1' HOUR;

-- CORRECT: Filter in CTE first
WITH recent_txs AS (
    SELECT hash, "to", value, block_time
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '1' HOUR
)
SELECT 
    t.hash,
    e.symbol,
    t.value / 1e18 AS eth_value
FROM recent_txs t
JOIN tokens.erc20 e 
    ON t."to" = e.contract_address 
    AND e.blockchain = 'ethereum'
LIMIT 100
```

### 16. Break large queries into smaller chunks
```sql
-- Process data in daily chunks
WITH daily_summary AS (
    SELECT 
        CAST(block_time AS DATE) AS day,
        COUNT(*) AS tx_count,
        SUM(gas_used * gas_price / 1e18) AS total_gas_eth
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '7' DAY
    GROUP BY 1
)
SELECT * FROM daily_summary
ORDER BY day DESC
```

### 17. Use materialized Spellbook tables when possible
```sql
-- Instead of raw table aggregation
-- Use pre-computed Spellbook tables
SELECT 
    day,
    SUM(amount_usd) AS volume
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '30' DAY
    AND blockchain = 'ethereum'
GROUP BY CAST(block_time AS DATE)
ORDER BY 1 DESC
```

### 18. Sample large datasets for exploration
```sql
-- Use modulo for random sampling (10% sample)
SELECT 
    hash,
    "from",
    value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND block_number % 10 = 0  -- ~10% sample
LIMIT 1000
```

### 19. Avoid correlated subqueries
```sql
-- WRONG: Correlated subquery (very slow)
-- SELECT hash, (SELECT COUNT(*) FROM ethereum.logs WHERE tx_hash = t.hash) AS log_count FROM ethereum.transactions t WHERE block_time >= NOW() - INTERVAL '1' HOUR;

-- CORRECT: Use JOIN
SELECT 
    t.hash,
    COUNT(l.tx_hash) AS log_count
FROM ethereum.transactions t
LEFT JOIN ethereum.logs l ON t.hash = l.tx_hash AND l.block_time >= NOW() - INTERVAL '1' HOUR
WHERE t.block_time >= NOW() - INTERVAL '1' HOUR
GROUP BY t.hash
LIMIT 100
```

### 20. Use index-friendly predicates
```sql
-- WRONG: Function on indexed column prevents index use
-- WHERE DATE_TRUNC('day', block_time) = CURRENT_DATE

-- CORRECT: Range comparison uses index
SELECT COUNT(*)
FROM ethereum.transactions
WHERE block_time >= CAST(CURRENT_DATE AS TIMESTAMP)
    AND block_time < CAST(CURRENT_DATE + INTERVAL '1' DAY AS TIMESTAMP)
```

---

## Finding Tokens and Contracts

### 21. Search token by name or symbol
```sql
SELECT 
    contract_address,
    symbol,
    name,
    decimals,
    blockchain
FROM tokens.erc20
WHERE (
    LOWER(symbol) LIKE '%usdc%' 
    OR LOWER(name) LIKE '%usd coin%'
)
    AND blockchain IN ('ethereum', 'polygon', 'arbitrum')
ORDER BY blockchain
```

### 22. Find contract by partial address
```sql
SELECT 
    contract_address,
    symbol,
    decimals
FROM tokens.erc20
WHERE CAST(contract_address AS VARCHAR) LIKE '%a0b86991%'
    AND blockchain = 'ethereum'
```

### 23. Get all tokens for a protocol
```sql
-- Find all tokens related to Uniswap
SELECT DISTINCT
    t.contract_address,
    t.symbol,
    t.name
FROM tokens.erc20 t
WHERE t.blockchain = 'ethereum'
    AND (
        LOWER(t.name) LIKE '%uniswap%'
        OR LOWER(t.symbol) LIKE '%uni%'
    )
LIMIT 50
```

### 24. Identify contract by recent activity
```sql
SELECT 
    "to" AS contract_address,
    COUNT(*) AS tx_count,
    COUNT(DISTINCT "from") AS unique_callers
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND "to" IS NOT NULL
GROUP BY 1
ORDER BY tx_count DESC
LIMIT 20
```

### 25. Check if address is a contract or EOA
```sql
SELECT 
    address,
    CASE 
        WHEN EXISTS (
            SELECT 1 FROM ethereum.traces 
            WHERE "to" = address 
                AND type = 'create'
                AND block_time >= NOW() - INTERVAL '365' DAY
        ) THEN 'CONTRACT'
        ELSE 'LIKELY EOA'
    END AS address_type
FROM (VALUES 
    (0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045),
    (0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48)
) AS t(address)
```

### 26. Find labeled addresses
```sql
SELECT 
    address,
    name,
    category,
    contributor
FROM labels.all
WHERE blockchain = 'ethereum'
    AND LOWER(name) LIKE '%binance%'
LIMIT 50
```

### 27. Get NFT collection contract info
```sql
SELECT DISTINCT
    nft_contract_address,
    collection,
    COUNT(*) AS sales_count,
    SUM(amount_usd) AS total_volume
FROM nft.trades
WHERE block_time >= NOW() - INTERVAL '30' DAY
    AND blockchain = 'ethereum'
    AND LOWER(collection) LIKE '%bored ape%'
GROUP BY 1, 2
```

### 28. Find DEX pool by token pair
```sql
SELECT 
    token_bought_symbol,
    token_sold_symbol,
    project,
    COUNT(*) AS trade_count,
    SUM(amount_usd) AS volume
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '7' DAY
    AND blockchain = 'ethereum'
    AND (
        (token_bought_symbol = 'WETH' AND token_sold_symbol = 'USDC')
        OR (token_bought_symbol = 'USDC' AND token_sold_symbol = 'WETH')
    )
GROUP BY 1, 2, 3
ORDER BY volume DESC
LIMIT 10
```

### 29. List all contracts deployed by address
```sql
SELECT 
    hash AS deployment_tx,
    block_time,
    contract_address
FROM ethereum.traces
WHERE "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
    AND type = 'create'
    AND success = true
ORDER BY block_time DESC
LIMIT 50
```

### 30. Verify token on multiple chains
```sql
SELECT 
    blockchain,
    contract_address,
    symbol,
    decimals
FROM tokens.erc20
WHERE symbol = 'USDC'
ORDER BY blockchain
```

---

## Cross-Chain Gotchas

### 31. Same token, different addresses per chain
```sql
-- USDC has different addresses on each chain
SELECT 
    blockchain,
    contract_address,
    symbol,
    decimals
FROM tokens.erc20
WHERE symbol = 'USDC'
ORDER BY blockchain
```

### 32. Different decimal standards per chain
```sql
-- Some chains use different decimal conventions
SELECT 
    blockchain,
    symbol,
    decimals,
    COUNT(*) AS token_count
FROM tokens.erc20
WHERE symbol IN ('USDC', 'USDT', 'WETH')
GROUP BY 1, 2, 3
ORDER BY blockchain, symbol
```

### 33. Cross-chain volume comparison (normalize correctly)
```sql
SELECT 
    blockchain,
    SUM(amount_usd) AS volume_usd,
    COUNT(*) AS trade_count
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain IN ('ethereum', 'arbitrum', 'polygon', 'optimism')
GROUP BY 1
ORDER BY volume_usd DESC
```

### 34. Wrapped token mappings
```sql
-- WETH/ETH equivalents per chain
SELECT 
    blockchain,
    contract_address,
    symbol
FROM tokens.erc20
WHERE symbol IN ('WETH', 'WMATIC', 'WAVAX', 'WBNB')
ORDER BY blockchain
```

### 35. Bridge transaction identification
```sql
-- Find potential bridge transactions
SELECT 
    hash,
    block_time,
    "from",
    "to",
    value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE "to" IN (
    0x40ec5B33f54e0E8A33A975908C5BA1c14e5BbbDf, -- Polygon Bridge
    0x99C9fc46f92E8a1c0deC1b1747d010903E884bE1, -- Optimism Bridge
    0x4Dbd4fc535Ac27206064B68FfCf827b0A60BAB3f  -- Arbitrum Bridge
)
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY block_time DESC
LIMIT 50
```

### 36. Chain-specific table names
```sql
-- Each chain has its own raw tables
-- ethereum.transactions, polygon.transactions, etc.
SELECT 'ethereum' AS chain, COUNT(*) AS tx_count
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR

UNION ALL

SELECT 'polygon' AS chain, COUNT(*) AS tx_count  
FROM polygon.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR

UNION ALL

SELECT 'arbitrum' AS chain, COUNT(*) AS tx_count
FROM arbitrum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
```

### 37. L2 gas differences
```sql
-- L2s have different gas models
SELECT 
    'ethereum' AS chain,
    AVG(gas_price / 1e9) AS avg_gas_gwei
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR

UNION ALL

SELECT 
    'arbitrum' AS chain,
    AVG(gas_price / 1e9) AS avg_gas_gwei
FROM arbitrum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
```

### 38. Native token naming differences
```sql
-- Different chains use different native token names
SELECT 
    blockchain,
    native_token,
    native_decimals
FROM (VALUES
    ('ethereum', 'ETH', 18),
    ('polygon', 'MATIC', 18),
    ('arbitrum', 'ETH', 18),
    ('optimism', 'ETH', 18),
    ('avalanche', 'AVAX', 18),
    ('bsc', 'BNB', 18)
) AS t(blockchain, native_token, native_decimals)
```

### 39. Cross-chain DEX aggregator trades
```sql
SELECT 
    blockchain,
    project,
    COUNT(*) AS trades,
    SUM(amount_usd) AS volume
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND project = '1inch'
GROUP BY 1, 2
ORDER BY volume DESC
```

### 40. Stablecoin peg check across chains
```sql
SELECT 
    blockchain,
    symbol,
    AVG(price) AS avg_price,
    MIN(price) AS min_price,
    MAX(price) AS max_price
FROM prices.usd
WHERE symbol IN ('USDC', 'USDT', 'DAI')
    AND minute >= NOW() - INTERVAL '24' HOUR
GROUP BY 1, 2
ORDER BY blockchain, symbol
```

---

## Missing Data and Indexing Delays

### 41. Check latest indexed block
```sql
SELECT 
    MAX(number) AS latest_block,
    MAX(time) AS latest_block_time,
    DATE_DIFF('minute', MAX(time), NOW()) AS minutes_behind
FROM ethereum.blocks
```

### 42. Check data freshness for trades
```sql
SELECT 
    MAX(block_time) AS latest_trade_time,
    DATE_DIFF('minute', MAX(block_time), NOW()) AS minutes_behind
FROM dex.trades
WHERE blockchain = 'ethereum'
```

### 43. Verify transaction exists in raw data
```sql
-- Use this when user says their tx is missing
SELECT 
    hash,
    block_number,
    block_time,
    success
FROM ethereum.transactions
WHERE hash = 0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef -- Replace with actual tx hash
```

### 44. Check decoded event lag
```sql
SELECT 
    'erc20_transfers' AS event_type,
    MAX(evt_block_time) AS latest_event,
    DATE_DIFF('minute', MAX(evt_block_time), NOW()) AS minutes_behind
FROM erc20_ethereum.evt_Transfer

UNION ALL

SELECT 
    'dex_trades' AS event_type,
    MAX(block_time) AS latest_event,
    DATE_DIFF('minute', MAX(block_time), NOW()) AS minutes_behind
FROM dex.trades
WHERE blockchain = 'ethereum'
```

### 45. Find gaps in block data
```sql
WITH block_numbers AS (
    SELECT number
    FROM ethereum.blocks
    WHERE time >= NOW() - INTERVAL '1' HOUR
),
expected AS (
    SELECT MIN(number) AS start_block, MAX(number) AS end_block
    FROM block_numbers
)
SELECT 
    (SELECT end_block - start_block + 1 FROM expected) AS expected_blocks,
    COUNT(*) AS actual_blocks,
    (SELECT end_block - start_block + 1 FROM expected) - COUNT(*) AS missing_blocks
FROM block_numbers
```

### 46. Compare raw vs decoded event counts
```sql
-- If counts differ significantly, decoding may be lagging
SELECT 
    'raw_transfer_logs' AS source,
    COUNT(*) AS count
FROM ethereum.logs
WHERE topic0 = 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
    AND block_time >= NOW() - INTERVAL '1' HOUR

UNION ALL

SELECT 
    'decoded_transfers' AS source,
    COUNT(*) AS count
FROM erc20_ethereum.evt_Transfer
WHERE evt_block_time >= NOW() - INTERVAL '1' HOUR
```

### 47. Check Spellbook refresh status
```sql
-- Spellbook tables update on different schedules
SELECT 
    MAX(block_time) AS latest_data
FROM dex.trades
WHERE blockchain = 'ethereum'
```

### 48. Verify data for specific contract exists
```sql
SELECT 
    COUNT(*) AS event_count,
    MIN(evt_block_time) AS first_event,
    MAX(evt_block_time) AS last_event
FROM erc20_ethereum.evt_Transfer
WHERE contract_address = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 -- USDC
    AND evt_block_time >= NOW() - INTERVAL '24' HOUR
```

### 49. Check for reorgs affecting data
```sql
-- Look for duplicate block numbers (sign of reorg)
SELECT 
    number,
    COUNT(*) AS block_count
FROM ethereum.blocks
WHERE time >= NOW() - INTERVAL '1' HOUR
GROUP BY number
HAVING COUNT(*) > 1
```

### 50. Debug: Why is my contract not showing in decoded tables?
```sql
-- Contract must be submitted for decoding
-- Check if it appears in raw logs
SELECT 
    tx_hash,
    block_time,
    topic0,
    BYTELENGTH(data) AS data_length
FROM ethereum.logs
WHERE contract_address = 0xYourContractAddress -- Replace with actual address
    AND block_time >= NOW() - INTERVAL '24' HOUR
LIMIT 20
-- If rows appear here but not in decoded tables, the contract needs to be submitted
```

---

*Last updated: March 2026*
