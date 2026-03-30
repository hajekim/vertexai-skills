# vertexaistudio-skills

**Vertex AI Studio** 기반 생성형 AI 개발을 위한 **Gemini CLI Skills** 모음입니다. Vertex AI Studio에서 제공하는 모델(Gemini, Imagen, Veo)을 `google-genai` Python SDK로 호출할 때 필요한 올바른 코드 패턴과 레퍼런스를 제공합니다.

> **범위:** 이 스킬은 Vertex AI Studio의 생성형 AI API(`generate_content`, `generate_images`, `generate_videos`)에 집중합니다. Vertex AI Pipelines, AutoML, Custom Training 등 MLOps 영역은 다루지 않습니다.

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
| `embedding-skills` | `skills/embedding-skills/SKILL.md` | 텍스트 임베딩, Task Type 선택, 멀티모달 임베딩, 배치 임베딩 |
| `translate-skills` | `skills/translate-skills/SKILL.md` | Translation LLM (Gemini 기반), NMT, 참조 문장 쌍, 문서 번역 |
| `speech-skills` | `skills/speech-skills/SKILL.md` | TTS (텍스트→음성), STT/Chirp 모델 (음성→텍스트), 프로덕션 Speech API 안내 |
| `computer-use-skills` | `skills/computer-use-skills/SKILL.md` | 브라우저 자동화, 액션 타입, 1000×1000 좌표 그리드, 안전 확인 패턴 |
| `url-context-skills` | `skills/url-context-skills/SKILL.md` | URL 컨텍스트 그라운딩, 최대 20 URL, 검색 상태 확인, 토큰 카운팅 |

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

비디오 생성 결과는 반드시 Cloud Storage에 저장됩니다. 미리 버킷을 생성해 주세요.

```bash
gcloud storage buckets create gs://your-bucket --location=us-central1
export OUTPUT_GCS_URI="gs://your-bucket/output/"
```

### 인증

```bash
gcloud auth application-default login
```

---

## 설치

Gemini CLI Skills는 **전역 설치**와 **프로젝트 단위 설치** 두 가지 방식을 지원합니다.

| 방식 | 경로 | 적용 범위 |
|------|------|----------|
| 전역 설치 | `~/.gemini/skills/` | 모든 프로젝트에서 사용 가능 |
| 프로젝트 단위 설치 | `<프로젝트 루트>/.gemini/skills/` | 해당 프로젝트에서만 사용 가능 |

> **우선순위:** 같은 이름의 스킬이 양쪽에 모두 있으면 프로젝트 단위 스킬이 우선 적용됩니다.

---

### 방법 1. 전역 설치 (모든 프로젝트에서 사용)

```bash
# 프로젝트 클론
git clone <repo-url> ~/sandbox/vertexai-skills
cd ~/sandbox/vertexai-skills

# Gemini CLI skills 디렉토리 생성 (없을 경우)
mkdir -p ~/.gemini/skills

# 심볼릭 링크 생성
ln -s "$(pwd)/skills/text-skills"          ~/.gemini/skills/text-skills
ln -s "$(pwd)/skills/image-skills"         ~/.gemini/skills/image-skills
ln -s "$(pwd)/skills/video-skills"         ~/.gemini/skills/video-skills
ln -s "$(pwd)/skills/thinking-skills"      ~/.gemini/skills/thinking-skills
ln -s "$(pwd)/skills/grounding-skills"     ~/.gemini/skills/grounding-skills
ln -s "$(pwd)/skills/model-skills"         ~/.gemini/skills/model-skills
ln -s "$(pwd)/skills/embedding-skills"     ~/.gemini/skills/embedding-skills
ln -s "$(pwd)/skills/translate-skills"     ~/.gemini/skills/translate-skills
ln -s "$(pwd)/skills/speech-skills"        ~/.gemini/skills/speech-skills
ln -s "$(pwd)/skills/computer-use-skills"  ~/.gemini/skills/computer-use-skills
ln -s "$(pwd)/skills/url-context-skills"   ~/.gemini/skills/url-context-skills
```

### 설치 확인

```bash
ls -la ~/.gemini/skills/
```

정상 출력 예시:
```
computer-use-skills -> /home/user/sandbox/vertexai-skills/skills/computer-use-skills
embedding-skills    -> /home/user/sandbox/vertexai-skills/skills/embedding-skills
grounding-skills    -> /home/user/sandbox/vertexai-skills/skills/grounding-skills
image-skills        -> /home/user/sandbox/vertexai-skills/skills/image-skills
model-skills        -> /home/user/sandbox/vertexai-skills/skills/model-skills
speech-skills       -> /home/user/sandbox/vertexai-skills/skills/speech-skills
text-skills         -> /home/user/sandbox/vertexai-skills/skills/text-skills
thinking-skills     -> /home/user/sandbox/vertexai-skills/skills/thinking-skills
translate-skills    -> /home/user/sandbox/vertexai-skills/skills/translate-skills
url-context-skills  -> /home/user/sandbox/vertexai-skills/skills/url-context-skills
video-skills        -> /home/user/sandbox/vertexai-skills/skills/video-skills
```

---

### 방법 2. 프로젝트 단위 설치 (특정 프로젝트에서만 사용)

Vertex AI 관련 작업을 하는 특정 프로젝트에만 스킬을 적용하고 싶을 때 사용합니다.

```bash
# 내 프로젝트 루트로 이동
cd ~/my-project

# 프로젝트 단위 skills 디렉토리 생성
mkdir -p .gemini/skills

# 필요한 스킬만 선택해서 심볼릭 링크 생성
SKILLS_REPO=~/sandbox/vertexai-skills

ln -s "$SKILLS_REPO/skills/text-skills"      .gemini/skills/text-skills
ln -s "$SKILLS_REPO/skills/model-skills"     .gemini/skills/model-skills
ln -s "$SKILLS_REPO/skills/grounding-skills" .gemini/skills/grounding-skills
# 필요한 스킬만 추가하면 됩니다
```

> **권장:** `.gemini/` 디렉토리를 `.gitignore`에 추가하거나, 팀 전체가 공유할 경우 그대로 커밋합니다.

### 설치 확인

```bash
ls -la .gemini/skills/
```

정상 출력 예시:
```
grounding-skills -> /home/user/sandbox/vertexai-skills/skills/grounding-skills
model-skills     -> /home/user/sandbox/vertexai-skills/skills/model-skills
text-skills      -> /home/user/sandbox/vertexai-skills/skills/text-skills
```

---

## 사용 방법

### 작동 원리

스킬은 별도 명령어 없이 **자동으로 활성화**됩니다. 두 가지 방식으로 트리거됩니다.

**① 키워드 매칭** — 메시지에 트리거 키워드가 포함되면 해당 스킬이 자동 로드됩니다.

```
"Veo로 비디오 생성하는 코드 작성해 주세요"  →  video-skills 활성화
"429 에러가 계속 나는데 어떻게 해결하나요?"  →  model-skills 활성화
```

**② 코드 컨텍스트** — 편집 중인 코드에 특정 import가 있으면 자동 활성화됩니다.

| 코드 내 import | 활성화 스킬 |
|---------------|-----------|
| `from google import genai` | `text-skills` |
| `ThinkingConfig` | `thinking-skills` |
| `GoogleSearch` | `grounding-skills` |

스킬이 로드되면 Gemini는 SKILL.md의 코드 패턴과 Best Practice를 참고해 답변합니다. 스킬 없이 질문하면 잘못된 API 패턴이 제안될 수 있습니다.

---

### text-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| 코드에 `from google import genai` 포함 | "이 코드에 스트리밍을 추가해 주세요" |
| `text generation`, `텍스트 생성` | "텍스트 생성 코드를 작성해 주세요" |
| `function calling`, `함수 호출` | "날씨 API를 함수 호출로 연결하는 코드를 작성해 주세요" |
| `structured output`, `구조화된 출력` | "JSON 형식으로 응답받는 코드를 작성해 주세요" |
| `code execution`, `코드 실행` | "모델이 직접 계산하는 코드 실행 예제를 작성해 주세요" |
| `system instruction`, `시스템 지시` | "시스템 지시로 역할을 설정하는 방법을 알려 주세요" |

### image-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| `Imagen`, `아이마젠` | "Imagen 4로 이미지 생성하는 코드를 작성해 주세요" |
| `image generation`, `이미지 생성` | "이미지 생성하는 코드를 만들어 주세요" |
| `edit image`, `이미지 편집` | "기존 이미지를 수채화 스타일로 편집해 주세요" |
| `Gemini image`, `제미나이 이미지` | "Gemini 이미지 모델로 생성하는 방법을 알려 주세요" |

### video-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| `Veo`, `비오` | "Veo로 비디오 생성하는 코드를 작성해 주세요" |
| `video generation`, `비디오 생성` | "텍스트로 영상 만드는 코드를 작성해 주세요" |
| `text to video`, `텍스트로 영상` | "프롬프트로 비디오를 생성해 주세요" |
| `image to video`, `이미지로 영상` | "이미지를 비디오로 변환하는 코드를 작성해 주세요" |
| `extend video`, `영상 연장` | "비디오 연장하는 방법을 알려 주세요" |

### thinking-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| `ThinkingConfig` 임포트 | "이 코드에 사고 예산을 설정해 주세요" |
| `thinking budget`, `생각 예산` | "thinking_budget으로 추론 깊이를 제어하는 방법을 알려 주세요" |
| `thinking level`, `사고 모드` | "Gemini 3에서 ThinkingLevel.HIGH 사용하는 코드를 작성해 주세요" |
| `thought signatures`, `사고 시그니처` | "함수 호출 시 Thought Signatures 처리하는 방법을 알려 주세요" |
| `reasoning model`, `추론 모델` | "추론 모델로 수학 문제를 풀게 해 주세요" |

### grounding-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| `GoogleSearch` 임포트 | "이 코드에 Google Search 그라운딩을 추가해 주세요" |
| `grounding`, `그라운딩` | "그라운딩으로 환각을 줄이는 방법을 알려 주세요" |
| `Vertex AI Search`, `버텍스 검색` | "내 문서로 RAG를 구성하는 코드를 작성해 주세요" |
| `Elasticsearch grounding` | "ES 인덱스를 그라운딩 소스로 사용하는 코드를 작성해 주세요" |
| `enterprise web search` | "컴플라이언스 환경에서 웹 그라운딩 사용하는 방법을 알려 주세요" |

### model-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| `model selection`, `모델 선택`, `어떤 모델`, `모델 추천` | "어떤 Gemini 모델을 써야 하나요?" |
| `model lifecycle`, `모델 라이프사이클` | "모델 은퇴일을 확인하고 싶습니다" |
| `deprecated model`, `retired model`, `모델 은퇴` | "1.5 모델을 아직 써도 되나요?" |
| `model migration`, `모델 마이그레이션`, `1.5 to 2.5` | "1.5에서 2.5로 마이그레이션하는 방법을 알려 주세요" |
| `SDK migration`, `SDK 마이그레이션` | "Vertex AI SDK에서 Gen AI SDK로 전환하는 코드를 작성해 주세요" |
| `standard paygo`, `usage tier`, `TPM`, `할당량`, `429 에러` | "Standard PayGo 티어 구조를 알려 주세요" |
| `provisioned throughput`, `프로비전드 처리량` | "Provisioned Throughput이 필요한지 판단하는 기준이 무엇인가요?" |

### embedding-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| `embed_content` 임포트 | "이 임베딩 코드에 task_type을 추가해 주세요" |
| `embedding`, `임베딩`, `텍스트 임베딩` | "텍스트 임베딩 코드를 작성해 주세요" |
| `vector embedding`, `벡터 임베딩` | "문서를 벡터로 변환하는 코드를 만들어 주세요" |
| `semantic similarity`, `의미적 유사도` | "두 문장의 유사도를 계산하는 코드를 작성해 주세요" |
| `multimodal embedding`, `멀티모달 임베딩` | "이미지와 텍스트를 함께 임베딩하는 방법을 알려 주세요" |
| `batch embedding`, `배치 임베딩` | "30,000개 문서를 배치로 임베딩하는 방법을 알려 주세요" |
| `task type`, `임베드` | "RETRIEVAL_QUERY와 RETRIEVAL_DOCUMENT의 차이가 무엇인가요?" |

### translate-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| `translate`, `번역`, `텍스트 번역` | "텍스트 번역하는 코드를 작성해 주세요" |
| `Translation LLM`, `cloud translate` | "Translation LLM API 사용하는 방법을 알려 주세요" |
| `NMT`, `언어 번역` | "NMT로 100개 언어 번역하는 코드를 작성해 주세요" |
| `다국어 번역`, `자동 번역` | "한국어→영어 번역 코드를 만들어 주세요" |

### speech-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| `text to speech`, `TTS`, `텍스트 음성 변환` | "TTS 코드를 작성해 주세요" |
| `speech to text`, `STT`, `음성 인식` | "Chirp 모델로 음성 인식하는 코드를 작성해 주세요" |
| `voice synthesis`, `음성 합성` | "음성 합성 API 사용 방법을 알려 주세요" |
| `Chirp`, `오디오 변환`, `음성 전사` | "WAV 파일을 텍스트로 변환하는 코드를 작성해 주세요" |

### computer-use-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| `ComputerUse` 임포트 | "이 코드에 컴퓨터 사용 도구를 추가해 주세요" |
| `computer use`, `컴퓨터 사용` | "Gemini로 브라우저를 자동화하는 코드를 작성해 주세요" |
| `browser automation`, `브라우저 자동화` | "웹 페이지 자동 클릭하는 코드를 만들어 주세요" |
| `GUI automation`, `web automation` | "화면을 보고 자동으로 폼을 채우는 방법을 알려 주세요" |

### url-context-skills 트리거

| 트리거 | 예시 요청 |
|--------|---------|
| `UrlContext` 임포트 | "이 코드에 URL 컨텍스트를 추가해 주세요" |
| `url context`, `URL 컨텍스트` | "URL 내용을 참조해서 답변하는 코드를 작성해 주세요" |
| `URL grounding`, `URL 그라운딩` | "특정 URL을 기반으로 요약하는 코드를 만들어 주세요" |
| `웹 페이지 읽기`, `링크 내용 참조` | "이 URL의 내용을 읽어서 분석해 주세요" |

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

### embedding-skills

```
환경 설정
  - 모델 선택 가이드 (gemini-embedding-001 / text-embedding-005 / multimodal)
§ 1. 텍스트 임베딩       — embed_content(), EmbedContentConfig, 요청 한도
§ 2. Task Type 선택      — 8가지 task_type 설명 및 비대칭 검색 패턴
§ 3. 멀티모달 임베딩     — gemini-embedding-2-preview (텍스트/이미지/비디오/오디오/PDF)
§ 4. 배치 임베딩         — BatchPredictionJob, JSONL 형식, 30,000개 한도
```

### translate-skills

```
환경 설정
  - Translation LLM vs NMT 선택 가이드
§ 1. Translation LLM     — REST API 호출, 참조 문장 쌍, 배치 1,024건
§ 2. NMT                 — Cloud Translation API, 문서 번역, 비교표
```

### speech-skills

```
환경 설정
  - Vertex AI Studio UI 전용 안내 (SDK 없음)
§ 1. TTS                 — UI 데모 안내 + Cloud TTS API 프로덕션 코드
§ 2. STT                 — Chirp 모델, WAV 16-bit PCM, 10MB/60s 제한
                           + Cloud Speech-to-Text API (단문/장문)
```

### computer-use-skills

```
환경 설정
  - Preview 기능 안내 (gemini-3-flash 전용)
§ 1. 기본 패턴           — ComputerUse(environment=ENVIRONMENT_BROWSER), safety_acknowledgement
§ 2. 액션 타입           — click_at, type_text_at, navigate, scroll, key_combination 등
                           1000×1000 좌표 그리드 스케일링
§ 3. 브라우저 자동화 루프 — Playwright 기반 멀티턴 스크린샷+액션 루프
§ 4. 안전 요구사항       — 인간 확인 패턴, 감사 로깅, 범위 제한
```

### url-context-skills

```
환경 설정
§ 1. 기본 URL 컨텍스트   — Tool(url_context=UrlContext()), URL을 contents에 포함
§ 2. 다중 URL            — 최대 20개 URL 동시 참조
§ 3. 검색 상태 확인      — url_retrieval_status (SUCCESS/FAILED)
§ 4. Google Search 병합  — UrlContext + GoogleSearch 동시 사용
§ 5. 지원 콘텐츠 타입    — HTML/JSON/PDF/이미지, 2단계 검색 방식, 토큰 카운팅
```

---

## 모델 선택 가이드

> 상세 내용은 `model-skills`를 참고하세요. 아래는 빠른 참조용 요약입니다.
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

모든 스킬이 동일한 초기화 패턴을 사용합니다.

```python
from google import genai
from google.genai.types import HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))
```

> `client`는 모듈 최상단에서 한 번만 초기화하세요. 함수마다 재생성하지 않아야 합니다.

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
│   ├── model-skills/
│   │   └── SKILL.md          # 모델 선택 및 마이그레이션 가이드
│   ├── embedding-skills/
│   │   └── SKILL.md          # 텍스트/멀티모달 임베딩 레퍼런스
│   ├── translate-skills/
│   │   └── SKILL.md          # Translation LLM / NMT 레퍼런스
│   ├── speech-skills/
│   │   └── SKILL.md          # TTS / STT (Chirp) 레퍼런스
│   ├── computer-use-skills/
│   │   └── SKILL.md          # 브라우저 자동화 레퍼런스
│   └── url-context-skills/
│       └── SKILL.md          # URL 컨텍스트 그라운딩 레퍼런스
└── README.md

# 배포 (심볼릭 링크)
~/.gemini/skills/text-skills          →  skills/text-skills/
~/.gemini/skills/image-skills         →  skills/image-skills/
~/.gemini/skills/video-skills         →  skills/video-skills/
~/.gemini/skills/thinking-skills      →  skills/thinking-skills/
~/.gemini/skills/grounding-skills     →  skills/grounding-skills/
~/.gemini/skills/model-skills         →  skills/model-skills/
~/.gemini/skills/embedding-skills     →  skills/embedding-skills/
~/.gemini/skills/translate-skills     →  skills/translate-skills/
~/.gemini/skills/speech-skills        →  skills/speech-skills/
~/.gemini/skills/computer-use-skills  →  skills/computer-use-skills/
~/.gemini/skills/url-context-skills   →  skills/url-context-skills/
```

---

## 스킬 업데이트

스킬 파일은 git으로 관리됩니다. 심볼릭 링크는 항상 최신 파일을 가리키므로 **별도 재배포 없이** 수정 즉시 반영됩니다.

```bash
# 스킬 수정 후
git add skills/<skill-name>/SKILL.md
git commit -m "feat: update <section> in <skill-name>"
# 완료 — Gemini CLI는 다음 실행 시 자동으로 업데이트된 내용을 로드합니다
```

---

## SDK 참고 문서

- [google-genai Python SDK](https://googleapis.github.io/python-genai/)
- [Vertex AI Gemini API](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/overview)
- [Imagen on Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs/image/overview)
- [Veo on Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs/video/overview)
