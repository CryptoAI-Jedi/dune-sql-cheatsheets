# Dune SQL Cheatsheet

**500 production-ready Dune SQL queries for Customer Solutions Specialists**

**Tags:** Dune Analytics, Trino SQL, Ethereum, Solana, DeFi, NFT, Web3 Data Analysis

---

## Overview

A comprehensive reference for debugging, analyzing, and troubleshooting blockchain data on Dune Analytics. Built for technical support roles, data analysts, and anyone working with on-chain data.

### Query Count
- **10 markdown files × 50 queries each = 500 total queries**
- All queries are production-ready and tested on Dune Analytics
- Uses Trino SQL syntax (Dune's query engine)

---

## Categories

### Batch 1 (Available Now)

1. **Raw Tables: Ethereum**  
   `ethereum.transactions`, `ethereum.logs`, `ethereum.traces`, `ethereum.blocks`  
   (50 queries) → [raw_tables_ethereum.md](raw_tables_ethereum.md)

2. **Raw Tables: Solana**  
   `solana.transactions`, `solana.instruction_calls`, `solana.account_activity`  
   (50 queries) → [raw_tables_solana.md](raw_tables_solana.md)

3. **Decoded Contracts**  
   ERC20 transfers, DEX trades, NFT trades, Aave, Uniswap, Compound events  
   (50 queries) → [decoded_contracts.md](decoded_contracts.md)

4. **Spellbook Abstractions**  
   `tokens.erc20`, `prices.usd`, `dex.trades`, `nft.trades`, `labels`  
   (50 queries) → [spellbook_abstractions.md](spellbook_abstractions.md)

5. **Query Optimization**  
   Timeout fixes, `DATE_TRUNC` filtering, avoiding `SELECT *`, CTEs vs subqueries  
   (50 patterns) → [query_optimization.md](query_optimization.md)

### Batch 2 (Coming Soon)

6. **Trino SQL Patterns**  
   Trino-specific functions, array operations, JSON parsing, bytea manipulation  
   (50 queries) → *trino_sql_patterns.md* 🔜

7. **Window Functions: On-Chain**  
   LAG/LEAD, running totals, ranking, moving averages for blockchain data  
   (50 queries) → *window_functions_onchain.md* 🔜

8. **Support Triage Queries**  
   Customer debugging templates, transaction lookup, wallet analysis  
   (50 queries) → *support_triage_queries.md* 🔜

9. **Dune API Reference**  
   API endpoints, query execution, result handling, rate limits  
   (50 patterns) → *dune_api_reference.md* 🔜

10. **Common Errors Reference**  
    Error messages, troubleshooting steps, edge cases  
    (50 entries) → *common_errors_reference.md* 🔜

---

## Quick Reference

### Essential Tables
| Category | Ethereum | Solana |
|----------|----------|--------|
| Transactions | `ethereum.transactions` | `solana.transactions` |
| Logs/Events | `ethereum.logs` | `solana.instruction_calls` |
| Account State | `ethereum.traces` | `solana.account_activity` |
| Blocks | `ethereum.blocks` | `solana.blocks` |

### Spellbook Tables
| Table | Description |
|-------|-------------|
| `dex.trades` | Aggregated DEX trades across protocols |
| `nft.trades` | NFT marketplace transactions |
| `tokens.erc20` | Token metadata (symbol, decimals) |
| `prices.usd` | Historical token prices |
| `labels.all` | Wallet/contract labels |

### Key Token Addresses (Ethereum)
```
USDC:  0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
USDT:  0xdAC17F958D2ee523a2206206994597C13D831ec7
WETH:  0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2
DAI:   0x6B175474E89094C44Da98b954EedfcdeCB5BE3830
LINK:  0x514910771AF9Ca656af840dff83E8264EcF986CA
```

### Key Token Addresses (Solana)
```
USDC:  EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
USDT:  Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB
SOL:   So11111111111111111111111111111111111111112
BONK:  DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263
```

---

## Query Patterns

### Time Filtering (Critical!)
```sql
-- Always filter on block_time first (indexed column)
WHERE block_time >= NOW() - INTERVAL '24' HOUR
```

### Token Decimal Handling
```sql
-- USDC/USDT (6 decimals)
value / 1e6 AS usdc_amount

-- ETH/WETH (18 decimals)  
value / 1e18 AS eth_amount

-- Dynamic decimals
value / POWER(10, decimals) AS normalized_amount
```

### Event Signatures
```sql
-- ERC20 Transfer
topic0 = 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef

-- ERC20 Approval
topic0 = 0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925

-- Uniswap V3 Swap
topic0 = 0xc42079f94a6350d7e6235f29174924f928cc2ac818eb64fed8004e115fbcca67
```

---

## Performance Tips

1. **Always filter by time** - `block_time` is indexed
2. **Avoid SELECT *** - Select only needed columns
3. **Use LIMIT during development** - Start small, scale up
4. **Prefer CTEs over subqueries** - Better readability and optimization
5. **Use APPROX functions** - `APPROX_DISTINCT` is faster than `COUNT(DISTINCT)`

---

## Resources

- [Dune Documentation](https://docs.dune.com/)
- [Trino SQL Reference](https://trino.io/docs/current/functions.html)
- [Dune Spellbook GitHub](https://github.com/duneanalytics/spellbook)
- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)
- [Solana Documentation](https://docs.solana.com/)

---

## License

MIT License - Feel free to use, modify, and share.

---

**Footer**  
Built by [CryptoAI-Jedi](https://github.com/CryptoAI-Jedi) | [View on GitHub](https://github.com/CryptoAI-Jedi/dune-sql-cheatsheet)
