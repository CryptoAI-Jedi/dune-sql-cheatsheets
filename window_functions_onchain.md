# Window Functions for On-Chain Data

**50 production-ready Dune SQL queries using window functions for blockchain analytics**

**Functions covered:** ROW_NUMBER, RANK, DENSE_RANK, LAG, LEAD, SUM() OVER(), AVG() OVER(), FIRST_VALUE, LAST_VALUE, NTILE

---

## ROW_NUMBER Applications

### 1. Get first transaction of each wallet
```sql
WITH ranked AS (
    SELECT 
        "from" AS wallet,
        hash,
        block_time,
        ROW_NUMBER() OVER(PARTITION BY "from" ORDER BY block_time) AS tx_rank
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '30' DAY
)
SELECT wallet, hash, block_time
FROM ranked
WHERE tx_rank = 1
LIMIT 100
```

### 2. Latest trade per token pair
```sql
WITH ranked AS (
    SELECT 
        token_bought_symbol,
        token_sold_symbol,
        tx_hash,
        amount_usd,
        block_time,
        ROW_NUMBER() OVER(
            PARTITION BY token_bought_symbol, token_sold_symbol 
            ORDER BY block_time DESC
        ) AS rn
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '24' HOUR
        AND blockchain = 'ethereum'
)
SELECT token_bought_symbol, token_sold_symbol, tx_hash, amount_usd, block_time
FROM ranked
WHERE rn = 1
ORDER BY amount_usd DESC
LIMIT 50
```

### 3. Number transactions per wallet per day
```sql
SELECT 
    "from" AS wallet,
    CAST(block_time AS DATE) AS tx_date,
    hash,
    ROW_NUMBER() OVER(
        PARTITION BY "from", CAST(block_time AS DATE) 
        ORDER BY block_time
    ) AS daily_tx_number
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '7' DAY
    AND "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
ORDER BY block_time DESC
```

### 4. Top 3 trades per DEX
```sql
WITH ranked AS (
    SELECT 
        project,
        tx_hash,
        amount_usd,
        block_time,
        ROW_NUMBER() OVER(PARTITION BY project ORDER BY amount_usd DESC) AS rn
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '24' HOUR
        AND blockchain = 'ethereum'
        AND amount_usd IS NOT NULL
)
SELECT project, tx_hash, amount_usd, block_time
FROM ranked
WHERE rn <= 3
ORDER BY project, rn
```

### 5. Deduplicate by keeping latest record
```sql
WITH deduped AS (
    SELECT 
        "from",
        "to",
        value,
        block_time,
        ROW_NUMBER() OVER(
            PARTITION BY "from", "to" 
            ORDER BY block_time DESC
        ) AS rn
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '24' HOUR
)
SELECT "from", "to", value / 1e18 AS eth_value, block_time
FROM deduped
WHERE rn = 1
LIMIT 100
```

---

## RANK and DENSE_RANK

### 6. Rank wallets by trading volume
```sql
SELECT 
    taker,
    SUM(amount_usd) AS total_volume,
    RANK() OVER(ORDER BY SUM(amount_usd) DESC) AS volume_rank
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '7' DAY
    AND blockchain = 'ethereum'
GROUP BY 1
ORDER BY volume_rank
LIMIT 100
```

### 7. DENSE_RANK for tied values
```sql
SELECT 
    project,
    COUNT(*) AS trade_count,
    DENSE_RANK() OVER(ORDER BY COUNT(*) DESC) AS popularity_rank
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
GROUP BY 1
ORDER BY popularity_rank
```

### 8. Rank tokens by transfer count
```sql
SELECT 
    contract_address,
    COUNT(*) AS transfer_count,
    RANK() OVER(ORDER BY COUNT(*) DESC) AS token_rank
FROM erc20_ethereum.evt_Transfer
WHERE evt_block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY token_rank
LIMIT 50
```

### 9. Rank NFT collections by sales
```sql
SELECT 
    nft_contract_address,
    SUM(amount_usd) AS total_sales,
    COUNT(*) AS sale_count,
    RANK() OVER(ORDER BY SUM(amount_usd) DESC) AS sales_rank,
    RANK() OVER(ORDER BY COUNT(*) DESC) AS volume_rank
FROM nft.trades
WHERE block_time >= NOW() - INTERVAL '7' DAY
    AND blockchain = 'ethereum'
GROUP BY 1
ORDER BY sales_rank
LIMIT 50
```

### 10. Percentile rank of trade sizes
```sql
SELECT 
    tx_hash,
    amount_usd,
    PERCENT_RANK() OVER(ORDER BY amount_usd) AS percentile
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
    AND amount_usd > 0
ORDER BY amount_usd DESC
LIMIT 100
```

---

## LAG and LEAD for Time-Series Analysis

### 11. Calculate time between transactions
```sql
SELECT 
    hash,
    block_time,
    LAG(block_time) OVER(ORDER BY block_time) AS prev_tx_time,
    DATE_DIFF('second', 
        LAG(block_time) OVER(ORDER BY block_time), 
        block_time
    ) AS seconds_since_last_tx
FROM ethereum.transactions
WHERE "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
    AND block_time >= NOW() - INTERVAL '30' DAY
ORDER BY block_time DESC
LIMIT 50
```

### 12. Price change between trades
```sql
SELECT 
    tx_hash,
    block_time,
    token_bought_symbol,
    amount_usd / token_bought_amount AS price,
    LAG(amount_usd / token_bought_amount) OVER(
        PARTITION BY token_bought_symbol 
        ORDER BY block_time
    ) AS prev_price,
    (amount_usd / token_bought_amount) - 
    LAG(amount_usd / token_bought_amount) OVER(
        PARTITION BY token_bought_symbol 
        ORDER BY block_time
    ) AS price_change
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
    AND token_bought_symbol = 'WETH'
    AND token_bought_amount > 0
ORDER BY block_time DESC
LIMIT 100
```

### 13. LEAD to get next transaction
```sql
SELECT 
    hash,
    block_time,
    value / 1e18 AS eth_value,
    LEAD(block_time) OVER(ORDER BY block_time) AS next_tx_time,
    LEAD(value / 1e18) OVER(ORDER BY block_time) AS next_tx_value
FROM ethereum.transactions
WHERE "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
    AND block_time >= NOW() - INTERVAL '30' DAY
ORDER BY block_time DESC
LIMIT 50
```

### 14. Day-over-day volume change
```sql
WITH daily_volume AS (
    SELECT 
        CAST(block_time AS DATE) AS day,
        SUM(amount_usd) AS volume
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '30' DAY
        AND blockchain = 'ethereum'
    GROUP BY 1
)
SELECT 
    day,
    volume,
    LAG(volume) OVER(ORDER BY day) AS prev_day_volume,
    ROUND(100.0 * (volume - LAG(volume) OVER(ORDER BY day)) / 
        NULLIF(LAG(volume) OVER(ORDER BY day), 0), 2) AS pct_change
FROM daily_volume
ORDER BY day DESC
```

### 15. Moving average comparison with LAG
```sql
WITH hourly_gas AS (
    SELECT 
        DATE_TRUNC('hour', block_time) AS hour,
        AVG(gas_price / 1e9) AS avg_gas_gwei
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '7' DAY
    GROUP BY 1
)
SELECT 
    hour,
    avg_gas_gwei,
    LAG(avg_gas_gwei, 24) OVER(ORDER BY hour) AS same_hour_yesterday,
    avg_gas_gwei - LAG(avg_gas_gwei, 24) OVER(ORDER BY hour) AS gas_diff
FROM hourly_gas
ORDER BY hour DESC
LIMIT 48
```

### 16. Identify consecutive transactions
```sql
SELECT 
    hash,
    block_number,
    nonce,
    LAG(nonce) OVER(PARTITION BY "from" ORDER BY nonce) AS prev_nonce,
    nonce - LAG(nonce) OVER(PARTITION BY "from" ORDER BY nonce) AS nonce_gap
FROM ethereum.transactions
WHERE "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
    AND block_time >= NOW() - INTERVAL '30' DAY
ORDER BY nonce DESC
LIMIT 50
```

### 17. Trade direction changes
```sql
WITH trades AS (
    SELECT 
        block_time,
        tx_hash,
        CASE WHEN token_bought_symbol = 'WETH' THEN 'BUY' ELSE 'SELL' END AS direction
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '24' HOUR
        AND blockchain = 'ethereum'
        AND (token_bought_symbol = 'WETH' OR token_sold_symbol = 'WETH')
)
SELECT 
    block_time,
    tx_hash,
    direction,
    LAG(direction) OVER(ORDER BY block_time) AS prev_direction,
    CASE 
        WHEN direction != LAG(direction) OVER(ORDER BY block_time) THEN 'CHANGED'
        ELSE 'SAME'
    END AS direction_change
FROM trades
ORDER BY block_time DESC
LIMIT 100
```

### 18. Predict next block gas based on trend
```sql
WITH block_gas AS (
    SELECT 
        number,
        gas_used,
        LAG(gas_used, 1) OVER(ORDER BY number) AS prev_gas,
        LAG(gas_used, 2) OVER(ORDER BY number) AS prev_prev_gas
    FROM ethereum.blocks
    WHERE time >= NOW() - INTERVAL '1' HOUR
)
SELECT 
    number,
    gas_used,
    (gas_used + prev_gas + prev_prev_gas) / 3 AS moving_avg,
    gas_used + (gas_used - prev_gas) AS linear_prediction
FROM block_gas
WHERE prev_prev_gas IS NOT NULL
ORDER BY number DESC
LIMIT 20
```

### 19. Session analysis with time gaps
```sql
WITH tx_gaps AS (
    SELECT 
        hash,
        block_time,
        DATE_DIFF('minute', 
            LAG(block_time) OVER(PARTITION BY "from" ORDER BY block_time),
            block_time
        ) AS minutes_since_last
    FROM ethereum.transactions
    WHERE "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
        AND block_time >= NOW() - INTERVAL '30' DAY
)
SELECT 
    hash,
    block_time,
    minutes_since_last,
    CASE WHEN minutes_since_last > 60 OR minutes_since_last IS NULL 
         THEN 'NEW_SESSION' ELSE 'CONTINUING' END AS session_status
FROM tx_gaps
ORDER BY block_time DESC
LIMIT 50
```

### 20. Compare with N periods ago
```sql
SELECT 
    CAST(block_time AS DATE) AS day,
    SUM(amount_usd) AS volume,
    LAG(SUM(amount_usd), 7) OVER(ORDER BY CAST(block_time AS DATE)) AS volume_7d_ago,
    LAG(SUM(amount_usd), 30) OVER(ORDER BY CAST(block_time AS DATE)) AS volume_30d_ago
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '60' DAY
    AND blockchain = 'ethereum'
GROUP BY 1
ORDER BY 1 DESC
LIMIT 30
```

---

## Running Totals and Cumulative Sums

### 21. Cumulative ETH transferred by wallet
```sql
SELECT 
    hash,
    block_time,
    value / 1e18 AS eth_value,
    SUM(value / 1e18) OVER(ORDER BY block_time) AS cumulative_eth_sent
FROM ethereum.transactions
WHERE "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
    AND block_time >= NOW() - INTERVAL '30' DAY
ORDER BY block_time DESC
```

### 22. Running balance calculation
```sql
WITH transfers AS (
    SELECT 
        block_time,
        tx_hash,
        CASE 
            WHEN "to" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 THEN value / 1e18
            ELSE -value / 1e18
        END AS net_flow
    FROM ethereum.transactions
    WHERE (
        "to" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
        OR "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
    )
        AND block_time >= NOW() - INTERVAL '30' DAY
)
SELECT 
    block_time,
    tx_hash,
    net_flow,
    SUM(net_flow) OVER(ORDER BY block_time) AS running_balance
FROM transfers
ORDER BY block_time DESC
LIMIT 100
```

### 23. Cumulative DEX volume by protocol
```sql
SELECT 
    project,
    CAST(block_time AS DATE) AS day,
    SUM(amount_usd) AS daily_volume,
    SUM(SUM(amount_usd)) OVER(
        PARTITION BY project 
        ORDER BY CAST(block_time AS DATE)
    ) AS cumulative_volume
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '30' DAY
    AND blockchain = 'ethereum'
GROUP BY 1, 2
ORDER BY project, day DESC
```

### 24. Running count of unique users
```sql
WITH first_trades AS (
    SELECT 
        taker,
        MIN(CAST(block_time AS DATE)) AS first_trade_date
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '30' DAY
        AND blockchain = 'ethereum'
    GROUP BY 1
)
SELECT 
    first_trade_date,
    COUNT(*) AS new_users,
    SUM(COUNT(*)) OVER(ORDER BY first_trade_date) AS cumulative_users
FROM first_trades
GROUP BY 1
ORDER BY 1 DESC
```

### 25. Cumulative gas spent
```sql
SELECT 
    CAST(block_time AS DATE) AS day,
    SUM(gas_used * gas_price / 1e18) AS daily_gas_eth,
    SUM(SUM(gas_used * gas_price / 1e18)) OVER(ORDER BY CAST(block_time AS DATE)) AS cumulative_gas_eth
FROM ethereum.transactions
WHERE "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
    AND block_time >= NOW() - INTERVAL '90' DAY
GROUP BY 1
ORDER BY 1 DESC
```

---

## Moving Averages and Rolling Statistics

### 26. 7-day moving average volume
```sql
WITH daily_volume AS (
    SELECT 
        CAST(block_time AS DATE) AS day,
        SUM(amount_usd) AS volume
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '60' DAY
        AND blockchain = 'ethereum'
    GROUP BY 1
)
SELECT 
    day,
    volume,
    AVG(volume) OVER(
        ORDER BY day 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d
FROM daily_volume
ORDER BY day DESC
LIMIT 30
```

### 27. Rolling 24-hour unique addresses
```sql
WITH hourly_unique AS (
    SELECT 
        DATE_TRUNC('hour', block_time) AS hour,
        COUNT(DISTINCT "from") AS unique_senders
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '7' DAY
    GROUP BY 1
)
SELECT 
    hour,
    unique_senders,
    AVG(unique_senders) OVER(
        ORDER BY hour 
        ROWS BETWEEN 23 PRECEDING AND CURRENT ROW
    ) AS rolling_24h_avg
FROM hourly_unique
ORDER BY hour DESC
LIMIT 48
```

### 28. Rolling standard deviation of gas prices
```sql
WITH hourly_gas AS (
    SELECT 
        DATE_TRUNC('hour', block_time) AS hour,
        AVG(gas_price / 1e9) AS avg_gas
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '7' DAY
    GROUP BY 1
)
SELECT 
    hour,
    avg_gas,
    AVG(avg_gas) OVER(ORDER BY hour ROWS BETWEEN 23 PRECEDING AND CURRENT ROW) AS rolling_mean,
    STDDEV(avg_gas) OVER(ORDER BY hour ROWS BETWEEN 23 PRECEDING AND CURRENT ROW) AS rolling_stddev
FROM hourly_gas
ORDER BY hour DESC
LIMIT 48
```

### 29. Rolling min/max trade sizes
```sql
WITH hourly_trades AS (
    SELECT 
        DATE_TRUNC('hour', block_time) AS hour,
        AVG(amount_usd) AS avg_trade_size
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '7' DAY
        AND blockchain = 'ethereum'
        AND amount_usd > 0
    GROUP BY 1
)
SELECT 
    hour,
    avg_trade_size,
    MIN(avg_trade_size) OVER(ORDER BY hour ROWS BETWEEN 23 PRECEDING AND CURRENT ROW) AS rolling_min,
    MAX(avg_trade_size) OVER(ORDER BY hour ROWS BETWEEN 23 PRECEDING AND CURRENT ROW) AS rolling_max
FROM hourly_trades
ORDER BY hour DESC
LIMIT 48
```

### 30. Exponential moving average approximation
```sql
WITH daily_volume AS (
    SELECT 
        CAST(block_time AS DATE) AS day,
        SUM(amount_usd) AS volume
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '60' DAY
        AND blockchain = 'ethereum'
    GROUP BY 1
),
ema AS (
    SELECT 
        day,
        volume,
        AVG(volume) OVER(ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS sma_7d,
        -- EMA approximation using weighted average
        SUM(volume * POWER(0.9, ROW_NUMBER() OVER(ORDER BY day DESC) - 1)) OVER(
            ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) / SUM(POWER(0.9, ROW_NUMBER() OVER(ORDER BY day DESC) - 1)) OVER(
            ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS ema_approx
    FROM daily_volume
)
SELECT day, volume, sma_7d
FROM ema
ORDER BY day DESC
LIMIT 30
```

---

## FIRST_VALUE and LAST_VALUE

### 31. First trade of each day
```sql
SELECT DISTINCT
    CAST(block_time AS DATE) AS day,
    FIRST_VALUE(tx_hash) OVER(
        PARTITION BY CAST(block_time AS DATE) 
        ORDER BY block_time
    ) AS first_tx,
    FIRST_VALUE(amount_usd) OVER(
        PARTITION BY CAST(block_time AS DATE) 
        ORDER BY block_time
    ) AS first_trade_amount
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '7' DAY
    AND blockchain = 'ethereum'
ORDER BY day DESC
```

### 32. Last price of each hour (OHLC close)
```sql
SELECT DISTINCT
    DATE_TRUNC('hour', block_time) AS hour,
    token_bought_symbol,
    FIRST_VALUE(amount_usd / NULLIF(token_bought_amount, 0)) OVER(
        PARTITION BY DATE_TRUNC('hour', block_time), token_bought_symbol
        ORDER BY block_time
    ) AS open_price,
    LAST_VALUE(amount_usd / NULLIF(token_bought_amount, 0)) OVER(
        PARTITION BY DATE_TRUNC('hour', block_time), token_bought_symbol
        ORDER BY block_time
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS close_price
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
    AND token_bought_symbol = 'WETH'
    AND token_bought_amount > 0
ORDER BY hour DESC
```

### 33. Compare current to first value in group
```sql
SELECT 
    tx_hash,
    block_time,
    amount_usd,
    FIRST_VALUE(amount_usd) OVER(
        PARTITION BY taker 
        ORDER BY block_time
    ) AS first_trade_amount,
    amount_usd - FIRST_VALUE(amount_usd) OVER(
        PARTITION BY taker 
        ORDER BY block_time
    ) AS diff_from_first
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '7' DAY
    AND blockchain = 'ethereum'
ORDER BY block_time DESC
LIMIT 100
```

### 34. First and last gas price of each block
```sql
SELECT DISTINCT
    block_number,
    FIRST_VALUE(gas_price / 1e9) OVER(
        PARTITION BY block_number 
        ORDER BY nonce
    ) AS first_gas_gwei,
    LAST_VALUE(gas_price / 1e9) OVER(
        PARTITION BY block_number 
        ORDER BY nonce
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_gas_gwei
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_number DESC
LIMIT 50
```

### 35. Track deviation from initial trade
```sql
WITH wallet_trades AS (
    SELECT 
        taker,
        block_time,
        amount_usd,
        FIRST_VALUE(amount_usd) OVER(
            PARTITION BY taker 
            ORDER BY block_time
        ) AS initial_trade
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '7' DAY
        AND blockchain = 'ethereum'
)
SELECT 
    taker,
    block_time,
    amount_usd,
    initial_trade,
    ROUND(100.0 * (amount_usd - initial_trade) / NULLIF(initial_trade, 0), 2) AS pct_change
FROM wallet_trades
ORDER BY block_time DESC
LIMIT 100
```

---

## NTILE for Distribution Analysis

### 36. Quartile analysis of trade sizes
```sql
SELECT 
    tx_hash,
    amount_usd,
    NTILE(4) OVER(ORDER BY amount_usd) AS quartile
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
    AND amount_usd > 0
ORDER BY amount_usd DESC
LIMIT 100
```

### 37. Decile ranking of wallets by volume
```sql
SELECT 
    taker,
    SUM(amount_usd) AS total_volume,
    NTILE(10) OVER(ORDER BY SUM(amount_usd)) AS decile
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '7' DAY
    AND blockchain = 'ethereum'
GROUP BY 1
ORDER BY total_volume DESC
LIMIT 100
```

### 38. Percentile buckets for gas prices
```sql
SELECT 
    hash,
    gas_price / 1e9 AS gas_gwei,
    NTILE(100) OVER(ORDER BY gas_price) AS gas_percentile
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY gas_price DESC
LIMIT 100
```

### 39. Equal distribution of users into cohorts
```sql
WITH user_volumes AS (
    SELECT 
        taker,
        SUM(amount_usd) AS total_volume,
        COUNT(*) AS trade_count
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '30' DAY
        AND blockchain = 'ethereum'
    GROUP BY 1
)
SELECT 
    taker,
    total_volume,
    trade_count,
    NTILE(5) OVER(ORDER BY total_volume DESC) AS volume_tier,
    CASE NTILE(5) OVER(ORDER BY total_volume DESC)
        WHEN 1 THEN 'Whale'
        WHEN 2 THEN 'Large'
        WHEN 3 THEN 'Medium'
        WHEN 4 THEN 'Small'
        WHEN 5 THEN 'Retail'
    END AS tier_name
FROM user_volumes
ORDER BY total_volume DESC
LIMIT 100
```

### 40. Time-based distribution into periods
```sql
SELECT 
    block_time,
    tx_hash,
    amount_usd,
    NTILE(24) OVER(ORDER BY block_time) AS hour_bucket
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND blockchain = 'ethereum'
ORDER BY block_time DESC
LIMIT 100
```

---

## Complex Window Patterns

### 41. Running distinct count (approximate)
```sql
WITH daily_users AS (
    SELECT DISTINCT 
        CAST(block_time AS DATE) AS day,
        taker
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '30' DAY
        AND blockchain = 'ethereum'
)
SELECT 
    day,
    COUNT(DISTINCT taker) AS daily_unique,
    SUM(COUNT(DISTINCT taker)) OVER(ORDER BY day) AS cumulative_unique_approx
FROM daily_users
GROUP BY day
ORDER BY day DESC
```

### 42. Gap and island detection
```sql
WITH numbered AS (
    SELECT 
        nonce,
        block_time,
        nonce - ROW_NUMBER() OVER(ORDER BY nonce) AS grp
    FROM ethereum.transactions
    WHERE "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
        AND block_time >= NOW() - INTERVAL '90' DAY
)
SELECT 
    MIN(nonce) AS island_start,
    MAX(nonce) AS island_end,
    COUNT(*) AS consecutive_txs,
    MIN(block_time) AS start_time,
    MAX(block_time) AS end_time
FROM numbered
GROUP BY grp
ORDER BY island_start DESC
LIMIT 20
```

### 43. Detect whale accumulation patterns
```sql
WITH transfers AS (
    SELECT 
        "to" AS wallet,
        block_time,
        value / 1e18 AS eth_value,
        SUM(value / 1e18) OVER(
            PARTITION BY "to" 
            ORDER BY block_time 
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS cumulative_received
    FROM ethereum.transactions
    WHERE value > 0
        AND block_time >= NOW() - INTERVAL '7' DAY
)
SELECT 
    wallet,
    block_time,
    eth_value,
    cumulative_received,
    CASE 
        WHEN cumulative_received > 1000 THEN 'WHALE'
        WHEN cumulative_received > 100 THEN 'LARGE'
        ELSE 'REGULAR'
    END AS wallet_tier
FROM transfers
WHERE cumulative_received > 100
ORDER BY block_time DESC
LIMIT 100
```

### 44. Multi-window comparison
```sql
SELECT 
    CAST(block_time AS DATE) AS day,
    SUM(amount_usd) AS volume,
    AVG(SUM(amount_usd)) OVER(ORDER BY CAST(block_time AS DATE) ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma_7d,
    AVG(SUM(amount_usd)) OVER(ORDER BY CAST(block_time AS DATE) ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) AS ma_30d,
    CASE 
        WHEN AVG(SUM(amount_usd)) OVER(ORDER BY CAST(block_time AS DATE) ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) >
             AVG(SUM(amount_usd)) OVER(ORDER BY CAST(block_time AS DATE) ROWS BETWEEN 29 PRECEDING AND CURRENT ROW)
        THEN 'BULLISH' ELSE 'BEARISH'
    END AS trend
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '60' DAY
    AND blockchain = 'ethereum'
GROUP BY 1
ORDER BY 1 DESC
LIMIT 30
```

### 45. Session-based analytics
```sql
WITH tx_sessions AS (
    SELECT 
        hash,
        block_time,
        value / 1e18 AS eth_value,
        SUM(CASE 
            WHEN DATE_DIFF('minute', LAG(block_time) OVER(ORDER BY block_time), block_time) > 60 
            OR LAG(block_time) OVER(ORDER BY block_time) IS NULL
            THEN 1 ELSE 0 
        END) OVER(ORDER BY block_time) AS session_id
    FROM ethereum.transactions
    WHERE "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
        AND block_time >= NOW() - INTERVAL '30' DAY
)
SELECT 
    session_id,
    COUNT(*) AS tx_count,
    SUM(eth_value) AS total_eth,
    MIN(block_time) AS session_start,
    MAX(block_time) AS session_end
FROM tx_sessions
GROUP BY session_id
ORDER BY session_id DESC
LIMIT 20
```

### 46. Window frame exclusion
```sql
SELECT 
    hash,
    block_time,
    gas_price / 1e9 AS gas_gwei,
    AVG(gas_price / 1e9) OVER(
        ORDER BY block_time 
        ROWS BETWEEN 5 PRECEDING AND 5 FOLLOWING
    ) AS local_avg,
    gas_price / 1e9 - AVG(gas_price / 1e9) OVER(
        ORDER BY block_time 
        ROWS BETWEEN 5 PRECEDING AND 5 FOLLOWING
    ) AS deviation
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 50
```

### 47. Cohort retention analysis
```sql
WITH first_trade AS (
    SELECT 
        taker,
        DATE_TRUNC('week', MIN(block_time)) AS cohort_week
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '90' DAY
        AND blockchain = 'ethereum'
    GROUP BY 1
),
weekly_activity AS (
    SELECT 
        t.taker,
        f.cohort_week,
        DATE_TRUNC('week', t.block_time) AS activity_week
    FROM dex.trades t
    JOIN first_trade f ON t.taker = f.taker
    WHERE t.block_time >= NOW() - INTERVAL '90' DAY
        AND t.blockchain = 'ethereum'
)
SELECT 
    cohort_week,
    COUNT(DISTINCT taker) AS cohort_size,
    COUNT(DISTINCT CASE WHEN activity_week = cohort_week THEN taker END) AS week_0,
    COUNT(DISTINCT CASE WHEN activity_week = cohort_week + INTERVAL '1' WEEK THEN taker END) AS week_1,
    COUNT(DISTINCT CASE WHEN activity_week = cohort_week + INTERVAL '2' WEEK THEN taker END) AS week_2
FROM weekly_activity
GROUP BY cohort_week
ORDER BY cohort_week DESC
LIMIT 12
```

### 48. Market share over time
```sql
WITH protocol_daily AS (
    SELECT 
        CAST(block_time AS DATE) AS day,
        project,
        SUM(amount_usd) AS volume
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '30' DAY
        AND blockchain = 'ethereum'
    GROUP BY 1, 2
)
SELECT 
    day,
    project,
    volume,
    SUM(volume) OVER(PARTITION BY day) AS total_daily_volume,
    ROUND(100.0 * volume / SUM(volume) OVER(PARTITION BY day), 2) AS market_share_pct
FROM protocol_daily
ORDER BY day DESC, volume DESC
LIMIT 100
```

### 49. Identify streak patterns
```sql
WITH daily_activity AS (
    SELECT 
        "from" AS wallet,
        CAST(block_time AS DATE) AS day,
        1 AS active
    FROM ethereum.transactions
    WHERE "from" = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
        AND block_time >= NOW() - INTERVAL '90' DAY
    GROUP BY 1, 2
),
streak AS (
    SELECT 
        wallet,
        day,
        day - CAST(ROW_NUMBER() OVER(PARTITION BY wallet ORDER BY day) AS INTEGER) * INTERVAL '1' DAY AS streak_group
    FROM daily_activity
)
SELECT 
    wallet,
    MIN(day) AS streak_start,
    MAX(day) AS streak_end,
    COUNT(*) AS streak_days
FROM streak
GROUP BY wallet, streak_group
HAVING COUNT(*) >= 3
ORDER BY streak_days DESC
LIMIT 20
```

### 50. Weighted moving average
```sql
WITH daily_volume AS (
    SELECT 
        CAST(block_time AS DATE) AS day,
        SUM(amount_usd) AS volume
    FROM dex.trades
    WHERE block_time >= NOW() - INTERVAL '30' DAY
        AND blockchain = 'ethereum'
    GROUP BY 1
),
weighted AS (
    SELECT 
        day,
        volume,
        ROW_NUMBER() OVER(ORDER BY day DESC) AS recency_rank
    FROM daily_volume
)
SELECT 
    day,
    volume,
    SUM(volume * (8 - LEAST(recency_rank, 7))) OVER(
        ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) / SUM(8 - LEAST(recency_rank, 7)) OVER(
        ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS weighted_ma_7d
FROM weighted
WHERE recency_rank <= 30
ORDER BY day DESC
```

---

*Last updated: March 2026*
