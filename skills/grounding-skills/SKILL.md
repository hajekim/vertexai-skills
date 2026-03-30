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

---

## § 1. Google Search 그라운딩

### 언제: 최신 공개 웹 정보로 응답을 보강할 때

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
                    exclude_domains=["domain.com"],   # 제외할 도메인 (선택)
                )
            )
        ],
    ),
)

print(response.text)
# 검색 출처 확인
for chunk in response.candidates[0].grounding_metadata.grounding_chunks:
    print(chunk.web.uri)
```

### Dynamic Retrieval (조건부 그라운딩)

```python
from google.genai.types import DynamicRetrievalConfig, GoogleSearch, Tool

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="What is the capital of France?",
    config=GenerateContentConfig(
        tools=[
            Tool(
                google_search=GoogleSearch(
                    dynamic_retrieval_config=DynamicRetrievalConfig(
                        dynamic_threshold=0.6,   # 0.0~1.0: 낮을수록 더 자주 검색
                    )
                )
            )
        ],
    ),
)
```

### GoogleSearch 파라미터

| 파라미터 | 타입 | 설명 |
|---------|------|------|
| `exclude_domains` | `list[str]` | 그라운딩에서 제외할 도메인 목록 |
| `dynamic_retrieval_config` | `DynamicRetrievalConfig` | 조건부 그라운딩 설정 |
| `dynamic_threshold` | `float` (0.0~1.0) | 검색 발동 임계값 |

> **원칙:**
> - 권장 temperature: `1.0`.
> - 일일 최대 1,000,000 쿼리 제한.
> - Search Suggestion HTML/CSS(`searchEntryPoint`)는 수정 없이 그대로 표시해야 한다.

---

## § 2. Google Maps 그라운딩

### 언제: 장소, 지역 정보, 주변 검색이 필요할 때

```python
from google import genai
from google.genai import types
from google.genai.types import (
    GenerateContentConfig,
    GoogleMaps,
    HttpOptions,
    Tool,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Where can I get the best espresso near me?",
    config=GenerateContentConfig(
        tools=[
            Tool(google_maps=GoogleMaps(
                enable_widget=False,   # True: 지도 위젯 토큰 반환
            ))
        ],
        tool_config=types.ToolConfig(
            retrieval_config=types.RetrievalConfig(
                lat_lng=types.LatLng(
                    latitude=37.5665,    # 서울 위도
                    longitude=126.9780,  # 서울 경도
                ),
                language_code="ko_KR",
            ),
        ),
    ),
)

print(response.text)
```

### GoogleMaps 파라미터

| 파라미터 | 타입 | 설명 |
|---------|------|------|
| `enable_widget` | `bool` | `True`이면 응답에 지도 위젯 토큰 포함 |
| `lat_lng.latitude` | `float` | 검색 기준 위도 |
| `lat_lng.longitude` | `float` | 검색 기준 경도 |
| `language_code` | `str` | 응답 언어 코드 (예: `"ko_KR"`, `"en_US"`) |

> **원칙:**
> - Maps 출처는 생성된 콘텐츠 바로 다음에 반드시 표시해야 한다.
> - Gemini 3 Pro / Gemini 3 Pro Image는 일일 5,000 쿼리 제한.
> - 금지 지역: 중국, 쿠바, 이란, 북한, 시리아 등.

---

## § 3. Vertex AI Search 그라운딩

### 언제: 내 문서나 웹사이트 데이터 스토어로 RAG를 구성할 때

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

# 데이터 스토어 경로 형식:
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
# 출처 확인
for chunk in response.candidates[0].grounding_metadata.grounding_chunks:
    print(chunk.retrieved_context.uri)
```

### 사전 요구사항

```bash
# 1. AI Applications API 활성화
gcloud services enable discoveryengine.googleapis.com

# 2. IAM 권한 부여
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:USER_EMAIL" \
  --role="roles/discoveryengine.viewer"
```

### 파라미터

| 파라미터 | 타입 | 설명 |
|---------|------|------|
| `VertexAISearch.datastore` | `str` | 데이터 스토어 전체 리소스 경로 |

> **원칙:**
> - 최대 10개 데이터 소스를 동시에 사용할 수 있다.
> - Google Search 그라운딩과 병행 사용 가능.
> - Gemini 2.5 이상에서는 `confidence_scores`가 제공되지 않는다.

---

## § 4. Elasticsearch 그라운딩

### 언제: 기존 Elasticsearch 인덱스를 그라운딩 소스로 활용할 때

```python
from google import genai
from google.genai.types import (
    Elasticsearch,
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
                    external_api=Elasticsearch(
                        api_spec="ELASTIC_SEARCH",
                        endpoint=ES_ENDPOINT,
                        api_auth={
                            "apiKeyConfig": {
                                "apiKeyString": f"ApiKey {ES_API_KEY}"   # "ApiKey " 접두사 필수
                            }
                        },
                        elastic_search_params={
                            "index": INDEX_NAME,
                            "searchTemplate": SEARCH_TEMPLATE,
                            "numHits": 5,            # 반환할 결과 수
                        },
                    )
                )
            )
        ],
    ),
)

print(response.text)
```

### Elasticsearch 파라미터

| 파라미터 | 설명 |
|---------|------|
| `api_spec` | 반드시 `"ELASTIC_SEARCH"` |
| `endpoint` | Elasticsearch 클러스터 URL |
| `apiKeyString` | 형식: `"ApiKey <your-key>"` — 접두사 `"ApiKey "` 필수 |
| `index` | 검색할 인덱스 이름 |
| `searchTemplate` | Elasticsearch 검색 템플릿 이름 |
| `numHits` | 반환할 결과 수 |

> **원칙:**
> - `apiKeyString` 값은 반드시 `"ApiKey "` 접두사를 포함해야 한다.
> - 텍스트 입력만 지원 (멀티모달 입력 불가).
> - 최대 10개 데이터 소스를 동시에 사용할 수 있다.
