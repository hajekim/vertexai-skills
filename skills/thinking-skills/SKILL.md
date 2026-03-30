---
name: thinking-skills
description: Use when working with thinking/reasoning models on Vertex AI using google-genai Python SDK — imports ThinkingConfig, or requests involving "thinking budget", "thinking level", "thought signatures", "reasoning model", "extended thinking". Korean: "사고 모드", "추론 모델", "생각 예산", "사고 시그니처", "씽킹".
version: 1.0.0
---

# Thinking Skills — google-genai Python SDK Reference

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

| Model | Thinking Mode | Can Disable | Max Tokens |
|-------|--------------|-------------|------------|
| `gemini-2.5-flash` | `thinking_budget` | ✅ (set to 0) | 24,576 |
| `gemini-2.5-flash-lite` | `thinking_budget` | ✅ (set to 0) | 24,576 |
| `gemini-2.5-pro` | `thinking_budget` | ❌ | 32,768 |
| `gemini-3-flash` | `thinking_level` | ✅ | — |
| `gemini-3.1-pro` | `thinking_level` | ❌ | — |

> **When to choose:**
> - Gemini 2.5 → `thinking_budget` for fine-grained token budget control
> - Gemini 3+ → `thinking_level` for selecting thinking intensity
> - `thinking_budget` and `thinking_level` cannot be set at the same time.

---

## § 1. Thinking Budget (Gemini 2.5)

### When: you want direct control over thinking depth via token budget

```python
from google import genai
from google.genai.types import GenerateContentConfig, HttpOptions, ThinkingConfig

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="solve x^2 + 4x + 4 = 0",
    config=GenerateContentConfig(
        thinking_config=ThinkingConfig(
            thinking_budget=1024,   # max tokens to spend on thinking
        ),
    ),
)

print(response.text)
# Check thinking token usage
print(f"Thoughts: {response.usage_metadata.thoughts_token_count}")
print(f"Total:    {response.usage_metadata.total_token_count}")
```

### thinking_budget Value Guide

| Value | Behavior |
|-------|----------|
| `1` ~ model max | Think within the specified token limit |
| `0` | Disable thinking (not supported on gemini-2.5-pro) |
| `-1` | Dynamic auto-control (default, up to 8,192) |

### thinking_budget Range by Model

| Model | Min | Max |
|-------|-----|-----|
| `gemini-2.5-flash` | 1 | 24,576 |
| `gemini-2.5-flash-lite` | 512 | 24,576 |
| `gemini-2.5-pro` | 128 | 32,768 |

> **Rule:**
> - Setting `thinking_budget=0` speeds up responses but may reduce reasoning quality.
> - `gemini-2.5-pro` does not support `thinking_budget=0`.
> - Fine-tuning is not supported with thinking enabled.

---

## § 2. Thinking Level (Gemini 3+)

### When: selecting thinking intensity on Gemini 3 or later models

```python
from google import genai
from google.genai.types import GenerateContentConfig, HttpOptions, ThinkingConfig, ThinkingLevel

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-3.1-pro",
    contents="How does AI work?",
    config=GenerateContentConfig(
        thinking_config=ThinkingConfig(
            thinking_level=ThinkingLevel.HIGH,
        ),
    ),
)

print(response.text)
```

### ThinkingLevel Enum Values

| Value | Token Usage | Suitable Tasks | Supported Models |
|-------|-------------|---------------|-----------------|
| `ThinkingLevel.MINIMAL` | Minimum | Simple classification, short answers | gemini-3-flash, gemini-3.1-flash-lite |
| `ThinkingLevel.LOW` | Low | General Q&A | All Gemini 3 |
| `ThinkingLevel.MEDIUM` | Moderate | Medium-complexity problems | All Gemini 3 |
| `ThinkingLevel.HIGH` | High | Complex reasoning, math, code | All Gemini 3 |

> **Rule:**
> - `ThinkingLevel.MINIMAL` requires Thought Signatures; missing signatures cause a 400 error.
> - `thinking_level` and `thinking_budget` cannot be set simultaneously.
> - `gemini-3.1-pro` cannot disable thinking.

---

## § 3. Viewing Thought Summaries

### When: you want to see the model's reasoning process alongside the response

```python
from google import genai
from google.genai.types import GenerateContentConfig, HttpOptions, ThinkingConfig

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-pro",
    contents="solve x^2 + 4x + 4 = 0",
    config=GenerateContentConfig(
        thinking_config=ThinkingConfig(
            include_thoughts=True,   # include thought summary
        ),
    ),
)

# Separate final answer from thought process
for part in response.candidates[0].content.parts:
    if part.thought:
        print(f"[Thought] {part.text}")
    else:
        print(f"[Answer]  {part.text}")
```

> **Rule:**
> - `include_thoughts=True` is best-effort — thought content may not always be returned.
> - Parts where `part.thought` is `True` are thought summaries; `False` parts are the final response.
> - Thought summaries are abbreviated — not the complete internal reasoning process.

---

## § 4. Thought Signatures (with Function Calling)

### When: maintaining thinking continuity in multi-turn conversations with function calls

Thought Signatures are encrypted tokens that preserve the model's thinking state between function calls. The SDK handles this automatically.

```python
from google import genai
from google.genai.types import (
    Content,
    FunctionDeclaration,
    GenerateContentConfig,
    HttpOptions,
    Part,
    ThinkingConfig,
    Tool,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))

# 1. Define the tool
get_weather = FunctionDeclaration(
    name="get_weather",
    description="Gets current weather for a location.",
    parameters={
        "type": "object",
        "properties": {"location": {"type": "string"}},
        "required": ["location"],
    },
)
tool = Tool(function_declarations=[get_weather])

# 2. First request — model requests a function call
prompt = "What's the weather in Seoul?"
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=prompt,
    config=GenerateContentConfig(
        tools=[tool],
        thinking_config=ThinkingConfig(include_thoughts=True),
    ),
)

# 3. Execute function (actual API call or mock data)
func_call = response.function_calls[0]
func_result = {"location": func_call.args["location"], "temperature": "18°C"}

# 4. Build history with function result — SDK handles signature automatically
history = [
    Content(role="user", parts=[Part(text=prompt)]),
    response.candidates[0].content,   # includes Thought Signature
    Content(
        role="tool",
        parts=[Part.from_function_response(name=func_call.name, response=func_result)],
    ),
]

# 5. Second request — thinking continuity preserved via signature
response2 = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=history,
    config=GenerateContentConfig(
        tools=[tool],
        thinking_config=ThinkingConfig(include_thoughts=True),
    ),
)

print(response2.text)
```

### Thought Signature Rules

| Rule | Description |
|------|-------------|
| SDK auto-handles | Passing `response.candidates[0].content` as-is includes the signature automatically |
| Do not modify Parts | Do not merge or modify a `Part` that contains a signature |
| Parallel function calls | Only the first `functionCall` Part contains the signature; all function calls must precede responses |
| Required for Gemini 3 | Gemini 3+ models return a 400 error if signatures are missing |

> **Rule:**
> - Using `chat.send_message()` handles signatures automatically.
> - When implementing directly via REST API, preserve signatures at the `Part` level.
> - Use `skip_thought_signature_validator` to skip signature validation, but this degrades performance.
