---
name: embedding-skills
description: Use when working with text or multimodal embeddings on Vertex AI using google-genai Python SDK — imports embed_content, or requests involving "embedding", "text embedding", "vector embedding", "semantic similarity", "multimodal embedding", "batch embedding", "embed_content", "task type". Korean: "임베딩", "텍스트 임베딩", "벡터 임베딩", "의미적 유사도", "멀티모달 임베딩", "배치 임베딩", "임베드".
version: 1.0.0
---

# Embedding Skills — google-genai Python SDK Reference

> Reference: [Text embeddings](https://cloud.google.com/vertex-ai/generative-ai/docs/embeddings/get-text-embeddings) · [Multimodal embeddings](https://cloud.google.com/vertex-ai/generative-ai/docs/embeddings/get-multimodal-embeddings) · [Batch embeddings](https://cloud.google.com/vertex-ai/generative-ai/docs/embeddings/batch-prediction-genai-embeddings)

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

### Model Selection Guide

| Model | Dimensions | Token Limit | Batch | Best For |
|-------|-----------|-------------|-------|----------|
| `gemini-embedding-001` | Up to 3072 | 2048 | ❌ | High-accuracy text retrieval |
| `text-embedding-005` | 768 | — | ✅ | General text embedding |
| `text-multilingual-embedding-002` | 768 | — | ✅ | Multilingual text embedding |
| `gemini-embedding-2-preview` | Up to 3072 | — | — | Multimodal (text/image/video/audio/PDF) |
| `multimodalembedding@001` | 1408 | — | — | Multimodal (text + image) |

> **Model selection tip:**
> - New projects → `gemini-embedding-001` (highest quality)
> - Multilingual support needed → `text-multilingual-embedding-002`
> - Images, video, audio, or PDF → `gemini-embedding-2-preview`

---

## § 1. Text Embeddings

### When: converting text into a vector representation for semantic search or similarity comparison

```python
from google import genai
from google.genai.types import EmbedContentConfig, HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.embed_content(
    model="gemini-embedding-001",
    contents="What is the meaning of life?",
    config=EmbedContentConfig(
        task_type="RETRIEVAL_DOCUMENT",   # specify the purpose of the embedding
        output_dimensionality=768,        # reduce dimensions (optional; max 3072)
    ),
)

print(response.embeddings[0].values)   # vector as a list of floats
```

### Batch Text Embedding (multiple texts at once)

```python
from google import genai
from google.genai.types import EmbedContentConfig, HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))

texts = [
    "How do I reset my password?",
    "What is the return policy?",
    "How can I track my order?",
]

response = client.models.embed_content(
    model="gemini-embedding-001",
    contents=texts,   # pass a list of strings
    config=EmbedContentConfig(
        task_type="RETRIEVAL_DOCUMENT",
    ),
)

for i, embedding in enumerate(response.embeddings):
    print(f"Text {i}: {len(embedding.values)} dimensions")
```

### Request Limits

| Limit | Value |
|-------|-------|
| Max texts per request | 250 |
| Max total tokens per request | 20,000 |
| Max tokens per text (`gemini-embedding-001`) | 2,048 |

> **Rule:**
> - Texts exceeding the token limit are silently truncated — split long documents into chunks first.
> - `output_dimensionality` reduces the embedding dimensions. Use lower values to save storage.
> - `gemini-embedding-001` does NOT support batch prediction jobs (§ 4).

---

## § 2. Task Types

### When: choosing the right task type improves embedding accuracy for your use case

| Task Type | Description | Typical Use Case |
|-----------|-------------|-----------------|
| `RETRIEVAL_DOCUMENT` | Index documents for retrieval | Document store, FAQ corpus |
| `RETRIEVAL_QUERY` | Encode a search query | Semantic search input |
| `CLASSIFICATION` | Classification-focused embedding | Sentiment analysis, category classification |
| `CLUSTERING` | Clustering-focused embedding | Topic modeling, document grouping |
| `SEMANTIC_SIMILARITY` | Measure similarity between two texts | Duplicate detection, paraphrase matching |
| `QUESTION_ANSWERING` | Encode a question for Q&A | Question → answer matching |
| `FACT_VERIFICATION` | Encode a claim for fact checking | Claim vs. source comparison |
| `CODE_RETRIEVAL_QUERY` | Encode a natural language code query | Code search |

```python
# Asymmetric retrieval: use RETRIEVAL_QUERY for the query, RETRIEVAL_DOCUMENT for documents
query_response = client.models.embed_content(
    model="gemini-embedding-001",
    contents="How do I fix a 429 error?",
    config=EmbedContentConfig(task_type="RETRIEVAL_QUERY"),
)

doc_response = client.models.embed_content(
    model="gemini-embedding-001",
    contents="A 429 error means rate limiting. Use exponential backoff to retry.",
    config=EmbedContentConfig(task_type="RETRIEVAL_DOCUMENT"),
)
```

> **Rule:**
> - For asymmetric retrieval (query ↔ document), always use different task types for query and document.
> - For symmetric similarity (text ↔ text), use `SEMANTIC_SIMILARITY` for both.

---

## § 3. Multimodal Embeddings

### When: embedding images, video, audio, or PDF alongside text in a unified vector space

```python
from google import genai
from google.genai.types import EmbedContentConfig, HttpOptions, Part

client = genai.Client(http_options=HttpOptions(api_version="v1"))

# Text + Image combined embedding
response = client.models.embed_content(
    model="gemini-embedding-2-preview",
    contents=[
        Part.from_text("A dog running in the park"),
        Part.from_uri(
            uri="gs://your-bucket/dog.jpg",
            mime_type="image/jpeg",
        ),
    ],
    config=EmbedContentConfig(
        output_dimensionality=1408,
    ),
)

print(response.embeddings[0].values)
```

### Supported Modalities

| Modality | Model | Notes |
|----------|-------|-------|
| Text | `gemini-embedding-2-preview` | Combined with other modalities |
| Image (JPEG/PNG) | `gemini-embedding-2-preview` | GCS URI or inline bytes |
| Video | `gemini-embedding-2-preview` | GCS URI |
| Audio | `gemini-embedding-2-preview` | GCS URI |
| PDF | `gemini-embedding-2-preview` | GCS URI |
| Text + Image | `multimodalembedding@001` | 1408 dimensions |

### `multimodalembedding@001` (text + image only)

```python
# For text-image embedding with the older multimodal model
response = client.models.embed_content(
    model="multimodalembedding@001",
    contents=[
        Part.from_text("A golden retriever"),
        Part.from_uri(
            uri="gs://your-bucket/dog.jpg",
            mime_type="image/jpeg",
        ),
    ],
)
```

> **Rule:**
> - `gemini-embedding-2-preview` is Preview — not recommended for production.
> - All embeddings from the same model share a unified vector space — cross-modal similarity comparisons are valid.

---

## § 4. Batch Embedding (Large Scale)

### When: embedding 1,000+ documents offline — not real-time

> **Note:** `gemini-embedding-001` does NOT support batch prediction. Use `text-embedding-005` or `text-multilingual-embedding-002`.

```python
from google.cloud import aiplatform

# Batch prediction uses the Vertex AI SDK (google-cloud-aiplatform), not google-genai
aiplatform.init(project="your-project-id", location="us-central1")

batch_prediction_job = aiplatform.BatchPredictionJob.create(
    job_display_name="text-embedding-batch",
    model_name="publishers/google/models/text-embedding-005",
    instances_format="jsonl",
    predictions_format="jsonl",
    gcs_source=["gs://your-bucket/input.jsonl"],
    gcs_destination_prefix="gs://your-bucket/output/",
)

print(batch_prediction_job.resource_name)
```

### Input JSONL Format

```jsonl
{"content": "First document to embed", "task_type": "RETRIEVAL_DOCUMENT"}
{"content": "Second document to embed", "task_type": "RETRIEVAL_DOCUMENT"}
{"content": "Third document to embed", "task_type": "RETRIEVAL_DOCUMENT"}
```

### Output JSONL Format

```jsonl
{"instance": {"content": "First document..."}, "predictions": [{"embeddings": {"values": [0.01, -0.02, ...]}}]}
```

### Batch Limits

| Limit | Value |
|-------|-------|
| Max prompts per job | 30,000 |
| Concurrent jobs per project | 1 |
| Input format | JSONL or BigQuery |
| Output format | JSONL or BigQuery |

> **Rule:**
> - Batch jobs run asynchronously. Poll `batch_prediction_job.state` until `JobState.JOB_STATE_SUCCEEDED`.
> - Only one batch job can run per project at a time.
> - Use `text-embedding-005` or `text-multilingual-embedding-002` for batch — `gemini-embedding-001` is not supported.
