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
export GOOGLE_CLOUD_LOCATION="us-central1"   # "global"은 일부 모델만 지원
export GOOGLE_GENAI_USE_VERTEXAI="True"
```

### 클라이언트 초기화 (모듈 수준에서 한 번만)

```python
from google import genai
from google.genai.types import HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))
```

### 대안: 직접 프로젝트 지정

```python
client = genai.Client(
    vertexai=True,
    project="your-project-id",
    location="us-central1",
)
```

> **원칙:** `client`는 모듈 최상단에서 한 번 초기화한다. 함수마다 재생성하지 않는다.
> 모든 설정은 `GenerateContentConfig`에 통합한다.

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
