# M2C モジュール仕様 v0.1

> ステータス: Draft
> 日付: 2026-04-01

## 概要

各M2Cモジュールは独立した分析ユニット。任意の言語で実装でき、ローカルプロセスまたはMCPツールサーバーとしてデプロイ可能。

---

## モジュール: `scene`

**目的**: シーン境界の検出と代表キーフレームの抽出

**対応メディア**: 動画

**リファレンス実装**: PySceneDetect

### 入力

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

### 出力

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

### 信頼度の計算

- 信頼度 = 1.0 - (曖昧なカット数 / 全カット数)
- 曖昧なカット = スコアが閾値の±10%以内

---

## モジュール: `transcription`

**目的**: 音声→テキスト変換 + 話者分離（ダイアライゼーション）

**対応メディア**: 動画、音声

**リファレンス実装**: WhisperX + pyannote

### 入力

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

### 出力

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

### 信頼度の計算

- セグメント信頼度 = 単語信頼度の平均
- モジュール信頼度 = セグメント信頼度の加重平均（持続時間で加重）

---

## モジュール: `visual`

**目的**: キーフレームの視覚的理解（物体検出、キャプション、構図）

**対応メディア**: 動画（キーフレーム経由）、画像

**リファレンス実装**: BLIP-3 / Open CLIP + YOLO

### 入力

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

動画の場合: `scene` モジュールのキーフレームパスが入力になる（依存関係）。

### 出力

```json
{
  "frames": [
    {
      "path": "/tmp/m2c/keyframe_000.jpg",
      "timestamp": 6.2,
      "caption": "明るいモダンキッチンで料理をする人。ステンレス家電が見える",
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

### 依存関係

- `scene`（動画の場合 — キーフレームパスが必要）

### 信頼度の計算

- フレーム信頼度 = キャプション信頼度と物体検出信頼度の平均
- モジュール信頼度 = フレーム信頼度の平均

---

## モジュール: `color`

**目的**: カラーパレット、支配色、ムード（雰囲気）の抽出

**対応メディア**: 動画（キーフレーム経由）、画像

**リファレンス実装**: MovieColorSchemer / Color Thief

### 出力

```json
{
  "palette": ["#F5E6D3", "#2C5F2D", "#E8A87C", "#97C1A9", "#4A4A4A"],
  "dominantColor": "#F5E6D3",
  "brightness": 0.72,
  "saturation": 0.45,
  "warmth": 0.68,
  "mood": "温かみ、ナチュラル、オーガニック"
}
```

### 信頼度の計算

- 常に 1.0（決定的アルゴリズムのため）

---

## モジュール: `acoustic`

**目的**: 音楽、無音、音響イベントの検出

**対応メディア**: 動画、音声

**リファレンス実装**: PANNs

### 出力

```json
{
  "bgmSegments": [
    { "start": 0.0, "end": 5.0, "mood": "アップテンポ", "confidence": 0.85 }
  ],
  "silenceSegments": [
    { "start": 45.0, "end": 46.5 }
  ],
  "events": [
    { "time": 30.0, "type": "拍手", "confidence": 0.72 },
    { "time": 60.0, "type": "笑い声", "confidence": 0.65 }
  ],
  "hasMusic": true,
  "musicRatio": 0.35
}
```

### 信頼度の計算

- モジュール信頼度 = イベント信頼度の平均

---

## モジュール: `pose`

**目的**: 人体ランドマーク、ジェスチャー、表情の検出

**対応メディア**: 動画（キーフレーム経由）、画像

**リファレンス実装**: MediaPipe

### 出力

```json
{
  "persons": [
    {
      "face": {
        "landmarks": [],
        "expression": "笑顔",
        "expressionConfidence": 0.91
      },
      "pose": {
        "landmarks": [],
        "gesture": "立ち、手でジェスチャー中",
        "gestureConfidence": 0.84
      }
    }
  ],
  "personCount": 1
}
```

### 依存関係

- `scene`（動画の場合 — キーフレームパスが必要）

### 信頼度の計算

- 人物信頼度 = 表情信頼度とジェスチャー信頼度の平均
- モジュール信頼度 = 人物信頼度の平均

---

## モジュール: `summary`

**目的**: 全モジュール出力を統合した自然言語サマリーの生成

**対応メディア**: 動画、画像、音声

**リファレンス実装**: LLM（OpenRouter / ローカル）

### 入力

先行実行した全モジュールの出力が `dependencyOutputs` として渡される。

### 出力

```json
{
  "summary": "料理Vlog。3つのレシピ（パスタ、サラダ、デザート）を紹介。明るいキッチンで撮影。ホスト1名が解説。BGMあり（アップテンポ）。全体的に明るくカジュアルな雰囲気。",
  "tags": ["料理", "Vlog", "レシピ", "パスタ", "サラダ", "キッチン"],
  "mood": "明るい、カジュアル、フレンドリー",
  "targetAudience": "料理好き、家庭料理ファン",
  "contentType": "チュートリアル、Vlog"
}
```

### 依存関係

- 実行された**全**モジュール

### 信頼度の計算

- サマリー信頼度 = 全入力モジュール信頼度の加重平均
- この値が `MediaContext.confidence` になる
