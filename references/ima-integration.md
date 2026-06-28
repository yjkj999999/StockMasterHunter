# IMA Knowledge Base Integration Guide

## Overview

This skill integrates with the IMA (Intelligent Media Assistant) knowledge base ecosystem for persistent, evolving domain knowledge on stock market analysis, Elliott Wave Theory, volume-price dynamics, and 100-bagger screening.

**API Base**: `https://ima.qq.com`
**API Prefix**: `openapi/wiki/v1`

## Setup & Authentication

### Step 1: Obtain Credentials
Visit https://ima.qq.com/agent-interface to get your Client ID and API Key.

### Step 2: Store Credentials

**Option A — Configuration File (Recommended):**
```bash
mkdir -p ~/.config/ima
echo "your_client_id" > ~/.config/ima/client_id
echo "your_api_key" > ~/.config/ima/api_key
```

**Option B — Environment Variables:**
```bash
export IMA_OPENAPI_CLIENTID="your_client_id"
export IMA_OPENAPI_APIKEY="your_api_key"
```

### Step 3: Load Credentials
The skill reads credentials in priority order: environment variables → config file.

## API Reference

All API calls use HTTP POST with JSON body. Authentication headers required on every request:

```
ima-openapi-clientid: <your_client_id>
ima-openapi-apikey: <your_api_key>
Content-Type: application/json
```

### API Helper Function

```bash
ima_api() {
  local path="$1" body="$2"
  curl -s -X POST "https://ima.qq.com/$path" \
    -H "ima-openapi-clientid: $IMA_CLIENT_ID" \
    -H "ima-openapi-apikey: $IMA_API_KEY" \
    -H "Content-Type: application/json" \
    -d "$body"
}
```

### 1. Search Knowledge Bases

Find knowledge bases containing stock market domain knowledge.

```bash
# Search by keyword
ima_api "openapi/wiki/v1/search_knowledge_base" '{"query": "股票", "cursor": "", "limit": 20}'

# List all knowledge bases
ima_api "openapi/wiki/v1/search_knowledge_base" '{"query": "", "cursor": "", "limit": 20}'
```

**Response Fields:**
- `id`: Knowledge base ID (use for subsequent operations)
- `name`: Human-readable name
- `description`: Knowledge base description

### 2. Get Knowledge Base Details

```bash
ima_api "openapi/wiki/v1/get_knowledge_base" '{"ids": ["<kb_id>"]}'
```

**Response Fields:**
- `name`, `cover_url`, `description`
- `recommended_questions`: Suggested queries for this knowledge base

### 3. Search Within a Knowledge Base

Perform semantic search within a specific knowledge base.

```bash
ima_api "openapi/wiki/v1/search_knowledge" '{
  "query": "波浪理论第三浪 主升浪",
  "knowledge_base_id": "<kb_id>",
  "cursor": ""
}'
```

**Response Fields:**
- `info_list[]`: Array of matched knowledge entries
  - `media_id`: Entry identifier
  - `title`: Entry title
  - `highlight_content`: Snippet with search terms highlighted
  - `parent_folder_id`: Parent folder (for context)

### 4. Browse Knowledge Base Contents

```bash
# Browse root directory
ima_api "openapi/wiki/v1/get_knowledge_list" '{
  "knowledge_base_id": "<kb_id>",
  "cursor": "",
  "limit": 50
}'

# Browse specific folder
ima_api "openapi/wiki/v1/get_knowledge_list" '{
  "knowledge_base_id": "<kb_id>",
  "folder_id": "<folder_id>",
  "cursor": "",
  "limit": 50
}'
```

### 5. Pagination

All list/search endpoints use cursor-based pagination:
1. Initial request: `cursor: ""`
2. Check `is_end` in response. If `false`, more data exists.
3. Use `next_cursor` as the `cursor` for the next request.
4. Stop when `is_end: true`.

## Knowledge Sync Protocol

### When to Sync
- On skill initialization (first load).
- When user explicitly requests: "sync knowledge base" or "同步知识库".
- After significant market cycle changes or new analytical frameworks are added to IMA.

### Sync Procedure

```bash
# Step 1: Discover stock-related knowledge bases
ima_api "openapi/wiki/v1/search_knowledge_base" '{"query": "股票", "cursor": "", "limit": 20}'

# Step 2: For each relevant knowledge base, search for domain concepts
DOMAINS=(
  "波浪理论 Elliott Wave 八浪循环"
  "量价关系 起涨点 奥德量"
  "百倍股 SQGLP 十倍股"
  "MACD 金叉 背离 均线多头"
  "斐波那契 黄金分割 1.618"
)

for query in "${DOMAINS[@]}"; do
  ima_api "openapi/wiki/v1/search_knowledge" "{
    \"query\": \"$query\",
    \"knowledge_base_id\": \"<kb_id>\",
    \"cursor\": \"\"
  }"
done

# Step 3: Diff new findings against cached knowledge in references/
# Step 4: Update reference files with new insights
# Step 5: Log sync timestamp
echo "$(date -Iseconds): Knowledge base sync complete. $(wc -l < new_insights.txt) new insights." >> sync.log
```

### Cached Queries for Rapid Retrieval

The skill maintains a cache of common queries for rapid context injection:

| Query | Reference File | Purpose |
|-------|---------------|---------|
| "波浪理论 三铁律" | `references/wave-theory.md` | Wave counting validation |
| "起涨点 放量突破 成交量20%" | `references/volume-price.md` | Entry signal confirmation |
| "SQGLP 百倍股 选股法则" | `references/hundred-bagger.md` | Genetic screening |
| "MACD 顶背离 底背离" | `references/volume-price.md` | Divergence detection |

## Python Integration Example

```python
import os
import json
import urllib.request

class IMAKnowledgeBase:
    def __init__(self):
        self.base_url = "https://ima.qq.com"
        self.client_id = os.environ.get("IMA_OPENAPI_CLIENTID") or \
                         open(os.path.expanduser("~/.config/ima/client_id")).read().strip()
        self.api_key = os.environ.get("IMA_OPENAPI_APIKEY") or \
                       open(os.path.expanduser("~/.config/ima/api_key")).read().strip()

    def _call(self, path, body):
        url = f"{self.base_url}/{path}"
        data = json.dumps(body).encode("utf-8")
        headers = {
            "ima-openapi-clientid": self.client_id,
            "ima-openapi-apikey": self.api_key,
            "Content-Type": "application/json"
        }
        req = urllib.request.Request(url, data=data, headers=headers, method="POST")
        with urllib.request.urlopen(req, timeout=15) as resp:
            result = json.loads(resp.read().decode("utf-8"))
            if result.get("retcode") != 0:
                raise Exception(f"IMA API error: {result.get('errmsg')}")
            return result.get("data", {})

    def search_knowledge_bases(self, query="股票"):
        return self._call("openapi/wiki/v1/search_knowledge_base", {
            "query": query, "cursor": "", "limit": 20
        })

    def search_knowledge(self, kb_id, query):
        return self._call("openapi/wiki/v1/search_knowledge", {
            "query": query,
            "knowledge_base_id": kb_id,
            "cursor": ""
        })
```

## Error Handling

All API responses follow the structure:
```json
{
  "retcode": 0,
  "errmsg": "",
  "data": { ... }
}
```

- `retcode = 0`: Success. Extract from `data`.
- `retcode != 0`: Failure. Display `errmsg` to user.
- On network error: Fallback to cached knowledge in reference files.

## Rate Limits & Best Practices

- Default limit per request: 50 items.
- Use cursor-based pagination, not offset-based.
- Cache search results for frequently queried concepts (valid for ~1 hour).
- Don't hammer the API — batch queries where possible.
