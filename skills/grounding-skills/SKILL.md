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
| Elasticsearch | `ExternalApi` (via `Retrieval`) | 기존 ES 인덱스 |
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
                enable_widget=False,   # True: 지도 위젯 토큰 반환
            ))
        ],
        tool_config=ToolConfig(
            retrieval_config=RetrievalConfig(
                lat_lng=LatLng(
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
                                "apiKeyString": f"ApiKey {ES_API_KEY}"   # "ApiKey " 접두사 필수
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

### Elasticsearch 파라미터

| 파라미터 | 위치 | 설명 |
|---------|------|------|
| `api_spec` | `ExternalApi` | 반드시 `"ELASTIC_SEARCH"` |
| `endpoint` | `ExternalApi` | Elasticsearch 클러스터 URL |
| `api_auth.apiKeyConfig.apiKeyString` | `ExternalApi` | 형식: `"ApiKey <your-key>"` — 접두사 `"ApiKey "` 필수 |
| `elastic_search_params.index` | `ExternalApi` | 검색할 인덱스 이름 |
| `elastic_search_params.searchTemplate` | `ExternalApi` | Elasticsearch 검색 템플릿 이름 |
| `elastic_search_params.numHits` | `ExternalApi` | 반환할 결과 수 |

> **원칙:**
> - `apiKeyString` 값은 반드시 `"ApiKey "` 접두사를 포함해야 한다.
> - 텍스트 입력만 지원 (멀티모달 입력 불가).
> - 최대 10개 데이터 소스를 동시에 사용할 수 있다.

---

## § 5. Custom Search API 그라운딩

### 언제: 자체 검색 API를 그라운딩 소스로 연결할 때

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
                        api_spec="SIMPLE_SEARCH",   # 현재 유일한 허용 값
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

### 자체 API 계약 (필수 준수)

Gemini가 POST로 호출하는 요청 형식:
```json
{ "query": "검색어 문자열" }
```

응답으로 반환해야 하는 형식:
```json
[
  { "snippet": "관련 정보 텍스트", "uri": "출처 URL" },
  { "snippet": "두 번째 정보", "uri": "출처2 URL" }
]
```
결과가 없으면 빈 배열 `[]` 반환.

### ExternalApi 파라미터

| 파라미터 | 설명 |
|---------|------|
| `api_spec` | 반드시 `"SIMPLE_SEARCH"` |
| `endpoint` | API Gateway 엔드포인트 URL |
| `apiKeyConfig.apiKeyString` | Gemini가 `?key=` 쿼리 파라미터로 전달할 API 키 |

> **원칙:**
> - API 응답 지연이 길어지면 전체 Gemini 응답 시간이 증가한다.
> - `snippet`의 품질이 그라운딩 응답 품질을 직접 결정한다.
> - 인증은 API 키 방식만 지원된다.

---

## § 6. Parallel Web Search 그라운딩

### 언제: Parallel AI의 LLM 최적화 웹 인덱스를 활용할 때

> **참고:** Parallel Web Search는 현재 REST API로만 사용할 수 있다.

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

# REST API 호출 예시 (gcloud 인증 사용)
# curl -X POST \
#   -H "Authorization: Bearer $(gcloud auth print-access-token)" \
#   -H "Content-Type: application/json" \
#   -d '<request_body>' \
#   "https://LOCATION-aiplatform.googleapis.com/v1/projects/PROJECT_ID/locations/LOCATION/publishers/google/models/MODEL_ID:generateContent"
```

### Parallel Web Search 파라미터

| 파라미터 | 기본값 | 범위 | 설명 |
|---------|--------|------|------|
| `api_key` | — | — | Parallel AI API 키 (필수) |
| `exclude_domains` | — | 최대 10개 | 제외할 도메인 |
| `include_domains` | — | 최대 10개 | 포함할 도메인 (제한) |
| `max_chars_per_result` | 30,000 | 1,000~100,000 | 결과당 최대 문자 수 |
| `max_chars_total` | 100,000 | 1,000~1,000,000 | 전체 최대 문자 수 |
| `max_results` | 10 | 1~20 | 최대 결과 수 |

> **원칙:**
> - 분당 60 프롬프트 기본 할당량.
> - Parallel의 별도 이용약관이 적용된다.
> - Pre-GA 서비스로 SLA가 없다.

---

## § 7. Enterprise Web Search 그라운딩

### 언제: 의료·금융·공공 분야 등 컴플라이언스 규제 환경에서 웹 그라운딩이 필요할 때

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
            Tool(enterprise_web_search=EnterpriseWebSearch())   # 파라미터 없음
        ],
    ),
)

print(response.text)
```

### Google Search vs Enterprise Web Search 비교

| 항목 | Google Search | Enterprise Web Search |
|------|--------------|----------------------|
| 인덱스 범위 | 전체 웹 (더 넓음) | 규제 산업 최적화 |
| 최신성 | 실시간 | 6시간/24시간 업데이트 |
| 데이터 로깅 | 표준 | 없음 |
| VPC-SC 지원 | — | ✅ |
| CMEK | — | 해당 없음 |
| 일일 쿼리 제한 | — | 5,000 (Gemini 3 Pro/Pro Image) |

> **원칙:**
> - `EnterpriseWebSearch()`는 파라미터가 없다.
> - Search Suggestion(`webSearchQueries`)은 반드시 앱에 표시해야 한다.
> - 컴플라이언스가 필요 없다면 더 넓은 인덱스를 제공하는 Google Search를 사용한다.
