---
name: thinking-skills
description: Use when working with thinking/reasoning models on Vertex AI using google-genai Python SDK — imports ThinkingConfig, or requests involving "thinking budget", "thinking level", "thought signatures", "reasoning model", "extended thinking". 한국어: "사고 모드", "추론 모델", "생각 예산", "사고 시그니처", "씽킹".
version: 1.0.0
---

# Thinking Skills — google-genai Python SDK 레퍼런스

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

### 모델 선택 가이드

| 모델 | 사고 방식 | 비활성화 가능 | 최대 토큰 |
|------|----------|-------------|---------|
| `gemini-2.5-flash` | `thinking_budget` | ✅ (0으로 설정) | 24,576 |
| `gemini-2.5-flash-lite` | `thinking_budget` | ✅ (0으로 설정) | 24,576 |
| `gemini-2.5-pro` | `thinking_budget` | ❌ | 32,768 |
| `gemini-3-flash` | `thinking_level` | ✅ | — |
| `gemini-3.1-pro` | `thinking_level` | ❌ | — |

> **선택 기준:**
> - Gemini 2.5 → `thinking_budget`으로 세밀한 토큰 예산 제어
> - Gemini 3+ → `thinking_level`로 사고 강도 선택
> - `thinking_budget`과 `thinking_level`을 동시에 지정할 수 없다.

---

## § 1. Thinking Budget 설정 (Gemini 2.5)

### 언제: 토큰 예산으로 사고 깊이를 직접 제어할 때

```python
from google import genai
from google.genai.types import GenerateContentConfig, HttpOptions, ThinkingConfig

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="solve x^2 + 4x + 4 = 0",
    config=GenerateContentConfig(
        thinking_config=ThinkingConfig(
            thinking_budget=1024,   # 사고에 사용할 최대 토큰 수
        ),
    ),
)

print(response.text)
# 사고 토큰 사용량 확인
print(f"Thoughts: {response.usage_metadata.thoughts_token_count}")
print(f"Total:    {response.usage_metadata.total_token_count}")
```

### thinking_budget 값 가이드

| 값 | 동작 |
|----|------|
| `1` ~ 모델 최대값 | 지정한 토큰 수 내에서 사고 |
| `0` | 사고 비활성화 (gemini-2.5-pro는 지원 안 함) |
| `-1` | 동적 자동 제어 (기본값, 최대 8,192) |

### 모델별 thinking_budget 범위

| 모델 | 최솟값 | 최댓값 |
|------|--------|--------|
| `gemini-2.5-flash` | 1 | 24,576 |
| `gemini-2.5-flash-lite` | 512 | 24,576 |
| `gemini-2.5-pro` | 128 | 32,768 |

> **원칙:**
> - `thinking_budget=0`으로 사고를 비활성화하면 응답 속도가 빨라지지만 추론 품질이 낮아질 수 있다.
> - `gemini-2.5-pro`는 `thinking_budget=0`을 지원하지 않는다.
> - 파인튜닝은 사고 활성화 상태에서 지원하지 않는다.

---

## § 2. Thinking Level 설정 (Gemini 3+)

### 언제: Gemini 3 이상 모델에서 사고 강도를 선택할 때

```python
from google import genai
from google.genai import types
from google.genai.types import HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-3.1-pro",
    contents="How does AI work?",
    config=types.GenerateContentConfig(
        thinking_config=types.ThinkingConfig(
            thinking_level=types.ThinkingLevel.HIGH,
        ),
    ),
)

print(response.text)
```

### ThinkingLevel 열거형 값

| 값 | 토큰 사용 | 적합한 태스크 | 지원 모델 |
|----|---------|------------|---------|
| `ThinkingLevel.MINIMAL` | 최소 | 간단한 분류, 단순 답변 | gemini-3-flash, gemini-3.1-flash-lite |
| `ThinkingLevel.LOW` | 적음 | 일반 질답 | 모든 Gemini 3 |
| `ThinkingLevel.MEDIUM` | 보통 | 중간 복잡도 문제 | 모든 Gemini 3 |
| `ThinkingLevel.HIGH` | 많음 | 복잡한 추론, 수학, 코드 | 모든 Gemini 3 |

> **원칙:**
> - `ThinkingLevel.MINIMAL`은 Thought Signatures가 필요하며, 없으면 400 오류가 발생한다.
> - `thinking_level`과 `thinking_budget`을 동시에 지정할 수 없다.
> - `gemini-3.1-pro`는 사고를 비활성화할 수 없다.

---

## § 3. Thought 요약 보기

### 언제: 모델의 사고 과정을 응답과 함께 확인할 때

```python
from google import genai
from google.genai.types import GenerateContentConfig, HttpOptions, ThinkingConfig

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-pro",
    contents="solve x^2 + 4x + 4 = 0",
    config=GenerateContentConfig(
        thinking_config=ThinkingConfig(
            include_thoughts=True,   # 사고 요약 포함
        ),
    ),
)

# 최종 답변과 사고 과정 분리
for part in response.candidates[0].content.parts:
    if part.thought:
        print(f"[Thought] {part.text}")
    else:
        print(f"[Answer]  {part.text}")
```

> **원칙:**
> - `include_thoughts=True`는 베스트 에포트(best-effort) 기능이다 — 항상 사고 내용이 반환되지 않을 수 있다.
> - `part.thought`가 `True`인 파트가 사고 요약, `False`인 파트가 최종 응답이다.
> - 사고 요약은 축약본이다. 완전한 내부 사고 과정이 아니다.
