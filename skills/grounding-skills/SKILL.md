---
name: grounding-skills
description: Use when working with grounding on Vertex AI using google-genai Python SDK — imports GoogleSearch, GoogleMaps, VertexAISearch, Elasticsearch, ExternalApi, EnterpriseWebSearch, or requests involving "grounding", "Google Search grounding", "RAG", "Vertex AI Search", "Elasticsearch grounding", "enterprise web search". Korean: "그라운딩", "검색 기반 생성", "구글 검색 연동", "버텍스 검색", "환각 방지", "외부 데이터 연결".
version: 1.0.0
---

# Grounding Skills — google-genai Python SDK Reference

## Environment Setup

### Install Libraries

```bash
pip install --upgrade google-genai
```

### Required Environment Variables

```bash
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"
export GOOGLE_GENAI_USE_VERTEXAI="True"
```

### Client Initialization

```python
from google import genai
from google.genai.types import HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))
```

### Grounding Method Selection Guide

| Method | Tool Class | Use Case |
|--------|-----------|----------|
| Google Search | `GoogleSearch` | Latest public web data |
| Google Maps | `GoogleMaps` | Location and geographic information |
| Vertex AI Search | `VertexAISearch` (via `Retrieval`) | Your own documents or website data |
| Elasticsearch | `ExternalApi` (via `Retrieval`) | Existing ES index |
| Custom Search API | `ExternalApi` (via `Retrieval`) | Your own search API |
| Parallel Web Search | REST only (`parallelAiSearch`) | LLM-optimized web index |
| Enterprise Web Search | `EnterpriseWebSearch` | Compliance-regulated environments |

> **Note:** Vertex AI RAG Engine grounding (`ground-responses-using-rag`) uses the `vertexai` SDK (`google-cloud-aiplatform`) and is outside the `google-genai` SDK scope of this skill.

---

## § 1. Google Search Grounding

### When: augmenting responses with the latest public web information

```python
from google import genai
from google.genai.types import (
    GenerateContentConfig,
    GoogleSearch,
    HttpOptions,
    Tool,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="When is the next total solar eclipse in the United States?",
    config=GenerateContentConfig(
        tools=[
            Tool(
                google_search=GoogleSearch(
                    exclude_domains=["domain.com"],   # domains to exclude (optional)
                )
            )
        ],
    ),
)

print(response.text)
# Check search sources
for chunk in response.candidates[0].grounding_metadata.grounding_chunks:
    print(chunk.web.uri)
```

### Dynamic Retrieval (Conditional Grounding)

```python
from google import genai
from google.genai.types import DynamicRetrievalConfig, GenerateContentConfig, GoogleSearch, HttpOptions, Tool

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="What is the capital of France?",
    config=GenerateContentConfig(
        tools=[
            Tool(
                google_search=GoogleSearch(
                    dynamic_retrieval_config=DynamicRetrievalConfig(
                        dynamic_threshold=0.6,   # 0.0~1.0: lower = search more often
                    )
                )
            )
        ],
    ),
)
```

### GoogleSearch Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `exclude_domains` | `list[str]` | List of domains to exclude from grounding |
| `dynamic_retrieval_config` | `DynamicRetrievalConfig` | Conditional grounding configuration |
| `dynamic_threshold` | `float` (0.0~1.0) | Threshold for triggering search |

> **Rule:**
> - Recommended temperature: `1.0`.
> - Maximum 1,000,000 queries per day.
> - Search Suggestion HTML/CSS (`searchEntryPoint`) must be displayed as-is without modification.

---

## § 2. Google Maps Grounding

### When: location queries, local search, or nearby place information

```python
from google import genai
from google.genai.types import (
    GenerateContentConfig,
    GoogleMaps,
    HttpOptions,
    LatLng,
    RetrievalConfig,
    Tool,
    ToolConfig,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Where can I get the best espresso near me?",
    config=GenerateContentConfig(
        tools=[
            Tool(google_maps=GoogleMaps(
                enable_widget=False,   # True: include map widget token in response
            ))
        ],
        tool_config=ToolConfig(
            retrieval_config=RetrievalConfig(
                lat_lng=LatLng(
                    latitude=37.5665,    # latitude
                    longitude=126.9780,  # longitude
                ),
                language_code="ko_KR",
            ),
        ),
    ),
)

print(response.text)
```

### GoogleMaps Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `enable_widget` | `bool` | If `True`, include map widget token in response |
| `lat_lng.latitude` | `float` | Reference latitude for search |
| `lat_lng.longitude` | `float` | Reference longitude for search |
| `language_code` | `str` | Response language code (e.g. `"ko_KR"`, `"en_US"`) |

> **Rule:**
> - Maps attribution must be displayed immediately after the generated content.
> - Gemini 3 Pro / Gemini 3 Pro Image: 5,000 queries per day limit.
> - Prohibited regions: China, Cuba, Iran, North Korea, Syria, etc.

---

## § 3. Vertex AI Search Grounding

### When: building RAG from your own documents or website data store

```python
from google import genai
from google.genai.types import (
    GenerateContentConfig,
    HttpOptions,
    Retrieval,
    Tool,
    VertexAISearch,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))

# Data store path format:
# projects/{PROJECT_ID}/locations/global/collections/default_collection/dataStores/{DATASTORE_ID}
DATASTORE_PATH = "projects/my-project/locations/global/collections/default_collection/dataStores/my-store"

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="What are the key features described in the documentation?",
    config=GenerateContentConfig(
        tools=[
            Tool(
                retrieval=Retrieval(
                    vertex_ai_search=VertexAISearch(
                        datastore=DATASTORE_PATH,
                    )
                )
            )
        ],
    ),
)

print(response.text)
# Check sources
for chunk in response.candidates[0].grounding_metadata.grounding_chunks:
    print(chunk.retrieved_context.uri)
```

### Prerequisites

```bash
# 1. Enable AI Applications API
gcloud services enable discoveryengine.googleapis.com

# 2. Grant IAM permissions
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:USER_EMAIL" \
  --role="roles/discoveryengine.viewer"
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `VertexAISearch.datastore` | `str` | Full resource path of the data store |

> **Rule:**
> - Up to 10 data sources can be used simultaneously.
> - Can be used alongside Google Search grounding.
> - `confidence_scores` are not provided for Gemini 2.5 and above.

---

## § 4. Elasticsearch Grounding

### When: using an existing Elasticsearch index as a grounding source

```python
from google import genai
from google.genai.types import (
    ExternalApi,
    GenerateContentConfig,
    HttpOptions,
    Retrieval,
    Tool,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))

ES_ENDPOINT = "https://your-cluster.es.io:443"
ES_API_KEY = "your-elasticsearch-api-key"
INDEX_NAME = "your-index"
SEARCH_TEMPLATE = "your-search-template"

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="What are the main features of product X?",
    config=GenerateContentConfig(
        tools=[
            Tool(
                retrieval=Retrieval(
                    external_api=ExternalApi(
                        api_spec="ELASTIC_SEARCH",
                        endpoint=ES_ENDPOINT,
                        api_auth={
                            "apiKeyConfig": {
                                "apiKeyString": f"ApiKey {ES_API_KEY}"   # "ApiKey " prefix required
                            }
                        },
                        elastic_search_params={
                            "index": INDEX_NAME,
                            "searchTemplate": SEARCH_TEMPLATE,
                            "numHits": 5,
                        },
                    )
                )
            )
        ],
    ),
)

print(response.text)
```

### Elasticsearch Parameters

| Parameter | Location | Description |
|-----------|----------|-------------|
| `api_spec` | `ExternalApi` | Must be `"ELASTIC_SEARCH"` |
| `endpoint` | `ExternalApi` | Elasticsearch cluster URL |
| `api_auth.apiKeyConfig.apiKeyString` | `ExternalApi` | Format: `"ApiKey <your-key>"` — `"ApiKey "` prefix required |
| `elastic_search_params.index` | `ExternalApi` | Index name to search |
| `elastic_search_params.searchTemplate` | `ExternalApi` | Elasticsearch search template name |
| `elastic_search_params.numHits` | `ExternalApi` | Number of results to return |

> **Rule:**
> - `apiKeyString` value must include the `"ApiKey "` prefix.
> - Text input only (multimodal input not supported).
> - Up to 10 data sources can be used simultaneously.

---

## § 5. Custom Search API Grounding

### When: connecting your own search API as a grounding source

```python
from google import genai
from google.genai.types import (
    ExternalApi,
    GenerateContentConfig,
    HttpOptions,
    Retrieval,
    Tool,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))

API_ENDPOINT = "https://your-api-gateway.example.com/v0/search"
API_KEY = "your-api-key"

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="What are the return policy details?",
    config=GenerateContentConfig(
        tools=[
            Tool(
                retrieval=Retrieval(
                    external_api=ExternalApi(
                        api_spec="SIMPLE_SEARCH",   # currently the only allowed value
                        endpoint=API_ENDPOINT,
                        api_auth={
                            "apiKeyConfig": {
                                "apiKeyString": API_KEY
                            }
                        },
                    )
                )
            )
        ],
    ),
)

print(response.text)
```

### API Contract (must comply)

Request format Gemini sends via POST:
```json
{ "query": "search query string" }
```

Required response format:
```json
[
  { "snippet": "relevant text", "uri": "source URL" },
  { "snippet": "second result", "uri": "source2 URL" }
]
```
Return empty array `[]` if no results.

### ExternalApi Parameters

| Parameter | Description |
|-----------|-------------|
| `api_spec` | Must be `"SIMPLE_SEARCH"` |
| `endpoint` | API Gateway endpoint URL |
| `apiKeyConfig.apiKeyString` | API key passed by Gemini as `?key=` query parameter |

> **Rule:**
> - High API response latency increases overall Gemini response time.
> - The quality of `snippet` directly determines grounding response quality.
> - Only API key authentication is supported.

---

## § 6. Parallel Web Search Grounding

### When: using Parallel AI's LLM-optimized web index

> **Note:** Parallel Web Search is currently only available via REST API.

```python
import json
import subprocess

PROJECT_ID = "your-project-id"
LOCATION = "us-central1"
MODEL_ID = "gemini-2.5-flash"
PARALLEL_API_KEY = "your-parallel-api-key"

request_body = {
    "contents": [{"role": "user", "parts": [{"text": "What is the latest news about AI?"}]}],
    "tools": [{
        "parallelAiSearch": {
            "api_key": PARALLEL_API_KEY,
            "customConfigs": {
                "source_policy": {
                    "exclude_domains": [],
                    "include_domains": [],
                },
                "excerpts": {
                    "max_chars_per_result": 30000,   # 1,000~100,000
                    "max_chars_total": 100000,        # 1,000~1,000,000
                },
                "max_results": 10,                   # 1~20
            }
        }
    }],
}

# REST API call example (using gcloud authentication)
# curl -X POST \
#   -H "Authorization: Bearer $(gcloud auth print-access-token)" \
#   -H "Content-Type: application/json" \
#   -d '<request_body>' \
#   "https://LOCATION-aiplatform.googleapis.com/v1/projects/PROJECT_ID/locations/LOCATION/publishers/google/models/MODEL_ID:generateContent"
```

### Parallel Web Search Parameters

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| `api_key` | — | — | Parallel AI API key (required) |
| `exclude_domains` | — | up to 10 | Domains to exclude |
| `include_domains` | — | up to 10 | Domains to include (restrict) |
| `max_chars_per_result` | 30,000 | 1,000~100,000 | Max characters per result |
| `max_chars_total` | 100,000 | 1,000~1,000,000 | Total max characters |
| `max_results` | 10 | 1~20 | Max number of results |

> **Rule:**
> - Default quota: 60 prompts per minute.
> - Parallel's separate terms of service apply.
> - Pre-GA service — no SLA.

---

## § 7. Enterprise Web Search Grounding

### When: web grounding is needed in compliance-regulated industries (healthcare, finance, public sector)

```python
from google import genai
from google.genai.types import (
    EnterpriseWebSearch,
    GenerateContentConfig,
    HttpOptions,
    Tool,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="When is the next total solar eclipse in the United States?",
    config=GenerateContentConfig(
        tools=[
            Tool(enterprise_web_search=EnterpriseWebSearch())   # no parameters
        ],
    ),
)

print(response.text)
```

### Google Search vs Enterprise Web Search

| Item | Google Search | Enterprise Web Search |
|------|--------------|----------------------|
| Index coverage | Full web (broader) | Optimized for regulated industries |
| Freshness | Real-time | Updated every 6h / 24h |
| Data logging | Standard | None |
| VPC-SC support | — | ✅ |
| CMEK | — | N/A |
| Daily query limit | — | 5,000 (Gemini 3 Pro/Pro Image) |

> **Rule:**
> - `EnterpriseWebSearch()` takes no parameters.
> - Search Suggestions (`webSearchQueries`) must be displayed in the app.
> - If compliance is not required, use Google Search for broader index coverage.
