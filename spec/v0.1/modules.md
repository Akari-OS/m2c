# M2C Module Specifications v0.1

> Status: Draft
> Date: 2026-04-01

## Overview

Each M2C module is an independent analysis unit. Modules can be implemented in any language, deployed as local processes or MCP tool servers.

---

## Module: `scene`

**Purpose**: Detect scene boundaries and extract representative keyframes.

**Supported Types**: video

**Reference Implementation**: PySceneDetect

### Input

```json
{
  "filePath": "/path/to/video.mp4",
  "options": {
    "algorithm": "adaptive",
    "threshold": 27.0,
    "minSceneDuration": 1.0
  }
}
```

### Output

```json
{
  "scenes": [
    {
      "index": 0,
      "start": 0.0,
      "end": 12.5,
      "keyframePath": "/tmp/m2c/keyframe_000.jpg",
      "keyframeTimestamp": 6.2
    }
  ],
  "totalScenes": 8,
  "averageDuration": 15.3
}
```

### Confidence Calculation

- Confidence = 1.0 - (ambiguous_cuts / total_cuts)
- Ambiguous cut = score within 10% of threshold

---

## Module: `transcription`

**Purpose**: Convert speech to timestamped text with speaker diarization.

**Supported Types**: video, audio

**Reference Implementation**: WhisperX + pyannote

### Input

```json
{
  "filePath": "/path/to/media.mp4",
  "options": {
    "language": "ja",
    "model": "large-v3",
    "diarize": true,
    "wordTimestamps": true
  }
}
```

### Output

```json
{
  "segments": [
    {
      "start": 0.5,
      "end": 3.2,
      "text": "こんにちは、今日は3つのレシピを紹介します。",
      "speaker": "SPEAKER_01",
      "words": [
        { "start": 0.5, "end": 0.9, "word": "こんにちは", "confidence": 0.95 }
      ],
      "confidence": 0.93
    }
  ],
  "speakers": [
    { "id": "SPEAKER_01", "ratio": 0.85 },
    { "id": "SPEAKER_02", "ratio": 0.15 }
  ],
  "language": "ja",
  "languageConfidence": 0.98
}
```

### Confidence Calculation

- Segment confidence = average word confidence
- Module confidence = average segment confidence, weighted by duration

---

## Module: `visual`

**Purpose**: Understand visual content of keyframes (objects, captions, composition).

**Supported Types**: video (via keyframes), image

**Reference Implementation**: BLIP-3 / Open CLIP + YOLO

### Input

```json
{
  "filePath": "/path/to/image.jpg",
  "options": {
    "captionModel": "blip3",
    "objectDetection": true,
    "objectModel": "yolov8"
  }
}
```

For video: input is the keyframe paths from the `scene` module (dependency).

### Output

```json
{
  "frames": [
    {
      "path": "/tmp/m2c/keyframe_000.jpg",
      "timestamp": 6.2,
      "caption": "A person cooking in a bright, modern kitchen with stainless steel appliances",
      "objects": [
        { "label": "person", "confidence": 0.97, "bbox": [0.1, 0.2, 0.5, 0.9] },
        { "label": "oven", "confidence": 0.89, "bbox": [0.6, 0.3, 0.9, 0.8] }
      ],
      "captionConfidence": 0.88
    }
  ],
  "objectFrequency": {
    "person": 12,
    "kitchen": 8,
    "food": 6
  }
}
```

### Dependencies

- `scene` (for video — needs keyframe paths)

### Confidence Calculation

- Frame confidence = average of caption confidence and mean object confidence
- Module confidence = average frame confidence

---

## Module: `color`

**Purpose**: Extract color palette, dominant colors, mood from visuals.

**Supported Types**: video (via keyframes), image

**Reference Implementation**: MovieColorSchemer / Color Thief

### Input

```json
{
  "filePath": "/path/to/image.jpg",
  "options": {
    "paletteSize": 5,
    "colorSpace": "oklch"
  }
}
```

### Output

```json
{
  "palette": ["#F5E6D3", "#2C5F2D", "#E8A87C", "#97C1A9", "#4A4A4A"],
  "dominantColor": "#F5E6D3",
  "brightness": 0.72,
  "saturation": 0.45,
  "warmth": 0.68,
  "mood": "warm, natural, organic"
}
```

### Confidence Calculation

- Always 1.0 (deterministic algorithm)

---

## Module: `acoustic`

**Purpose**: Detect music, silence, and audio events.

**Supported Types**: video, audio

**Reference Implementation**: PANNs

### Input

```json
{
  "filePath": "/path/to/audio.mp3",
  "options": {
    "eventThreshold": 0.5
  }
}
```

### Output

```json
{
  "bgmSegments": [
    { "start": 0.0, "end": 5.0, "mood": "upbeat", "confidence": 0.85 },
    { "start": 120.0, "end": 180.0, "mood": "calm", "confidence": 0.78 }
  ],
  "silenceSegments": [
    { "start": 45.0, "end": 46.5 }
  ],
  "events": [
    { "time": 30.0, "type": "applause", "confidence": 0.72 },
    { "time": 60.0, "type": "laughter", "confidence": 0.65 }
  ],
  "hasMusic": true,
  "musicRatio": 0.35
}
```

### Confidence Calculation

- Module confidence = average event confidence

---

## Module: `pose`

**Purpose**: Detect human body landmarks, gestures, and facial expressions.

**Supported Types**: video (via keyframes), image

**Reference Implementation**: MediaPipe

### Input

```json
{
  "filePath": "/path/to/image.jpg",
  "options": {
    "detectFace": true,
    "detectPose": true,
    "detectHands": true
  }
}
```

### Output

```json
{
  "persons": [
    {
      "face": {
        "landmarks": [...],
        "expression": "smiling",
        "expressionConfidence": 0.91
      },
      "pose": {
        "landmarks": [...],
        "gesture": "standing, gesturing with hands",
        "gestureConfidence": 0.84
      }
    }
  ],
  "personCount": 1
}
```

### Dependencies

- `scene` (for video — needs keyframe paths)

### Confidence Calculation

- Per-person confidence = average of expression and gesture confidence
- Module confidence = average person confidence

---

## Module: `summary`

**Purpose**: Aggregate all module outputs into a unified natural-language summary.

**Supported Types**: video, image, audio

**Reference Implementation**: LLM (OpenRouter / local)

### Input

All outputs from previously executed modules are passed as `dependencyOutputs`.

### Output

```json
{
  "summary": "料理Vlog。3つのレシピ（パスタ、サラダ、デザート）を紹介。明るいキッチンで撮影。ホスト1名が解説。BGMあり（アップテンポ）。全体的に明るくカジュアルな雰囲気。",
  "tags": ["料理", "Vlog", "レシピ", "パスタ", "サラダ", "キッチン"],
  "mood": "bright, casual, friendly",
  "targetAudience": "cooking enthusiasts, home cooks",
  "contentType": "tutorial, vlog"
}
```

### Dependencies

- ALL other executed modules

### Confidence Calculation

- Summary confidence = weighted average of all input module confidences
- This becomes the `MediaContext.confidence` value
