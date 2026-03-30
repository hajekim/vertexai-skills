---
name: translate-skills
description: Use when working with translation on Vertex AI using google-genai Python SDK or REST API — requests involving "translate", "translation", "translate text", "language translation", "NMT", "Translation LLM", "cloud translate". Korean: "번역", "텍스트 번역", "언어 번역", "자동 번역", "다국어 번역".
version: 1.0.0
---

# Translate Skills — Vertex AI Translation Reference

> Reference: [Translation overview](https://cloud.google.com/vertex-ai/generative-ai/docs/translate/translate-text)

---

## Environment Setup

### Install Libraries

```bash
pip install --upgrade google-genai
pip install google-cloud-translate  # for advanced NMT features
```

### Required Environment Variables

```bash
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"
export GOOGLE_GENAI_USE_VERTEXAI="True"
```

### Translation Method Selection Guide

| Method | API | Latency | Languages | Best For |
|--------|-----|---------|-----------|----------|
| **Translation LLM** | REST `cloud-translate-text` | ~1–3s | 20+ | High-quality, Gemini-powered translation |
| **NMT (Neural Machine Translation)** | Cloud Translation API | ~100ms | 100+ | Low-latency, broad language coverage |

> **When to choose:**
> - High translation quality, cultural nuance, or document context → Translation LLM
> - Real-time or low-latency requirements, broader language support → NMT

---

## § 1. Translation LLM (Gemini-Powered)

### When: high-quality translation with context awareness and cultural nuance

```python
import json
import subprocess
import os

PROJECT_ID = os.environ["GOOGLE_CLOUD_PROJECT"]
LOCATION = "us-central1"

request_body = {
    "instances": [
        {
            "content": "The weather is nice today.",
        }
    ],
    "parameters": {
        "sourceLanguageCode": "en",
        "targetLanguageCode": "ko",
    },
}

# REST API call using gcloud authentication
result = subprocess.run(
    [
        "curl", "-s", "-X", "POST",
        "-H", f"Authorization: Bearer $(gcloud auth print-access-token)",
        "-H", "Content-Type: application/json",
        "-d", json.dumps(request_body),
        f"https://{LOCATION}-aiplatform.googleapis.com/v1/projects/{PROJECT_ID}/locations/{LOCATION}/publishers/google/models/cloud-translate-text:predict",
    ],
    capture_output=True,
    text=True,
    shell=False,
)

print(result.stdout)
```

### Recommended: Using requests library

```python
import os
import requests
import google.auth
import google.auth.transport.requests

PROJECT_ID = os.environ["GOOGLE_CLOUD_PROJECT"]
LOCATION = "us-central1"

# Get access token
credentials, _ = google.auth.default()
auth_request = google.auth.transport.requests.Request()
credentials.refresh(auth_request)

url = (
    f"https://{LOCATION}-aiplatform.googleapis.com/v1"
    f"/projects/{PROJECT_ID}/locations/{LOCATION}"
    f"/publishers/google/models/cloud-translate-text:predict"
)

payload = {
    "instances": [
        {"content": "The weather is nice today."},
        {"content": "I would like to make a reservation."},
    ],
    "parameters": {
        "sourceLanguageCode": "en",
        "targetLanguageCode": "ko",
    },
}

response = requests.post(
    url,
    headers={
        "Authorization": f"Bearer {credentials.token}",
        "Content-Type": "application/json",
    },
    json=payload,
)

data = response.json()
for prediction in data.get("predictions", []):
    print(prediction["translations"][0]["translatedText"])
```

### With Reference Pairs (Terminology Customization)

```python
payload = {
    "instances": [
        {"content": "Please submit the pull request for review."},
    ],
    "parameters": {
        "sourceLanguageCode": "en",
        "targetLanguageCode": "ko",
        "referenceSentencePairLists": [
            {
                "referenceSentencePairs": [
                    {
                        "sourceReferenceSentence": "pull request",
                        "targetReferenceSentence": "풀 리퀘스트",  # force specific terminology
                    },
                    {
                        "sourceReferenceSentence": "code review",
                        "targetReferenceSentence": "코드 리뷰",
                    },
                ]
            }
        ],
    },
}
```

### Translation LLM Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `sourceLanguageCode` | `str` | Source language BCP-47 code (e.g. `"en"`, `"ko"`, `"ja"`) |
| `targetLanguageCode` | `str` | Target language BCP-47 code |
| `mimeType` | `str` | `"text/plain"` (default) or `"text/html"` |
| `referenceSentencePairLists` | `list` | Custom terminology/style pairs to guide translation |

> **Rule:**
> - Batch up to 1,024 instances per request.
> - `referenceSentencePairLists` accepts up to 3 lists, each with up to 3 pairs.
> - Source language auto-detection is not supported — always provide `sourceLanguageCode`.

---

## § 2. NMT (Neural Machine Translation)

### When: real-time, low-latency translation with broad language support

```python
from google.cloud import translate_v2 as translate

client = translate.Client()

result = client.translate(
    "The weather is nice today.",
    source_language="en",
    target_language="ko",
)

print(result["translatedText"])   # 오늘 날씨가 좋네요.
```

### Batch Translation with NMT

```python
from google.cloud import translate_v3

PROJECT_ID = "your-project-id"
client = translate_v3.TranslationServiceClient()
parent = f"projects/{PROJECT_ID}/locations/us-central1"

response = client.translate_document(
    request={
        "parent": parent,
        "source_language_code": "en-US",
        "target_language_codes": ["ko"],
        "input_configs": [
            {
                "gcs_source": {"input_uri": "gs://your-bucket/document.pdf"},
                "mime_type": "application/pdf",
            }
        ],
        "output_config": {
            "gcs_destination": {"output_uri_prefix": "gs://your-bucket/output/"}
        },
    }
)
```

### NMT vs Translation LLM Comparison

| Item | Translation LLM | NMT |
|------|----------------|-----|
| Quality | High (context-aware) | Good |
| Latency | ~1–3 seconds | ~100ms |
| Supported languages | 20+ | 100+ |
| Terminology control | ✅ (reference pairs) | ✅ (glossaries) |
| Document translation | — | ✅ (PDF, DOCX, etc.) |
| API | Vertex AI predict | Cloud Translation API |

> **Rule:**
> - For user-facing real-time translation, prefer NMT for low latency.
> - For content quality where latency is acceptable, use Translation LLM.
> - Document translation (PDF, DOCX) is only supported by NMT.
