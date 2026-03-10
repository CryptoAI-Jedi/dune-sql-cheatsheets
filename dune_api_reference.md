# Dune API Reference

**50 production-ready examples for the Dune Analytics API**

**Topics covered:** API endpoints, execution flow, status codes, rate limits, common errors, Python/curl examples, pagination, CSV exports, query parameters

---

## API Basics

### 1. Execute a saved query (Python)
```python
import requests

API_KEY = "your_api_key"
QUERY_ID = 1234567

response = requests.post(
    f"https://api.dune.com/api/v1/query/{QUERY_ID}/execute",
    headers={"X-Dune-API-Key": API_KEY}
)
execution_id = response.json()["execution_id"]
print(f"Execution ID: {execution_id}")
```

### 2. Execute a saved query (curl)
```bash
curl -X POST "https://api.dune.com/api/v1/query/1234567/execute" \
  -H "X-Dune-API-Key: your_api_key"
```

### 3. Check execution status (Python)
```python
import requests
import time

API_KEY = "your_api_key"
EXECUTION_ID = "01HF..."

def check_status(execution_id):
    response = requests.get(
        f"https://api.dune.com/api/v1/execution/{execution_id}/status",
        headers={"X-Dune-API-Key": API_KEY}
    )
    return response.json()

# Poll until complete
while True:
    status = check_status(EXECUTION_ID)
    state = status["state"]
    print(f"State: {state}")
    
    if state == "QUERY_STATE_COMPLETED":
        break
    elif state == "QUERY_STATE_FAILED":
        raise Exception(f"Query failed: {status}")
    
    time.sleep(2)
```

### 4. Check execution status (curl)
```bash
curl "https://api.dune.com/api/v1/execution/01HF.../status" \
  -H "X-Dune-API-Key: your_api_key"
```

### 5. Get execution results (Python)
```python
import requests

API_KEY = "your_api_key"
EXECUTION_ID = "01HF..."

response = requests.get(
    f"https://api.dune.com/api/v1/execution/{EXECUTION_ID}/results",
    headers={"X-Dune-API-Key": API_KEY}
)

data = response.json()
rows = data["result"]["rows"]
print(f"Got {len(rows)} rows")
```

### 6. Get execution results (curl)
```bash
curl "https://api.dune.com/api/v1/execution/01HF.../results" \
  -H "X-Dune-API-Key: your_api_key"
```

### 7. Complete execute-and-wait flow (Python)
```python
import requests
import time

API_KEY = "your_api_key"
QUERY_ID = 1234567

def execute_query(query_id, parameters=None):
    # Execute
    payload = {"query_parameters": parameters} if parameters else {}
    response = requests.post(
        f"https://api.dune.com/api/v1/query/{query_id}/execute",
        headers={"X-Dune-API-Key": API_KEY},
        json=payload
    )
    execution_id = response.json()["execution_id"]
    
    # Poll for completion
    while True:
        status_response = requests.get(
            f"https://api.dune.com/api/v1/execution/{execution_id}/status",
            headers={"X-Dune-API-Key": API_KEY}
        )
        state = status_response.json()["state"]
        
        if state == "QUERY_STATE_COMPLETED":
            break
        elif state == "QUERY_STATE_FAILED":
            raise Exception("Query failed")
        
        time.sleep(2)
    
    # Get results
    results_response = requests.get(
        f"https://api.dune.com/api/v1/execution/{execution_id}/results",
        headers={"X-Dune-API-Key": API_KEY}
    )
    return results_response.json()["result"]["rows"]

# Usage
rows = execute_query(QUERY_ID)
print(f"Retrieved {len(rows)} rows")
```

### 8. Execute with query parameters (Python)
```python
import requests

API_KEY = "your_api_key"
QUERY_ID = 1234567

# Parameters must match those defined in the query
parameters = {
    "wallet_address": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
    "days_back": 30
}

response = requests.post(
    f"https://api.dune.com/api/v1/query/{QUERY_ID}/execute",
    headers={"X-Dune-API-Key": API_KEY},
    json={"query_parameters": parameters}
)
print(response.json())
```

### 9. Execute with parameters (curl)
```bash
curl -X POST "https://api.dune.com/api/v1/query/1234567/execute" \
  -H "X-Dune-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"query_parameters": {"wallet_address": "0xd8dA...", "days_back": 30}}'
```

### 10. Get latest results without re-executing
```python
import requests

API_KEY = "your_api_key"
QUERY_ID = 1234567

# Get cached results (doesn't trigger new execution)
response = requests.get(
    f"https://api.dune.com/api/v1/query/{QUERY_ID}/results",
    headers={"X-Dune-API-Key": API_KEY}
)

data = response.json()
print(f"Results from: {data['execution_ended_at']}")
```

---

## Pagination and Large Results

### 11. Paginate results (Python)
```python
import requests

API_KEY = "your_api_key"
EXECUTION_ID = "01HF..."

def get_all_results(execution_id, limit=1000):
    all_rows = []
    offset = 0
    
    while True:
        response = requests.get(
            f"https://api.dune.com/api/v1/execution/{execution_id}/results",
            headers={"X-Dune-API-Key": API_KEY},
            params={"limit": limit, "offset": offset}
        )
        
        data = response.json()
        rows = data["result"]["rows"]
        all_rows.extend(rows)
        
        if len(rows) < limit:
            break  # No more pages
        
        offset += limit
    
    return all_rows

rows = get_all_results(EXECUTION_ID)
print(f"Total rows: {len(rows)}")
```

### 12. Paginate results (curl)
```bash
# First page
curl "https://api.dune.com/api/v1/execution/01HF.../results?limit=1000&offset=0" \
  -H "X-Dune-API-Key: your_api_key"

# Second page
curl "https://api.dune.com/api/v1/execution/01HF.../results?limit=1000&offset=1000" \
  -H "X-Dune-API-Key: your_api_key"
```

### 13. Get result count before fetching
```python
import requests

API_KEY = "your_api_key"
EXECUTION_ID = "01HF..."

# First get just metadata
response = requests.get(
    f"https://api.dune.com/api/v1/execution/{EXECUTION_ID}/results",
    headers={"X-Dune-API-Key": API_KEY},
    params={"limit": 0}  # Get metadata only
)

metadata = response.json()["result"]["metadata"]
total_rows = metadata["total_row_count"]
print(f"Total rows available: {total_rows}")
```

### 14. Sort results by column
```python
import requests

API_KEY = "your_api_key"
EXECUTION_ID = "01HF..."

# Note: Sorting should typically be done in the SQL query itself
# API sorting is limited
response = requests.get(
    f"https://api.dune.com/api/v1/execution/{EXECUTION_ID}/results",
    headers={"X-Dune-API-Key": API_KEY},
    params={
        "limit": 100,
        "sort_by": "volume_usd desc"  # Check API docs for current support
    }
)
```

### 15. Filter specific columns
```python
import requests

API_KEY = "your_api_key"
EXECUTION_ID = "01HF..."

# Get only specific columns
response = requests.get(
    f"https://api.dune.com/api/v1/execution/{EXECUTION_ID}/results",
    headers={"X-Dune-API-Key": API_KEY},
    params={
        "columns": "wallet,volume_usd,trade_count"
    }
)
```

---

## CSV Export

### 16. Export results as CSV (Python)
```python
import requests

API_KEY = "your_api_key"
EXECUTION_ID = "01HF..."

response = requests.get(
    f"https://api.dune.com/api/v1/execution/{EXECUTION_ID}/results/csv",
    headers={"X-Dune-API-Key": API_KEY}
)

# Save to file
with open("results.csv", "w") as f:
    f.write(response.text)
```

### 17. Export results as CSV (curl)
```bash
curl "https://api.dune.com/api/v1/execution/01HF.../results/csv" \
  -H "X-Dune-API-Key: your_api_key" \
  -o results.csv
```

### 18. Stream large CSV results
```python
import requests

API_KEY = "your_api_key"
EXECUTION_ID = "01HF..."

# Stream to avoid loading entire file in memory
with requests.get(
    f"https://api.dune.com/api/v1/execution/{EXECUTION_ID}/results/csv",
    headers={"X-Dune-API-Key": API_KEY},
    stream=True
) as r:
    r.raise_for_status()
    with open("large_results.csv", "wb") as f:
        for chunk in r.iter_content(chunk_size=8192):
            f.write(chunk)
```

### 19. Load CSV directly into pandas
```python
import pandas as pd
import requests
from io import StringIO

API_KEY = "your_api_key"
EXECUTION_ID = "01HF..."

response = requests.get(
    f"https://api.dune.com/api/v1/execution/{EXECUTION_ID}/results/csv",
    headers={"X-Dune-API-Key": API_KEY}
)

df = pd.read_csv(StringIO(response.text))
print(df.head())
```

### 20. CSV with custom delimiter handling
```python
import pandas as pd
import requests
from io import StringIO

API_KEY = "your_api_key"
EXECUTION_ID = "01HF..."

response = requests.get(
    f"https://api.dune.com/api/v1/execution/{EXECUTION_ID}/results/csv",
    headers={"X-Dune-API-Key": API_KEY}
)

# Handle potential issues with CSV
df = pd.read_csv(
    StringIO(response.text),
    quoting=1,  # QUOTE_ALL - handle embedded commas
    escapechar='\\'
)
```

---

## Error Handling

### 21. Handle rate limiting (Python)
```python
import requests
import time

API_KEY = "your_api_key"

def api_request(url, method="GET", **kwargs):
    headers = {"X-Dune-API-Key": API_KEY}
    
    for attempt in range(5):
        if method == "GET":
            response = requests.get(url, headers=headers, **kwargs)
        else:
            response = requests.post(url, headers=headers, **kwargs)
        
        if response.status_code == 429:  # Rate limited
            wait_time = int(response.headers.get("Retry-After", 60))
            print(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)
            continue
        
        return response
    
    raise Exception("Max retries exceeded")
```

### 22. Handle common API errors
```python
import requests

API_KEY = "your_api_key"
QUERY_ID = 1234567

response = requests.post(
    f"https://api.dune.com/api/v1/query/{QUERY_ID}/execute",
    headers={"X-Dune-API-Key": API_KEY}
)

if response.status_code == 200:
    print("Success:", response.json())
elif response.status_code == 400:
    print("Bad request - check query parameters")
elif response.status_code == 401:
    print("Unauthorized - check API key")
elif response.status_code == 403:
    print("Forbidden - query may be private or exceeded quota")
elif response.status_code == 404:
    print("Not found - query ID doesn't exist")
elif response.status_code == 429:
    print("Rate limited - slow down requests")
elif response.status_code == 500:
    print("Server error - try again later")
else:
    print(f"Unknown error: {response.status_code}")
```

### 23. Handle query timeout
```python
import requests
import time

API_KEY = "your_api_key"
EXECUTION_ID = "01HF..."
TIMEOUT_SECONDS = 300  # 5 minutes

start_time = time.time()
while True:
    elapsed = time.time() - start_time
    if elapsed > TIMEOUT_SECONDS:
        raise TimeoutError(f"Query exceeded {TIMEOUT_SECONDS}s timeout")
    
    response = requests.get(
        f"https://api.dune.com/api/v1/execution/{EXECUTION_ID}/status",
        headers={"X-Dune-API-Key": API_KEY}
    )
    state = response.json()["state"]
    
    if state == "QUERY_STATE_COMPLETED":
        print(f"Completed in {elapsed:.1f}s")
        break
    elif state == "QUERY_STATE_FAILED":
        error = response.json().get("error", "Unknown error")
        raise Exception(f"Query failed: {error}")
    
    time.sleep(2)
```

### 24. Validate API key
```python
import requests

API_KEY = "your_api_key"

# Simple validation by checking a known endpoint
response = requests.get(
    "https://api.dune.com/api/v1/query/1/results",
    headers={"X-Dune-API-Key": API_KEY}
)

if response.status_code == 401:
    print("Invalid API key")
elif response.status_code in [200, 404]:  # 404 just means query not found
    print("API key is valid")
```

### 25. Handle malformed response
```python
import requests

API_KEY = "your_api_key"
QUERY_ID = 1234567

try:
    response = requests.post(
        f"https://api.dune.com/api/v1/query/{QUERY_ID}/execute",
        headers={"X-Dune-API-Key": API_KEY}
    )
    response.raise_for_status()
    data = response.json()
    
    if "execution_id" not in data:
        raise ValueError(f"Unexpected response format: {data}")
    
    print(f"Execution ID: {data['execution_id']}")
    
except requests.exceptions.JSONDecodeError:
    print(f"Invalid JSON response: {response.text}")
except requests.exceptions.RequestException as e:
    print(f"Request failed: {e}")
```

---

## Advanced Usage

### 26. Cancel running execution
```python
import requests

API_KEY = "your_api_key"
EXECUTION_ID = "01HF..."

response = requests.post(
    f"https://api.dune.com/api/v1/execution/{EXECUTION_ID}/cancel",
    headers={"X-Dune-API-Key": API_KEY}
)

if response.status_code == 200:
    print("Execution cancelled")
else:
    print(f"Failed to cancel: {response.text}")
```

### 27. Cancel execution (curl)
```bash
curl -X POST "https://api.dune.com/api/v1/execution/01HF.../cancel" \
  -H "X-Dune-API-Key: your_api_key"
```

### 28. Get query metadata
```python
import requests

API_KEY = "your_api_key"
QUERY_ID = 1234567

response = requests.get(
    f"https://api.dune.com/api/v1/query/{QUERY_ID}",
    headers={"X-Dune-API-Key": API_KEY}
)

query_info = response.json()
print(f"Query name: {query_info.get('name')}")
print(f"Parameters: {query_info.get('parameters')}")
```

### 29. Async execution with callback pattern
```python
import requests
import asyncio
import aiohttp

API_KEY = "your_api_key"

async def execute_and_wait(query_id, parameters=None):
    async with aiohttp.ClientSession() as session:
        # Execute
        payload = {"query_parameters": parameters} if parameters else {}
        async with session.post(
            f"https://api.dune.com/api/v1/query/{query_id}/execute",
            headers={"X-Dune-API-Key": API_KEY},
            json=payload
        ) as resp:
            data = await resp.json()
            execution_id = data["execution_id"]
        
        # Poll
        while True:
            async with session.get(
                f"https://api.dune.com/api/v1/execution/{execution_id}/status",
                headers={"X-Dune-API-Key": API_KEY}
            ) as resp:
                status = await resp.json()
                if status["state"] == "QUERY_STATE_COMPLETED":
                    break
                elif status["state"] == "QUERY_STATE_FAILED":
                    raise Exception("Query failed")
            await asyncio.sleep(2)
        
        # Get results
        async with session.get(
            f"https://api.dune.com/api/v1/execution/{execution_id}/results",
            headers={"X-Dune-API-Key": API_KEY}
        ) as resp:
            data = await resp.json()
            return data["result"]["rows"]

# Usage
# rows = asyncio.run(execute_and_wait(1234567))
```

### 30. Batch execute multiple queries
```python
import requests
import concurrent.futures
import time

API_KEY = "your_api_key"
QUERY_IDS = [1234567, 1234568, 1234569]

def execute_single(query_id):
    # Execute
    response = requests.post(
        f"https://api.dune.com/api/v1/query/{query_id}/execute",
        headers={"X-Dune-API-Key": API_KEY}
    )
    execution_id = response.json()["execution_id"]
    
    # Wait
    while True:
        status = requests.get(
            f"https://api.dune.com/api/v1/execution/{execution_id}/status",
            headers={"X-Dune-API-Key": API_KEY}
        ).json()
        
        if status["state"] == "QUERY_STATE_COMPLETED":
            break
        elif status["state"] == "QUERY_STATE_FAILED":
            raise Exception(f"Query {query_id} failed")
        time.sleep(2)
    
    # Get results
    results = requests.get(
        f"https://api.dune.com/api/v1/execution/{execution_id}/results",
        headers={"X-Dune-API-Key": API_KEY}
    ).json()
    
    return query_id, results["result"]["rows"]

# Execute in parallel (respect rate limits!)
with concurrent.futures.ThreadPoolExecutor(max_workers=3) as executor:
    futures = {executor.submit(execute_single, qid): qid for qid in QUERY_IDS}
    for future in concurrent.futures.as_completed(futures):
        query_id, rows = future.result()
        print(f"Query {query_id}: {len(rows)} rows")
```

---

## SQL Query Examples for API Use

### 31. Parameterized wallet query
```sql
-- Query ID: Save this query in Dune with parameters
-- Parameter: wallet_address (text)
-- Parameter: days_back (number)

SELECT 
    hash,
    block_time,
    value / 1e18 AS eth_value,
    gas_used * gas_price / 1e18 AS fee_eth
FROM ethereum.transactions
WHERE "from" = {{wallet_address}}
    AND block_time >= NOW() - INTERVAL '{{days_back}}' DAY
ORDER BY block_time DESC
LIMIT 1000
```

### 32. Token transfer query with parameters
```sql
-- Parameter: token_address (text)
-- Parameter: min_amount (number)

SELECT 
    evt_tx_hash,
    evt_block_time,
    "from",
    "to",
    value / POWER(10, t.decimals) AS amount
FROM erc20_ethereum.evt_Transfer e
JOIN tokens.erc20 t 
    ON e.contract_address = t.contract_address 
    AND t.blockchain = 'ethereum'
WHERE e.contract_address = {{token_address}}
    AND value / POWER(10, t.decimals) >= {{min_amount}}
    AND evt_block_time >= NOW() - INTERVAL '24' HOUR
ORDER BY evt_block_time DESC
```

### 33. DEX volume by date range
```sql
-- Parameter: start_date (date)
-- Parameter: end_date (date)
-- Parameter: blockchain (text)

SELECT 
    CAST(block_time AS DATE) AS day,
    project,
    SUM(amount_usd) AS volume_usd,
    COUNT(*) AS trade_count
FROM dex.trades
WHERE block_time >= CAST('{{start_date}}' AS TIMESTAMP)
    AND block_time < CAST('{{end_date}}' AS TIMESTAMP) + INTERVAL '1' DAY
    AND blockchain = '{{blockchain}}'
GROUP BY 1, 2
ORDER BY 1, 3 DESC
```

### 34. NFT sales for collection
```sql
-- Parameter: collection_address (text)

SELECT 
    tx_hash,
    block_time,
    buyer,
    seller,
    amount_usd,
    token_id
FROM nft.trades
WHERE nft_contract_address = {{collection_address}}
    AND block_time >= NOW() - INTERVAL '30' DAY
ORDER BY block_time DESC
```

### 35. Gas price analytics
```sql
-- Parameter: hours_back (number)

SELECT 
    DATE_TRUNC('hour', block_time) AS hour,
    COUNT(*) AS tx_count,
    AVG(gas_price / 1e9) AS avg_gas_gwei,
    APPROX_PERCENTILE(gas_price / 1e9, 0.5) AS median_gas_gwei,
    APPROX_PERCENTILE(gas_price / 1e9, 0.95) AS p95_gas_gwei
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '{{hours_back}}' HOUR
GROUP BY 1
ORDER BY 1 DESC
```

---

## Rate Limits and Best Practices

### 36. Implement exponential backoff
```python
import requests
import time
import random

API_KEY = "your_api_key"

def api_request_with_backoff(url, max_retries=5):
    for attempt in range(max_retries):
        response = requests.get(url, headers={"X-Dune-API-Key": API_KEY})
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 429:
            # Exponential backoff with jitter
            wait_time = (2 ** attempt) + random.uniform(0, 1)
            print(f"Rate limited. Waiting {wait_time:.1f}s...")
            time.sleep(wait_time)
        else:
            response.raise_for_status()
    
    raise Exception("Max retries exceeded")
```

### 37. Check remaining rate limit
```python
import requests

API_KEY = "your_api_key"

response = requests.get(
    "https://api.dune.com/api/v1/query/1/results",
    headers={"X-Dune-API-Key": API_KEY}
)

# Rate limit headers (check current API docs for exact header names)
print(f"Limit: {response.headers.get('X-RateLimit-Limit')}")
print(f"Remaining: {response.headers.get('X-RateLimit-Remaining')}")
print(f"Reset: {response.headers.get('X-RateLimit-Reset')}")
```

### 38. Cache results locally
```python
import requests
import json
import os
from datetime import datetime, timedelta

API_KEY = "your_api_key"
CACHE_DIR = "./cache"

def get_cached_or_fetch(query_id, max_age_hours=1):
    cache_file = f"{CACHE_DIR}/{query_id}.json"
    
    # Check cache
    if os.path.exists(cache_file):
        stat = os.stat(cache_file)
        age = datetime.now() - datetime.fromtimestamp(stat.st_mtime)
        
        if age < timedelta(hours=max_age_hours):
            with open(cache_file) as f:
                return json.load(f)
    
    # Fetch fresh data
    response = requests.get(
        f"https://api.dune.com/api/v1/query/{query_id}/results",
        headers={"X-Dune-API-Key": API_KEY}
    )
    data = response.json()
    
    # Cache it
    os.makedirs(CACHE_DIR, exist_ok=True)
    with open(cache_file, 'w') as f:
        json.dump(data, f)
    
    return data
```

### 39. Use connection pooling
```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

API_KEY = "your_api_key"

# Create session with connection pooling
session = requests.Session()
retry_strategy = Retry(
    total=3,
    backoff_factor=1,
    status_forcelist=[429, 500, 502, 503, 504]
)
adapter = HTTPAdapter(max_retries=retry_strategy, pool_connections=10, pool_maxsize=10)
session.mount("https://", adapter)
session.headers.update({"X-Dune-API-Key": API_KEY})

# Use session for all requests
response = session.get("https://api.dune.com/api/v1/query/1234567/results")
```

### 40. Monitor API usage
```python
import requests
from collections import defaultdict
from datetime import datetime

API_KEY = "your_api_key"

# Simple usage tracker
usage_log = defaultdict(int)

def tracked_request(endpoint):
    usage_log[datetime.now().strftime("%Y-%m-%d %H")] += 1
    
    response = requests.get(
        f"https://api.dune.com/api/v1/{endpoint}",
        headers={"X-Dune-API-Key": API_KEY}
    )
    return response

def print_usage():
    print("API Usage by hour:")
    for hour, count in sorted(usage_log.items()):
        print(f"  {hour}: {count} requests")
```

---

## Integration Examples

### 41. Save to database
```python
import requests
import sqlite3

API_KEY = "your_api_key"
EXECUTION_ID = "01HF..."

# Fetch data
response = requests.get(
    f"https://api.dune.com/api/v1/execution/{EXECUTION_ID}/results",
    headers={"X-Dune-API-Key": API_KEY}
)
rows = response.json()["result"]["rows"]

# Save to SQLite
conn = sqlite3.connect("dune_data.db")
cursor = conn.cursor()

# Create table (adjust columns as needed)
cursor.execute('''
    CREATE TABLE IF NOT EXISTS dex_trades (
        tx_hash TEXT,
        block_time TEXT,
        project TEXT,
        amount_usd REAL
    )
''')

# Insert data
for row in rows:
    cursor.execute(
        "INSERT INTO dex_trades VALUES (?, ?, ?, ?)",
        (row["tx_hash"], row["block_time"], row["project"], row["amount_usd"])
    )

conn.commit()
conn.close()
```

### 42. Send to webhook on completion
```python
import requests
import time

API_KEY = "your_api_key"
QUERY_ID = 1234567
WEBHOOK_URL = "https://your-webhook.com/endpoint"

# Execute query
response = requests.post(
    f"https://api.dune.com/api/v1/query/{QUERY_ID}/execute",
    headers={"X-Dune-API-Key": API_KEY}
)
execution_id = response.json()["execution_id"]

# Wait for completion
while True:
    status = requests.get(
        f"https://api.dune.com/api/v1/execution/{execution_id}/status",
        headers={"X-Dune-API-Key": API_KEY}
    ).json()
    
    if status["state"] == "QUERY_STATE_COMPLETED":
        # Get results and send to webhook
        results = requests.get(
            f"https://api.dune.com/api/v1/execution/{execution_id}/results",
            headers={"X-Dune-API-Key": API_KEY}
        ).json()
        
        requests.post(WEBHOOK_URL, json={
            "query_id": QUERY_ID,
            "execution_id": execution_id,
            "row_count": len(results["result"]["rows"]),
            "data": results["result"]["rows"][:100]  # First 100 rows
        })
        break
    
    time.sleep(5)
```

### 43. Schedule recurring queries
```python
import schedule
import time
import requests

API_KEY = "your_api_key"
QUERY_ID = 1234567

def run_query():
    print(f"Running query {QUERY_ID}...")
    response = requests.post(
        f"https://api.dune.com/api/v1/query/{QUERY_ID}/execute",
        headers={"X-Dune-API-Key": API_KEY}
    )
    print(f"Started execution: {response.json()['execution_id']}")

# Run every hour
schedule.every(1).hour.do(run_query)

# Keep running
while True:
    schedule.run_pending()
    time.sleep(60)
```

### 44. Export to Google Sheets
```python
import requests
import gspread
from oauth2client.service_account import ServiceAccountCredentials

API_KEY = "your_api_key"
EXECUTION_ID = "01HF..."

# Get Dune data
response = requests.get(
    f"https://api.dune.com/api/v1/execution/{EXECUTION_ID}/results",
    headers={"X-Dune-API-Key": API_KEY}
)
rows = response.json()["result"]["rows"]

# Setup Google Sheets
scope = ['https://spreadsheets.google.com/feeds']
creds = ServiceAccountCredentials.from_json_keyfile_name('credentials.json', scope)
client = gspread.authorize(creds)

# Write to sheet
sheet = client.open("Dune Data").sheet1
sheet.clear()

# Write headers
if rows:
    headers = list(rows[0].keys())
    sheet.insert_row(headers, 1)
    
    # Write data
    for i, row in enumerate(rows, start=2):
        sheet.insert_row(list(row.values()), i)
```

### 45. Create Slack alert
```python
import requests

API_KEY = "your_api_key"
EXECUTION_ID = "01HF..."
SLACK_WEBHOOK = "https://hooks.slack.com/services/..."

# Get results
response = requests.get(
    f"https://api.dune.com/api/v1/execution/{EXECUTION_ID}/results",
    headers={"X-Dune-API-Key": API_KEY}
)
rows = response.json()["result"]["rows"]

# Check threshold
total_volume = sum(row.get("volume_usd", 0) for row in rows)

if total_volume > 1000000:
    requests.post(SLACK_WEBHOOK, json={
        "text": f"🚨 High volume alert!\nTotal: ${total_volume:,.0f}"
    })
```

---

## Debugging and Monitoring

### 46. Log all API calls
```python
import requests
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

API_KEY = "your_api_key"

class DuneClient:
    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = "https://api.dune.com/api/v1"
        
    def _request(self, method, endpoint, **kwargs):
        url = f"{self.base_url}/{endpoint}"
        logger.info(f"{method} {url}")
        
        response = requests.request(
            method, url,
            headers={"X-Dune-API-Key": self.api_key},
            **kwargs
        )
        
        logger.info(f"Response: {response.status_code}")
        return response
    
    def execute(self, query_id, parameters=None):
        payload = {"query_parameters": parameters} if parameters else {}
        return self._request("POST", f"query/{query_id}/execute", json=payload)

client = DuneClient(API_KEY)
```

### 47. Measure execution time
```python
import requests
import time

API_KEY = "your_api_key"
QUERY_ID = 1234567

# Execute
start = time.time()
response = requests.post(
    f"https://api.dune.com/api/v1/query/{QUERY_ID}/execute",
    headers={"X-Dune-API-Key": API_KEY}
)
execution_id = response.json()["execution_id"]

# Wait
while True:
    status = requests.get(
        f"https://api.dune.com/api/v1/execution/{execution_id}/status",
        headers={"X-Dune-API-Key": API_KEY}
    ).json()
    
    if status["state"] in ["QUERY_STATE_COMPLETED", "QUERY_STATE_FAILED"]:
        break
    time.sleep(2)

elapsed = time.time() - start
print(f"Total time: {elapsed:.1f}s")
print(f"State: {status['state']}")
```

### 48. Compare query performance
```python
import requests
import time

API_KEY = "your_api_key"

def benchmark_query(query_id, runs=3):
    times = []
    
    for i in range(runs):
        start = time.time()
        
        # Execute
        response = requests.post(
            f"https://api.dune.com/api/v1/query/{query_id}/execute",
            headers={"X-Dune-API-Key": API_KEY}
        )
        execution_id = response.json()["execution_id"]
        
        # Wait
        while True:
            status = requests.get(
                f"https://api.dune.com/api/v1/execution/{execution_id}/status",
                headers={"X-Dune-API-Key": API_KEY}
            ).json()
            
            if status["state"] == "QUERY_STATE_COMPLETED":
                break
            time.sleep(1)
        
        elapsed = time.time() - start
        times.append(elapsed)
        print(f"Run {i+1}: {elapsed:.1f}s")
    
    print(f"Average: {sum(times)/len(times):.1f}s")
    return times

# benchmark_query(1234567)
```

### 49. Debug parameter issues
```python
import requests

API_KEY = "your_api_key"
QUERY_ID = 1234567

# First, get query metadata to see expected parameters
query_info = requests.get(
    f"https://api.dune.com/api/v1/query/{QUERY_ID}",
    headers={"X-Dune-API-Key": API_KEY}
).json()

print("Expected parameters:")
for param in query_info.get("parameters", []):
    print(f"  - {param['name']} ({param['type']})")

# Now execute with parameters
parameters = {
    "wallet_address": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045"
}

response = requests.post(
    f"https://api.dune.com/api/v1/query/{QUERY_ID}/execute",
    headers={"X-Dune-API-Key": API_KEY},
    json={"query_parameters": parameters}
)
print(f"Execution response: {response.json()}")
```

### 50. Full production client class
```python
import requests
import time
from typing import Optional, Dict, List, Any

class DuneAPI:
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.dune.com/api/v1"
        self.session = requests.Session()
        self.session.headers.update({"X-Dune-API-Key": api_key})
    
    def execute_query(
        self, 
        query_id: int, 
        parameters: Optional[Dict] = None,
        timeout: int = 300
    ) -> List[Dict[str, Any]]:
        # Execute a query and return results.
        
        # Start execution
        payload = {"query_parameters": parameters} if parameters else {}
        resp = self.session.post(
            f"{self.base_url}/query/{query_id}/execute",
            json=payload
        )
        resp.raise_for_status()
        execution_id = resp.json()["execution_id"]
        
        # Wait for completion
        start = time.time()
        while time.time() - start < timeout:
            status_resp = self.session.get(
                f"{self.base_url}/execution/{execution_id}/status"
            )
            status = status_resp.json()
            
            if status["state"] == "QUERY_STATE_COMPLETED":
                break
            elif status["state"] == "QUERY_STATE_FAILED":
                raise Exception(f"Query failed: {status}")
            
            time.sleep(2)
        else:
            raise TimeoutError(f"Query timed out after {timeout}s")
        
        # Get results
        results_resp = self.session.get(
            f"{self.base_url}/execution/{execution_id}/results"
        )
        return results_resp.json()["result"]["rows"]
    
    def get_latest_results(self, query_id: int) -> List[Dict[str, Any]]:
        # Get cached results without re-executing.
        resp = self.session.get(
            f"{self.base_url}/query/{query_id}/results"
        )
        resp.raise_for_status()
        return resp.json()["result"]["rows"]

# Usage:
# client = DuneAPI("your_api_key")
# rows = client.execute_query(1234567, {"wallet": "0x..."})
```

---

*Last updated: March 2026*
