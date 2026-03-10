# Raw Tables: Ethereum

**50 production-ready Dune SQL queries for Ethereum raw tables**

**Tables covered:** `ethereum.transactions`, `ethereum.logs`, `ethereum.traces`, `ethereum.blocks`

---

## ethereum.transactions

### 1. Get latest 100 transactions
```sql
SELECT 
    hash,
    block_number,
    "from",
    "to",
    value / 1e18 AS eth_value,
    gas_used,
    gas_price / 1e9 AS gas_price_gwei
FROM ethereum.transactions
ORDER BY block_number DESC
LIMIT 100
```

### 2. Transactions from a specific address (Vitalik)
```sql
SELECT 
    hash,
    block_time,
    "to",
    value / 1e18 AS eth_sent,
    gas_used * gas_price / 1e18 AS fee_eth
FROM ethereum.transactions
WHERE "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
    AND block_time >= NOW() - INTERVAL '7' DAY
ORDER BY block_time DESC
```

### 3. Top gas spenders in last 24 hours
```sql
SELECT 
    "from" AS wallet,
    COUNT(*) AS tx_count,
    SUM(gas_used * gas_price) / 1e18 AS total_gas_eth
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 3 DESC
LIMIT 50
```

### 4. Failed transactions analysis
```sql
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    COUNT(*) AS total_txs,
    SUM(CASE WHEN success = false THEN 1 ELSE 0 END) AS failed_txs,
    ROUND(100.0 * SUM(CASE WHEN success = false THEN 1 ELSE 0 END) / COUNT(*), 2) AS fail_rate
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 1 DESC
```

### 5. Contract creation transactions
```sql
SELECT 
    hash,
    block_time,
    "from" AS deployer,
    gas_used,
    value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE "to" IS NULL
    AND block_time >= NOW() - INTERVAL '7' DAY
ORDER BY block_time DESC
LIMIT 100
```

### 6. High-value ETH transfers (> 100 ETH)
```sql
SELECT 
    hash,
    block_time,
    "from",
    "to",
    value / 1e18 AS eth_value
FROM ethereum.transactions
WHERE value > 100 * 1e18
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY value DESC
```

### 7. Transaction count by type (EIP-1559 vs legacy)
```sql
SELECT 
    DATE_TRUNC('day', block_time) AS day,
    SUM(CASE WHEN type = '2' THEN 1 ELSE 0 END) AS eip1559_txs,
    SUM(CASE WHEN type IN ('0', '1') THEN 1 ELSE 0 END) AS legacy_txs
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 1
```

### 8. Average gas price by hour
```sql
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    AVG(gas_price) / 1e9 AS avg_gas_gwei,
    MAX(gas_price) / 1e9 AS max_gas_gwei,
    MIN(gas_price) / 1e9 AS min_gas_gwei
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 1 DESC
```

### 9. Uniswap V3 Router transactions
```sql
SELECT 
    hash,
    block_time,
    "from",
    value / 1e18 AS eth_value,
    gas_used
FROM ethereum.transactions
WHERE "to" = 0xE592427A0AEce92De3Edee1F18E0157C05861564 -- Uniswap V3 Router
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
```

### 10. Transactions to USDC contract
```sql
SELECT 
    hash,
    block_time,
    "from",
    BYTEARRAY_SUBSTRING(data, 1, 4) AS method_id,
    gas_used
FROM ethereum.transactions
WHERE "to" = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 -- USDC
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 11. Gas efficiency by sender
```sql
SELECT 
    "from" AS sender,
    COUNT(*) AS tx_count,
    AVG(gas_used) AS avg_gas_used,
    AVG(gas_price) / 1e9 AS avg_gas_price_gwei
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
HAVING COUNT(*) >= 10
ORDER BY 3 DESC
LIMIT 50
```

### 12. Nonce gaps detection
```sql
SELECT 
    "from",
    nonce,
    LAG(nonce) OVER (PARTITION BY "from" ORDER BY nonce) AS prev_nonce,
    nonce - LAG(nonce) OVER (PARTITION BY "from" ORDER BY nonce) AS gap
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
ORDER BY nonce
```

### 13. MEV bot transactions (high gas priority)
```sql
SELECT 
    hash,
    block_time,
    "from",
    "to",
    max_priority_fee_per_gas / 1e9 AS priority_fee_gwei
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
    AND max_priority_fee_per_gas > 50 * 1e9
ORDER BY max_priority_fee_per_gas DESC
LIMIT 100
```

### 14. Transaction throughput by block
```sql
SELECT 
    block_number,
    COUNT(*) AS tx_count,
    SUM(gas_used) AS total_gas,
    MAX(block_time) AS block_time
FROM ethereum.transactions
WHERE block_number >= (SELECT MAX(block_number) - 100 FROM ethereum.transactions)
GROUP BY 1
ORDER BY 1 DESC
```

### 15. Wallet activity frequency
```sql
SELECT 
    "from" AS wallet,
    MIN(block_time) AS first_tx,
    MAX(block_time) AS last_tx,
    COUNT(*) AS total_txs,
    DATE_DIFF('day', MIN(block_time), MAX(block_time)) AS active_days
FROM ethereum.transactions
WHERE "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
GROUP BY 1
```

---

## ethereum.logs

### 16. ERC20 Transfer events (last hour)
```sql
SELECT 
    block_time,
    contract_address AS token,
    BYTEARRAY_SUBSTRING(topic1, 13, 20) AS "from",
    BYTEARRAY_SUBSTRING(topic2, 13, 20) AS "to",
    BYTEARRAY_TO_UINT256(data) AS raw_amount
FROM ethereum.logs
WHERE topic0 = 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef -- Transfer
    AND block_time >= NOW() - INTERVAL '1' HOUR
LIMIT 100
```

### 17. USDC Transfer events
```sql
SELECT 
    block_time,
    tx_hash,
    BYTEARRAY_SUBSTRING(topic1, 13, 20) AS sender,
    BYTEARRAY_SUBSTRING(topic2, 13, 20) AS recipient,
    BYTEARRAY_TO_UINT256(data) / 1e6 AS usdc_amount
FROM ethereum.logs
WHERE contract_address = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
    AND topic0 = 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
```

### 18. Uniswap V3 Swap events
```sql
SELECT 
    block_time,
    tx_hash,
    contract_address AS pool,
    BYTEARRAY_SUBSTRING(topic1, 13, 20) AS sender,
    BYTEARRAY_SUBSTRING(topic2, 13, 20) AS recipient
FROM ethereum.logs
WHERE topic0 = 0xc42079f94a6350d7e6235f29174924f928cc2ac818eb64fed8004e115fbcca67 -- Swap
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 19. Approval events for WETH
```sql
SELECT 
    block_time,
    tx_hash,
    BYTEARRAY_SUBSTRING(topic1, 13, 20) AS owner,
    BYTEARRAY_SUBSTRING(topic2, 13, 20) AS spender,
    BYTEARRAY_TO_UINT256(data) / 1e18 AS approved_amount
FROM ethereum.logs
WHERE contract_address = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 -- WETH
    AND topic0 = 0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925 -- Approval
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 20. NFT Transfer events (ERC721)
```sql
SELECT 
    block_time,
    contract_address AS nft_contract,
    tx_hash,
    BYTEARRAY_SUBSTRING(topic1, 13, 20) AS "from",
    BYTEARRAY_SUBSTRING(topic2, 13, 20) AS "to",
    BYTEARRAY_TO_UINT256(topic3) AS token_id
FROM ethereum.logs
WHERE topic0 = 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
    AND topic3 IS NOT NULL -- ERC721 has indexed tokenId
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 21. Event count by contract
```sql
SELECT 
    contract_address,
    COUNT(*) AS event_count
FROM ethereum.logs
WHERE block_time >= NOW() - INTERVAL '1' HOUR
GROUP BY 1
ORDER BY 2 DESC
LIMIT 50
```

### 22. Deposit events (WETH wrapping)
```sql
SELECT 
    block_time,
    tx_hash,
    BYTEARRAY_SUBSTRING(topic1, 13, 20) AS depositor,
    BYTEARRAY_TO_UINT256(data) / 1e18 AS eth_amount
FROM ethereum.logs
WHERE contract_address = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 -- WETH
    AND topic0 = 0xe1fffcc4923d04b559f4d29a8bfc6cda04eb5b0d3c460751c2402c5c5cc9109c -- Deposit
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY block_time DESC
```

### 23. Sync events (DEX liquidity pools)
```sql
SELECT 
    block_time,
    contract_address AS pool,
    BYTEARRAY_TO_UINT256(BYTEARRAY_SUBSTRING(data, 1, 32)) AS reserve0,
    BYTEARRAY_TO_UINT256(BYTEARRAY_SUBSTRING(data, 33, 32)) AS reserve1
FROM ethereum.logs
WHERE topic0 = 0x1c411e9a96e071241c2f21f7726b17ae89e3cab4c78be50e062b03a9fffbbad1 -- Sync
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 24. PairCreated events (new liquidity pools)
```sql
SELECT 
    block_time,
    tx_hash,
    contract_address AS factory,
    BYTEARRAY_SUBSTRING(topic1, 13, 20) AS token0,
    BYTEARRAY_SUBSTRING(topic2, 13, 20) AS token1
FROM ethereum.logs
WHERE topic0 = 0x0d3648bd0f6ba80134a33ba9275ac585d9d315f0ad8355cddefde31afa28d0e9 -- PairCreated
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY block_time DESC
```

### 25. Logs from specific transaction
```sql
SELECT 
    log_index,
    contract_address,
    topic0,
    topic1,
    topic2,
    data
FROM ethereum.logs
WHERE tx_hash = 0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
ORDER BY log_index
```

---

## ethereum.traces

### 26. Internal ETH transfers
```sql
SELECT 
    block_time,
    tx_hash,
    "from",
    "to",
    value / 1e18 AS eth_value,
    call_type
FROM ethereum.traces
WHERE value > 0
    AND block_time >= NOW() - INTERVAL '1' HOUR
    AND call_type = 'call'
ORDER BY value DESC
LIMIT 100
```

### 27. Failed internal calls
```sql
SELECT 
    block_time,
    tx_hash,
    "from",
    "to",
    error,
    call_type
FROM ethereum.traces
WHERE success = false
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 28. Contract calls depth analysis
```sql
SELECT 
    tx_hash,
    MAX(CARDINALITY(trace_address)) AS max_depth,
    COUNT(*) AS total_traces
FROM ethereum.traces
WHERE block_time >= NOW() - INTERVAL '1' HOUR
GROUP BY 1
ORDER BY 2 DESC
LIMIT 50
```

### 29. SELFDESTRUCT operations
```sql
SELECT 
    block_time,
    tx_hash,
    "from" AS destroyed_contract,
    "to" AS beneficiary,
    value / 1e18 AS eth_sent
FROM ethereum.traces
WHERE type = 'suicide'
    AND block_time >= NOW() - INTERVAL '7' DAY
ORDER BY block_time DESC
```

### 30. Contract creation via CREATE2
```sql
SELECT 
    block_time,
    tx_hash,
    "from" AS factory,
    address AS new_contract,
    gas_used
FROM ethereum.traces
WHERE type = 'create'
    AND call_type = 'create2'
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 31. Delegatecall usage
```sql
SELECT 
    block_time,
    tx_hash,
    "from",
    "to" AS implementation,
    gas_used
FROM ethereum.traces
WHERE call_type = 'delegatecall'
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 32. Static calls (read-only)
```sql
SELECT 
    block_time,
    tx_hash,
    "from",
    "to",
    BYTEARRAY_SUBSTRING(input, 1, 4) AS method_id
FROM ethereum.traces
WHERE call_type = 'staticcall'
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 33. ETH received by contract
```sql
SELECT 
    DATE_TRUNC('day', block_time) AS day,
    SUM(value) / 1e18 AS eth_received
FROM ethereum.traces
WHERE "to" = 0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D -- Uniswap V2 Router
    AND value > 0
    AND block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 1
```

### 34. Gas usage by trace type
```sql
SELECT 
    call_type,
    COUNT(*) AS trace_count,
    AVG(gas_used) AS avg_gas,
    SUM(gas_used) AS total_gas
FROM ethereum.traces
WHERE block_time >= NOW() - INTERVAL '1' HOUR
GROUP BY 1
ORDER BY 4 DESC
```

### 35. Reverted traces with errors
```sql
SELECT 
    block_time,
    tx_hash,
    "from",
    "to",
    error,
    BYTEARRAY_SUBSTRING(input, 1, 4) AS method_id
FROM ethereum.traces
WHERE success = false
    AND error IS NOT NULL
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

---

## ethereum.blocks

### 36. Latest blocks overview
```sql
SELECT 
    number,
    time AS block_time,
    gas_used,
    gas_limit,
    ROUND(100.0 * gas_used / gas_limit, 2) AS gas_utilization,
    base_fee_per_gas / 1e9 AS base_fee_gwei
FROM ethereum.blocks
ORDER BY number DESC
LIMIT 100
```

### 37. Block time analysis
```sql
SELECT 
    number,
    time AS block_time,
    time - LAG(time) OVER (ORDER BY number) AS time_since_last
FROM ethereum.blocks
WHERE number >= (SELECT MAX(number) - 100 FROM ethereum.blocks)
ORDER BY number DESC
```

### 38. Average block metrics by hour
```sql
SELECT 
    DATE_TRUNC('hour', time) AS hour,
    COUNT(*) AS blocks,
    AVG(gas_used) AS avg_gas_used,
    AVG(base_fee_per_gas) / 1e9 AS avg_base_fee_gwei
FROM ethereum.blocks
WHERE time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 1 DESC
```

### 39. Blocks with highest gas usage
```sql
SELECT 
    number,
    time AS block_time,
    gas_used,
    gas_limit,
    base_fee_per_gas / 1e9 AS base_fee_gwei
FROM ethereum.blocks
WHERE time >= NOW() - INTERVAL '24' HOUR
ORDER BY gas_used DESC
LIMIT 20
```

### 40. Base fee volatility
```sql
SELECT 
    number,
    base_fee_per_gas / 1e9 AS base_fee_gwei,
    LAG(base_fee_per_gas) OVER (ORDER BY number) / 1e9 AS prev_base_fee,
    ROUND(100.0 * (base_fee_per_gas - LAG(base_fee_per_gas) OVER (ORDER BY number)) 
        / LAG(base_fee_per_gas) OVER (ORDER BY number), 2) AS pct_change
FROM ethereum.blocks
WHERE number >= (SELECT MAX(number) - 50 FROM ethereum.blocks)
ORDER BY number DESC
```

### 41. Daily block production
```sql
SELECT 
    DATE_TRUNC('day', time) AS day,
    COUNT(*) AS blocks,
    AVG(gas_used) AS avg_gas_used,
    SUM(gas_used) AS total_gas_used
FROM ethereum.blocks
WHERE time >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 1
```

### 42. Empty blocks detection
```sql
SELECT 
    number,
    time AS block_time,
    gas_used
FROM ethereum.blocks
WHERE gas_used < 21000
    AND time >= NOW() - INTERVAL '7' DAY
ORDER BY time DESC
```

### 43. Block size distribution
```sql
SELECT 
    CASE 
        WHEN gas_used < 5000000 THEN 'Low (<5M)'
        WHEN gas_used < 15000000 THEN 'Medium (5-15M)'
        ELSE 'High (>15M)'
    END AS gas_bucket,
    COUNT(*) AS block_count
FROM ethereum.blocks
WHERE time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 2 DESC
```

### 44. Miner/validator rewards estimation
```sql
SELECT 
    DATE_TRUNC('day', time) AS day,
    COUNT(*) AS blocks,
    SUM(gas_used * base_fee_per_gas) / 1e18 AS burned_eth
FROM ethereum.blocks
WHERE time >= NOW() - INTERVAL '7' DAY
GROUP BY 1
ORDER BY 1
```

### 45. Block difficulty (pre-merge historical)
```sql
SELECT 
    DATE_TRUNC('day', time) AS day,
    AVG(difficulty) AS avg_difficulty
FROM ethereum.blocks
WHERE time BETWEEN DATE '2022-01-01' AND DATE '2022-09-15'
GROUP BY 1
ORDER BY 1
```

---

## Cross-Table Queries

### 46. Transaction with all logs
```sql
SELECT 
    t.hash,
    t.block_time,
    t."from",
    t."to",
    l.contract_address,
    l.topic0
FROM ethereum.transactions t
LEFT JOIN ethereum.logs l ON t.hash = l.tx_hash
WHERE t.hash = 0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
ORDER BY l.log_index
```

### 47. Failed transactions with revert reasons
```sql
SELECT 
    t.hash,
    t.block_time,
    t."from",
    t."to",
    tr.error
FROM ethereum.transactions t
JOIN ethereum.traces tr ON t.hash = tr.tx_hash
WHERE t.success = false
    AND tr.trace_address = ARRAY[]
    AND t.block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY t.block_time DESC
LIMIT 50
```

### 48. Block with transaction count
```sql
SELECT 
    b.number,
    b.time,
    b.gas_used,
    b.base_fee_per_gas / 1e9 AS base_fee_gwei,
    COUNT(t.hash) AS tx_count
FROM ethereum.blocks b
LEFT JOIN ethereum.transactions t ON b.number = t.block_number
WHERE b.number >= (SELECT MAX(number) - 10 FROM ethereum.blocks)
GROUP BY 1, 2, 3, 4
ORDER BY 1 DESC
```

### 49. High-gas transactions with traces
```sql
SELECT 
    t.hash,
    t.gas_used,
    COUNT(tr.tx_hash) AS trace_count,
    MAX(CARDINALITY(tr.trace_address)) AS max_depth
FROM ethereum.transactions t
JOIN ethereum.traces tr ON t.hash = tr.tx_hash
WHERE t.block_time >= NOW() - INTERVAL '1' HOUR
    AND t.gas_used > 500000
GROUP BY 1, 2
ORDER BY 2 DESC
LIMIT 50
```

### 50. Complete transaction analysis
```sql
WITH tx AS (
    SELECT 
        hash,
        block_time,
        "from",
        "to",
        value / 1e18 AS eth_value,
        gas_used,
        gas_price / 1e9 AS gas_price_gwei,
        success
    FROM ethereum.transactions
    WHERE hash = 0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
),
events AS (
    SELECT 
        tx_hash,
        COUNT(*) AS event_count
    FROM ethereum.logs
    WHERE tx_hash = 0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
    GROUP BY 1
),
traces AS (
    SELECT 
        tx_hash,
        COUNT(*) AS trace_count,
        SUM(CASE WHEN success = false THEN 1 ELSE 0 END) AS failed_traces
    FROM ethereum.traces
    WHERE tx_hash = 0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
    GROUP BY 1
)
SELECT 
    tx.*,
    COALESCE(e.event_count, 0) AS events,
    COALESCE(t.trace_count, 0) AS traces,
    COALESCE(t.failed_traces, 0) AS failed_traces
FROM tx
LEFT JOIN events e ON tx.hash = e.tx_hash
LEFT JOIN traces t ON tx.hash = t.tx_hash
```

---

**Footer**  
Built by [CryptoAI-Jedi](https://github.com/CryptoAI-Jedi) | [Dune SQL Cheatsheet](https://github.com/CryptoAI-Jedi/dune-sql-cheatsheet)
