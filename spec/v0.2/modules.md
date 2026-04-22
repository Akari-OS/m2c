# M2C Module Specifications v0.2

> Status: Draft
> Date: 2026-04-01

## 1. Overview

Each M2C module is an independent analysis unit. Modules can be implemented in any language, deployed as local processes, MCP tool servers, or remote APIs.

Modules are classified into two categories (inspired by UniVA):

| Category | Description | Example |
|----------|-------------|---------|
| **Atom** | Single-function, minimal dependency | `scene`, `color`, `transcription` |
| **Workflow** | Composes multiple atoms | `full_video_analysis` |

---

## 2. Module Registry

### 2.1 Module Manifest

Every M2C module MUST declare a manifest:

```json
{
  "$schema": "https://github.com/Akari-OS/m2c/schema/v0.2/module-manifest.json",
  "id": "scene",
  "name": "Scene Detection",
  "version": "1.0.0",
  "category": "atom",
  "description": "Detect scene boundaries and extract representative keyframes",
  "supportedTypes": ["video"],
  "dependencies": [],
  "confidenceWeight": 1.0,
  "capabilities": {
    "algorithms": ["adaptive", "content", "hash", "threshold"],
    "outputArtifacts": true,
    "streaming": false
  },
  "estimateCost": {
    "perMinute": { "cost": 0, "costUnit": "local", "timeMs": 2000 }
  },
  "author": "M2C Core",
  "license": "Apache-2.0",
  "homepage": "https://github.com/Akari-OS/m2c/modules/scene"
}
```

### 2.2 Module Discovery

Providers MUST support module listing:

```
GET /m2c/modules → M2CModuleManifest[]
```

Or via capability negotiation (see protocol.md §6).

### 2.3 Custom Modules

Third-party modules SHOULD use reverse-domain naming:

```json
{ "id": "com.example.brand-detection" }
```

Core modules (defined in this spec) use simple names: `scene`, `transcription`, etc.

---

## 3. Core Modules

### 3.1 Confidence Weights

| Module | Default Weight | Rationale |
|--------|---------------|-----------|
| scene | 1.0 | Structural foundation |
| transcription | 1.5 | Critical for text-heavy content |
| visual | 1.0 | Standard visual understanding |
| color | 0.5 | Supplementary aesthetic info |
| acoustic | 0.8 | Important for audio context |
| pose | 0.5 | Supplementary human analysis |
| summary | 2.0 | Integrates all other modules |
| text_extract | 1.5 | Critical for documents |

---

### Module: `scene`

**Category**: Atom
**Supported Types**: video
**Dependencies**: none
**Reference Implementations**: PySceneDetect, FFmpeg scene filter

**Input Options**:
```json
{ "algorithm": "adaptive", "threshold": 27.0, "minSceneDuration": 1.0 }
```

**Output** (`data` field):
```json
{
  "scenes": [
    { "index": 0, "start": 0.0, "end": 12.5, "keyframeUri": "file:///tmp/m2c/kf_000.jpg", "keyframeTimestamp": 6.2 }
  ],
  "totalScenes": 8,
  "averageDuration": 15.3
}
```

**Confidence**: `1.0 - (ambiguous_cuts / total_cuts)`. Ambiguous = score within ±10% of threshold.

---

### Module: `transcription`

**Category**: Atom
**Supported Types**: video, audio, music
**Dependencies**: none
**Reference Implementations**: WhisperX, Faster Whisper, any STT API

**Input Options**:
```json
{ "language": "ja", "model": "large-v3", "diarize": true, "wordTimestamps": true }
```

**Output**:
```json
{
  "segments": [
    { "start": 0.5, "end": 3.2, "text": "Hello, today we'll cover three recipes.", "speaker": "SPEAKER_01", "confidence": 0.93 }
  ],
  "speakers": [ { "id": "SPEAKER_01", "ratio": 0.85 } ],
  "language": "ja",
  "languageConfidence": 0.98
}
```

**Confidence**: Duration-weighted average of segment confidences.

---

### Module: `visual`

**Category**: Atom
**Supported Types**: video (via keyframes), image, vector, presentation
**Dependencies**: `scene` (for video)
**Reference Implementations**: BLIP-3, Open CLIP + YOLO, any Vision LLM

**Output**:
```json
{
  "frames": [
    {
      "uri": "file:///tmp/m2c/kf_000.jpg",
      "timestamp": 6.2,
      "caption": "A person cooking in a bright, modern kitchen",
      "objects": [ { "label": "person", "confidence": 0.97, "bbox": [0.1, 0.2, 0.5, 0.9] } ],
      "captionConfidence": 0.88
    }
  ],
  "objectFrequency": { "person": 12, "kitchen": 8, "food": 6 }
}
```

**Confidence**: Average of (caption confidence + mean object confidence) per frame.

---

### Module: `color`

**Category**: Atom
**Supported Types**: video (via keyframes), image, vector
**Dependencies**: `scene` (for video)
**Reference Implementations**: MovieColorSchemer, Color Thief

**Output**:
```json
{
  "palette": ["#F5E6D3", "#2C5F2D", "#E8A87C"],
  "dominantColor": "#F5E6D3",
  "brightness": 0.72,
  "saturation": 0.45,
  "warmth": 0.68,
  "mood": "warm, natural"
}
```

**Confidence**: SHOULD account for input quality. `confidence = 1.0 - noise_penalty - compression_penalty`. For clean inputs, approaches 1.0.

---

### Module: `acoustic`

**Category**: Atom
**Supported Types**: video, audio, music
**Dependencies**: none
**Reference Implementations**: PANNs, any audio classification model

**Output**:
```json
{
  "bgmSegments": [ { "start": 0.0, "end": 5.0, "mood": "upbeat", "confidence": 0.85 } ],
  "silenceSegments": [ { "start": 45.0, "end": 46.5 } ],
  "events": [ { "time": 30.0, "type": "applause", "confidence": 0.72 } ],
  "hasMusic": true,
  "musicRatio": 0.35
}
```

**Confidence**: Average event confidence.

---

### Module: `pose`

**Category**: Atom
**Supported Types**: video (via keyframes), image
**Dependencies**: `scene` (for video)
**Reference Implementations**: MediaPipe, any pose estimation model

**Output**:
```json
{
  "persons": [
    {
      "face": { "expression": "smiling", "expressionConfidence": 0.91 },
      "pose": { "gesture": "standing, gesturing with hands", "gestureConfidence": 0.84 }
    }
  ],
  "personCount": 1
}
```

**Confidence**: Average of expression and gesture confidences per person.

---

### Module: `text_extract`

**Category**: Atom
**Supported Types**: document, presentation, spreadsheet
**Dependencies**: none
**Reference Implementations**: Apache Tika, pdfplumber, python-pptx

**Output**:
```json
{
  "text": "Full extracted text...",
  "pages": [
    { "index": 0, "text": "Page 1 content...", "tables": [], "images": [] }
  ],
  "totalPages": 12,
  "wordCount": 3500,
  "language": "en"
}
```

**Confidence**: Based on extraction completeness (encrypted/scanned pages reduce score).

---

### Module: `summary`

**Category**: Workflow (depends on all)
**Supported Types**: all
**Dependencies**: `"*"` (all executed modules)
**Reference Implementations**: Any LLM (OpenAI, Anthropic, local models, etc.)

**Input**: All outputs from previously executed modules via `dependencyOutputs`.

**Output**:
```json
{
  "summary": "Cooking vlog featuring 3 recipes. Bright kitchen setting, single host. Upbeat BGM throughout.",
  "tags": ["cooking", "vlog", "recipe"],
  "moods": ["bright", "casual"],
  "targetAudience": "cooking enthusiasts",
  "contentType": "tutorial"
}
```

**Dependencies**: The `"*"` wildcard means this module receives ALL other module outputs. Implementations MUST pass all available `dependencyOutputs`.

**Confidence**: Weighted average of all input module confidences. This value becomes `MediaContext.confidence`.
