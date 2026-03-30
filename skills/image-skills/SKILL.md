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
