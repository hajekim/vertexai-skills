---
name: video-skills
description: Use when working with video generation on Vertex AI using google-genai Python SDK — imports Veo models, or requests involving "video generation", "generate video", "text to video", "image to video", "extend video", "Veo". 한국어: "비디오 생성", "영상 만들기", "텍스트로 영상", "이미지로 영상", "비오", "영상 연장".
version: 1.0.0
---

# Video Skills — google-genai Python SDK 레퍼런스

## 환경 설정

### 라이브러리 설치

```bash
pip install --upgrade google-genai
```

### 필수 환경변수 및 GCS 버킷

```bash
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"
export GOOGLE_GENAI_USE_VERTEXAI="True"
# 생성된 비디오는 GCS 버킷에 저장됨 — 필수
export OUTPUT_GCS_URI="gs://your-bucket/output/"
```

### 클라이언트 초기화

```python
from google import genai

client = genai.Client()
output_gcs_uri = "gs://your-bucket/output/"  # 모든 생성 작업에 필요
```

### 모델 목록

| 모델 | 특징 |
|------|------|
| `veo-3.1-generate-001` | 최신 안정 버전 (권장) |
| `veo-3.1-fast-generate-001` | 빠른 생성 |
| `veo-3.0-generate-001` | 이전 안정 버전 |
| `veo-2.0-generate-001` | 레거시 (객체 삽입/제거 전용) |

### 공통 패턴: 폴링 루프

모든 비디오 생성은 비동기 LRO(Long Running Operation)로 동작한다:

```python
import time

# 작업 시작 후 완료까지 대기
while not operation.done:
    time.sleep(15)
    operation = client.operations.get(operation)

for video in operation.result.generated_videos:
    print(video.video.uri)  # GCS URI
```

> **원칙:** 비디오 생성은 항상 비동기다. `operation.done`이 True가 될 때까지 15초 간격으로 폴링한다.

---

## § 1. 텍스트 → 비디오

### 언제: 텍스트 프롬프트만으로 비디오를 생성할 때

```python
import time
from google import genai
from google.genai.types import GenerateVideosConfig

client = genai.Client()
output_gcs_uri = "gs://your-bucket/output/"

operation = client.models.generate_videos(
    model="veo-3.1-generate-001",
    prompt="A cat reading a book in a cozy library, warm lighting, cinematic shot",
    config=GenerateVideosConfig(
        aspect_ratio="16:9",       # "16:9" (가로) 또는 "9:16" (세로/모바일)
        number_of_videos=1,        # 생성할 비디오 수 (1-4)
        duration_seconds=8,        # 비디오 길이 (4, 6, 8 중 선택)
        output_gcs_uri=output_gcs_uri,
    ),
)

# 완료까지 대기
while not operation.done:
    time.sleep(15)
    operation = client.operations.get(operation)

for video in operation.result.generated_videos:
    print(video.video.uri)   # gs://your-bucket/output/xxx.mp4
```

### 파라미터 가이드

| 파라미터 | 값 | 설명 |
|---------|-----|------|
| `aspect_ratio` | `"16:9"` / `"9:16"` | 가로형 / 세로형(모바일) |
| `number_of_videos` | 1-4 | 생성할 비디오 수 |
| `duration_seconds` | 4, 6, 8 | 길이 (초) |

> **원칙:**
> - `output_gcs_uri`는 필수다. 비디오는 로컬이 아닌 GCS에 저장된다.
> - 생성 시간은 보통 2-5분 소요된다.

---

## § 2. 이미지 → 비디오

### 언제: 정적 이미지에 움직임을 추가하여 비디오로 만들 때

```python
import time
from google import genai
from google.genai.types import GenerateVideosConfig, Image

client = genai.Client()
output_gcs_uri = "gs://your-bucket/output/"

operation = client.models.generate_videos(
    model="veo-3.1-generate-001",
    prompt="The flowers sway gently in the breeze, petals trembling",
    image=Image(
        gcs_uri="gs://your-bucket/input/flowers.png",   # GCS 경로 필수
        mime_type="image/png",                           # "image/png" 또는 "image/jpeg"
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

> **원칙:**
> - 입력 이미지는 반드시 GCS에 업로드 후 `gcs_uri`로 참조한다.
> - 권장 해상도: 720p 이상.
> - 프롬프트는 이미지에 추가할 **동작(motion)**만 설명한다. 이미지 내용을 재설명하지 않는다.

---

## § 3. 첫/마지막 프레임으로 비디오

### 언제: 시작 이미지와 끝 이미지 사이의 전환을 자동 생성할 때

```python
import time
from google import genai
from google.genai.types import GenerateVideosConfig, Image

client = genai.Client()
output_gcs_uri = "gs://your-bucket/output/"

operation = client.models.generate_videos(
    model="veo-3.1-generate-001",
    prompt="A smooth natural transition, cinematic feel",
    image=Image(
        gcs_uri="gs://your-bucket/first-frame.png",   # 시작 프레임
        mime_type="image/png",
    ),
    config=GenerateVideosConfig(
        aspect_ratio="16:9",
        last_frame=Image(
            gcs_uri="gs://your-bucket/last-frame.png",   # 마지막 프레임
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

> **원칙:**
> - `image` (첫 프레임)은 `generate_videos()` 직접 파라미터로 전달한다.
> - `last_frame` (마지막 프레임)은 `GenerateVideosConfig` 안에 전달한다.
> - 두 이미지의 해상도와 비율을 일치시킨다.
