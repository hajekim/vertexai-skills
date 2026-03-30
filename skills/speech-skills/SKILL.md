---
name: speech-skills
description: Use when working with speech synthesis or speech recognition on Vertex AI — requests involving "text to speech", "speech to text", "TTS", "STT", "voice synthesis", "audio transcription", "Chirp", "speech generation", "speech recognition". Korean: "텍스트 음성 변환", "음성 인식", "음성 합성", "TTS", "STT", "음성 전사", "오디오 변환".
version: 1.0.0
---

# Speech Skills — Vertex AI Studio Speech Reference

> Reference: [Text to speech](https://cloud.google.com/vertex-ai/generative-ai/docs/speech/text-to-speech) · [Speech to text](https://cloud.google.com/vertex-ai/generative-ai/docs/speech/speech-to-text)

---

## Important: UI-Only Features in Vertex AI Studio

> **Both Text-to-Speech and Speech-to-Text in Vertex AI Studio are currently UI-only features.**
> There is no SDK or REST API available through the Vertex AI Studio interface.
> For production use, use the dedicated Google Cloud Speech APIs listed below.

| Feature | Vertex AI Studio | Production Alternative |
|---------|-----------------|----------------------|
| Text to Speech | UI demo only | [Cloud Text-to-Speech API](https://cloud.google.com/text-to-speech) |
| Speech to Text | UI demo only | [Cloud Speech-to-Text API](https://cloud.google.com/speech-to-text) |

---

## § 1. Text to Speech (TTS)

### Vertex AI Studio UI

In Vertex AI Studio, TTS lets you:
- Type text and generate audio output
- Select voice type and language
- Play back and download the generated WAV file

### Production: Cloud Text-to-Speech API

```bash
pip install google-cloud-texttospeech
```

```python
from google.cloud import texttospeech

client = texttospeech.TextToSpeechClient()

synthesis_input = texttospeech.SynthesisInput(text="Hello, how can I help you today?")

voice = texttospeech.VoiceSelectionParams(
    language_code="en-US",
    ssml_gender=texttospeech.SsmlVoiceGender.NEUTRAL,
)

audio_config = texttospeech.AudioConfig(
    audio_encoding=texttospeech.AudioEncoding.MP3,
)

response = client.synthesize_speech(
    input=synthesis_input,
    voice=voice,
    audio_config=audio_config,
)

with open("output.mp3", "wb") as out:
    out.write(response.audio_content)
    print("Audio saved to output.mp3")
```

### TTS Voice Options

| Parameter | Options | Description |
|-----------|---------|-------------|
| `language_code` | `"en-US"`, `"ko-KR"`, `"ja-JP"`, etc. | BCP-47 language code |
| `ssml_gender` | `NEUTRAL`, `MALE`, `FEMALE` | Voice gender |
| `name` | e.g. `"en-US-Neural2-A"` | Specific voice model (optional) |
| `audio_encoding` | `MP3`, `LINEAR16`, `OGG_OPUS` | Output audio format |

> **Rule:**
> - Neural2 and Studio voices offer higher quality than Standard voices.
> - For Korean, use `"ko-KR"` language code with Neural2 voices for best results.

---

## § 2. Speech to Text (STT)

### Vertex AI Studio UI — Chirp Model

In Vertex AI Studio, STT uses the **Chirp** model:
- Upload an audio file (WAV format, 16-bit PCM)
- Maximum file size: 10MB
- Maximum duration: 60 seconds
- Results are displayed in the UI

### Input Requirements for Vertex AI Studio STT

| Requirement | Specification |
|-------------|--------------|
| Format | WAV (16-bit PCM) |
| Max file size | 10 MB |
| Max duration | 60 seconds |
| Channel | Mono or Stereo |

### Production: Cloud Speech-to-Text API

```bash
pip install google-cloud-speech
```

```python
from google.cloud import speech

client = speech.SpeechClient()

# Read local audio file
with open("audio.wav", "rb") as audio_file:
    content = audio_file.read()

audio = speech.RecognitionAudio(content=content)

config = speech.RecognitionConfig(
    encoding=speech.RecognitionConfig.AudioEncoding.LINEAR16,
    sample_rate_hertz=16000,
    language_code="en-US",
    model="chirp",   # use Chirp model for best accuracy
)

response = client.recognize(config=config, audio=audio)

for result in response.results:
    print(f"Transcript: {result.alternatives[0].transcript}")
    print(f"Confidence: {result.alternatives[0].confidence:.2f}")
```

### Long Audio (> 60 seconds): Async Recognition

```python
from google.cloud import speech

client = speech.SpeechClient()

# For long audio files, use GCS URI and async recognition
audio = speech.RecognitionAudio(uri="gs://your-bucket/long_audio.wav")

config = speech.RecognitionConfig(
    encoding=speech.RecognitionConfig.AudioEncoding.LINEAR16,
    sample_rate_hertz=16000,
    language_code="ko-KR",
    model="chirp",
    enable_automatic_punctuation=True,
)

operation = client.long_running_recognize(config=config, audio=audio)
print("Waiting for operation to complete...")

response = operation.result(timeout=600)

for result in response.results:
    print(f"Transcript: {result.alternatives[0].transcript}")
```

### STT Configuration Options

| Parameter | Description |
|-----------|-------------|
| `model` | `"chirp"` (recommended), `"default"`, `"phone_call"`, `"video"` |
| `language_code` | BCP-47 language code (e.g. `"en-US"`, `"ko-KR"`) |
| `enable_automatic_punctuation` | Auto-insert punctuation in transcript |
| `enable_word_time_offsets` | Include word-level timestamps |
| `max_alternatives` | Number of alternative transcripts to return |

> **Rule:**
> - Use `"chirp"` model for best accuracy across languages.
> - For audio longer than 60 seconds, use `long_running_recognize()` with a GCS URI.
> - Vertex AI Studio STT supports WAV 16-bit PCM only — convert other formats before uploading.
> - For production workloads, always use the Cloud Speech-to-Text API directly, not the Vertex AI Studio UI.
