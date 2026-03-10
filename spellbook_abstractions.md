# Spellbook Abstractions

**50 production-ready Dune SQL queries for Dune Spellbook tables**

**Covers:** `tokens.erc20`, `prices.usd`, `dex.trades`, `nft.trades`, `labels`, `balances`, and more

---

## tokens.erc20

### 1. Get token metadata
```sql
SELECT 
    contract_address,
    symbol,
    decimals,
    blockchain
FROM tokens.erc20
WHERE blockchain = 'ethereum'
    AND symbol = 'USDC'
```

### 2. Search tokens by symbol pattern
```sql
SELECT 
    contract_address,
    symbol,
    decimals,
    blockchain
FROM tokens.erc20
WHERE blockchain = 'ethereum'
    AND symbol LIKE '%ETH%'
LIMIT 50
```

### 3. All stablecoins on Ethereum
```sql
SELECT 
    contract_address,
    symbol,
    decimals
FROM tokens.erc20
WHERE blockchain = 'ethereum'
    AND symbol IN ('USDC', 'USDT', 'DAI', 'FRAX', 'TUSD', 'BUSD', 'USDP', 'GUSD')
```

### 4. Tokens with 18 decimals
```sql
SELECT 
    contract_address,
    symbol
FROM tokens.erc20
WHERE blockchain = 'ethereum'
    AND decimals = 18
LIMIT 100
```

### 5. Cross-chain token lookup
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

### 6. Token count by blockchain
```sql
SELECT 
    blockchain,
    COUNT(*) AS token_count
FROM tokens.erc20
GROUP BY 1
ORDER BY 2 DESC
```

### 7. Recently added tokens (by address pattern)
```sql
SELECT 
    contract_address,
    symbol,
    decimals,
    blockchain
FROM tokens.erc20
WHERE blockchain = 'ethereum'
ORDER BY contract_address DESC
LIMIT 50
```

### 8. Tokens with unusual decimals
```sql
SELECT 
    contract_address,
    symbol,
    decimals
FROM tokens.erc20
WHERE blockchain = 'ethereum'
    AND decimals NOT IN (6, 8, 18)
LIMIT 50
```

### 9. WETH addresses across chains
```sql
SELECT 
    blockchain,
    contract_address,
    symbol
FROM tokens.erc20
WHERE symbol IN ('WETH', 'wETH')
ORDER BY blockchain
```

### 10. Native wrapped token addresses
```sql
SELECT 
    blockchain,
    contract_address,
    symbol
FROM tokens.erc20
WHERE symbol IN ('WETH', 'WMATIC', 'WAVAX', 'WBNB', 'WFTM', 'WSOL')
ORDER BY blockchain
```

---

## prices.usd

### 11. Current token prices
```sql
SELECT 
    blockchain,
    contract_address,
    symbol,
    price,
    minute AS price_time
FROM prices.usd
WHERE blockchain = 'ethereum'
    AND symbol = 'WETH'
ORDER BY minute DESC
LIMIT 1
```

### 12. Historical price (specific token, time range)
```sql
SELECT 
    minute,
    price
FROM prices.usd
WHERE blockchain = 'ethereum'
    AND contract_address = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 -- WETH
    AND minute >= NOW() - INTERVAL '24' HOUR
ORDER BY minute DESC
```

### 13. OHLC price data (hourly)
```sql
SELECT 
    DATE_TRUNC('hour', minute) AS hour,
    MIN(price) AS low,
    MAX(price) AS high,
    (ARRAY_AGG(price ORDER BY minute ASC))[1] AS open,
    (ARRAY_AGG(price ORDER BY minute DESC))[1] AS close
FROM prices.usd
WHERE blockchain = 'ethereum'
    AND symbol = 'WETH'
    AND minute >= NOW() - INTERVAL '7' DAY
GROUP BY 1
ORDER BY 1 DESC
```

### 14. Price at specific timestamp
```sql
SELECT 
    price,
    minute
FROM prices.usd
WHERE blockchain = 'ethereum'
    AND contract_address = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2
    AND minute <= TIMESTAMP '2024-01-01 12:00:00'
ORDER BY minute DESC
LIMIT 1
```

### 15. Multiple token prices comparison
```sql
SELECT 
    symbol,
    price,
    minute
FROM prices.usd
WHERE blockchain = 'ethereum'
    AND symbol IN ('WETH', 'WBTC', 'LINK', 'UNI', 'AAVE')
    AND minute >= NOW() - INTERVAL '1' HOUR
ORDER BY minute DESC
LIMIT 50
```

### 16. Token price volatility
```sql
SELECT 
    DATE_TRUNC('day', minute) AS day,
    MIN(price) AS daily_low,
    MAX(price) AS daily_high,
    (MAX(price) - MIN(price)) / MIN(price) * 100 AS volatility_pct
FROM prices.usd
WHERE blockchain = 'ethereum'
    AND symbol = 'WETH'
    AND minute >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 1 DESC
```

### 17. Stablecoin peg check
```sql
SELECT 
    symbol,
    AVG(price) AS avg_price,
    MIN(price) AS min_price,
    MAX(price) AS max_price,
    STDDEV(price) AS price_stddev
FROM prices.usd
WHERE blockchain = 'ethereum'
    AND symbol IN ('USDC', 'USDT', 'DAI')
    AND minute >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
```

### 18. Price feed availability
```sql
SELECT 
    blockchain,
    symbol,
    COUNT(*) AS data_points,
    MIN(minute) AS earliest_price,
    MAX(minute) AS latest_price
FROM prices.usd
WHERE symbol = 'WETH'
GROUP BY 1, 2
ORDER BY 1
```

### 19. Token price in ETH terms
```sql
WITH eth_price AS (
    SELECT price AS eth_usd, minute
    FROM prices.usd
    WHERE blockchain = 'ethereum'
        AND symbol = 'WETH'
        AND minute >= NOW() - INTERVAL '24' HOUR
),
token_price AS (
    SELECT price AS token_usd, minute
    FROM prices.usd
    WHERE blockchain = 'ethereum'
        AND symbol = 'LINK'
        AND minute >= NOW() - INTERVAL '24' HOUR
)
SELECT 
    t.minute,
    t.token_usd,
    e.eth_usd,
    t.token_usd / e.eth_usd AS link_eth_price
FROM token_price t
JOIN eth_price e ON DATE_TRUNC('minute', t.minute) = DATE_TRUNC('minute', e.minute)
ORDER BY t.minute DESC
LIMIT 100
```

### 20. BTC/ETH ratio over time
```sql
WITH btc AS (
    SELECT minute, price AS btc_price
    FROM prices.usd
    WHERE blockchain = 'ethereum'
        AND symbol = 'WBTC'
        AND minute >= NOW() - INTERVAL '30' DAY
),
eth AS (
    SELECT minute, price AS eth_price
    FROM prices.usd
    WHERE blockchain = 'ethereum'
        AND symbol = 'WETH'
        AND minute >= NOW() - INTERVAL '30' DAY
)
SELECT 
    DATE_TRUNC('day', b.minute) AS day,
    AVG(b.btc_price / e.eth_price) AS btc_eth_ratio
FROM btc b
JOIN eth e ON DATE_TRUNC('minute', b.minute) = DATE_TRUNC('minute', e.minute)
GROUP BY 1
ORDER BY 1
```

---

## dex.trades (Spellbook)

### 21. DEX volume by chain (24h)
```sql
SELECT 
    blockchain,
    SUM(amount_usd) AS total_volume_usd,
    COUNT(*) AS trade_count
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 2 DESC
```

### 22. Top trading pairs
```sql
SELECT 
    token_pair,
    COUNT(*) AS trade_count,
    SUM(amount_usd) AS total_volume_usd
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 3 DESC
LIMIT 20
```

### 23. Average trade size by DEX
```sql
SELECT 
    project,
    AVG(amount_usd) AS avg_trade_size,
    APPROX_PERCENTILE(amount_usd, 0.5) AS median_trade_size
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '24' HOUR
    AND amount_usd > 0
GROUP BY 1
ORDER BY 2 DESC
```

### 24. Trading hours analysis
```sql
SELECT 
    HOUR(block_time) AS hour_utc,
    SUM(amount_usd) AS volume_usd,
    COUNT(*) AS trade_count
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '7' DAY
GROUP BY 1
ORDER BY 1
```

### 25. Token buy/sell pressure
```sql
SELECT 
    token_bought_address,
    token_bought_symbol,
    SUM(amount_usd) AS total_bought_usd,
    COUNT(*) AS buy_count
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '24' HOUR
    AND token_bought_symbol IS NOT NULL
GROUP BY 1, 2
ORDER BY 3 DESC
LIMIT 20
```

### 26. DEX market share trend
```sql
SELECT 
    DATE_TRUNC('day', block_time) AS day,
    project,
    SUM(amount_usd) AS volume_usd
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC
```

### 27. Slippage estimation (price impact)
```sql
SELECT 
    tx_hash,
    project,
    token_bought_symbol,
    token_sold_symbol,
    amount_usd,
    CASE 
        WHEN amount_usd < 1000 THEN 'Small (<$1k)'
        WHEN amount_usd < 10000 THEN 'Medium ($1k-$10k)'
        WHEN amount_usd < 100000 THEN 'Large ($10k-$100k)'
        ELSE 'Whale (>$100k)'
    END AS trade_size_bucket
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY amount_usd DESC
LIMIT 100
```

### 28. New token trading activity
```sql
SELECT 
    token_bought_address,
    token_bought_symbol,
    MIN(block_time) AS first_trade,
    COUNT(*) AS trade_count,
    SUM(amount_usd) AS total_volume
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '7' DAY
GROUP BY 1, 2
HAVING MIN(block_time) >= NOW() - INTERVAL '24' HOUR
ORDER BY 5 DESC
LIMIT 50
```

### 29. Cross-DEX arbitrage opportunities
```sql
WITH latest_trades AS (
    SELECT 
        token_bought_address,
        token_sold_address,
        project,
        amount_usd / token_bought_amount AS price,
        block_time
    FROM dex.trades
    WHERE blockchain = 'ethereum'
        AND block_time >= NOW() - INTERVAL '5' MINUTE
        AND token_bought_amount > 0
)
SELECT 
    token_bought_address,
    MIN(price) AS min_price,
    MAX(price) AS max_price,
    (MAX(price) - MIN(price)) / MIN(price) * 100 AS spread_pct
FROM latest_trades
GROUP BY 1
HAVING COUNT(DISTINCT project) > 1
ORDER BY 4 DESC
LIMIT 20
```

### 30. Trader PnL estimation
```sql
WITH buys AS (
    SELECT 
        taker AS trader,
        token_bought_address AS token,
        SUM(token_bought_amount) AS total_bought,
        SUM(amount_usd) AS total_cost_usd
    FROM dex.trades
    WHERE blockchain = 'ethereum'
        AND block_time >= NOW() - INTERVAL '7' DAY
    GROUP BY 1, 2
),
sells AS (
    SELECT 
        taker AS trader,
        token_sold_address AS token,
        SUM(token_sold_amount) AS total_sold,
        SUM(amount_usd) AS total_revenue_usd
    FROM dex.trades
    WHERE blockchain = 'ethereum'
        AND block_time >= NOW() - INTERVAL '7' DAY
    GROUP BY 1, 2
)
SELECT 
    b.trader,
    b.token,
    b.total_cost_usd,
    COALESCE(s.total_revenue_usd, 0) AS total_revenue_usd,
    COALESCE(s.total_revenue_usd, 0) - b.total_cost_usd AS estimated_pnl
FROM buys b
LEFT JOIN sells s ON b.trader = s.trader AND b.token = s.token
WHERE b.total_cost_usd > 1000
ORDER BY 5 DESC
LIMIT 50
```

---

## nft.trades (Spellbook)

### 31. NFT market overview
```sql
SELECT 
    blockchain,
    project,
    SUM(amount_usd) AS total_volume_usd,
    COUNT(*) AS sales_count,
    COUNT(DISTINCT buyer) AS unique_buyers
FROM nft.trades
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1, 2
ORDER BY 3 DESC
```

### 32. Collection ranking
```sql
SELECT 
    nft_contract_address,
    COUNT(*) AS sales,
    SUM(amount_usd) AS total_volume_usd,
    AVG(amount_usd) AS avg_price_usd,
    MIN(amount_usd) AS floor_price_usd
FROM nft.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 3 DESC
LIMIT 20
```

### 33. NFT flipper profits
```sql
WITH buys AS (
    SELECT 
        buyer,
        nft_contract_address,
        token_id,
        amount_usd AS buy_price,
        block_time AS buy_time
    FROM nft.trades
    WHERE blockchain = 'ethereum'
        AND block_time >= NOW() - INTERVAL '30' DAY
),
sells AS (
    SELECT 
        seller,
        nft_contract_address,
        token_id,
        amount_usd AS sell_price,
        block_time AS sell_time
    FROM nft.trades
    WHERE blockchain = 'ethereum'
        AND block_time >= NOW() - INTERVAL '30' DAY
)
SELECT 
    b.buyer AS flipper,
    b.nft_contract_address,
    b.token_id,
    b.buy_price,
    s.sell_price,
    s.sell_price - b.buy_price AS profit_usd,
    DATE_DIFF('hour', b.buy_time, s.sell_time) AS hold_hours
FROM buys b
JOIN sells s ON b.buyer = s.seller 
    AND b.nft_contract_address = s.nft_contract_address
    AND b.token_id = s.token_id
    AND s.sell_time > b.buy_time
WHERE s.sell_price > b.buy_price
ORDER BY 6 DESC
LIMIT 100
```

### 34. Marketplace fee comparison
```sql
SELECT 
    project,
    SUM(platform_fee_amount) AS total_fees_eth,
    SUM(royalty_fee_amount) AS total_royalties_eth,
    COUNT(*) AS trade_count
FROM nft.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '7' DAY
GROUP BY 1
ORDER BY 2 DESC
```

### 35. NFT holder distribution
```sql
SELECT 
    buyer AS current_holder,
    COUNT(DISTINCT nft_contract_address || CAST(token_id AS VARCHAR)) AS unique_nfts
FROM nft.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 2 DESC
LIMIT 50
```

---

## Labels

### 36. Known wallet labels
```sql
SELECT 
    address,
    name,
    category,
    blockchain
FROM labels.all
WHERE blockchain = 'ethereum'
    AND category = 'cex'
LIMIT 50
```

### 37. CEX deposit addresses
```sql
SELECT 
    address,
    name
FROM labels.all
WHERE blockchain = 'ethereum'
    AND name LIKE '%Binance%'
LIMIT 100
```

### 38. DEX contract labels
```sql
SELECT 
    address,
    name,
    category
FROM labels.all
WHERE blockchain = 'ethereum'
    AND category = 'dex'
ORDER BY name
LIMIT 100
```

### 39. Bridge contract labels
```sql
SELECT 
    address,
    name,
    category
FROM labels.all
WHERE blockchain = 'ethereum'
    AND (category = 'bridge' OR name LIKE '%Bridge%')
LIMIT 50
```

### 40. Label a transaction's participants
```sql
WITH tx_addresses AS (
    SELECT "from" AS address FROM ethereum.transactions WHERE hash = 0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
    UNION
    SELECT "to" AS address FROM ethereum.transactions WHERE hash = 0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
)
SELECT 
    t.address,
    l.name,
    l.category
FROM tx_addresses t
LEFT JOIN labels.all l ON t.address = l.address AND l.blockchain = 'ethereum'
```

---

## Balances and Advanced Queries

### 41. Token holder balances
```sql
SELECT 
    address,
    SUM(amount) AS balance
FROM tokens.transfers
WHERE blockchain = 'ethereum'
    AND contract_address = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 -- USDC
GROUP BY 1
HAVING SUM(amount) > 0
ORDER BY 2 DESC
LIMIT 100
```

### 42. Daily active wallets
```sql
SELECT 
    DATE_TRUNC('day', block_time) AS day,
    COUNT(DISTINCT taker) AS unique_traders
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 1
```

### 43. Token transfer volume
```sql
SELECT 
    DATE_TRUNC('day', block_time) AS day,
    token_address,
    COUNT(*) AS transfer_count,
    SUM(amount_usd) AS volume_usd
FROM tokens.transfers
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '7' DAY
    AND token_address = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
GROUP BY 1, 2
ORDER BY 1 DESC
```

### 44. Protocol TVL estimation
```sql
SELECT 
    DATE_TRUNC('day', block_time) AS day,
    SUM(CASE WHEN "to" = 0x7d2768dE32b0b80b7a3454c06BdAc94A69DDc7A9 THEN amount_usd ELSE 0 END) AS deposits,
    SUM(CASE WHEN "from" = 0x7d2768dE32b0b80b7a3454c06BdAc94A69DDc7A9 THEN amount_usd ELSE 0 END) AS withdrawals
FROM tokens.transfers
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 1
```

### 45. Cross-chain volume comparison
```sql
SELECT 
    blockchain,
    DATE_TRUNC('day', block_time) AS day,
    SUM(amount_usd) AS dex_volume_usd
FROM dex.trades
WHERE block_time >= NOW() - INTERVAL '7' DAY
GROUP BY 1, 2
ORDER BY 1, 2
```

### 46. Whale wallet tracking
```sql
SELECT 
    "from" AS whale_address,
    COUNT(*) AS tx_count,
    SUM(amount_usd) AS total_volume_usd
FROM tokens.transfers
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '24' HOUR
    AND amount_usd > 100000
GROUP BY 1
ORDER BY 3 DESC
LIMIT 20
```

### 47. Smart contract interaction frequency
```sql
SELECT 
    "to" AS contract_address,
    l.name,
    COUNT(*) AS interaction_count
FROM ethereum.transactions t
LEFT JOIN labels.all l ON t."to" = l.address AND l.blockchain = 'ethereum'
WHERE t.block_time >= NOW() - INTERVAL '24' HOUR
    AND t."to" IS NOT NULL
GROUP BY 1, 2
ORDER BY 3 DESC
LIMIT 50
```

### 48. Gas token correlation
```sql
WITH gas_data AS (
    SELECT 
        DATE_TRUNC('hour', block_time) AS hour,
        AVG(gas_price) / 1e9 AS avg_gas_gwei
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '7' DAY
    GROUP BY 1
),
price_data AS (
    SELECT 
        DATE_TRUNC('hour', minute) AS hour,
        AVG(price) AS eth_price
    FROM prices.usd
    WHERE blockchain = 'ethereum'
        AND symbol = 'WETH'
        AND minute >= NOW() - INTERVAL '7' DAY
    GROUP BY 1
)
SELECT 
    g.hour,
    g.avg_gas_gwei,
    p.eth_price
FROM gas_data g
JOIN price_data p ON g.hour = p.hour
ORDER BY 1 DESC
```

### 49. MEV bundle detection
```sql
SELECT 
    block_number,
    COUNT(*) AS tx_count,
    COUNT(DISTINCT "from") AS unique_senders,
    MAX(gas_price) / MIN(gas_price) AS gas_price_ratio
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
GROUP BY 1
HAVING MAX(gas_price) / MIN(gas_price) > 10
ORDER BY 4 DESC
LIMIT 20
```

### 50. Complete market overview dashboard
```sql
WITH dex_stats AS (
    SELECT 
        SUM(amount_usd) AS dex_volume_24h,
        COUNT(*) AS dex_trades_24h
    FROM dex.trades
    WHERE blockchain = 'ethereum'
        AND block_time >= NOW() - INTERVAL '24' HOUR
),
nft_stats AS (
    SELECT 
        SUM(amount_usd) AS nft_volume_24h,
        COUNT(*) AS nft_sales_24h
    FROM nft.trades
    WHERE blockchain = 'ethereum'
        AND block_time >= NOW() - INTERVAL '24' HOUR
),
gas_stats AS (
    SELECT 
        AVG(gas_price) / 1e9 AS avg_gas_gwei,
        COUNT(*) AS total_txs
    FROM ethereum.transactions
    WHERE block_time >= NOW() - INTERVAL '24' HOUR
)
SELECT 
    d.dex_volume_24h,
    d.dex_trades_24h,
    n.nft_volume_24h,
    n.nft_sales_24h,
    g.avg_gas_gwei,
    g.total_txs
FROM dex_stats d, nft_stats n, gas_stats g
```

---

**Footer**  
Built by [CryptoAI-Jedi](https://github.com/CryptoAI-Jedi) | [Dune SQL Cheatsheet](https://github.com/CryptoAI-Jedi/dune-sql-cheatsheet)
