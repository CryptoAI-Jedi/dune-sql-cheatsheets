# Decoded Contracts

**50 production-ready Dune SQL queries for decoded contract events**

**Covers:** ERC20 transfers, DEX trades (`dex.trades`), NFT trades (`nft.trades`), Aave, Uniswap, Compound decoded events

---

## ERC20 Token Events

### 1. USDC transfers (last hour)
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    "from",
    "to",
    value / 1e6 AS usdc_amount
FROM erc20_ethereum.evt_Transfer
WHERE contract_address = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 -- USDC
    AND evt_block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY evt_block_time DESC
LIMIT 100
```

### 2. WETH transfers
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    "from",
    "to",
    value / 1e18 AS weth_amount
FROM erc20_ethereum.evt_Transfer
WHERE contract_address = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 -- WETH
    AND evt_block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY evt_block_time DESC
LIMIT 100
```

### 3. Large USDT transfers (> $100k)
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    "from",
    "to",
    value / 1e6 AS usdt_amount
FROM erc20_ethereum.evt_Transfer
WHERE contract_address = 0xdAC17F958D2ee523a2206206994597C13D831ec7 -- USDT
    AND value > 100000 * 1e6
    AND evt_block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY value DESC
LIMIT 100
```

### 4. Top token senders
```sql
SELECT 
    "from" AS sender,
    COUNT(*) AS transfer_count,
    SUM(value) / 1e6 AS total_usdc
FROM erc20_ethereum.evt_Transfer
WHERE contract_address = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
    AND evt_block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 3 DESC
LIMIT 50
```

### 5. Token approval events
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    owner,
    spender,
    value / 1e18 AS approved_amount
FROM erc20_ethereum.evt_Approval
WHERE contract_address = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 -- WETH
    AND evt_block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY evt_block_time DESC
LIMIT 100
```

### 6. Unlimited approvals (potential risk)
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    owner,
    spender,
    contract_address AS token
FROM erc20_ethereum.evt_Approval
WHERE value = CAST(POWER(2, 256) - 1 AS UINT256) -- Max uint256
    AND evt_block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY evt_block_time DESC
LIMIT 100
```

### 7. DAI transfers volume by hour
```sql
SELECT 
    DATE_TRUNC('hour', evt_block_time) AS hour,
    COUNT(*) AS transfer_count,
    SUM(value) / 1e18 AS total_dai
FROM erc20_ethereum.evt_Transfer
WHERE contract_address = 0x6B175474E89094C44Da98b954EescdeCB5BE3830 -- DAI
    AND evt_block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 1 DESC
```

### 8. LINK token transfers
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    "from",
    "to",
    value / 1e18 AS link_amount
FROM erc20_ethereum.evt_Transfer
WHERE contract_address = 0x514910771AF9Ca656af840dff83E8264EcF986CA -- LINK
    AND evt_block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY evt_block_time DESC
LIMIT 100
```

### 9. Stablecoin flow comparison
```sql
SELECT 
    contract_address,
    CASE contract_address
        WHEN 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 THEN 'USDC'
        WHEN 0xdAC17F958D2ee523a2206206994597C13D831ec7 THEN 'USDT'
        WHEN 0x6B175474E89094C44Da98b954EedfcdeCB5BE3830 THEN 'DAI'
    END AS stablecoin,
    COUNT(*) AS transfer_count
FROM erc20_ethereum.evt_Transfer
WHERE contract_address IN (
    0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48,
    0xdAC17F958D2ee523a2206206994597C13D831ec7,
    0x6B175474E89094C44Da98b954EedfcdeCB5BE3830
)
    AND evt_block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 3 DESC
```

### 10. Zero-value transfers (potential airdrops/dust)
```sql
SELECT 
    evt_block_time,
    contract_address,
    "from",
    "to"
FROM erc20_ethereum.evt_Transfer
WHERE value = 0
    AND evt_block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY evt_block_time DESC
LIMIT 100
```

---

## DEX Trades (dex.trades)

### 11. Recent DEX trades
```sql
SELECT 
    block_time,
    tx_hash,
    project,
    version,
    token_bought_symbol,
    token_sold_symbol,
    amount_usd
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 12. Top DEX by volume (24h)
```sql
SELECT 
    project,
    COUNT(*) AS trade_count,
    SUM(amount_usd) AS total_volume_usd
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 3 DESC
```

### 13. WETH-USDC trades
```sql
SELECT 
    block_time,
    tx_hash,
    project,
    token_bought_amount,
    token_bought_symbol,
    token_sold_amount,
    token_sold_symbol,
    amount_usd
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND (
        (token_bought_address = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 
         AND token_sold_address = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48)
        OR 
        (token_sold_address = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 
         AND token_bought_address = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48)
    )
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 14. Whale trades (> $100k)
```sql
SELECT 
    block_time,
    tx_hash,
    project,
    taker,
    token_bought_symbol,
    token_sold_symbol,
    amount_usd
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND amount_usd > 100000
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY amount_usd DESC
LIMIT 100
```

### 15. Uniswap V3 trades only
```sql
SELECT 
    block_time,
    tx_hash,
    token_pair,
    token_bought_amount,
    token_bought_symbol,
    token_sold_amount,
    token_sold_symbol,
    amount_usd
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND project = 'uniswap'
    AND version = '3'
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 16. Hourly DEX volume
```sql
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    project,
    SUM(amount_usd) AS volume_usd,
    COUNT(*) AS trade_count
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC
```

### 17. Trader activity analysis
```sql
SELECT 
    taker AS trader,
    COUNT(*) AS trade_count,
    SUM(amount_usd) AS total_volume_usd,
    AVG(amount_usd) AS avg_trade_size
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 3 DESC
LIMIT 50
```

### 18. Most traded tokens
```sql
SELECT 
    token_bought_symbol AS token,
    COUNT(*) AS buy_count,
    SUM(amount_usd) AS total_volume_usd
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '24' HOUR
    AND token_bought_symbol IS NOT NULL
GROUP BY 1
ORDER BY 3 DESC
LIMIT 20
```

### 19. DEX trades on Arbitrum
```sql
SELECT 
    block_time,
    tx_hash,
    project,
    token_bought_symbol,
    token_sold_symbol,
    amount_usd
FROM dex.trades
WHERE blockchain = 'arbitrum'
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 20. Cross-chain DEX comparison
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

---

## NFT Trades (nft.trades)

### 21. Recent NFT sales
```sql
SELECT 
    block_time,
    tx_hash,
    project,
    nft_contract_address,
    token_id,
    amount_usd,
    buyer,
    seller
FROM nft.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 22. Top NFT marketplaces by volume
```sql
SELECT 
    project,
    COUNT(*) AS trade_count,
    SUM(amount_usd) AS total_volume_usd
FROM nft.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 3 DESC
```

### 23. BAYC sales
```sql
SELECT 
    block_time,
    tx_hash,
    token_id,
    amount_usd,
    buyer,
    seller,
    project AS marketplace
FROM nft.trades
WHERE blockchain = 'ethereum'
    AND nft_contract_address = 0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D -- BAYC
    AND block_time >= NOW() - INTERVAL '30' DAY
ORDER BY block_time DESC
```

### 24. High-value NFT sales (> $10k)
```sql
SELECT 
    block_time,
    tx_hash,
    nft_contract_address,
    token_id,
    amount_usd,
    project
FROM nft.trades
WHERE blockchain = 'ethereum'
    AND amount_usd > 10000
    AND block_time >= NOW() - INTERVAL '7' DAY
ORDER BY amount_usd DESC
LIMIT 100
```

### 25. NFT buyer analysis
```sql
SELECT 
    buyer,
    COUNT(*) AS purchase_count,
    SUM(amount_usd) AS total_spent_usd,
    AVG(amount_usd) AS avg_purchase_price
FROM nft.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '7' DAY
GROUP BY 1
ORDER BY 3 DESC
LIMIT 50
```

### 26. OpenSea trades
```sql
SELECT 
    block_time,
    tx_hash,
    nft_contract_address,
    token_id,
    amount_usd,
    buyer,
    seller
FROM nft.trades
WHERE blockchain = 'ethereum'
    AND project = 'opensea'
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 27. Blur marketplace activity
```sql
SELECT 
    block_time,
    tx_hash,
    nft_contract_address,
    token_id,
    amount_usd
FROM nft.trades
WHERE blockchain = 'ethereum'
    AND project = 'blur'
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 28. Daily NFT volume trend
```sql
SELECT 
    DATE_TRUNC('day', block_time) AS day,
    SUM(amount_usd) AS total_volume_usd,
    COUNT(*) AS trade_count,
    COUNT(DISTINCT buyer) AS unique_buyers
FROM nft.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 1
```

### 29. Collection floor price tracking
```sql
SELECT 
    DATE_TRUNC('day', block_time) AS day,
    MIN(amount_usd) AS floor_price_usd,
    AVG(amount_usd) AS avg_price_usd,
    COUNT(*) AS sales
FROM nft.trades
WHERE blockchain = 'ethereum'
    AND nft_contract_address = 0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D -- BAYC
    AND block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 1
```

### 30. Wash trading detection (same buyer/seller)
```sql
SELECT 
    block_time,
    tx_hash,
    nft_contract_address,
    token_id,
    amount_usd,
    buyer,
    seller
FROM nft.trades
WHERE blockchain = 'ethereum'
    AND buyer = seller
    AND block_time >= NOW() - INTERVAL '7' DAY
ORDER BY block_time DESC
```

---

## Aave Decoded Events

### 31. Aave V3 deposits
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    reserve AS asset,
    "user",
    amount / 1e18 AS deposit_amount,
    onBehalfOf
FROM aave_v3_ethereum.Pool_evt_Supply
WHERE evt_block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY evt_block_time DESC
LIMIT 100
```

### 32. Aave V3 borrows
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    reserve AS asset,
    "user",
    amount / 1e18 AS borrow_amount,
    interestRateMode
FROM aave_v3_ethereum.Pool_evt_Borrow
WHERE evt_block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY evt_block_time DESC
LIMIT 100
```

### 33. Aave V3 liquidations
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    collateralAsset,
    debtAsset,
    "user",
    liquidatedCollateralAmount,
    liquidator
FROM aave_v3_ethereum.Pool_evt_LiquidationCall
WHERE evt_block_time >= NOW() - INTERVAL '7' DAY
ORDER BY evt_block_time DESC
LIMIT 100
```

### 34. Aave repayments
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    reserve AS asset,
    "user",
    amount / 1e18 AS repay_amount,
    useATokens
FROM aave_v3_ethereum.Pool_evt_Repay
WHERE evt_block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY evt_block_time DESC
LIMIT 100
```

### 35. Aave flash loans
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    asset,
    amount / 1e18 AS flash_loan_amount,
    premium / 1e18 AS fee_paid,
    initiator
FROM aave_v3_ethereum.Pool_evt_FlashLoan
WHERE evt_block_time >= NOW() - INTERVAL '7' DAY
ORDER BY evt_block_time DESC
LIMIT 100
```

---

## Uniswap Decoded Events

### 36. Uniswap V3 swaps
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    contract_address AS pool,
    sender,
    recipient,
    amount0,
    amount1,
    sqrtPriceX96
FROM uniswap_v3_ethereum.Pair_evt_Swap
WHERE evt_block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY evt_block_time DESC
LIMIT 100
```

### 37. Uniswap V3 liquidity additions
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    contract_address AS pool,
    owner,
    tickLower,
    tickUpper,
    amount AS liquidity,
    amount0,
    amount1
FROM uniswap_v3_ethereum.Pair_evt_Mint
WHERE evt_block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY evt_block_time DESC
LIMIT 100
```

### 38. Uniswap V3 liquidity removals
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    contract_address AS pool,
    owner,
    tickLower,
    tickUpper,
    amount AS liquidity,
    amount0,
    amount1
FROM uniswap_v3_ethereum.Pair_evt_Burn
WHERE evt_block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY evt_block_time DESC
LIMIT 100
```

### 39. Uniswap V2 swaps
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    contract_address AS pair,
    sender,
    amount0In,
    amount1In,
    amount0Out,
    amount1Out,
    "to"
FROM uniswap_v2_ethereum.Pair_evt_Swap
WHERE evt_block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY evt_block_time DESC
LIMIT 100
```

### 40. New Uniswap V3 pool creations
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    token0,
    token1,
    fee,
    pool
FROM uniswap_v3_ethereum.Factory_evt_PoolCreated
WHERE evt_block_time >= NOW() - INTERVAL '7' DAY
ORDER BY evt_block_time DESC
LIMIT 50
```

---

## Compound Decoded Events

### 41. Compound V3 supply events
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    "from",
    dst,
    amount
FROM compound_v3_ethereum.cUSDCv3_evt_Supply
WHERE evt_block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY evt_block_time DESC
LIMIT 100
```

### 42. Compound V3 withdraw events
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    src,
    "to",
    amount
FROM compound_v3_ethereum.cUSDCv3_evt_Withdraw
WHERE evt_block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY evt_block_time DESC
LIMIT 100
```

### 43. Compound liquidations
```sql
SELECT 
    evt_block_time,
    evt_tx_hash,
    absorber,
    borrower,
    asset,
    collateralAbsorbed
FROM compound_v3_ethereum.cUSDCv3_evt_AbsorbCollateral
WHERE evt_block_time >= NOW() - INTERVAL '30' DAY
ORDER BY evt_block_time DESC
LIMIT 100
```

---

## Cross-Protocol Analysis

### 44. DeFi activity by protocol
```sql
WITH aave_activity AS (
    SELECT 'Aave Supply' AS action, evt_block_time, evt_tx_hash
    FROM aave_v3_ethereum.Pool_evt_Supply
    WHERE evt_block_time >= NOW() - INTERVAL '24' HOUR
),
uniswap_activity AS (
    SELECT 'Uniswap Swap' AS action, evt_block_time, evt_tx_hash
    FROM uniswap_v3_ethereum.Pair_evt_Swap
    WHERE evt_block_time >= NOW() - INTERVAL '24' HOUR
)
SELECT action, COUNT(*) AS event_count
FROM (
    SELECT * FROM aave_activity
    UNION ALL
    SELECT * FROM uniswap_activity
)
GROUP BY 1
ORDER BY 2 DESC
```

### 45. Flash loan usage across protocols
```sql
SELECT 
    'Aave V3' AS protocol,
    COUNT(*) AS flash_loan_count,
    SUM(amount) / 1e18 AS total_volume
FROM aave_v3_ethereum.Pool_evt_FlashLoan
WHERE evt_block_time >= NOW() - INTERVAL '7' DAY
```

### 46. DEX aggregator routing
```sql
SELECT 
    block_time,
    tx_hash,
    project,
    token_bought_symbol,
    token_sold_symbol,
    amount_usd
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND tx_hash IN (
        SELECT DISTINCT tx_hash 
        FROM dex.trades
        WHERE blockchain = 'ethereum'
            AND block_time >= NOW() - INTERVAL '1' HOUR
        GROUP BY tx_hash
        HAVING COUNT(*) > 2 -- Multi-hop trades
    )
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY tx_hash, block_time
```

### 47. Arbitrage detection
```sql
SELECT 
    tx_hash,
    MIN(block_time) AS trade_time,
    COUNT(*) AS hop_count,
    ARRAY_AGG(project ORDER BY block_time) AS dex_path,
    ARRAY_AGG(token_bought_symbol ORDER BY block_time) AS tokens_bought
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND block_time >= NOW() - INTERVAL '1' HOUR
GROUP BY tx_hash
HAVING COUNT(*) >= 3
ORDER BY hop_count DESC
LIMIT 50
```

### 48. Stablecoin DEX volume
```sql
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    token_bought_symbol AS stablecoin,
    SUM(amount_usd) AS volume_usd
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND token_bought_symbol IN ('USDC', 'USDT', 'DAI', 'FRAX')
    AND block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC
```

### 49. NFT to DeFi correlation
```sql
WITH nft_activity AS (
    SELECT 
        DATE_TRUNC('hour', block_time) AS hour,
        SUM(amount_usd) AS nft_volume
    FROM nft.trades
    WHERE blockchain = 'ethereum'
        AND block_time >= NOW() - INTERVAL '24' HOUR
    GROUP BY 1
),
dex_activity AS (
    SELECT 
        DATE_TRUNC('hour', block_time) AS hour,
        SUM(amount_usd) AS dex_volume
    FROM dex.trades
    WHERE blockchain = 'ethereum'
        AND block_time >= NOW() - INTERVAL '24' HOUR
    GROUP BY 1
)
SELECT 
    n.hour,
    n.nft_volume,
    d.dex_volume
FROM nft_activity n
JOIN dex_activity d ON n.hour = d.hour
ORDER BY 1 DESC
```

### 50. Protocol revenue comparison
```sql
SELECT 
    'Uniswap V3' AS protocol,
    SUM(amount_usd * 0.003) AS estimated_fees_usd -- 0.3% average fee
FROM dex.trades
WHERE blockchain = 'ethereum'
    AND project = 'uniswap'
    AND version = '3'
    AND block_time >= NOW() - INTERVAL '24' HOUR
UNION ALL
SELECT 
    'Blur' AS protocol,
    SUM(amount_usd * 0.005) AS estimated_fees_usd -- 0.5% marketplace fee
FROM nft.trades
WHERE blockchain = 'ethereum'
    AND project = 'blur'
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY 2 DESC
```

---

**Footer**  
Built by [CryptoAI-Jedi](https://github.com/CryptoAI-Jedi) | [Dune SQL Cheatsheet](https://github.com/CryptoAI-Jedi/dune-sql-cheatsheet)
