---
name: video-skills
description: Use when working with video generation on Vertex AI using google-genai Python SDK — imports Veo models, or requests involving "video generation", "generate video", "text to video", "image to video", "extend video", "Veo". Korean: "비디오 생성", "영상 만들기", "텍스트로 영상", "이미지로 영상", "비오", "영상 연장".
version: 1.0.0
---

# Video Skills — google-genai Python SDK Reference

## Environment Setup

### Install Libraries

```bash
pip install --upgrade google-genai
```

### Required Environment Variables and GCS Bucket

```bash
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"
export GOOGLE_GENAI_USE_VERTEXAI="True"
# Generated videos are stored in a GCS bucket — required
export OUTPUT_GCS_URI="gs://your-bucket/output/"
```

### Client Initialization

```python
from google import genai
from google.genai.types import HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))
output_gcs_uri = "gs://your-bucket/output/"  # required for all generation operations
```

### Model List

| Model | Description |
|-------|-------------|
| `veo-3.1-generate-001` | Latest stable version (recommended) |
| `veo-3.1-fast-generate-001` | Faster generation |
| `veo-3.0-generate-001` | Previous stable version |
| `veo-2.0-generate-001` | Legacy (object insert/remove only) |

### Common Pattern: Polling Loop

All video generation runs as an async LRO (Long Running Operation):

```python
import time

# operation = client.models.generate_videos(...)  # start the job
# Wait until complete
while not operation.done:
    time.sleep(15)
    operation = client.operations.get(operation)

for video in operation.result.generated_videos:
    print(video.video.uri)  # GCS URI
```

> **Rule:** Video generation is always async. Poll every 15 seconds until `operation.done` is True.

---

## § 1. Text to Video

### When: generating video from a text prompt only

```python
import time
from google import genai
from google.genai.types import GenerateVideosConfig, HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))
output_gcs_uri = "gs://your-bucket/output/"

operation = client.models.generate_videos(
    model="veo-3.1-generate-001",
    prompt="A cat reading a book in a cozy library, warm lighting, cinematic shot",
    config=GenerateVideosConfig(
        aspect_ratio="16:9",       # "16:9" (landscape) or "9:16" (portrait/mobile)
        number_of_videos=1,        # number of videos to generate (1–4)
        duration_seconds=8,        # video duration (4, 6, or 8 seconds)
        output_gcs_uri=output_gcs_uri,
    ),
)

# Wait until complete
while not operation.done:
    time.sleep(15)
    operation = client.operations.get(operation)

for video in operation.result.generated_videos:
    print(video.video.uri)   # gs://your-bucket/output/xxx.mp4
```

### Parameter Guide

| Parameter | Values | Description |
|-----------|--------|-------------|
| `aspect_ratio` | `"16:9"` / `"9:16"` | Landscape / Portrait (mobile) |
| `number_of_videos` | 1–4 | Number of videos to generate |
| `duration_seconds` | 4, 6, 8 | Duration in seconds |

> **Rule:**
> - `output_gcs_uri` is required. Videos are stored in GCS, not locally.
> - Generation typically takes 2–5 minutes.

---

## § 2. Image to Video

### When: adding motion to a static image to create a video

```python
import time
from google import genai
from google.genai.types import GenerateVideosConfig, HttpOptions, Image

client = genai.Client(http_options=HttpOptions(api_version="v1"))
output_gcs_uri = "gs://your-bucket/output/"

operation = client.models.generate_videos(
    model="veo-3.1-generate-001",
    prompt="The flowers sway gently in the breeze, petals trembling",
    image=Image(
        gcs_uri="gs://your-bucket/input/flowers.png",   # GCS path required
        mime_type="image/png",                           # "image/png" or "image/jpeg"
    ),
    config=GenerateVideosConfig(
        aspect_ratio="16:9",
        output_gcs_uri=output_gcs_uri,
    ),
)

while not operation.done:
    time.sleep(15)
    operation = client.operations.get(operation)

for video in operation.result.generated_videos:
    print(video.video.uri)
```

> **Rule:**
> - Input image must be uploaded to GCS and referenced via `gcs_uri`.
> - Recommended resolution: 720p or higher.
> - The prompt should describe only the **motion** to add. Do not re-describe the image content.

---

## § 3. First / Last Frame Video

### When: automatically generating the transition between a start image and an end image

```python
import time
from google import genai
from google.genai.types import GenerateVideosConfig, HttpOptions, Image

client = genai.Client(http_options=HttpOptions(api_version="v1"))
output_gcs_uri = "gs://your-bucket/output/"

operation = client.models.generate_videos(
    model="veo-3.1-generate-001",
    prompt="A smooth natural transition, cinematic feel",
    image=Image(
        gcs_uri="gs://your-bucket/first-frame.png",   # first frame
        mime_type="image/png",
    ),
    config=GenerateVideosConfig(
        aspect_ratio="16:9",
        last_frame=Image(
            gcs_uri="gs://your-bucket/last-frame.png",   # last frame
            mime_type="image/png",
        ),
        output_gcs_uri=output_gcs_uri,
    ),
)

while not operation.done:
    time.sleep(15)
    operation = client.operations.get(operation)

for video in operation.result.generated_videos:
    print(video.video.uri)
```

> **Rule:**
> - `image` (first frame) is passed directly to `generate_videos()`.
> - `last_frame` (last frame) is passed inside `GenerateVideosConfig`.
> - Match the resolution and aspect ratio of both images.

---

## § 4. Video Extension

### When: naturally extending an existing video

```python
import time
from google import genai
from google.genai.types import GenerateVideosConfig, HttpOptions, Video

client = genai.Client(http_options=HttpOptions(api_version="v1"))
output_gcs_uri = "gs://your-bucket/output/"

operation = client.models.generate_videos(
    model="veo-3.1-generate-001",
    prompt="Continue the scene naturally, same lighting and style",
    video=Video(
        uri="gs://your-bucket/input/original.mp4",   # source video GCS path
        mime_type="video/mp4",
    ),
    config=GenerateVideosConfig(
        output_gcs_uri=output_gcs_uri,
    ),
)

while not operation.done:
    time.sleep(15)
    operation = client.operations.get(operation)

for video in operation.result.generated_videos:
    print(video.video.uri)
```

### Input Video Requirements

| Item | Condition |
|------|-----------|
| Format | MP4 only |
| Duration | 1–30 seconds |
| Frame rate | 24fps |
| Resolution | 720p, 1080p, 4K |
| Aspect ratio | 9:16 or 16:9 |

### Output Specs

- Format: MP4
- Extension length: 7 seconds
- Resolution / aspect ratio: matches input

> **Rule:**
> - The `Video` object must specify both `uri` (GCS) and `mime_type="video/mp4"`.
> - The prompt should explicitly state to maintain the original style.

---

## § 5. Advanced Techniques

### 5-1. Reference Image for Style / Character Consistency

Use when you need to maintain consistent appearance of a person, character, or product, or apply a specific style:

```python
import time
from google import genai
from google.genai.types import (
    GenerateVideosConfig,
    HttpOptions,
    Image,
    VideoGenerationReferenceImage,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))
output_gcs_uri = "gs://your-bucket/output/"

operation = client.models.generate_videos(
    model="veo-3.1-generate-001",
    prompt="A person walking through a sunny park, cheerful mood",
    config=GenerateVideosConfig(
        reference_images=[
            VideoGenerationReferenceImage(
                image=Image(
                    gcs_uri="gs://your-bucket/person.png",
                    mime_type="image/png",
                ),
                reference_type="asset",   # "asset": preserve person/character/product appearance
            ),
        ],
        aspect_ratio="16:9",
        output_gcs_uri=output_gcs_uri,
    ),
)

while not operation.done:
    time.sleep(15)
    operation = client.operations.get(operation)

for video in operation.result.generated_videos:
    print(video.video.uri)
```

| `reference_type` | Use Case | Supported Models |
|-----------------|----------|-----------------|
| `"asset"` | Preserve person/character/product appearance | veo-3.1 |
| `"style"` | Apply video art style | veo-2.0 only |

### 5-2. Insert Objects

> **Note:** `veo-2.0-generate-001` only

```python
import time
from google import genai
from google.genai.types import (
    GenerateVideosConfig,
    HttpOptions,
    Video,
    Image,
    VideoGenerationMask,
    VideoGenerationMaskMode,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))
output_gcs_uri = "gs://your-bucket/output/"

operation = client.models.generate_videos(
    model="veo-2.0-generate-001",
    prompt="a sheep standing in the field",
    video=Video(
        uri="gs://your-bucket/input.mp4",
        mime_type="video/mp4",
    ),
    config=GenerateVideosConfig(
        mask=VideoGenerationMask(
            image=Image(
                gcs_uri="gs://your-bucket/mask.png",   # mask marking insertion area
                mime_type="image/png",
            ),
            mask_mode=VideoGenerationMaskMode.INSERT,
        ),
        output_gcs_uri=output_gcs_uri,
    ),
)

while not operation.done:
    time.sleep(15)
    operation = client.operations.get(operation)

for video in operation.result.generated_videos:
    print(video.video.uri)
```

### 5-3. Remove Objects

> **Note:** `veo-2.0-generate-001` only

```python
import time
from google import genai
from google.genai.types import (
    GenerateVideosConfig,
    HttpOptions,
    Video,
    Image,
    VideoGenerationMask,
    VideoGenerationMaskMode,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))
output_gcs_uri = "gs://your-bucket/output/"

operation = client.models.generate_videos(
    model="veo-2.0-generate-001",
    prompt="clean background without the object",
    video=Video(
        uri="gs://your-bucket/input.mp4",
        mime_type="video/mp4",
    ),
    config=GenerateVideosConfig(
        mask=VideoGenerationMask(
            image=Image(
                gcs_uri="gs://your-bucket/mask.png",   # mask marking the object to remove
                mime_type="image/png",
            ),
            mask_mode=VideoGenerationMaskMode.REMOVE,
        ),
        output_gcs_uri=output_gcs_uri,
    ),
)

while not operation.done:
    time.sleep(15)
    operation = client.operations.get(operation)

for video in operation.result.generated_videos:
    print(video.video.uri)
```

> **Rule:**
> - Up to 3 reference images can be provided (asset type).
> - Object insert/remove is only supported on `veo-2.0-generate-001`.
> - Mask images must be PNG/JPEG/WebP; mark the target area in white.

---

## § 6. Prompt Guide & Best Practices

### 7 Components of a Video Prompt

| Component | Description | Example |
|-----------|-------------|---------|
| **Subject** | Main character or object | `"a seasoned detective"`, `"a golden retriever"` |
| **Action** | Movement or behavior | `"walking slowly"`, `"leaves rustling"` |
| **Scene** | Background or environment | `"foggy Victorian street at night"` |
| **Camera angle** | Camera perspective | `"low-angle shot"`, `"bird's-eye view"`, `"Dutch angle"` |
| **Camera movement** | Camera motion | `"slow pan left"`, `"dolly zoom"`, `"aerial drone shot"` |
| **Visual style** | Aesthetic style | `"photorealistic"`, `"anime style"`, `"cinematic lighting"` |
| **Temporal** | Time effects | `"slow motion"`, `"time-lapse"`, `"fast-paced"` |

### Prompt Writing Guidelines

**Be specific and clear:**
```
# Bad
"a man looking sad"

# Good
"Low-angle close-up shot of a man with a somber expression, rain falling in background, cinematic lighting"
```

**Dialogue expression:**
```
# Bad (using quotes)
'A woman saying "My name is Clara"'

# Good (using colon)
"A woman says: My name is Clara"
```

**Focus on one scene:**
- Do not include multiple scenes in one prompt
- Limit each video to a single scene

**For image-to-video:**
- Do not re-describe the source image content
- Only describe the **motion** to add

### Disabling the Prompt Rewriter

> **Note:** Only supported on `veo-2.0-generate-001`. Cannot be disabled on Veo 3/3.1.

```python
config=GenerateVideosConfig(
    enhance_prompt=False,    # default True (model auto-improves the prompt)
    output_gcs_uri=output_gcs_uri,
)
```

Disabling the rewriter may reduce output quality and prompt fidelity.

### Improving Prompts with Gemini

```python
from google import genai
from google.genai.types import GenerateContentConfig, HttpOptions

text_client = genai.Client(http_options=HttpOptions(api_version="v1"))

refined = text_client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Refine this video prompt for Veo: 'a cat in a garden'",
    config=GenerateContentConfig(
        system_instruction="You are a Veo video prompt expert. Make prompts cinematic, specific, and detailed. Keep under 200 words.",
    ),
)
print(refined.text)  # pass the refined prompt to Veo
```

---

## § 7. Responsible AI & Safety

### Prohibited Content

Prohibited under the Generative AI Prohibited Use Policy:
- Child safety threats
- Violent or extremist content
- Sexually explicit content
- Hate speech / toxic language
- Unauthorized realistic depictions of celebrities or real individuals

### Handling Safety Filter Errors

```python
import time
from google import genai
from google.genai.types import GenerateVideosConfig, HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))
output_gcs_uri = "gs://your-bucket/output/"

operation = client.models.generate_videos(
    model="veo-3.1-generate-001",
    prompt="Your video prompt here",
    config=GenerateVideosConfig(output_gcs_uri=output_gcs_uri),
)

while not operation.done:
    time.sleep(15)
    operation = client.operations.get(operation)

# Check result and handle safety errors
if operation.result and operation.result.generated_videos:
    for video in operation.result.generated_videos:
        print(video.video.uri)
elif operation.error:
    print(f"Error: {operation.error.message}")
    # Example safety-related error codes:
    # 58061214 / 17301594 → child safety
    # 90789179 / 43188360 → sexual content
    # 61493863 / 56562880 → violence
    # 57734940 / 22137204 → hate speech
    # 29310472 / 15236754 → celebrity depiction
```

> **Rule:**
> - If blocked by a safety filter, revise the prompt.
> - Some videos may be blocked while others are returned (partial results).
> - Realistic depictions of celebrities or real individuals require separate approval.
