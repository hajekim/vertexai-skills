---
name: model-skills
description: >
  Vertex AI Studio model selection, migration, and pricing guide. Triggered by:
  "model selection", "model lifecycle", "deprecated model", "model migration",
  "gemini migration", "1.5 to 2.5", "model retirement", "which model",
  "SDK migration", "standard paygo", "usage tier", "quota", "TPM", "RPM", "429 error",
  "provisioned throughput". Korean: "모델 선택", "모델 라이프사이클", "deprecated 모델",
  "모델 마이그레이션", "어떤 모델", "모델 추천", "SDK 마이그레이션", "모델 은퇴",
  "스탠다드 페이고", "사용량 티어", "할당량", "429 에러", "프로비전드 처리량".
version: 1.0.0
---

# Model Skills — Vertex AI Studio Model Selection and Migration Guide

> Reference: [Model versions and lifecycle](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/model-versions) · [Migration guide](https://cloud.google.com/vertex-ai/generative-ai/docs/migrate) · [Standard PayGo](https://cloud.google.com/vertex-ai/generative-ai/docs/standard-paygo)

---

## Model Lifecycle Concepts

| Status | Meaning | Production Suitable |
|--------|---------|-------------------|
| **Stable (GA)** | Publicly released. New access blocked 1 month before retirement date | ✅ Recommended |
| **Preview** | Under validation. API may change | ⚠️ Not recommended |
| **Retired** | Returns 404 after retirement date. Unusable | ❌ Not available |

---

## § 1. LLM Model Selection

### Current Stable Models (as of 2025)

| Model ID | Released | Retires | Description |
|----------|----------|---------|-------------|
| `gemini-2.5-flash` | 2025-06-17 | 2026-06-17 | Speed/cost balance **(recommended default for new projects)** |
| `gemini-2.5-pro` | 2025-06-17 | 2026-06-17 | High-quality complex reasoning |
| `gemini-2.5-flash-lite` | 2025-07-22 | 2026-07-22 | Lightweight, cost-optimized |
| `gemini-2.0-flash-001` | 2025-02-05 | 2026-06-01 | Existing projects only — new use blocked since 2026-03-06 |

### Retired Models — Already End-of-Life, Replace Immediately

> API calls return **404 error**. Replace now if still in use.

| Model ID | Retired | Replacement |
|----------|---------|-------------|
| `gemini-1.5-pro-001` | 2025-05-24 | `gemini-2.5-pro` |
| `gemini-1.5-pro-002` | 2025-09-24 | `gemini-2.5-pro` |
| `gemini-1.5-flash-001` | 2025-05-24 | `gemini-2.5-flash` |
| `gemini-1.5-flash-002` | 2025-09-24 | `gemini-2.5-flash` |
| `gemini-1.0-pro-001` | 2025-04-21 | `gemini-2.5-flash` |
| `gemini-1.0-pro-002` | 2025-04-21 | `gemini-2.5-flash` |
| `gemini-1.0-pro-vision-001` | 2025-04-21 | `gemini-2.5-flash` |

### Selection by Use Case

```
Minimize cost              → gemini-2.5-flash-lite
Speed/quality balance      → gemini-2.5-flash      ← most use cases
High-quality reasoning     → gemini-2.5-pro
Thinking (budget control)  → gemini-2.5-flash / gemini-2.5-pro  (thinking_budget)
Thinking (level control)   → gemini-3-flash / gemini-3.1-pro    (thinking_level)
```

### Auto-update Alias Warning

| Alias | Currently Points To | Warning |
|-------|-------------------|---------|
| `gemini-2.5-pro` | `gemini-2.5-pro-001` | Auto-upgrades when a new version is released |
| `gemini-2.5-flash` | `gemini-2.5-flash-001` | Same |
| `gemini-2.0-flash` | `gemini-2.0-flash-001` | Same |

> **In production, use pinned version IDs like `gemini-2.5-flash-001` instead of aliases.** Aliases auto-upgrade and may change behavior.

---

## § 2. Image Model Selection

### Current Stable Models

| Model ID | API | Released | Retires | Description |
|----------|-----|----------|---------|-------------|
| `imagen-4.0-generate-001` | `generate_images()` | 2025-08-14 | 2026-06-30 | Highest quality static images **(recommended)** |
| `imagen-3.0-generate-002` | `generate_images()` | 2025-01-29 | 2026-06-30 | Imagen 3 stable version |
| `gemini-2.5-flash-image` | `generate_content()` | — | — | Image generation + editing, lightweight |

### Preview Models (not recommended for production)

| Model ID | API | Description |
|----------|-----|-------------|
| `gemini-3-pro-image-preview` | `generate_content()` | Highest quality generation + editing |
| `gemini-3.1-flash-image-preview` | `generate_content()` | Latest, fast generation + editing |

### Selection by Use Case

```
Highest quality static images       → imagen-4.0-generate-001   (generate_images)
Image generation + mixed with text  → gemini-2.5-flash-image    (generate_content)
Latest editing features (experimental) → gemini-3.1-flash-image-preview
```

---

## § 3. Video Model Selection

### Current Stable Models

| Model ID | Released | Retires | Description |
|----------|----------|---------|-------------|
| `veo-3.1-generate-001` | 2025-11-17 | TBD | Latest stable version **(recommended)** |
| `veo-3.1-fast-generate-001` | 2025-11-17 | TBD | Faster generation, lower cost |
| `veo-3.0-generate-001` | 2025-07-29 | 2026-06-30 | Previous stable version |
| `veo-2.0-generate-001` | — | — | Object insert/remove and `enhance_prompt` only |

### Selection by Use Case

```
General video generation      → veo-3.1-generate-001
Fast generation / cost saving → veo-3.1-fast-generate-001
Object insert or remove       → veo-2.0-generate-001  (exclusive to 2.0)
enhance_prompt usage          → veo-2.0-generate-001
```

---

## § 4. Migration Guide

### 4-1. SDK Migration (Required)

The Vertex AI SDK (`google-cloud-aiplatform`) will stop supporting Gemini after June 2026. Migrate to the Gen AI SDK.

```bash
# Remove old SDK
pip uninstall google-cloud-aiplatform

# Install Gen AI SDK
pip install --upgrade google-genai
```

```python
# Before (Vertex AI SDK)
import vertexai
from vertexai.generative_models import GenerativeModel

vertexai.init(project="my-project", location="us-central1")
model = GenerativeModel("gemini-1.5-flash")
response = model.generate_content("Hello")

# After (Gen AI SDK)
from google import genai
from google.genai.types import HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Hello",
)
```

### 4-2. Model ID Changes

```python
# 1.5 → 2.5 direct replacement (1.5 is fully retired — returning 404)
"gemini-1.5-flash-001"  →  "gemini-2.5-flash"   # retired 2025-05-24
"gemini-1.5-flash-002"  →  "gemini-2.5-flash"   # retired 2025-09-24
"gemini-1.5-pro-001"    →  "gemini-2.5-pro"     # retired 2025-05-24
"gemini-1.5-pro-002"    →  "gemini-2.5-pro"     # retired 2025-09-24

# 1.0 → 2.5 (1.0 retired 2025-04-21)
"gemini-1.0-pro-001"         →  "gemini-2.5-flash"
"gemini-1.0-pro-vision-001"  →  "gemini-2.5-flash"

# 2.0 → 2.5 (new projects — new access blocked since 2026-03-06)
"gemini-2.0-flash-001"  →  "gemini-2.5-flash"
```

### 4-3. Breaking Changes Checklist

#### Thinking parameter (when upgrading to Gemini 3 or later)

```python
# Before (Gemini 2.5 — thinking_budget)
from google.genai.types import GenerateContentConfig, ThinkingConfig

config = GenerateContentConfig(
    thinking_config=ThinkingConfig(thinking_budget=1024),
)

# After (Gemini 3+ — thinking_level)
from google.genai.types import GenerateContentConfig, ThinkingConfig, ThinkingLevel

config = GenerateContentConfig(
    thinking_config=ThinkingConfig(thinking_level=ThinkingLevel.HIGH),
)
```

#### Remove Top-K parameter (not supported after Gemini 1.0 Pro Vision)

```python
# Before
config = GenerateContentConfig(top_k=40)  # worked on some models

# After — remove top_k, use only temperature / top_p
config = GenerateContentConfig(
    temperature=1.0,
    top_p=0.95,
)
```

#### Dynamic Retrieval → Google Search Grounding

```python
# Before (Dynamic Retrieval — deprecated)
from google.genai.types import DynamicRetrievalConfig, GoogleSearchRetrieval, Tool

tool = Tool(google_search_retrieval=GoogleSearchRetrieval(
    dynamic_retrieval_config=DynamicRetrievalConfig(dynamic_threshold=0.7),
))

# After (Google Search Grounding)
from google.genai.types import GoogleSearch, Tool

tool = Tool(google_search=GoogleSearch())
```

#### temperature default warning (Gemini 3 and later)

```python
# Gemini 3+ models: keep temperature default at 1.0
# Lowering it may cause repetitive responses or performance degradation

config = GenerateContentConfig(
    temperature=1.0,  # explicitly set to 1.0 (maintain default)
)
```

### 4-4. Recommended Migration Procedure (7 steps)

```
1. Document evaluation requirements
   └─ Collect existing eval data; organize by RAG/Tool use/Agent component

2. Upgrade code
   └─ Switch to Gen AI SDK → update model IDs → fix breaking changes

3. Run offline evaluation
   └─ Repeat existing evaluations (use Gen AI Evaluation Service)

4. Adjust prompts
   └─ Iterative improvement via hill climbing; review system instructions (remove contradictions)

5. Load test
   └─ Confirm whether to pre-purchase Provisioned Throughput

6. Online evaluation (optional)
   └─ A/B test or canary deployment

7. Production deployment
```

### 4-5. Pre-migration Checklist

| Item | Details |
|------|---------|
| **Security & compliance** | Obtain InfoSec/regulatory approval first |
| **Regional availability** | Check supported regions per model (`us-central1` recommended) |
| **Pricing change** | Gemini 2+ models bill by token, not character |
| **Fine-tuning** | Existing tuned models cannot be reused; new tuning required |
| **Image/PDF tokenization** | Gemini 3+: Pan and Scan → variable sequence length |
| **PDF processing** | Gemini 3+: OCR disabled by default |
| **Image Segmentation** | Not supported on Gemini 3+ |

> **Warning:** Do not perform a full migration all at once without thorough testing. Gradual rollout is recommended.

---

## § 5. Standard PayGo — Pricing, Quotas, and Throughput

> Reference: [Standard PayGo](https://cloud.google.com/vertex-ai/generative-ai/docs/standard-paygo)

### 5-1. What is Standard PayGo?

Consumption-based pricing with no upfront commitment. Throughput limits automatically increase as your rolling 30-day organizational spend grows across three tiers.

### 5-2. Usage Tiers

Tier is determined by **rolling 30-day total spend across all eligible Vertex AI services** in your organization.

#### Gemini Pro Models (`gemini-2.5-pro`, `gemini-3-pro`, etc.)

| Tier | 30-Day Spend | Baseline TPM |
|------|-------------|-------------|
| Tier 1 | $10 – $250 | 500,000 |
| Tier 2 | $250 – $2,000 | 1,000,000 |
| Tier 3 | > $2,000 | 2,000,000 |

#### Gemini Flash & Flash-Lite Models (`gemini-2.5-flash`, `gemini-2.5-flash-lite`, etc.)

| Tier | 30-Day Spend | Baseline TPM |
|------|-------------|-------------|
| Tier 1 | $10 – $250 | 2,000,000 |
| Tier 2 | $250 – $2,000 | 4,000,000 |
| Tier 3 | > $2,000 | 10,000,000 |

> **Key facts:**
> - TPM limits are **independent per model** — `gemini-2.5-flash` and `gemini-2.5-pro` each have their own separate limit.
> - System-wide RPM cap: **30,000 RPM per model per region** (applies across all tiers).
> - There is no per-tier RPM cap — only the global 30,000 RPM system limit.

### 5-3. Models Supporting Standard PayGo

**With usage tiers (GA Gemini models):**
- `gemini-3.1-pro`, `gemini-3.1-flash-lite`, `gemini-3.1-flash-image`
- `gemini-3-pro`, `gemini-3-flash`, `gemini-3-pro-image`
- `gemini-2.5-pro`, `gemini-2.5-flash`, `gemini-2.5-flash-lite`
- All supervised fine-tuned variants of the above

**Standard PayGo without tiers:**
- `gemini-2.5-flash-image`, `imagen-4.0-*` (all variants), Virtual Try-On

### 5-4. Spend Calculation — What Counts Toward Your Tier

| Category | Included |
|----------|----------|
| Gemini model usage | ✅ Text, image, audio, video (including batch and tuned models) |
| Gemini features | ✅ Context caching, storage, priority tiers |
| Vertex AI compute | ✅ CPU, GPU, TPU instances |
| Provisioned Throughput commitments | ✅ |

### 5-5. Traffic Behavior and Best Practices

> **Important:** Traffic is **not strictly capped** at the Baseline TPM. Vertex AI allows bursting beyond the limit on a best-effort basis. A 429 error indicates **temporary high resource contention**, not a fixed quota violation.

**To maximize throughput and minimize throttling:**

```
1. Use the global endpoint (not a regional endpoint)
   → Access a larger multi-region capacity pool

2. Spread requests evenly across each minute
   → Avoid second-level spikes even if per-minute average is within limits

3. Implement exponential backoff on 429 errors
   → Do not retry immediately; back off and retry gradually
```

**Exponential backoff pattern:**

```python
import time
import random
from google import genai
from google.genai.types import HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))

def generate_with_backoff(prompt: str, max_retries: int = 5) -> str:
    for attempt in range(max_retries):
        try:
            response = client.models.generate_content(
                model="gemini-2.5-flash",
                contents=prompt,
            )
            return response.text
        except Exception as e:
            if "429" in str(e) or "Resource exhausted" in str(e):
                if attempt == max_retries - 1:
                    raise
                wait = (2 ** attempt) + random.uniform(0, 1)  # exponential backoff + jitter
                print(f"Rate limited. Retrying in {wait:.1f}s (attempt {attempt + 1}/{max_retries})")
                time.sleep(wait)
            else:
                raise
    return ""
```

### 5-6. Standard PayGo vs Provisioned Throughput

| Item | Standard PayGo | Provisioned Throughput |
|------|---------------|----------------------|
| Commitment | None (pay per use) | Pre-purchased capacity |
| Throughput | Dynamic (burst allowed) | Dedicated and assured |
| SLA | None | Provided |
| Performance variability | Higher during congestion | Low |
| Cost | Lower at low/medium volume | Predictable at high volume |
| **Best for** | Development, variable workloads | Mission-critical production |

> **When to choose Provisioned Throughput:** consistent high-volume traffic, strict latency SLAs, or workloads that cannot tolerate throttling.

### 5-7. Monitoring Quota Usage

Use Cloud Monitoring's Metrics Explorer to track real-time token consumption at the organization level.

```bash
# View current quota usage via gcloud
gcloud monitoring metrics list \
  --filter="metric.type:aiplatform.googleapis.com/quota"
```

> For multi-project observability, configure observability scopes in Cloud Monitoring.
