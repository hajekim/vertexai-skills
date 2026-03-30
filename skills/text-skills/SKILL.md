---
name: text-skills
description: Use when working with google-genai Python SDK on Vertex AI — code imports `from google import genai`, or requests involving "text generation", "chat", "function calling", "structured output", "code execution", "system instruction", "generation parameters". Korean: "텍스트 생성", "함수 호출", "구조화된 출력", "코드 실행", "시스템 지시", "생성 파라미터", "채팅".
version: 1.0.0
---

# Text Skills — google-genai Python SDK Reference

## Environment Setup

### Install Libraries

```bash
pip install --upgrade google-genai   # latest stable: 1.69.0 (as of 2026-03)
```

### Required Environment Variables

```bash
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"   # "global" only supported by some models
export GOOGLE_GENAI_USE_VERTEXAI="True"
```

### Client Initialization (once at module level)

```python
from google import genai
from google.genai.types import HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))
```

### Alternative: Explicit Project Configuration

```python
client = genai.Client(
    vertexai=True,
    project="your-project-id",
    location="us-central1",
)
```

> **Rule:** Initialize `client` once at the top of the module. Never recreate it per function call.
> Consolidate all settings into `GenerateContentConfig`.

---

## § 1. Text Generation

### When: simple text response needed

### Non-streaming

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

### Streaming (multi-turn chat)

```python
chat = client.chats.create(model="gemini-2.5-flash")

for chunk in chat.send_message_stream("Why is the sky blue?"):
    print(chunk.text, end="")

# Continue the conversation
for chunk in chat.send_message_stream("Tell me more."):
    print(chunk.text, end="")
```

> **When to choose:**
> - Display response incrementally → streaming
> - Process the full response before using it → non-streaming

---

## § 2. System Instruction

### When: you need to fix the model's role, tone, or output format

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

### System Instruction in Multi-turn Chat

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

> **Rule:** Set system instruction via `GenerateContentConfig.system_instruction`.
> Do not prepend it directly to the prompt.

---

## § 3. Function Calling

### When: you need to connect external APIs, databases, or real-time data to the model

### Step 1: Declare the Function

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

### Step 2: Pass Tool to Model and Receive Function Call

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

### Step 3: Execute Function and Return Result

```python
# Execute the actual function (example)
api_result = {"location": "Seoul", "temperature": 15, "condition": "Cloudy"}

# Return result to the model
final_response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=[
        types.Content(role="user", parts=[types.Part(text="What is the weather in Seoul?")]),
        response.candidates[0].content,  # model's function call request
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

> **Rule:**
> - Check `response.function_calls` to detect function call requests.
> - Wrap function results with `Part.from_function_response()`.
> - Up to 512 function declarations per request.

---

## § 4. Structured Output

### When: you need consistent JSON-formatted responses (for parsing, data processing, etc.)

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

### Supported Types

| JSON Schema Type | Example |
|-----------------|---------|
| `STRING` | string value |
| `NUMBER` | integer or float |
| `BOOLEAN` | True/False |
| `ARRAY` | list |
| `OBJECT` | dictionary |

> **Rule:**
> - `response_mime_type` must be `"application/json"`.
> - Declare `required` fields so the model always populates them.
> - Parse the response with `json.loads(response.text)`.

---

## § 5. Generation Parameters

### When: you need to control creativity, length, or repetition of responses

```python
from google import genai
from google.genai.types import HttpOptions, GenerateContentConfig

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Write a creative story about a robot.",
    config=GenerateContentConfig(
        temperature=1.0,          # 0.0 (deterministic) ~ 2.0 (creative). Default 1.0 recommended
        top_p=0.95,               # cumulative probability threshold; use alongside temperature
        top_k=20,                 # consider only top K tokens
        max_output_tokens=500,    # maximum output tokens (100 tokens ≈ 60–80 words)
        stop_sequences=["END"],   # stop generation when this string appears
        seed=42,                  # seed for reproducible output
        presence_penalty=0.0,     # penalize already-seen tokens (-2.0 ~ 2.0)
        frequency_penalty=0.0,    # penalize repeated tokens (-2.0 ~ 2.0)
    ),
)
print(response.text)
```

### Parameter Selection Guide

| Goal | Setting |
|------|---------|
| Factual, precise responses | `temperature=0.0` |
| General conversation | `temperature=1.0` (default) |
| Creative writing | `temperature=1.5~2.0` |
| Limit response length | `max_output_tokens=200` |
| Suppress repetition | `frequency_penalty=0.5~1.0` |

> **Rule:** Set all parameters in `GenerateContentConfig` at once.
> `temperature` and `top_p` interact — adjust one at a time.

---

## § 6. Code Execution

### When: the model needs to directly run math calculations, data processing, or algorithm verification

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

# Code generated by the model
print(response.executable_code)

# Code execution result
print(response.code_execution_result)

# Final text response
print(response.text)
```

### Pre-installed Libraries

`numpy`, `pandas`, `matplotlib`, `sympy`, and other major scientific libraries are included.
Custom package installation is not supported.

### Code Execution vs Function Calling

| | Code Execution | Function Calling |
|--|---------------|-----------------|
| Executor | API backend (sandboxed) | Your application |
| Use case | Computation, algorithms | External APIs, databases |
| Libraries | Pre-installed only | Unlimited |
| Single-request complete | Yes | No (loop required) |

> **Rule:** For pure computation or data processing that doesn't need external services, prefer Code Execution.
