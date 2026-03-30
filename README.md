# vertexai-skills

**Vertex AI Studio** 기반 생성형 AI 개발을 위한 **Gemini CLI Skills** 모음. Vertex AI Studio에서 제공하는 모델(Gemini, Imagen, Veo)을 `google-genai` Python SDK로 호출할 때 필요한 올바른 코드 패턴과 레퍼런스를 제공한다.

> **범위:** 이 스킬은 Vertex AI Studio의 생성형 AI API(`generate_content`, `generate_images`, `generate_videos`)에 집중한다. Vertex AI Pipelines, AutoML, Custom Training 등 MLOps 영역은 다루지 않는다.

---

## 스킬 목록

| 스킬 | 파일 | 다루는 내용 |
|------|------|------------|
| `text-skills` | `skills/text-skills/SKILL.md` | 텍스트 생성, 시스템 지시, 함수 호출, 구조화된 출력, 생성 파라미터, 코드 실행 |
| `image-skills` | `skills/image-skills/SKILL.md` | Imagen 4 이미지 생성, Gemini 이미지 생성/편집, Best Practices, Limitations, Safety |
| `video-skills` | `skills/video-skills/SKILL.md` | Veo 텍스트→비디오, 이미지→비디오, 프레임 보간, 비디오 연장, 고급 기법, Safety |
| `thinking-skills` | `skills/thinking-skills/SKILL.md` | Thinking Budget, Thinking Level, Thought 요약, Thought Signatures |
| `grounding-skills` | `skills/grounding-skills/SKILL.md` | Google Search, Maps, Vertex AI Search, Elasticsearch, Custom API, Parallel, Enterprise Web |
| `model-skills` | `skills/model-skills/SKILL.md` | 모델 라이프사이클, LLM/이미지/비디오 모델 선택, 마이그레이션 가이드, Standard PayGo 사용량 티어/할당량/429 처리 |

---

## 사전 요구사항

### 라이브러리

```bash
pip install --upgrade google-genai        # 모든 스킬 공통
pip install --upgrade pillow              # image-skills: 이미지 파일 저장 시 필요
```

> 검증된 버전: `google-genai 1.69.0` (2026-03 기준)

### 필수 환경변수

```bash
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"   # "global"은 일부 모델만 지원
export GOOGLE_GENAI_USE_VERTEXAI="True"
```

### GCS 버킷 (video-skills 전용)

비디오 생성 결과는 반드시 Cloud Storage에 저장된다. 미리 버킷을 생성해야 한다:

```bash
gcloud storage buckets create gs://your-bucket --location=us-central1
export OUTPUT_GCS_URI="gs://your-bucket/output/"
```

### 인증

```bash
gcloud auth application-default login
```

---

## 설치 (심볼릭 링크 배포)

스킬 파일은 프로젝트 내 `skills/` 디렉토리에서 관리하고, Gemini CLI가 읽을 수 있도록 `~/.gemini/skills/`에 심볼릭 링크를 생성한다.

```bash
# 프로젝트 클론
git clone <repo-url> ~/sandbox/vertexai-skills
cd ~/sandbox/vertexai-skills

# Gemini CLI skills 디렉토리 생성 (없을 경우)
mkdir -p ~/.gemini/skills

# 심볼릭 링크 생성
ln -s "$(pwd)/skills/text-skills"      ~/.gemini/skills/text-skills
ln -s "$(pwd)/skills/image-skills"     ~/.gemini/skills/image-skills
ln -s "$(pwd)/skills/video-skills"     ~/.gemini/skills/video-skills
ln -s "$(pwd)/skills/thinking-skills"  ~/.gemini/skills/thinking-skills
ln -s "$(pwd)/skills/grounding-skills" ~/.gemini/skills/grounding-skills
ln -s "$(pwd)/skills/model-skills"     ~/.gemini/skills/model-skills
```

### 설치 확인

```bash
ls -la ~/.gemini/skills/
```

정상 출력 예시:
```
grounding-skills -> /home/user/sandbox/vertexai-skills/skills/grounding-skills
image-skills     -> /home/user/sandbox/vertexai-skills/skills/image-skills
model-skills     -> /home/user/sandbox/vertexai-skills/skills/model-skills
text-skills      -> /home/user/sandbox/vertexai-skills/skills/text-skills
thinking-skills  -> /home/user/sandbox/vertexai-skills/skills/thinking-skills
video-skills     -> /home/user/sandbox/vertexai-skills/skills/video-skills
```

---

## 사용 방법

### 작동 원리

스킬은 별도 명령어 없이 **자동으로 활성화**된다. 두 가지 방식으로 트리거된다.

**① 키워드 매칭** — 메시지에 트리거 키워드가 포함되면 해당 스킬이 자동 로드된다.

```
"Veo로 비디오 생성하는 코드 써줘"  →  video-skills 활성화
"429 에러가 계속 나는데 어떻게 해?"  →  model-skills 활성화
```

**② 코드 컨텍스트** — 편집 중인 코드에 특정 import가 있으면 자동 활성화된다.

| 코드 내 import | 활성화 스킬 |
|---------------|-----------|
| `from google import genai` | `text-skills` |
| `ThinkingConfig` | `thinking-skills` |
| `GoogleSearch` | `grounding-skills` |

스킬이 로드되면 Gemini는 SKILL.md의 코드 패턴과 Best Practice를 참고해서 답변한다. 스킬 없이 질문하면 잘못된 API 패턴이 제안될 수 있다.

---

### text-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| 코드에 `from google import genai` 포함 | "이 코드에 스트리밍을 추가해줘" |
| `text generation`, `텍스트 생성` | "텍스트 생성 코드 작성해줘" |
| `function calling`, `함수 호출` | "날씨 API를 함수 호출로 연결하는 코드 써줘" |
| `structured output`, `구조화된 출력` | "JSON 형식으로 응답받는 코드 써줘" |
| `code execution`, `코드 실행` | "모델이 직접 계산하는 코드 실행 예제 써줘" |
| `system instruction`, `시스템 지시` | "시스템 지시로 역할을 설정하는 방법 알려줘" |

### image-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| `Imagen`, `아이마젠` | "Imagen 4로 이미지 생성하는 코드 써줘" |
| `image generation`, `이미지 생성` | "이미지 생성하는 코드 만들어줘" |
| `edit image`, `이미지 편집` | "기존 이미지를 수채화 스타일로 편집해줘" |
| `Gemini image`, `제미나이 이미지` | "Gemini 이미지 모델로 생성하는 방법 알려줘" |

### video-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| `Veo`, `비오` | "Veo로 비디오 생성하는 코드 써줘" |
| `video generation`, `비디오 생성` | "텍스트로 영상 만드는 코드 써줘" |
| `text to video`, `텍스트로 영상` | "프롬프트로 비디오 생성해줘" |
| `image to video`, `이미지로 영상` | "이미지를 비디오로 변환하는 코드 써줘" |
| `extend video`, `영상 연장` | "비디오 연장하는 방법 알려줘" |

### thinking-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| `ThinkingConfig` 임포트 | "이 코드에 사고 예산을 설정해줘" |
| `thinking budget`, `생각 예산` | "thinking_budget으로 추론 깊이 제어하는 법 알려줘" |
| `thinking level`, `사고 모드` | "Gemini 3에서 ThinkingLevel.HIGH 사용하는 코드 써줘" |
| `thought signatures`, `사고 시그니처` | "함수 호출 시 Thought Signatures 처리하는 방법" |
| `reasoning model`, `추론 모델` | "추론 모델로 수학 문제 풀게 해줘" |

### grounding-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| `GoogleSearch` 임포트 | "이 코드에 Google Search 그라운딩 추가해줘" |
| `grounding`, `그라운딩` | "그라운딩으로 환각을 줄이는 방법 알려줘" |
| `Vertex AI Search`, `버텍스 검색` | "내 문서로 RAG 구성하는 코드 써줘" |
| `Elasticsearch grounding` | "ES 인덱스를 그라운딩 소스로 쓰는 코드 써줘" |
| `enterprise web search` | "컴플라이언스 환경에서 웹 그라운딩 사용하는 방법" |

### model-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| `model selection`, `모델 선택`, `어떤 모델`, `모델 추천` | "어떤 Gemini 모델을 써야 하나요?" |
| `model lifecycle`, `모델 라이프사이클` | "모델 은퇴일 확인하고 싶어요" |
| `deprecated model`, `retired model`, `모델 은퇴` | "1.5 모델 아직 써도 되나요?" |
| `model migration`, `모델 마이그레이션`, `1.5 to 2.5` | "1.5에서 2.5로 마이그레이션하는 방법" |
| `SDK migration`, `SDK 마이그레이션` | "Vertex AI SDK에서 Gen AI SDK로 전환하는 코드 써줘" |
| `standard paygo`, `usage tier`, `TPM`, `할당량`, `429 에러` | "Standard PayGo 티어 구조 알려줘" |
| `provisioned throughput`, `프로비전드 처리량` | "Provisioned Throughput이 필요한지 판단하는 기준이 뭔가요?" |

---

## 스킬 내용 구조

### text-skills (352줄)

```
환경 설정
  - 라이브러리 설치
  - 환경변수
  - 클라이언트 초기화 (HttpOptions api_version="v1")
§ 1. 텍스트 생성         — 논스트리밍 / 스트리밍 채팅
§ 2. 시스템 지시          — GenerateContentConfig.system_instruction
§ 3. 함수 호출            — FunctionDeclaration → Tool → function_response 루프
§ 4. 구조화된 출력        — response_mime_type + response_schema (JSON Schema)
§ 5. 생성 파라미터        — temperature, top_p, top_k, max_output_tokens 등
§ 6. 코드 실행            — ToolCodeExecution, executable_code, code_execution_result
```

### image-skills (323줄)

```
환경 설정
  - 라이브러리 설치 (google-genai + pillow)
  - 모델 선택 가이드 (Imagen 4 vs Gemini Image)
§ 1. Imagen 4 이미지 생성  — generate_images() + GenerateImagesConfig
§ 2. Gemini 이미지 생성    — generate_content() + Modality.IMAGE
§ 3. Gemini 이미지 편집    — 이미지 입력 + 멀티턴 편집
§ 4. Best Practices        — 구체적 프롬프트, 카메라 용어, 반복 개선
§ 5. Limitations           — 언어 지원, 입력 제한, 출력 한계
§ 6. Responsible AI & Safety — 금지 콘텐츠, finish_reason SAFETY 처리
```

### video-skills (523줄)

```
환경 설정
  - GCS 버킷 필수 안내
  - 모델 목록 (veo-3.1-generate-001 권장)
  - 공통 폴링 루프 패턴 (LRO)
§ 1. 텍스트 → 비디오       — generate_videos() + GenerateVideosConfig
§ 2. 이미지 → 비디오       — Image(gcs_uri=...) 입력 패턴
§ 3. 첫/마지막 프레임      — image + last_frame 파라미터
§ 4. 비디오 연장            — Video(uri=..., mime_type="video/mp4")
§ 5. 고급 기법              — 레퍼런스 이미지, 객체 삽입/제거 (veo-2.0 전용)
§ 6. 프롬프트 가이드       — 7가지 구성 요소, 발화 표현, enhance_prompt
§ 7. Responsible AI & Safety — 금지 콘텐츠, operation.error 처리
```

### thinking-skills

```
환경 설정
  - 모델 선택 가이드 (thinking_budget vs thinking_level)
§ 1. Thinking Budget 설정  — ThinkingConfig(thinking_budget=N), 0/-1 옵션
§ 2. Thinking Level 설정   — ThinkingLevel.MINIMAL/LOW/MEDIUM/HIGH (Gemini 3+)
§ 3. Thought 요약 보기     — include_thoughts=True, part.thought 분기
§ 4. Thought Signatures    — 함수 호출 멀티턴에서 사고 연속성 유지
```

### grounding-skills

```
환경 설정
  - 7가지 그라운딩 방식 선택 가이드
§ 1. Google Search 그라운딩       — GoogleSearch, Dynamic Retrieval
§ 2. Google Maps 그라운딩         — GoogleMaps, LatLng, ToolConfig
§ 3. Vertex AI Search 그라운딩    — Retrieval + VertexAISearch(datastore=...)
§ 4. Elasticsearch 그라운딩       — ExternalApi(api_spec="ELASTIC_SEARCH")
§ 5. Custom Search API 그라운딩   — ExternalApi(api_spec="SIMPLE_SEARCH"), API 계약
§ 6. Parallel Web Search 그라운딩 — REST only, parallelAiSearch
§ 7. Enterprise Web Search 그라운딩 — EnterpriseWebSearch(), 컴플라이언스 환경
```

### model-skills

```
§ 1. LLM 모델 선택      — Stable/Retired 모델 표, 용도별 선택 기준, Alias 주의사항
§ 2. 이미지 모델 선택    — Imagen vs Gemini Image, Preview 모델 구분
§ 3. 비디오 모델 선택    — veo-3.1 권장, veo-2.0 전용 기능
§ 4. 마이그레이션 가이드  — SDK 전환 코드, Breaking Changes, 7단계 절차, 확인사항
§ 5. Standard PayGo    — 사용량 티어(TPM), 30일 지출 기준, 429 에러 처리, Provisioned Throughput 비교
```

---

## 모델 선택 가이드

> 상세 내용은 `model-skills`를 참고한다. 아래는 빠른 참조용 요약이다.
>
> 참고: [Model versions and lifecycle](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/model-versions) · [Migration guide](https://cloud.google.com/vertex-ai/generative-ai/docs/migrate) · [Standard PayGo](https://cloud.google.com/vertex-ai/generative-ai/docs/standard-paygo)

### 모델 라이프사이클 개념

| 상태 | 의미 |
|------|------|
| **Stable (GA)** | 프로덕션 사용 권장. 은퇴일 1개월 전부터 신규 접근 차단 |
| **Preview** | 기능 검증 중. 프로덕션 사용 비권장 |
| **Deprecated** | 은퇴일 이후 영구 비활성화 |

### LLM (텍스트 생성 / 그라운딩 / 사고 모델)

| 모델 | 상태 | 은퇴일 | 권장 용도 |
|------|------|--------|----------|
| `gemini-2.5-flash` | **Stable ✅** | 2026-06-17 | 일반 텍스트 생성, 그라운딩 **(기본 권장)** |
| `gemini-2.5-pro` | **Stable ✅** | 2026-06-17 | 복잡한 추론, 고품질 응답 |
| `gemini-2.5-flash-lite` | **Stable ✅** | 2026-07-22 | 비용 최적화, 고속 처리 |
| `gemini-2.0-flash-001` | Stable (신규 제한) | 2026-06-01 | 기존 프로젝트만. 신규는 2.5-flash로 |
| `gemini-1.5-pro-002` | **Retired ❌** | 2025-09-24 (종료) | API 호출 시 404. 즉시 교체 필요 |
| `gemini-1.5-flash-002` | **Retired ❌** | 2025-09-24 (종료) | 동일 |
| `gemini-1.0-pro-*` | **Retired ❌** | 2025-04-21 (종료) | 동일 |

> **Thinking Budget vs Thinking Level**
> - `thinking_budget` (토큰 수 직접 제어): `gemini-2.5-flash`, `gemini-2.5-pro`
> - `thinking_level` (MINIMAL/LOW/MEDIUM/HIGH): Gemini 3 이상 (`gemini-3-flash`, `gemini-3.1-pro`)

### 이미지 생성

| 모델 | API | 상태 | 권장 용도 |
|------|-----|------|----------|
| `imagen-4.0-generate-001` | `generate_images()` | **Stable ✅** | 최고 품질 정적 이미지 **(권장)** |
| `imagen-3.0-generate-002` | `generate_images()` | **Stable ✅** | Imagen 3 안정 버전 |
| `gemini-2.5-flash-image` | `generate_content()` | Stable | 이미지 생성 + 편집, 경량 |
| `gemini-3-pro-image-preview` | `generate_content()` | Preview | 이미지 생성 + 편집, 최고 품질 |
| `gemini-3.1-flash-image-preview` | `generate_content()` | Preview | 이미지 생성 + 편집, 최신 |

### 비디오 생성

| 모델 | 상태 | 은퇴일 | 권장 용도 |
|------|------|--------|----------|
| `veo-3.1-generate-001` | **Stable ✅** | 미정 | 최신 안정 버전 **(권장)** |
| `veo-3.1-fast-generate-001` | **Stable ✅** | 미정 | 빠른 생성 |
| `veo-3.0-generate-001` | Stable | 2026-06-30 | 이전 안정 버전 |
| `veo-2.0-generate-001` | Stable (제한적) | — | 객체 삽입/제거, `enhance_prompt` 전용 |

### 마이그레이션 핵심 체크리스트

- [ ] `gemini-1.5-*` 사용 중 → `gemini-2.5-flash` 또는 `gemini-2.5-pro`로 즉시 교체 (**이미 은퇴, API 404 반환 중**)
- [ ] `gemini-2.0-flash-001` 신규 사용 → `gemini-2.5-flash`로 교체
- [ ] `thinking_budget` 사용 코드에서 Gemini 3 모델로 업그레이드 시 → `thinking_level`로 변경
- [ ] Vertex AI SDK(`google-cloud-aiplatform`) → Gen AI SDK(`google-genai`)로 마이그레이션 권장
- [ ] 지역 가용성 확인: `us-central1` 권장 (`global`은 일부 모델만 지원)

---

## 공통 클라이언트 초기화 패턴

모든 스킬이 동일한 초기화 패턴을 사용한다:

```python
from google import genai
from google.genai.types import HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))
```

> `client`는 모듈 최상단에서 한 번만 초기화한다. 함수마다 재생성하지 않는다.

대안 (환경변수 대신 직접 지정):

```python
client = genai.Client(
    vertexai=True,
    project="your-project-id",
    location="us-central1",
)
```

---

## 프로젝트 구조

```
vertexai-skills/
├── skills/
│   ├── text-skills/
│   │   └── SKILL.md          # 텍스트 생성 레퍼런스
│   ├── image-skills/
│   │   └── SKILL.md          # 이미지 생성/편집 레퍼런스
│   ├── video-skills/
│   │   └── SKILL.md          # 비디오 생성 레퍼런스
│   ├── thinking-skills/
│   │   └── SKILL.md          # 사고 모델 제어 레퍼런스
│   ├── grounding-skills/
│   │   └── SKILL.md          # 그라운딩 레퍼런스 (7가지 소스)
│   └── model-skills/
│       └── SKILL.md          # 모델 선택 및 마이그레이션 가이드
├── .gitignore                 # .claude/, docs/ 제외
└── README.md

# 배포 (심볼릭 링크)
~/.gemini/skills/text-skills      →  skills/text-skills/
~/.gemini/skills/image-skills     →  skills/image-skills/
~/.gemini/skills/video-skills     →  skills/video-skills/
~/.gemini/skills/thinking-skills  →  skills/thinking-skills/
~/.gemini/skills/grounding-skills →  skills/grounding-skills/
~/.gemini/skills/model-skills     →  skills/model-skills/
```

---

## 스킬 업데이트

스킬 파일은 git으로 관리된다. 심볼릭 링크는 항상 최신 파일을 가리키므로 **별도 재배포 없이** 수정 즉시 반영된다.

```bash
# 스킬 수정 후
git add skills/<skill-name>/SKILL.md
git commit -m "feat: update <section> in <skill-name>"
# 완료 — Gemini CLI는 다음 실행 시 자동으로 업데이트된 내용을 로드
```

---

## SDK 참고 문서

- [google-genai Python SDK](https://googleapis.github.io/python-genai/)
- [Vertex AI Gemini API](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/overview)
- [Imagen on Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs/image/overview)
- [Veo on Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs/video/overview)
