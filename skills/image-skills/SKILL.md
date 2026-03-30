---
name: image-skills
description: Use when working with image generation or editing on Vertex AI using google-genai Python SDK — imports Imagen or Gemini image models, or requests involving "image generation", "generate image", "edit image", "Imagen", "Gemini image". Korean: "이미지 생성", "이미지 편집", "이미지 만들어줘", "아이마젠", "제미나이 이미지".
version: 1.0.0
---

# Image Skills — google-genai Python SDK Reference

## Environment Setup

### Install Libraries

```bash
pip install --upgrade google-genai pillow   # pillow: required for saving image files
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

| Model | Use Case | API |
|-------|----------|-----|
| `imagen-4.0-generate-001` | Highest quality static images | `generate_images()` |
| `gemini-2.5-flash-image` | Generation + editing, lightweight | `generate_content()` |
| `gemini-3-pro-image-preview` | Generation + editing, highest quality | `generate_content()` |
| `gemini-3.1-flash-image-preview` | Generation + editing, latest lightweight | `generate_content()` |

> **When to choose:**
> - High-quality static images only → Imagen 4
> - Editing, multi-turn, or mixed text+image → Gemini Image models

---

## § 1. Imagen 4 Image Generation

### When: you need the highest quality static images

```python
from google import genai
from google.genai.types import GenerateImagesConfig, HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_images(
    model="imagen-4.0-generate-001",
    prompt="A serene mountain lake at sunset with reflections",
    config=GenerateImagesConfig(
        number_of_images=1,        # number of images to generate (1–4)
        aspect_ratio="1:1",        # "1:1", "4:3", "3:4", "16:9", "9:16"
        language="ko",             # prompt language hint (optional)
    ),
)

# Save images
for i, generated in enumerate(response.generated_images):
    generated.image.save(f"output_{i}.png")
    print(f"Saved output_{i}.png")
```

> **Rule:**
> - `generate_images()` is Imagen-only. Cannot be used with Gemini models.
> - Results are returned as `response.generated_images` list.
> - `pillow` library is required for saving.

---

## § 2. Gemini Image Generation

### When: you need images alongside text descriptions, or multimodal responses

```python
from google import genai
from google.genai.types import GenerateContentConfig, HttpOptions, Modality
from PIL import Image
from io import BytesIO
import os

client = genai.Client(http_options=HttpOptions(api_version="v1"))

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

### Using a High-Quality Model

```python
response = client.models.generate_content(
    model="gemini-3-pro-image-preview",   # highest quality
    contents="Generate a photorealistic portrait of a golden retriever in a park.",
    config=GenerateContentConfig(
        response_modalities=[Modality.TEXT, Modality.IMAGE],
    ),
)
```

> **Rule:**
> - `response_modalities=[Modality.TEXT, Modality.IMAGE]` is required.
> - Iterate response parts and branch on `part.text` vs `part.inline_data`.
> - Image data is returned as `part.inline_data.data` (bytes).

---

## § 3. Gemini Image Editing

### When: you need to modify an existing image or apply a style transformation

```python
from google import genai
from google.genai.types import GenerateContentConfig, HttpOptions, Modality
from PIL import Image
from io import BytesIO

client = genai.Client(http_options=HttpOptions(api_version="v1"))

# Load source image
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

### Multi-turn Editing (sequential refinement)

```python
chat = client.chats.create(
    model="gemini-3-pro-image-preview",
    config=GenerateContentConfig(
        response_modalities=[Modality.TEXT, Modality.IMAGE],
    ),
)

# First edit
source_image = Image.open("input.png")
response1 = chat.send_message([source_image, "Make the sky more dramatic with dark clouds."])

# Second edit (continuing from previous result)
response2 = chat.send_message("Now add lightning bolts in the sky.")

for part in response2.candidates[0].content.parts:
    if part.inline_data:
        Image.open(BytesIO(part.inline_data.data)).save("final.png")
```

> **Rule:**
> - Pass the source image as the first element of the `contents` list.
> - Use `client.chats.create()` to maintain session for multi-turn editing.
> - Keep total request size under 50MB.

---

## § 4. Best Practices

### Prompt Writing Guidelines

**Be specific and descriptive**
```
# Bad
"a warrior"

# Good
"A medieval elf archer with leather arm guards, fingerless gloves, and a wooden quiver"
```

**State the purpose clearly**
```
# Bad
"make a logo"

# Good
"A minimalist premium logo for a skincare brand, clean lines, pastel tones"
```

**Use positive descriptions**
```
# Bad
"no cars in the street"

# Good
"an empty cobblestone street"
```

### Camera and Composition Terms

| Goal | Example Terms |
|------|--------------|
| Wide scene | `wide-angle shot`, `establishing shot` |
| Portrait close-up | `close-up portrait`, `macro shot` |
| Dramatic atmosphere | `low-angle shot`, `Dutch angle` |
| Natural feel | `eye-level shot`, `candid style` |

### Iterative Refinement

If the initial result isn't perfect, adjust with follow-up prompts:
```
"Make the lighting warmer"
"Make the expression more serious"
"Add more detail to the background"
```

### Step-by-step Composition (for complex scenes)

Build complex images incrementally:
1. Set background: `"A foggy Victorian street at night"`
2. Add main subject: `"Add a detective in a trench coat"`
3. Add details: `"Add gas lamps and wet cobblestones"`

> **Rule:** Ambiguous prompts may return text-only responses. Explicitly request image generation.

---

## § 5. Limitations

### Language Support

| Model | Supported Languages |
|-------|-------------------|
| `gemini-2.5-flash-image` | EN, es-MX, ja-JP, zh-CN, hi-IN |
| `gemini-3-pro-image-preview` | AR, DE, EN, ES, FR, HI, ID, IT, JA, KO, PT, RU, UK, VI, ZH |

### Input Limitations

- Audio/video input not supported (image + text only)
- Max images per request: Gemini 2.5 Flash → 3 / Gemini 3 Pro → 14
- Total request size: under 50MB

### Output Limitations

- Fewer images than requested may be returned (due to safety filters)
- For text within images: generate text first → then generate image (two-step approach)
- Ambiguous prompts may return text only → explicitly request image generation

---

## § 6. Responsible AI & Safety

### Prohibited Content

The following categories are prohibited under the Generative AI Prohibited Use Policy:
- Child exploitation content
- Violent extremism / terrorism
- Non-consensual intimate images
- Self-harm promotion
- Sexually explicit content
- Hate speech
- Harassment / cyberbullying

### Handling Safety Filter Errors

```python
from google import genai
from google.genai.types import GenerateContentConfig, HttpOptions, Modality
from PIL import Image
from io import BytesIO

client = genai.Client(http_options=HttpOptions(api_version="v1"))

response = client.models.generate_content(
    model="gemini-3-pro-image-preview",
    contents="Generate an image...",
    config=GenerateContentConfig(
        response_modalities=[Modality.TEXT, Modality.IMAGE],
    ),
)

# Check if blocked by safety filter
candidate = response.candidates[0]
if candidate.finish_reason.name == "SAFETY":
    print("Safety filter blocked this request")
    for rating in candidate.safety_ratings:
        if rating.blocked:
            print(f"Blocked by: {rating.category}")
else:
    # Normal processing
    for part in candidate.content.parts:
        if part.inline_data:
            Image.open(BytesIO(part.inline_data.data)).save("output.png")
```

> **Rule:**
> - If `finish_reason` is `SAFETY`, revise the prompt.
> - Safety filter sensitivity can be adjusted per application requirements.
