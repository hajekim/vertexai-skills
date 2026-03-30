---
name: grounding-skills
description: Use when working with grounding on Vertex AI using google-genai Python SDK — imports GoogleSearch, GoogleMaps, VertexAISearch, Elasticsearch, ExternalApi, EnterpriseWebSearch, or requests involving "grounding", "Google Search grounding", "RAG", "Vertex AI Search", "Elasticsearch grounding", "enterprise web search". 한국어: "그라운딩", "검색 기반 생성", "구글 검색 연동", "버텍스 검색", "환각 방지", "외부 데이터 연결".
version: 1.0.0
---

# Grounding Skills — google-genai Python SDK 레퍼런스

## 환경 설정

### 라이브러리 설치

```bash
pip install --upgrade google-genai
```

### 필수 환경변수

```bash
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"
export GOOGLE_GENAI_USE_VERTEXAI="True"
```

### 클라이언트 초기화

```python
from google import genai
from google.genai.types import HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))
```

### 그라운딩 방식 선택 가이드

| 방식 | Tool 클래스 | 용도 |
|------|------------|------|
| Google Search | `GoogleSearch` | 최신 공개 웹 데이터 |
| Google Maps | `GoogleMaps` | 장소/지리 정보 |
| Vertex AI Search | `VertexAISearch` (via `Retrieval`) | 내 문서/웹사이트 데이터 |
| Elasticsearch | `Elasticsearch` (via `Retrieval`) | 기존 ES 인덱스 |
| Custom Search API | `ExternalApi` (via `Retrieval`) | 자체 검색 API |
| Parallel Web Search | REST only (`parallelAiSearch`) | LLM 최적화 웹 인덱스 |
| Enterprise Web Search | `EnterpriseWebSearch` | 컴플라이언스 규제 환경 |

> **참고:** Vertex AI RAG Engine 그라운딩(`ground-responses-using-rag`)은 `vertexai` SDK(`google-cloud-aiplatform`)를 사용하며, 이 스킬의 `google-genai` SDK 범위에 포함되지 않는다.
