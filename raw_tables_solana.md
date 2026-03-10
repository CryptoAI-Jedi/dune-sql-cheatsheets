# Raw Tables: Solana

**50 production-ready Dune SQL queries for Solana raw tables**

**Tables covered:** `solana.transactions`, `solana.instruction_calls`, `solana.account_activity`

---

## solana.transactions

### 1. Latest 100 transactions
```sql
SELECT 
    id AS signature,
    block_time,
    block_slot,
    signer,
    success,
    fee / 1e9 AS fee_sol
FROM solana.transactions
ORDER BY block_time DESC
LIMIT 100
```

### 2. Transactions from specific wallet
```sql
SELECT 
    id AS signature,
    block_time,
    success,
    fee / 1e9 AS fee_sol
FROM solana.transactions
WHERE signer = 'vines1vzrYbzLMRdu58ou5XTby4qAqVRLmqo36NKPTg'
    AND block_time >= NOW() - INTERVAL '7' DAY
ORDER BY block_time DESC
```

### 3. Failed transactions analysis
```sql
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    COUNT(*) AS total_txs,
    SUM(CASE WHEN success = false THEN 1 ELSE 0 END) AS failed_txs,
    ROUND(100.0 * SUM(CASE WHEN success = false THEN 1 ELSE 0 END) / COUNT(*), 2) AS fail_rate
FROM solana.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 1 DESC
```

### 4. Transaction fees over time
```sql
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    AVG(fee) / 1e9 AS avg_fee_sol,
    MAX(fee) / 1e9 AS max_fee_sol,
    SUM(fee) / 1e9 AS total_fees_sol
FROM solana.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 1 DESC
```

### 5. Top signers by transaction count
```sql
SELECT 
    signer,
    COUNT(*) AS tx_count,
    SUM(CASE WHEN success THEN 1 ELSE 0 END) AS successful_txs,
    SUM(fee) / 1e9 AS total_fees_sol
FROM solana.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 2 DESC
LIMIT 50
```

### 6. Transaction throughput by slot
```sql
SELECT 
    block_slot,
    COUNT(*) AS tx_count,
    SUM(CASE WHEN success THEN 1 ELSE 0 END) AS successful,
    MIN(block_time) AS slot_time
FROM solana.transactions
WHERE block_slot >= (SELECT MAX(block_slot) - 100 FROM solana.transactions)
GROUP BY 1
ORDER BY 1 DESC
```

### 7. Compute unit analysis
```sql
SELECT 
    id AS signature,
    block_time,
    signer,
    compute_unit_consumed,
    fee / 1e9 AS fee_sol
FROM solana.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY compute_unit_consumed DESC
LIMIT 100
```

### 8. Transaction size distribution
```sql
SELECT 
    CASE 
        WHEN CARDINALITY(instructions) <= 2 THEN '1-2 instructions'
        WHEN CARDINALITY(instructions) <= 5 THEN '3-5 instructions'
        WHEN CARDINALITY(instructions) <= 10 THEN '6-10 instructions'
        ELSE '10+ instructions'
    END AS instruction_bucket,
    COUNT(*) AS tx_count
FROM solana.transactions
WHERE block_time >= NOW() - INTERVAL '1' HOUR
GROUP BY 1
ORDER BY 2 DESC
```

### 9. Priority fee transactions
```sql
SELECT 
    id AS signature,
    block_time,
    signer,
    fee / 1e9 AS fee_sol
FROM solana.transactions
WHERE fee > 5000 -- More than base fee
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY fee DESC
LIMIT 100
```

### 10. Recent transactions involving Jupiter
```sql
SELECT 
    t.id AS signature,
    t.block_time,
    t.signer,
    t.success,
    t.fee / 1e9 AS fee_sol
FROM solana.transactions t
WHERE t.block_time >= NOW() - INTERVAL '1' HOUR
    AND CONTAINS(t.account_keys, 'JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4')
ORDER BY t.block_time DESC
LIMIT 100
```

### 11. Transactions with specific account
```sql
SELECT 
    id AS signature,
    block_time,
    signer,
    success
FROM solana.transactions
WHERE CONTAINS(account_keys, 'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v') -- USDC
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 12. Error analysis
```sql
SELECT 
    error,
    COUNT(*) AS error_count
FROM solana.transactions
WHERE success = false
    AND block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 2 DESC
LIMIT 20
```

### 13. Multi-signer transactions (versioned)
```sql
SELECT 
    id AS signature,
    block_time,
    signer,
    CARDINALITY(signatures) AS num_signatures,
    success
FROM solana.transactions
WHERE CARDINALITY(signatures) > 1
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 50
```

### 14. Daily transaction metrics
```sql
SELECT 
    DATE_TRUNC('day', block_time) AS day,
    COUNT(*) AS total_txs,
    SUM(CASE WHEN success THEN 1 ELSE 0 END) AS successful_txs,
    SUM(fee) / 1e9 AS total_fees_sol,
    AVG(compute_unit_consumed) AS avg_compute_units
FROM solana.transactions
WHERE block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 1
```

### 15. Unique signers per hour
```sql
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    COUNT(DISTINCT signer) AS unique_signers,
    COUNT(*) AS total_txs
FROM solana.transactions
WHERE block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 1 DESC
```

---

## solana.instruction_calls

### 16. Latest instruction calls
```sql
SELECT 
    tx_id,
    block_time,
    executing_account,
    inner_instruction_index,
    data
FROM solana.instruction_calls
ORDER BY block_time DESC
LIMIT 100
```

### 17. Jupiter swap instructions
```sql
SELECT 
    tx_id,
    block_time,
    inner_instruction_index,
    account_arguments
FROM solana.instruction_calls
WHERE executing_account = 'JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4'
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 18. Raydium AMM instructions
```sql
SELECT 
    tx_id,
    block_time,
    inner_instruction_index,
    data
FROM solana.instruction_calls
WHERE executing_account = '675kPX9MHTjS2zt1qfr1NYHuzeLXfQM9H24wFSUt1Mp8' -- Raydium V4
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 19. Token Program transfer instructions
```sql
SELECT 
    tx_id,
    block_time,
    BYTEARRAY_SUBSTRING(data, 1, 1) AS instruction_type,
    account_arguments
FROM solana.instruction_calls
WHERE executing_account = 'TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA'
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 20. System Program instructions
```sql
SELECT 
    tx_id,
    block_time,
    BYTEARRAY_SUBSTRING(data, 1, 4) AS instruction_type,
    account_arguments
FROM solana.instruction_calls
WHERE executing_account = '11111111111111111111111111111111'
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 21. Programs by instruction count
```sql
SELECT 
    executing_account AS program,
    COUNT(*) AS instruction_count
FROM solana.instruction_calls
WHERE block_time >= NOW() - INTERVAL '1' HOUR
GROUP BY 1
ORDER BY 2 DESC
LIMIT 50
```

### 22. Nested instruction depth
```sql
SELECT 
    tx_id,
    MAX(inner_instruction_index) AS max_depth,
    COUNT(*) AS total_instructions
FROM solana.instruction_calls
WHERE block_time >= NOW() - INTERVAL '1' HOUR
GROUP BY 1
ORDER BY 2 DESC
LIMIT 50
```

### 23. Marinade Finance staking
```sql
SELECT 
    tx_id,
    block_time,
    data,
    account_arguments
FROM solana.instruction_calls
WHERE executing_account = 'MarBmsSgKXdrN1egZf5sqe1TMai9K1rChYNDJgjq7aD'
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 24. Orca Whirlpool swaps
```sql
SELECT 
    tx_id,
    block_time,
    account_arguments
FROM solana.instruction_calls
WHERE executing_account = 'whirLbMiicVdio4qvUfM5KAg6Ct8VwpYzGff3uctyCc'
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 25. Associated Token Account creations
```sql
SELECT 
    tx_id,
    block_time,
    account_arguments
FROM solana.instruction_calls
WHERE executing_account = 'ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL'
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 26. Memo program usage
```sql
SELECT 
    tx_id,
    block_time,
    CAST(data AS VARCHAR) AS memo_text
FROM solana.instruction_calls
WHERE executing_account IN ('MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr', 'Memo1UhkJRfHyvLMcVucJwxXeuD728EqVDDwQDxFMNo')
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 27. Serum/OpenBook DEX orders
```sql
SELECT 
    tx_id,
    block_time,
    data,
    account_arguments
FROM solana.instruction_calls
WHERE executing_account = 'srmqPvymJeFKQ4zGQed1GFppgkRHL9kaELCbyksJtPX'
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 28. Metaplex NFT mints
```sql
SELECT 
    tx_id,
    block_time,
    account_arguments
FROM solana.instruction_calls
WHERE executing_account = 'metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s'
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 29. Instructions by hour
```sql
SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    executing_account AS program,
    COUNT(*) AS instruction_count
FROM solana.instruction_calls
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND executing_account IN (
        'JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4',
        '675kPX9MHTjS2zt1qfr1NYHuzeLXfQM9H24wFSUt1Mp8',
        'whirLbMiicVdio4qvUfM5KAg6Ct8VwpYzGff3uctyCc'
    )
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC
```

### 30. SPL Token-2022 instructions
```sql
SELECT 
    tx_id,
    block_time,
    data,
    account_arguments
FROM solana.instruction_calls
WHERE executing_account = 'TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb'
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY block_time DESC
LIMIT 100
```

---

## solana.account_activity

### 31. Account balance changes
```sql
SELECT 
    block_time,
    tx_id,
    address,
    pre_balance / 1e9 AS pre_sol,
    post_balance / 1e9 AS post_sol,
    (post_balance - pre_balance) / 1e9 AS change_sol
FROM solana.account_activity
WHERE address = 'vines1vzrYbzLMRdu58ou5XTby4qAqVRLmqo36NKPTg'
    AND block_time >= NOW() - INTERVAL '7' DAY
ORDER BY block_time DESC
```

### 32. Large SOL transfers
```sql
SELECT 
    block_time,
    tx_id,
    address,
    (post_balance - pre_balance) / 1e9 AS change_sol
FROM solana.account_activity
WHERE ABS(post_balance - pre_balance) > 1000 * 1e9 -- > 1000 SOL
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY ABS(post_balance - pre_balance) DESC
LIMIT 100
```

### 33. Token account activity
```sql
SELECT 
    block_time,
    tx_id,
    address,
    token_mint_address,
    pre_token_balance,
    post_token_balance,
    post_token_balance - pre_token_balance AS token_change
FROM solana.account_activity
WHERE token_mint_address = 'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v' -- USDC
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 34. USDT transfers
```sql
SELECT 
    block_time,
    tx_id,
    address,
    pre_token_balance / 1e6 AS pre_usdt,
    post_token_balance / 1e6 AS post_usdt,
    (post_token_balance - pre_token_balance) / 1e6 AS usdt_change
FROM solana.account_activity
WHERE token_mint_address = 'Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB' -- USDT
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 35. Account creation events
```sql
SELECT 
    block_time,
    tx_id,
    address,
    post_balance / 1e9 AS initial_balance_sol
FROM solana.account_activity
WHERE pre_balance = 0
    AND post_balance > 0
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 36. Account closure events
```sql
SELECT 
    block_time,
    tx_id,
    address,
    pre_balance / 1e9 AS final_balance_sol
FROM solana.account_activity
WHERE pre_balance > 0
    AND post_balance = 0
    AND block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 37. Top accounts by activity
```sql
SELECT 
    address,
    COUNT(*) AS activity_count,
    SUM(ABS(post_balance - pre_balance)) / 1e9 AS total_sol_moved
FROM solana.account_activity
WHERE block_time >= NOW() - INTERVAL '1' HOUR
GROUP BY 1
ORDER BY 2 DESC
LIMIT 50
```

### 38. Token mints by activity
```sql
SELECT 
    token_mint_address,
    COUNT(*) AS transfer_count,
    COUNT(DISTINCT address) AS unique_accounts
FROM solana.account_activity
WHERE token_mint_address IS NOT NULL
    AND block_time >= NOW() - INTERVAL '1' HOUR
GROUP BY 1
ORDER BY 2 DESC
LIMIT 50
```

### 39. Wrapped SOL activity
```sql
SELECT 
    block_time,
    tx_id,
    address,
    pre_token_balance / 1e9 AS pre_wsol,
    post_token_balance / 1e9 AS post_wsol
FROM solana.account_activity
WHERE token_mint_address = 'So11111111111111111111111111111111111111112' -- Wrapped SOL
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

### 40. Bonk token transfers
```sql
SELECT 
    block_time,
    tx_id,
    address,
    (post_token_balance - pre_token_balance) / 1e5 AS bonk_change
FROM solana.account_activity
WHERE token_mint_address = 'DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263' -- BONK
    AND block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY block_time DESC
LIMIT 100
```

---

## Cross-Table Queries

### 41. Transaction with account changes
```sql
SELECT 
    t.id AS signature,
    t.block_time,
    t.signer,
    aa.address,
    (aa.post_balance - aa.pre_balance) / 1e9 AS sol_change
FROM solana.transactions t
JOIN solana.account_activity aa ON t.id = aa.tx_id
WHERE t.id = '5VERv8NMvzbJMEkV8xnrLkEaWRtSz9CosKDYjCJjBRnbJLgp8uirBgmQpjKhoR4tjF3ZpRzrFmBV6UjKdiSZkQUW'
```

### 42. Failed transactions with instructions
```sql
SELECT 
    t.id AS signature,
    t.block_time,
    t.error,
    i.executing_account,
    i.inner_instruction_index
FROM solana.transactions t
JOIN solana.instruction_calls i ON t.id = i.tx_id
WHERE t.success = false
    AND t.block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY t.block_time DESC
LIMIT 100
```

### 43. Jupiter swaps with token changes
```sql
SELECT 
    t.id AS signature,
    t.block_time,
    t.signer,
    aa.token_mint_address,
    (aa.post_token_balance - aa.pre_token_balance) AS token_change
FROM solana.transactions t
JOIN solana.instruction_calls i ON t.id = i.tx_id
JOIN solana.account_activity aa ON t.id = aa.tx_id
WHERE i.executing_account = 'JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4'
    AND aa.token_mint_address IS NOT NULL
    AND t.block_time >= NOW() - INTERVAL '1' HOUR
ORDER BY t.block_time DESC
LIMIT 100
```

### 44. Program revenue analysis
```sql
SELECT 
    i.executing_account AS program,
    COUNT(DISTINCT i.tx_id) AS tx_count,
    SUM(t.fee) / 1e9 AS total_fees_sol
FROM solana.instruction_calls i
JOIN solana.transactions t ON i.tx_id = t.id
WHERE i.block_time >= NOW() - INTERVAL '24' HOUR
GROUP BY 1
ORDER BY 3 DESC
LIMIT 20
```

### 45. Whale wallet monitoring
```sql
SELECT 
    aa.address,
    aa.block_time,
    aa.tx_id,
    (aa.post_balance - aa.pre_balance) / 1e9 AS sol_change,
    t.success
FROM solana.account_activity aa
JOIN solana.transactions t ON aa.tx_id = t.id
WHERE aa.address = 'vines1vzrYbzLMRdu58ou5XTby4qAqVRLmqo36NKPTg'
    AND aa.block_time >= NOW() - INTERVAL '7' DAY
ORDER BY aa.block_time DESC
```

### 46. Token transfer with transaction details
```sql
SELECT 
    t.id AS signature,
    t.block_time,
    t.signer,
    aa.address AS token_account,
    aa.token_mint_address,
    (aa.post_token_balance - aa.pre_token_balance) AS amount_change
FROM solana.transactions t
JOIN solana.account_activity aa ON t.id = aa.tx_id
WHERE aa.token_mint_address = 'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v' -- USDC
    AND ABS(aa.post_token_balance - aa.pre_token_balance) > 1000 * 1e6 -- > 1000 USDC
    AND t.block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY ABS(aa.post_token_balance - aa.pre_token_balance) DESC
LIMIT 100
```

### 47. DEX aggregator comparison
```sql
SELECT 
    executing_account AS dex_program,
    CASE 
        WHEN executing_account = 'JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4' THEN 'Jupiter'
        WHEN executing_account = '675kPX9MHTjS2zt1qfr1NYHuzeLXfQM9H24wFSUt1Mp8' THEN 'Raydium'
        WHEN executing_account = 'whirLbMiicVdio4qvUfM5KAg6Ct8VwpYzGff3uctyCc' THEN 'Orca'
        ELSE 'Other'
    END AS dex_name,
    COUNT(*) AS instruction_count
FROM solana.instruction_calls
WHERE block_time >= NOW() - INTERVAL '24' HOUR
    AND executing_account IN (
        'JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4',
        '675kPX9MHTjS2zt1qfr1NYHuzeLXfQM9H24wFSUt1Mp8',
        'whirLbMiicVdio4qvUfM5KAg6Ct8VwpYzGff3uctyCc'
    )
GROUP BY 1
ORDER BY 3 DESC
```

### 48. Account activity summary
```sql
WITH activity AS (
    SELECT 
        address,
        COUNT(*) AS total_activity,
        SUM(CASE WHEN post_balance > pre_balance THEN 1 ELSE 0 END) AS deposits,
        SUM(CASE WHEN post_balance < pre_balance THEN 1 ELSE 0 END) AS withdrawals,
        SUM(post_balance - pre_balance) / 1e9 AS net_sol_change
    FROM solana.account_activity
    WHERE block_time >= NOW() - INTERVAL '24' HOUR
    GROUP BY 1
)
SELECT *
FROM activity
ORDER BY total_activity DESC
LIMIT 50
```

### 49. Compute unit efficiency by program
```sql
SELECT 
    i.executing_account AS program,
    COUNT(DISTINCT i.tx_id) AS tx_count,
    AVG(t.compute_unit_consumed) AS avg_compute_units,
    SUM(t.fee) / 1e9 / COUNT(DISTINCT i.tx_id) AS avg_fee_per_tx
FROM solana.instruction_calls i
JOIN solana.transactions t ON i.tx_id = t.id
WHERE i.block_time >= NOW() - INTERVAL '1' HOUR
GROUP BY 1
HAVING COUNT(DISTINCT i.tx_id) >= 10
ORDER BY 3 DESC
LIMIT 20
```

### 50. Complete transaction analysis
```sql
WITH tx_base AS (
    SELECT 
        id,
        block_time,
        signer,
        success,
        fee / 1e9 AS fee_sol,
        compute_unit_consumed
    FROM solana.transactions
    WHERE id = '5VERv8NMvzbJMEkV8xnrLkEaWRtSz9CosKDYjCJjBRnbJLgp8uirBgmQpjKhoR4tjF3ZpRzrFmBV6UjKdiSZkQUW'
),
instructions AS (
    SELECT 
        tx_id,
        COUNT(*) AS instruction_count,
        COUNT(DISTINCT executing_account) AS unique_programs
    FROM solana.instruction_calls
    WHERE tx_id = '5VERv8NMvzbJMEkV8xnrLkEaWRtSz9CosKDYjCJjBRnbJLgp8uirBgmQpjKhoR4tjF3ZpRzrFmBV6UjKdiSZkQUW'
    GROUP BY 1
),
account_changes AS (
    SELECT 
        tx_id,
        COUNT(*) AS accounts_affected,
        SUM(CASE WHEN token_mint_address IS NOT NULL THEN 1 ELSE 0 END) AS token_transfers
    FROM solana.account_activity
    WHERE tx_id = '5VERv8NMvzbJMEkV8xnrLkEaWRtSz9CosKDYjCJjBRnbJLgp8uirBgmQpjKhoR4tjF3ZpRzrFmBV6UjKdiSZkQUW'
    GROUP BY 1
)
SELECT 
    t.*,
    i.instruction_count,
    i.unique_programs,
    a.accounts_affected,
    a.token_transfers
FROM tx_base t
LEFT JOIN instructions i ON t.id = i.tx_id
LEFT JOIN account_changes a ON t.id = a.tx_id
```

---

**Footer**  
Built by [CryptoAI-Jedi](https://github.com/CryptoAI-Jedi) | [Dune SQL Cheatsheet](https://github.com/CryptoAI-Jedi/dune-sql-cheatsheet)
