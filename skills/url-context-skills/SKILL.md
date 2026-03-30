---
name: url-context-skills
description: Use when working with URL context grounding on Vertex AI using google-genai Python SDK — imports UrlContext, or requests involving "url context", "URL grounding", "fetch URL", "web page context", "browse URL", "read URL". Korean: "URL 컨텍스트", "URL 그라운딩", "웹 페이지 읽기", "URL 기반 응답", "링크 내용 참조".
version: 1.0.0
---

# URL Context Skills — google-genai Python SDK Reference

> Reference: [URL context](https://cloud.google.com/vertex-ai/generative-ai/docs/grounding/url-context)

---

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

---

## § 1. Basic URL Context

### When: grounding a response using content from one or more URLs

```python
from google import genai
from google.genai.types import (
    GenerateContentConfig,
    HttpOptions,
    Tool,
    UrlContext,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=[
        "Summarize the key points from this documentation: "
        "https://cloud.google.com/vertex-ai/generative-ai/docs/overview"
    ],
    config=GenerateContentConfig(
        tools=[
            Tool(url_context=UrlContext())   # no parameters required
        ],
    ),
)

print(response.text)
```

> **Rule:**
> - Include the URL directly in the `contents` text — Gemini detects and fetches it.
> - `UrlContext()` takes no parameters.
> - Maximum 20 URLs per request.

---

## § 2. Multiple URLs

### When: comparing or synthesizing content from multiple web pages

```python
from google import genai
from google.genai.types import GenerateContentConfig, HttpOptions, Tool, UrlContext

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=(
        "Compare the pricing models described in these two pages:\n"
        "1. https://cloud.google.com/vertex-ai/pricing\n"
        "2. https://cloud.google.com/vertex-ai/generative-ai/docs/standard-paygo"
    ),
    config=GenerateContentConfig(
        tools=[Tool(url_context=UrlContext())],
    ),
)

print(response.text)
```

---

## § 3. Checking URL Retrieval Status

### When: verifying which URLs were successfully fetched and used

```python
from google import genai
from google.genai.types import GenerateContentConfig, HttpOptions, Tool, UrlContext

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="What are the main features at https://example.com?",
    config=GenerateContentConfig(
        tools=[Tool(url_context=UrlContext())],
    ),
)

print(response.text)

# Check URL retrieval status
if response.candidates[0].grounding_metadata:
    metadata = response.candidates[0].grounding_metadata
    if hasattr(metadata, "url_retrieval_status"):
        for status in metadata.url_retrieval_status:
            print(f"URL: {status.url}")
            print(f"Status: {status.status}")   # SUCCESS or FAILED
```

### URL Retrieval Status Values

| Status | Meaning |
|--------|---------|
| `SUCCESS` | URL was fetched and content used for grounding |
| `FAILED` | URL could not be fetched (access denied, timeout, etc.) |

---

## § 4. URL Context with Google Search (Combined)

### When: combining live URL fetching with Google Search grounding

```python
from google import genai
from google.genai.types import (
    GenerateContentConfig,
    GoogleSearch,
    HttpOptions,
    Tool,
    UrlContext,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=(
        "Based on the content at https://example.com/product "
        "and the latest reviews online, summarize pros and cons."
    ),
    config=GenerateContentConfig(
        tools=[
            Tool(url_context=UrlContext()),
            Tool(google_search=GoogleSearch()),
        ],
    ),
)

print(response.text)
```

---

## § 5. Supported Content Types and Retrieval Behavior

### Supported URL Content Types

| Content Type | Supported |
|-------------|-----------|
| HTML web pages | ✅ |
| JSON data | ✅ |
| PDF documents | ✅ |
| Images (JPEG, PNG, etc.) | ✅ |
| Plain text | ✅ |
| Protected/login-required pages | ❌ |
| Dynamic JavaScript-rendered content | ⚠️ Partial |

### Two-Stage Retrieval Process

URL Context uses a two-stage retrieval approach:

1. **Index stage:** Searches a pre-built index of crawled URLs for relevant content chunks
2. **Live fetch stage:** If the URL is not in the index or content is too recent, fetches the URL directly

> **Rule:**
> - URLs requiring authentication or login are not accessible.
> - Very recent content (not yet indexed) triggers a live fetch — latency may be higher.
> - Include URLs directly in your prompt text — do not pass them as separate parameters.
> - Maximum 20 URLs per request; additional URLs beyond the limit are ignored.
> - URL context counts toward your token usage — long pages increase cost.

### Token Counting with URL Context

```python
from google import genai
from google.genai.types import GenerateContentConfig, HttpOptions, Tool, UrlContext

client = genai.Client(http_options=HttpOptions(api_version="v1"))

# Count tokens before sending (includes URL content)
count_response = client.models.count_tokens(
    model="gemini-2.5-flash",
    contents="Summarize: https://cloud.google.com/vertex-ai/generative-ai/docs/overview",
    config=GenerateContentConfig(
        tools=[Tool(url_context=UrlContext())],
    ),
)

print(f"Estimated tokens: {count_response.total_tokens}")
```
