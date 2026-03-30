# Vertex AI Skills Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `google-genai` Python SDK로 Vertex AI를 사용하는 Gemini CLI 레퍼런스 가이드 스킬(`SKILL.md`) 작성 및 배포.

**Architecture:** 단일 `SKILL.md` 파일에 환경 설정 + 6개 섹션(텍스트 생성, 시스템 지시, 함수 호출, 구조화된 출력, 생성 파라미터, 코드 실행)을 계층적으로 구성. 프로젝트 내 소스를 두고 `~/.gemini/skills/`에 심볼릭 링크로 배포.

**Tech Stack:** `google-genai` Python SDK, Gemini CLI skills 시스템 (SKILL.md frontmatter)

---

## File Map

| 파일 | 역할 |
|------|------|
| `skills/vertexai-skills/SKILL.md` | 스킬 본문 (소스) |
| `~/.gemini/skills/vertexai-skills` → `skills/vertexai-skills/` | Gemini CLI 배포용 심볼릭 링크 |

---

## Task 1: SKILL.md 스캐폴딩 — frontmatter + 환경 설정 섹션

**Files:**
- Create: `skills/vertexai-skills/SKILL.md`

- [ ] **Step 1: frontmatter + 환경 설정 섹션 작성**

`skills/vertexai-skills/SKILL.md`를 아래 내용으로 생성:

```markdown
---
name: vertexai-skills
description: Use when working with google-genai Python SDK on Vertex AI — code imports `from google import genai`, or requests involving "Vertex AI", "Gemini on Vertex", "function calling", "structured output", "code execution", "system instruction", "generation parameters". 한국어: "버텍스 AI", "함수 호출", "구조화된 출력", "코드 실행", "시스템 지시", "생성 파라미터", "제미나이 버텍스".
version: 1.0.0
---

# Vertex AI Skills — google-genai Python SDK 레퍼런스

## 환경 설정

### 필수 환경변수

```bash
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"   # 또는 "global"
export GOOGLE_GENAI_USE_VERTEXAI="True"
```

### 클라이언트 초기화 (모듈 수준에서 한 번만)

```python
from google import genai
from google.genai.types import HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))
```

> **원칙:** `client`는 모듈 최상단에서 한 번 초기화한다. 함수마다 재생성하지 않는다.
> 모든 설정은 `GenerateContentConfig`에 통합한다.
```

- [ ] **Step 2: 파일 존재 확인**

```bash
cat skills/vertexai-skills/SKILL.md
```

Expected: frontmatter와 환경 설정 섹션이 출력됨.

- [ ] **Step 3: Commit**

```bash
git init  # 아직 git 저장소가 없는 경우
git add skills/vertexai-skills/SKILL.md
git commit -m "feat: scaffold vertexai-skills SKILL.md with env setup"
```

---

## Task 2: § 1 텍스트 생성 섹션 추가

**Files:**
- Modify: `skills/vertexai-skills/SKILL.md`

- [ ] **Step 1: 텍스트 생성 섹션 추가**

`SKILL.md`의 환경 설정 섹션 아래에 추가:

```markdown
---

## § 1. 텍스트 생성

### 언제: 단순 텍스트 응답이 필요할 때

### 논스트리밍

```python
from google import genai
from google.genai.types import HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="How does AI work?",
)
print(response.text)
```

### 스트리밍 (멀티턴 채팅)

```python
chat = client.chats.create(model="gemini-2.5-flash")

for chunk in chat.send_message_stream("Why is the sky blue?"):
    print(chunk.text, end="")

# 이어서 대화 계속
for chunk in chat.send_message_stream("Tell me more."):
    print(chunk.text, end="")
```

> **선택 기준:**
> - 응답을 즉시 화면에 표시해야 하면 → 스트리밍
> - 응답 전체를 처리한 후 사용해야 하면 → 논스트리밍
```

- [ ] **Step 2: 문법 확인**

```bash
head -80 skills/vertexai-skills/SKILL.md
```

Expected: frontmatter, 환경 설정, § 1 섹션이 순서대로 보임.

- [ ] **Step 3: Commit**

```bash
git add skills/vertexai-skills/SKILL.md
git commit -m "feat: add §1 text generation section to vertexai-skills"
```

---

## Task 3: § 2 시스템 지시 섹션 추가

**Files:**
- Modify: `skills/vertexai-skills/SKILL.md`

- [ ] **Step 1: 시스템 지시 섹션 추가**

`SKILL.md`의 § 1 아래에 추가:

```markdown
---

## § 2. 시스템 지시 (System Instruction)

### 언제: 모델의 역할, 말투, 출력 형식을 고정해야 할 때

```python
from google import genai
from google.genai.types import HttpOptions, GenerateContentConfig

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Write a haiku about the sea.",
    config=GenerateContentConfig(
        system_instruction="You are a poet. Always respond in Korean.",
    ),
)
print(response.text)
```

### 멀티턴 채팅에서의 시스템 지시

```python
chat = client.chats.create(
    model="gemini-2.5-flash",
    config=GenerateContentConfig(
        system_instruction="You are a helpful assistant that responds only in bullet points.",
    ),
)

response = chat.send_message("What are the benefits of exercise?")
print(response.text)
```

> **원칙:** 시스템 지시는 `GenerateContentConfig.system_instruction`에 설정한다.
> 프롬프트 앞에 직접 붙이지 않는다.
```

- [ ] **Step 2: Commit**

```bash
git add skills/vertexai-skills/SKILL.md
git commit -m "feat: add §2 system instruction section to vertexai-skills"
```

---

## Task 4: § 3 함수 호출 섹션 추가

**Files:**
- Modify: `skills/vertexai-skills/SKILL.md`

- [ ] **Step 1: 함수 호출 섹션 추가**

`SKILL.md`의 § 2 아래에 추가:

```markdown
---

## § 3. 함수 호출 (Function Calling)

### 언제: 외부 API, 데이터베이스, 실시간 정보를 모델과 연결해야 할 때

### Step 1: 함수 선언

```python
from google import genai
from google.genai import types
from google.genai.types import HttpOptions, GenerateContentConfig

client = genai.Client(http_options=HttpOptions(api_version="v1"))

get_weather_func = types.FunctionDeclaration(
    name="get_current_weather",
    description="Get the current weather in a given location.",
    parameters={
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "City name, e.g. 'Seoul'",
            }
        },
        "required": ["location"],
    },
)
```

### Step 2: 모델에 도구 전달 및 함수 호출 응답 수신

```python
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="What is the weather in Seoul?",
    config=GenerateContentConfig(
        tools=[types.Tool(function_declarations=[get_weather_func])],
    ),
)

function_call = response.function_calls[0]
print(function_call.name)   # "get_current_weather"
print(function_call.args)   # {"location": "Seoul"}
```

### Step 3: 함수 실행 → 결과 반환

```python
# 실제 함수 실행 (예시)
api_result = {"location": "Seoul", "temperature": 15, "condition": "Cloudy"}

# 결과를 모델에 반환
final_response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=[
        types.Content(role="user", parts=[types.Part(text="What is the weather in Seoul?")]),
        response.candidates[0].content,  # 모델의 함수 호출 요청
        types.Content(
            role="user",
            parts=[
                types.Part.from_function_response(
                    name=function_call.name,
                    response={"contents": api_result},
                )
            ],
        ),
    ],
    config=GenerateContentConfig(
        tools=[types.Tool(function_declarations=[get_weather_func])],
    ),
)
print(final_response.text)
```

> **원칙:**
> - `response.function_calls`로 함수 호출 여부를 확인한다.
> - 함수 결과는 반드시 `Part.from_function_response()`로 감싸서 반환한다.
> - 한 요청에 최대 512개 함수 선언 가능.
```

- [ ] **Step 2: Commit**

```bash
git add skills/vertexai-skills/SKILL.md
git commit -m "feat: add §3 function calling section to vertexai-skills"
```

---

## Task 5: § 4 구조화된 출력 섹션 추가

**Files:**
- Modify: `skills/vertexai-skills/SKILL.md`

- [ ] **Step 1: 구조화된 출력 섹션 추가**

`SKILL.md`의 § 3 아래에 추가:

```markdown
---

## § 4. 구조화된 출력 (Controlled Output)

### 언제: JSON 형식의 일관된 응답이 필요할 때 (파싱, 데이터 처리 등)

```python
from google import genai
from google.genai.types import HttpOptions, GenerateContentConfig

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response_schema = {
    "type": "ARRAY",
    "items": {
        "type": "OBJECT",
        "properties": {
            "name": {"type": "STRING"},
            "price": {"type": "NUMBER"},
            "in_stock": {"type": "BOOLEAN"},
        },
        "required": ["name", "price", "in_stock"],
    },
}

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="List 3 popular fruits with their approximate price per kg and availability.",
    config=GenerateContentConfig(
        response_mime_type="application/json",
        response_schema=response_schema,
    ),
)

import json
data = json.loads(response.text)
print(data)
# [{"name": "Apple", "price": 3.5, "in_stock": true}, ...]
```

### 지원 타입

| JSON Schema 타입 | Python 예시 |
|-----------------|-------------|
| `STRING` | 문자열 |
| `NUMBER` | 정수/실수 |
| `BOOLEAN` | True/False |
| `ARRAY` | 리스트 |
| `OBJECT` | 딕셔너리 |

> **원칙:**
> - `response_mime_type`은 반드시 `"application/json"`으로 설정한다.
> - `required` 필드를 명시해야 모델이 해당 필드를 반드시 채운다.
> - 응답은 `json.loads(response.text)`로 파싱한다.
```

- [ ] **Step 2: Commit**

```bash
git add skills/vertexai-skills/SKILL.md
git commit -m "feat: add §4 controlled output section to vertexai-skills"
```

---

## Task 6: § 5 생성 파라미터 섹션 추가

**Files:**
- Modify: `skills/vertexai-skills/SKILL.md`

- [ ] **Step 1: 생성 파라미터 섹션 추가**

`SKILL.md`의 § 4 아래에 추가:

```markdown
---

## § 5. 생성 파라미터 (Generation Parameters)

### 언제: 응답의 창의성, 길이, 반복성을 조절해야 할 때

```python
from google import genai
from google.genai.types import HttpOptions, GenerateContentConfig

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Write a creative story about a robot.",
    config=GenerateContentConfig(
        temperature=1.0,          # 0.0(결정적) ~ 2.0(창의적). 기본값 1.0 권장
        top_p=0.95,               # 누적 확률 임계값. temperature와 함께 사용
        top_k=20,                 # 상위 K개 토큰만 고려
        max_output_tokens=500,    # 최대 출력 토큰 수 (100토큰 ≈ 60~80 단어)
        stop_sequences=["END"],   # 이 문자열 등장 시 생성 중단
        seed=42,                  # 재현 가능한 출력을 위한 시드
        presence_penalty=0.0,     # 이미 등장한 토큰 억제 (-2.0 ~ 2.0)
        frequency_penalty=0.0,    # 반복 토큰 억제 (-2.0 ~ 2.0)
    ),
)
print(response.text)
```

### 파라미터 선택 가이드

| 목적 | 설정 |
|------|------|
| 정확한 사실 기반 응답 | `temperature=0.0` |
| 일반적인 대화 | `temperature=1.0` (기본값) |
| 창의적 글쓰기 | `temperature=1.5~2.0` |
| 응답 길이 제한 | `max_output_tokens=200` |
| 반복 억제 | `frequency_penalty=0.5~1.0` |

> **원칙:** 모든 파라미터는 `GenerateContentConfig`에 한 번에 설정한다.
> `temperature`와 `top_p`는 함께 사용 시 상호작용하므로 하나씩 조정한다.
```

- [ ] **Step 2: Commit**

```bash
git add skills/vertexai-skills/SKILL.md
git commit -m "feat: add §5 generation parameters section to vertexai-skills"
```

---

## Task 7: § 6 코드 실행 섹션 추가

**Files:**
- Modify: `skills/vertexai-skills/SKILL.md`

- [ ] **Step 1: 코드 실행 섹션 추가**

`SKILL.md`의 § 5 아래에 추가:

```markdown
---

## § 6. 코드 실행 (Code Execution)

### 언제: 수학 계산, 데이터 처리, 알고리즘 검증을 모델이 직접 실행해야 할 때

```python
from google import genai
from google.genai.types import (
    HttpOptions,
    Tool,
    ToolCodeExecution,
    GenerateContentConfig,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Calculate the 20th Fibonacci number, then find the nearest palindrome.",
    config=GenerateContentConfig(
        tools=[Tool(code_execution=ToolCodeExecution())],
        temperature=0,
    ),
)

# 모델이 생성한 코드
print(response.executable_code)

# 코드 실행 결과
print(response.code_execution_result)

# 최종 텍스트 응답
print(response.text)
```

### 사전 설치된 라이브러리

`numpy`, `pandas`, `matplotlib`, `sympy` 등 주요 과학 라이브러리가 기본 포함됨.
커스텀 패키지 설치는 불가.

### Function Calling과의 차이

| | Code Execution | Function Calling |
|--|---------------|-----------------|
| 실행 주체 | API 백엔드 (격리 환경) | 내 애플리케이션 |
| 용도 | 계산, 알고리즘 | 외부 API, DB 연동 |
| 라이브러리 | 사전 설치된 것만 | 제한 없음 |
| 단일 요청 완결 | O | X (루프 필요) |

> **원칙:** 외부 서비스 연동이 필요 없는 순수 계산/데이터 처리는 Code Execution을 우선 사용한다.
```

- [ ] **Step 2: Commit**

```bash
git add skills/vertexai-skills/SKILL.md
git commit -m "feat: add §6 code execution section to vertexai-skills"
```

---

## Task 8: 심볼릭 링크 생성 및 최종 검증

**Files:**
- Symlink: `~/.gemini/skills/vertexai-skills` → `<project>/skills/vertexai-skills/`

- [ ] **Step 1: 프로젝트 절대 경로 확인**

```bash
pwd
```

Expected: `/home/ext_hajekim_google_com/sandbox/vertexai-skills` (또는 실제 경로)

- [ ] **Step 2: 심볼릭 링크 생성**

```bash
ln -s /home/ext_hajekim_google_com/sandbox/vertexai-skills/skills/vertexai-skills \
      /home/ext_hajekim_google_com/.gemini/skills/vertexai-skills
```

- [ ] **Step 3: 심볼릭 링크 확인**

```bash
ls -la ~/.gemini/skills/ | grep vertexai-skills
```

Expected: `vertexai-skills -> /home/ext_hajekim_google_com/sandbox/vertexai-skills/skills/vertexai-skills`

- [ ] **Step 4: SKILL.md 접근 확인**

```bash
head -10 ~/.gemini/skills/vertexai-skills/SKILL.md
```

Expected: frontmatter (`---`, `name: vertexai-skills`, 등)가 보임.

- [ ] **Step 5: 전체 섹션 구조 확인**

```bash
grep "^## " ~/.gemini/skills/vertexai-skills/SKILL.md
```

Expected:
```
## 환경 설정
## § 1. 텍스트 생성
## § 2. 시스템 지시 (System Instruction)
## § 3. 함수 호출 (Function Calling)
## § 4. 구조화된 출력 (Controlled Output)
## § 5. 생성 파라미터 (Generation Parameters)
## § 6. 코드 실행 (Code Execution)
```

- [ ] **Step 6: 최종 Commit**

```bash
git add .
git commit -m "feat: deploy vertexai-skills via symlink to ~/.gemini/skills"
```
