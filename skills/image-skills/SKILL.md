---
name: image-skills
description: Use when working with image generation or editing on Vertex AI using google-genai Python SDK — imports Imagen or Gemini image models, or requests involving "image generation", "generate image", "edit image", "Imagen", "Gemini image". 한국어: "이미지 생성", "이미지 편집", "이미지 만들어줘", "아이마젠", "제미나이 이미지".
version: 1.0.0
---

# Image Skills — google-genai Python SDK 레퍼런스

## 환경 설정

### 라이브러리 설치

```bash
pip install --upgrade google-genai pillow   # pillow: 이미지 파일 저장에 필요
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

client = genai.Client()
```

### 모델 선택 가이드

| 모델 | 용도 | API |
|------|------|-----|
| `imagen-4.0-generate-001` | 최고 품질 정적 이미지 | `generate_images()` |
| `gemini-2.5-flash-image` | 생성 + 편집, 경량 | `generate_content()` |
| `gemini-3-pro-image-preview` | 생성 + 편집, 최고 품질 | `generate_content()` |
| `gemini-3.1-flash-image-preview` | 생성 + 편집, 최신 경량 | `generate_content()` |

> **선택 기준:**
> - 고품질 정적 이미지만 필요 → Imagen 4
> - 편집, 멀티턴, 텍스트+이미지 혼합 → Gemini Image 모델

---

## § 1. Imagen 4 이미지 생성

### 언제: 최고 품질의 정적 이미지가 필요할 때

```python
from google import genai
from google.genai.types import GenerateImagesConfig

client = genai.Client()

response = client.models.generate_images(
    model="imagen-4.0-generate-001",
    prompt="A serene mountain lake at sunset with reflections",
    config=GenerateImagesConfig(
        number_of_images=1,        # 생성할 이미지 수 (1-4)
        aspect_ratio="1:1",        # "1:1", "4:3", "3:4", "16:9", "9:16"
        language="ko",             # 프롬프트 언어 힌트 (선택)
    ),
)

# 이미지 저장
for i, generated in enumerate(response.generated_images):
    generated.image.save(f"output_{i}.png")
    print(f"Saved output_{i}.png")
```

> **원칙:**
> - `generate_images()`는 Imagen 전용 API다. Gemini 모델에는 사용할 수 없다.
> - 결과는 `response.generated_images` 리스트로 반환된다.
> - 저장 시 `pillow` 라이브러리가 필요하다.

---

## § 2. Gemini 이미지 생성

### 언제: 텍스트 설명과 함께 이미지가 필요하거나 멀티모달 응답이 필요할 때

```python
from google import genai
from google.genai.types import GenerateContentConfig, Modality
from PIL import Image
from io import BytesIO
import os

client = genai.Client()

response = client.models.generate_content(
    model="gemini-2.5-flash-image",
    contents="Generate an image of the Eiffel tower with fireworks in the background.",
    config=GenerateContentConfig(
        response_modalities=[Modality.TEXT, Modality.IMAGE],
    ),
)

os.makedirs("output", exist_ok=True)
for i, part in enumerate(response.candidates[0].content.parts):
    if part.text:
        print(part.text)
    elif part.inline_data:
        image = Image.open(BytesIO(part.inline_data.data))
        image.save(f"output/image_{i}.png")
        print(f"Saved output/image_{i}.png")
```

### 고품질 모델 사용 시

```python
response = client.models.generate_content(
    model="gemini-3-pro-image-preview",   # 최고 품질
    contents="Generate a photorealistic portrait of a golden retriever in a park.",
    config=GenerateContentConfig(
        response_modalities=[Modality.TEXT, Modality.IMAGE],
    ),
)
```

> **원칙:**
> - `response_modalities=[Modality.TEXT, Modality.IMAGE]`는 필수다.
> - 응답 파트를 순회하며 `part.text`와 `part.inline_data`를 분기 처리한다.
> - 이미지 데이터는 `part.inline_data.data` (bytes)로 반환된다.

---

## § 3. Gemini 이미지 편집

### 언제: 기존 이미지를 수정하거나 스타일을 변환해야 할 때

```python
from google import genai
from google.genai.types import GenerateContentConfig, Modality
from PIL import Image
from io import BytesIO

client = genai.Client()

# 원본 이미지 로드
source_image = Image.open("input.png")

response = client.models.generate_content(
    model="gemini-3-pro-image-preview",
    contents=[source_image, "Edit this image to make it look like a watercolor painting."],
    config=GenerateContentConfig(
        response_modalities=[Modality.TEXT, Modality.IMAGE],
    ),
)

for part in response.candidates[0].content.parts:
    if part.text:
        print(part.text)
    elif part.inline_data:
        edited = Image.open(BytesIO(part.inline_data.data))
        edited.save("edited.png")
        print("Saved edited.png")
```

### 멀티턴 편집 (순차적 수정)

```python
chat = client.chats.create(
    model="gemini-3-pro-image-preview",
    config=GenerateContentConfig(
        response_modalities=[Modality.TEXT, Modality.IMAGE],
    ),
)

# 1차 편집
source_image = Image.open("input.png")
response1 = chat.send_message([source_image, "Make the sky more dramatic with dark clouds."])

# 2차 편집 (이전 결과에 이어서)
response2 = chat.send_message("Now add lightning bolts in the sky.")

for part in response2.candidates[0].content.parts:
    if part.inline_data:
        Image.open(BytesIO(part.inline_data.data)).save("final.png")
```

> **원칙:**
> - 편집 프롬프트에 원본 이미지를 `contents` 리스트 첫 번째 요소로 전달한다.
> - 멀티턴 편집은 `client.chats.create()`로 세션을 유지한다.
> - 요청 전체 파일 크기는 50MB 이하로 유지한다.

---

## § 4. Best Practices

### 프롬프트 작성 원칙

**구체적으로 묘사한다**
```
# 나쁜 예
"a warrior"

# 좋은 예
"A medieval elf archer with leather arm guards, fingerless gloves, and a wooden quiver"
```

**목적을 명시한다**
```
# 나쁜 예
"make a logo"

# 좋은 예
"A minimalist premium logo for a skincare brand, clean lines, pastel tones"
```

**긍정적 표현을 사용한다**
```
# 나쁜 예
"no cars in the street"

# 좋은 예
"an empty cobblestone street"
```

### 카메라/구도 용어 활용

| 목적 | 용어 예시 |
|------|---------|
| 넓은 장면 | `wide-angle shot`, `establishing shot` |
| 인물 클로즈업 | `close-up portrait`, `macro shot` |
| 극적 분위기 | `low-angle shot`, `Dutch angle` |
| 자연스러운 느낌 | `eye-level shot`, `candid style` |

### 반복적 개선

초기 결과가 완벽하지 않으면 후속 프롬프트로 조정한다:
```
"Make the lighting warmer"
"Make the expression more serious"
"Add more detail to the background"
```

### 단계별 구성 (복잡한 장면)

복잡한 이미지는 순서대로 지시한다:
1. 배경 설정: `"A foggy Victorian street at night"`
2. 주요 피사체 추가: `"Add a detective in a trench coat"`
3. 세부 요소 추가: `"Add gas lamps and wet cobblestones"`

> **원칙:** 프롬프트가 모호하면 이미지 없이 텍스트만 반환될 수 있다. 이미지 생성을 명시적으로 요청한다.

---

## § 5. Limitations

### 언어 지원

| 모델 | 지원 언어 |
|------|---------|
| `gemini-2.5-flash-image` | EN, es-MX, ja-JP, zh-CN, hi-IN |
| `gemini-3-pro-image-preview` | AR, DE, EN, ES, FR, HI, ID, IT, JA, KO, PT, RU, UK, VI, ZH |

### 입력 제한

- 오디오/비디오 입력 불가 (이미지 + 텍스트만 지원)
- 요청당 최대 이미지 수: Gemini 2.5 Flash → 3개 / Gemini 3 Pro → 14개
- 전체 요청 크기: 50MB 이하

### 출력 한계

- 요청한 이미지 수보다 적게 생성될 수 있음 (안전 필터로 인해)
- 이미지 내 텍스트 생성 시: 텍스트 먼저 생성 → 이미지 생성 2단계 접근 필요
- 프롬프트가 모호하면 이미지 없이 텍스트만 반환될 수 있음 → 명시적으로 이미지 요청

---

## § 6. Responsible AI & Safety

### 금지 콘텐츠

다음 카테고리는 Generative AI Prohibited Use Policy에 의해 금지됨:
- 아동 착취 콘텐츠
- 폭력적 극단주의 / 테러리즘
- 비동의 친밀 이미지
- 자해 조장
- 성적으로 노골적인 콘텐츠
- 혐오 발언
- 괴롭힘 / 사이버불링

### Safety 필터 에러 처리

```python
from google import genai
from google.genai.types import GenerateContentConfig, Modality
from PIL import Image
from io import BytesIO

client = genai.Client()

response = client.models.generate_content(
    model="gemini-3-pro-image-preview",
    contents="Generate an image...",
    config=GenerateContentConfig(
        response_modalities=[Modality.TEXT, Modality.IMAGE],
    ),
)

# 안전 필터 차단 여부 확인
candidate = response.candidates[0]
if candidate.finish_reason.name == "SAFETY":
    print("Safety filter blocked this request")
    for rating in candidate.safety_ratings:
        if rating.blocked:
            print(f"Blocked by: {rating.category}")
else:
    # 정상 처리
    for part in candidate.content.parts:
        if part.inline_data:
            Image.open(BytesIO(part.inline_data.data)).save("output.png")
```

> **원칙:**
> - `finish_reason`이 `SAFETY`이면 프롬프트를 수정해야 한다.
> - 안전 필터 민감도는 애플리케이션 요구사항에 따라 조정 가능하다.
