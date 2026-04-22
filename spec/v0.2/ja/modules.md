# M2C モジュール仕様 v0.2

> ステータス: ドラフト
> 日付: 2026-04-01

## 1. 概要

M2C の各モジュールは独立した分析ユニットである。モジュールは任意の言語で実装でき、ローカルプロセス、MCP ツールサーバー、またはリモート API としてデプロイできる。

モジュールは（UniVA に着想を得た）2 つのカテゴリに分類される:

| カテゴリ | 説明 | 例 |
|----------|------|-----|
| **Atom** | 単機能、依存関係が最小限 | `scene`, `color`, `transcription` |
| **Workflow** | 複数の Atom を組み合わせる | `full_video_analysis` |

---

## 2. モジュールレジストリ

### 2.1 M2CModuleManifest

すべての M2C モジュールは Manifest を宣言しなければならない（MUST）:

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

### 2.2 モジュールディスカバリ

プロバイダはモジュール一覧の取得をサポートしなければならない（MUST）:

```
GET /m2c/modules → M2CModuleManifest[]
```

または Capability Negotiation を使用する（protocol.md §6 参照）。

### 2.3 カスタムモジュール

サードパーティモジュールは逆ドメイン命名を使用すべきである（SHOULD）:

```json
{ "id": "com.example.brand-detection" }
```

コアモジュール（本仕様で定義）はシンプルな名前を使用する: `scene`, `transcription` など。

---

## 3. コアモジュール

### 3.1 信頼度の重み

| モジュール | デフォルト重み | 根拠 |
|-----------|--------------|------|
| scene | 1.0 | 構造の基盤 |
| transcription | 1.5 | テキスト主体のコンテンツに不可欠 |
| visual | 1.0 | 標準的な視覚理解 |
| color | 0.5 | 補助的な美的情報 |
| acoustic | 0.8 | 音声コンテキストに重要 |
| pose | 0.5 | 補助的な人物分析 |
| summary | 2.0 | 他のすべてのモジュールを統合 |
| text_extract | 1.5 | ドキュメントに不可欠 |

---

### モジュール: `scene`

**カテゴリ**: Atom
**対応タイプ**: video
**依存関係**: なし
**リファレンス実装**: PySceneDetect, FFmpeg scene filter

**入力オプション**:
```json
{ "algorithm": "adaptive", "threshold": 27.0, "minSceneDuration": 1.0 }
```

**出力**（`data` フィールド）:
```json
{
  "scenes": [
    { "index": 0, "start": 0.0, "end": 12.5, "keyframeUri": "file:///tmp/m2c/kf_000.jpg", "keyframeTimestamp": 6.2 }
  ],
  "totalScenes": 8,
  "averageDuration": 15.3
}
```

**信頼度**: `1.0 - (ambiguous_cuts / total_cuts)`。曖昧（ambiguous）= スコアが閾値の ±10% 以内のカット。

---

### モジュール: `transcription`

**カテゴリ**: Atom
**対応タイプ**: video, audio, music
**依存関係**: なし
**リファレンス実装**: WhisperX, Faster Whisper, 任意の STT API

**入力オプション**:
```json
{ "language": "ja", "model": "large-v3", "diarize": true, "wordTimestamps": true }
```

**出力**:
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

**信頼度**: セグメント信頼度の時間加重平均。

---

### モジュール: `visual`

**カテゴリ**: Atom
**対応タイプ**: video（キーフレーム経由）, image, vector, presentation
**依存関係**: `scene`（video の場合）
**リファレンス実装**: BLIP-3, Open CLIP + YOLO, 任意の Vision LLM

**出力**:
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

**信頼度**: フレームごとの（キャプション信頼度 + オブジェクト信頼度の平均）の全体平均。

---

### モジュール: `color`

**カテゴリ**: Atom
**対応タイプ**: video（キーフレーム経由）, image, vector
**依存関係**: `scene`（video の場合）
**リファレンス実装**: MovieColorSchemer, Color Thief

**出力**:
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

**信頼度**: 入力品質を考慮すべきである（SHOULD）。`confidence = 1.0 - noise_penalty - compression_penalty`。クリーンな入力であれば 1.0 に近づく。

---

### モジュール: `acoustic`

**カテゴリ**: Atom
**対応タイプ**: video, audio, music
**依存関係**: なし
**リファレンス実装**: PANNs, 任意の音声分類モデル

**出力**:
```json
{
  "bgmSegments": [ { "start": 0.0, "end": 5.0, "mood": "upbeat", "confidence": 0.85 } ],
  "silenceSegments": [ { "start": 45.0, "end": 46.5 } ],
  "events": [ { "time": 30.0, "type": "applause", "confidence": 0.72 } ],
  "hasMusic": true,
  "musicRatio": 0.35
}
```

**信頼度**: イベント信頼度の平均。

---

### モジュール: `pose`

**カテゴリ**: Atom
**対応タイプ**: video（キーフレーム経由）, image
**依存関係**: `scene`（video の場合）
**リファレンス実装**: MediaPipe, 任意のポーズ推定モデル

**出力**:
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

**信頼度**: 人物ごとの表情信頼度とジェスチャー信頼度の平均。

---

### モジュール: `text_extract`

**カテゴリ**: Atom
**対応タイプ**: document, presentation, spreadsheet
**依存関係**: なし
**リファレンス実装**: Apache Tika, pdfplumber, python-pptx

**出力**:
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

**信頼度**: 抽出の完全性に基づく（暗号化・スキャンされたページはスコアを下げる）。

---

### モジュール: `summary`

**カテゴリ**: Workflow（すべてに依存）
**対応タイプ**: すべて
**依存関係**: `"*"`（実行されたすべてのモジュール）
**リファレンス実装**: 任意の LLM（OpenAI, Anthropic, ローカルモデルなど）

**入力**: `dependencyOutputs` を通じて、事前に実行されたすべてのモジュールの出力を受け取る。

**出力**:
```json
{
  "summary": "Cooking vlog featuring 3 recipes. Bright kitchen setting, single host. Upbeat BGM throughout.",
  "tags": ["cooking", "vlog", "recipe"],
  "moods": ["bright", "casual"],
  "targetAudience": "cooking enthusiasts",
  "contentType": "tutorial"
}
```

**依存関係**: `"*"` ワイルドカードは、このモジュールが他のすべてのモジュール出力を受け取ることを意味する。実装は利用可能なすべての `dependencyOutputs` を渡さなければならない（MUST）。

**信頼度**: すべての入力モジュール信頼度の加重平均。この値が `MediaContext.confidence` となる。
