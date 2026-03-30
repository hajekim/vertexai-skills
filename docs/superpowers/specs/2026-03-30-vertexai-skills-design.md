# Vertex AI Skills — Design Spec
**Date:** 2026-03-30
**Status:** Approved

---

## 1. 목적

Gemini CLI가 `google-genai` Python SDK로 Vertex AI 코드를 작성할 때 자동으로 로드되는 레퍼런스 가이드 스킬. 올바른 SDK 사용 패턴, 클라이언트 초기화, 6가지 핵심 기능 코드 예제를 제공한다.

---

## 2. 파일 구조

```
vertexai-skills/                        ← 프로젝트 루트
├── docs/
│   └── superpowers/
│       └── specs/
│           └── 2026-03-30-vertexai-skills-design.md
└── skills/
    └── vertexai-skills/
        └── SKILL.md                    ← 스킬 본문 (단일 파일)

# 배포: 심볼릭 링크
~/.gemini/skills/vertexai-skills/  →  <project>/skills/vertexai-skills/
```

---

## 3. 스킬 메타데이터

```yaml
name: vertexai-skills
description: |
  Use when working with google-genai Python SDK on Vertex AI — code imports
  `from google import genai`, or requests involving "Vertex AI", "Gemini on Vertex",
  "function calling", "structured output", "code execution", "system instruction",
  "generation parameters". 한국어: "버텍스 AI", "함수 호출", "구조화된 출력",
  "코드 실행", "시스템 지시", "생성 파라미터"
version: 1.0.0
```

**트리거 조건 (A+B):**
- Vertex AI SDK 코드 작업 시 (`from google import genai`, `genai.Client()` 등)
- 특정 기능 키워드 요청 시 (function calling, structured output, code execution 등)

---

## 4. SKILL.md 내용 구조

### 환경 설정
- 필수 환경변수: `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION`, `GOOGLE_GENAI_USE_VERTEXAI=True`
- 클라이언트 초기화: `genai.Client(http_options=HttpOptions(api_version="v1"))` — 모듈 수준에서 한 번만

### § 1. 텍스트 생성
- 논스트리밍: `client.models.generate_content()`
- 스트리밍: `client.chats.create()` + `send_message_stream()`

### § 2. 시스템 지시 (System Instruction)
- `GenerateContentConfig(system_instruction=...)` 설정
- 멀티턴 채팅에서의 활용 패턴

### § 3. 함수 호출 (Function Calling)
- `types.FunctionDeclaration` + `types.Tool` 정의
- `response.function_calls` 추출 → 실행 → `Part.from_function_response()` 반환 루프

### § 4. 구조화된 출력 (Controlled Output)
- `response_mime_type: "application/json"`
- `response_schema`: JSON Schema 딕셔너리 형태로 정의

### § 5. 생성 파라미터
- `temperature`, `top_p`, `top_k`, `max_output_tokens`, `stop_sequences`
- `presence_penalty`, `frequency_penalty`, `seed`

### § 6. 코드 실행 (Code Execution)
- `Tool(code_execution=ToolCodeExecution())` 설정
- `response.executable_code` / `response.code_execution_result` 결과 처리

### 핵심 원칙
- 모든 설정은 `GenerateContentConfig`에 통합
- 클라이언트는 모듈 수준에서 한 번만 초기화

---

## 5. 설계 결정 사항

| 결정 | 이유 |
|------|------|
| 단일 스킬 파일 (Monolithic X, Structured 섹션) | 유저 요청이 "레퍼런스 가이드 하나". 섹션 구조로 가독성 확보 |
| `google-genai` SDK 전용 (ai-platform SDK 제외) | 유저 지정. 두 SDK 혼용 시 혼란 방지 |
| Python SDK 전용 (REST/Node.js 제외) | 유저 지정. 범위 집중 |
| 프로젝트 내 소스 + 심볼릭 링크 배포 | 스킬 소스를 git으로 관리하면서 글로벌 접근 가능 |
| Claude Code 심볼릭 링크 없음 | 유저가 A만 선택 (C 제외) |

---

## 6. 구현 범위

- [x] `skills/vertexai-skills/SKILL.md` 작성 (6개 섹션 + 환경 설정)
- [x] `~/.gemini/skills/vertexai-skills` 심볼릭 링크 생성
- [ ] (선택) `~/.claude/skills/vertexai-skills` 심볼릭 링크 (향후 확장 시)
