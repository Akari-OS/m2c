# M2C — Media to Context プロトコル

> あらゆるメディアを、AIが理解できる構造化コンテキストに変換する。

## M2C とは？

M2C（Media to Context）は、AIがメディアを分析・理解する方法を標準化するオープンプロトコル。動画だけでなく、**あらゆるメディア**に対応: ドキュメント、プレゼンテーション、音声、音楽、3Dモデル、ベクターグラフィックスなど。

4つの領域を定義:

1. **分析** — 何をどう分析するか（モジュラーパイプライン）
2. **スキーマ** — 結果をどう構造化するか（MediaContext JSON）
3. **供給** — コンテキストをAIにどう渡すか（JIT、トークンバジェット、隔離）
4. **相互運用** — 既存標準とどう連携するか（IPTC、C2PA、XMP）

```
あらゆるメディア → [M2C パイプライン] → 構造化コンテキスト (JSON) → AIが理解
```

## なぜ必要？

現在のAIツールはメディアを**盲目的**に扱っている。ファイルパスは受け取るが、中身を*理解*していない。

- **M2C以前**: 「動画ファイルです。編集して」→ AIは推測するしかない
- **M2C以後**: 「12シーン、2名の話者、暖色パレット、料理コンテンツ」→ AIが理解して提案できる

### M2C の位置づけ

```
IPTC 2025.1  → AI生成コンテンツのラベル     （生成メタデータ）
C2PA         → コンテンツの来歴証明          （真正性）
M2C          → AI分析結果の構造化            （理解） ← NEW
```

**AI分析結果**をカバーする既存標準は存在しない。M2C がその隙間を埋める。

## コアコンセプト

### 対応メディアタイプ

| タイプ | 例 |
|--------|-----|
| 動画 | .mp4, .mov, .mkv |
| 画像 | .jpg, .png, .webp |
| 音声 | .mp3, .wav, .flac |
| 音楽 | .mp3, .midi |
| ドキュメント | .pdf, .docx, .md |
| プレゼンテーション | .pptx, .key |
| スプレッドシート | .xlsx, .csv |
| ベクター | .svg, .ai |
| 3D | .gltf, .obj |
| カスタム | `x-` プレフィックス |

### 分析モジュール

各分析機能は独立した交換可能なモジュール:

| モジュール | 機能 | OSS例 |
|-----------|------|-------|
| `scene` | シーン境界検出 | PySceneDetect |
| `transcription` | 音声→テキスト + 話者分離 | WhisperX |
| `visual` | 画像理解 | BLIP / CLIP |
| `color` | カラーパレット + ムード | MovieColorSchemer |
| `acoustic` | 音楽/音響イベント | PANNs |
| `text_extract` | ドキュメントテキスト/表 | Apache Tika |
| `summary` | 統合サマリー | 任意のLLM |

### コンテキスト供給（4つの柱）

| 柱 | 機能 |
|----|------|
| **Write** | 分析 → 構造化JSON |
| **Select** | タスクに合ったコンテキストを選択（JIT） |
| **Compress** | トークンバジェットに収まるよう要約 |
| **Isolate** | モダリティ/時間/対象ごとに分離 |

### 信頼スコア

すべての結果に信頼スコア（0.0〜1.0）。品質ゲートで低品質コンテキストがAIに到達するのを防止。

## アーキテクチャ

```
┌──────────────────────────────────────────────┐
│              M2C パイプライン                   │
│                                              │
│  メディア URI                                  │
│      ↓                                       │
│  Capability Negotiation                      │
│      ↓                                       │
│  ディスパッチャ（タイプ+精度 → モジュール選定）  │
│      ↓                                       │
│  モジュール1 → 結果1 ─┐                       │
│  モジュール2 → 結果2 ─┤→ 集約 → コンテキスト   │
│  モジュールN → 結果N ─┘                       │
│      ↓                                       │
│  品質ゲート（信頼度チェック）                    │
│      ↓                                       │
│  MediaContext JSON                            │
│      ↓                                       │
│  コンテキスト供給（Select→Compress→Isolate）   │
│      ↓                                       │
│  AI コンシューマ                               │
└──────────────────────────────────────────────┘
```

## 仕様書

```
spec/
├── v0.2/                    ← 最新
│   ├── protocol.md          ← コアプロトコル仕様
│   ├── schema.json          ← MediaContext JSON Schema
│   ├── modules.md           ← モジュールインターフェース + Registry
│   ├── supply.md            ← コンテキスト供給プロトコル（4つの柱）
│   ├── interop.md           ← IPTC/C2PA/XMP/Schema.org マッピング
│   └── ja/                  ← 日本語版
└── v0.1/                    ← 旧バージョン
```

## 設計原則

| 原則 | 説明 |
|------|------|
| **コンテキストファースト** | コンテキストなしにAI操作は行わない |
| **モジュラー** | 分析モジュールは交換可能 |
| **ローカルファースト** | 完全ローカル実行をサポート |
| **セキュアバイデフォルト** | メディアデータには認証必須 |
| **標準互換** | IPTC、C2PA、XMP と連携 |
| **コスト意識** | 事前見積もり、ユーザーが深さを選択 |

## サンプル

[`examples/`](examples/) ディレクトリにMediaContext出力のサンプルがあります:

- [`video-cooking.mediacontext.json`](examples/video-cooking.mediacontext.json) — 料理チュートリアル動画をStandard精度で分析した結果

## SDK

> Coming Soon

| 言語 | 状態 |
|------|------|
| TypeScript | 予定 |
| Python | 予定 |

## 既知の実装

| プロジェクト | 状態 | 説明 |
|------------|------|------|
| *あなたのプロジェクト* | — | [実装を登録する](https://github.com/) |

## ライセンス

[Apache 2.0](LICENSE)

## リンク

- [プロトコル仕様 (v0.2)](spec/v0.2/protocol.md)
- [コンテキスト供給仕様](spec/v0.2/supply.md)
- [相互運用仕様](spec/v0.2/interop.md)
- [日本語版プロトコル仕様](spec/v0.2/ja/protocol.md)

### 参考文献

- [TwelveLabs: 動画のコンテキストエンジニアリング](https://www.twelvelabs.io/blog/context-engineering-for-video-understanding)
- [Anthropic: AIエージェントのコンテキストエンジニアリング](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [MCP (Model Context Protocol)](https://modelcontextprotocol.io)
- [UniVA: Universal Video Agent](https://github.com/univa-agent/univa)
