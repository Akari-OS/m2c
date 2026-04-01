# M2C プロトコル仕様 v0.1

> ステータス: Draft
> 日付: 2026-04-01

## 1. 概要

M2C（Media to Context）は、生のメディアファイル（動画・画像・音声）をAIが理解できる構造化コンテキストに変換するオープンプロトコル。

### 1.1 設計原則

| 原則 | 説明 |
|------|------|
| **コンテキストファースト** | コンテキストなしにAI操作は行わない。メディアは使用前に分析必須 |
| **モジュラー** | 各分析機能は独立した交換可能なモジュール |
| **階層的** | コンテキストは3層構造: フレーム → シーン → メディア全体 |
| **スコア付き** | すべての結果に信頼スコア。品質ゲートで低品質を排除 |
| **コスト意識** | 分析コストを事前に見積もり。ユーザーが精度レベルを選択 |

### 1.2 なぜ M2C が必要か

現在のAIツールはメディアを「盲目的」に扱っている。ファイルパスは受け取るが、中身を*理解*していない。

```
従来: 「動画ファイルです。編集して」→ AIは推測するしかない
M2C: 「12シーン、2名の話者、暖色パレット、料理コンテンツ」→ AIが理解して提案できる
```

### 1.3 用語定義

| 用語 | 定義 |
|------|------|
| **メディア** | 動画、画像、または音声ファイル |
| **モジュール** | 独立した分析ユニット（例: シーン検出、文字起こし） |
| **コンテキスト** | メディアの中身を記述した構造化JSON |
| **ディスパッチャ** | メディアタイプに応じてモジュールを選定するオーケストレーター |
| **精度レベル** | 分析の深さ（Quick / Standard / Deep） |
| **品質ゲート** | 信頼スコアの閾値チェック |

---

## 2. 分析パイプライン

### 2.1 フロー

```
1. 入力:    メディアファイル + 精度レベル
2. 選定:    メディアタイプ → モジュール群の選定
3. 実行:    選定されたモジュールを実行（可能な限り並列）
4. 集約:    各モジュール出力を MediaContext に統合
5. スコア:  総合信頼スコアを計算
6. ゲート:  品質閾値をチェック
7. 出力:    MediaContext JSON
```

### 2.2 ディスパッチルール

メディアタイプと精度レベルに応じてモジュールを自動選定する。

#### メディアタイプ別の必須モジュール

| メディア | Quick | Standard | Deep |
|---------|-------|----------|------|
| **動画** | scene, summary | + transcription, color | + visual, acoustic, pose |
| **画像** | visual, summary | + color | + pose |
| **音声** | transcription, summary | + acoustic | + speaker_diarization |

#### 実行順序

依存関係がなければ並列実行。`summary` は全モジュール完了後に実行。

```
scene ──────────────┐
transcription ──────┤
visual ─────────────┤→ summary（全結果を集約）
color ──────────────┤
acoustic ───────────┘
```

### 2.3 精度レベル

| レベル | 用途 | 推定コスト | 推定時間 |
|--------|------|----------|---------|
| **Quick** | ブラウジング、プレビュー、基本分類 | 低（0-1 AC/分） | 10秒以内 |
| **Standard** | 編集、検索、AI提案 | 中（2-4 AC/分） | 30秒〜2分 |
| **Deep** | AI駆動編集、スタイル分析、完全理解 | 高（5-10 AC/分） | 1〜5分 |

AC = Analysis Credit（実装定義の課金単位）

---

## 3. モジュールインターフェース

### 3.1 共通インターフェース

すべてのM2Cモジュールは以下のインターフェースを実装する:

```typescript
interface M2CModule {
  /** モジュール固有ID */
  id: string;
  /** 表示名 */
  name: string;
  /** セマンティックバージョン */
  version: string;
  /** 対応メディアタイプ */
  supportedTypes: ('video' | 'image' | 'audio')[];
  /** 依存モジュール（先に実行が必要なモジュールID） */
  dependencies: string[];

  /** メディアを分析 */
  analyze(input: M2CModuleInput): Promise<M2CModuleOutput>;

  /** 実行前のコスト見積もり */
  estimate(input: M2CModuleInput): Promise<M2CEstimate>;
}

interface M2CModuleInput {
  /** メディアファイルの絶対パス */
  filePath: string;
  /** メディアタイプ */
  mediaType: 'video' | 'image' | 'audio';
  /** 長さ（秒）。画像は0 */
  duration: number;
  /** モジュール固有のオプション */
  options?: Record<string, unknown>;
  /** 依存モジュールの出力（あれば） */
  dependencyOutputs?: Record<string, M2CModuleOutput>;
}

interface M2CModuleOutput {
  /** モジュールID */
  moduleId: string;
  /** モジュールバージョン */
  moduleVersion: string;
  /** 実行時間（ミリ秒） */
  durationMs: number;
  /** このモジュールの信頼スコア（0.0〜1.0） */
  confidence: number;
  /** モジュール固有の構造化データ */
  data: unknown;
}

interface M2CEstimate {
  /** 推定コスト（AC単位） */
  cost: number;
  /** 推定時間（ミリ秒） */
  timeMs: number;
}
```

### 3.2 MCP 互換性

M2CモジュールはMCP（Model Context Protocol）ツールサーバーとして公開できる。これにより、MCP対応の任意のAIシステムからM2C分析モジュールを呼び出せる。

---

## 4. コンテキストスキーマ（MediaContext）

### 4.1 4つの柱

TwelveLabs の4柱モデルを M2C に適用:

| 柱 | M2C での実現 |
|----|------------|
| **Write（構造化）** | モジュールが構造化JSONをMediaContextに書き込む |
| **Select（フィルタ）** | 消費者がタスクに応じたレイヤー（L0-L3）を選択 |
| **Compress（圧縮）** | summaryモジュールがコンパクトな表現を生成 |
| **Isolate（分離）** | 各モダリティ（映像/音声/テキスト）を独立管理 |

### 4.2 3層構造

```
L0: タグ       — tags, mood（検索・フィルタ用）
L1: サマリー    — summary, tags, colorPalette, speakers, duration
L2: シーン      — L1 + scenes[].summary, scenes[].objects
L3: フル       — 全フィールド（transcription セグメント含む）
```

コンテキストの消費者（パートナーAI等）はタスクに応じて適切なレイヤーだけを取得する（JIT供給）。

| タスク | 推奨レイヤー | 理由 |
|--------|------------|------|
| 検索・ブラウジング | L0 | 最小トークン |
| フロー提案 | L1 | 提案に十分 |
| シーン編集 | L2 | シーンレベルの詳細が必要 |
| 字幕作業 | L3 | 全文書き起こしが必要 |

### 4.3 スキーマ定義

完全な JSON Schema は [schema.json](../schema.json) を参照。

コア構造:

```typescript
interface MediaContext {
  /** スキーマバージョン */
  schemaVersion: number;
  /** メディアタイプ */
  mediaType: 'video' | 'image' | 'audio';

  // ── L0: タグ ──
  tags: string[];
  mood: string;

  // ── L1: サマリー ──
  summary: string;
  colorPalette: string[];
  speakers: SpeakerInfo[];
  duration: number;

  // ── L2: シーン（動画のみ） ──
  scenes: SceneContext[];

  // ── L3: フル（モダリティ別） ──
  transcription: TranscriptionSegment[];
  acoustic: AcousticContext;
  visual: VisualContext;
  style: StyleProfile | null;

  // ── メタ ──
  confidence: number;
  analyzedModules: string[];
  precisionLevel: 'quick' | 'standard' | 'deep';
  analyzedAt: string;
}
```

---

## 5. 信頼スコアと品質ゲート

### 5.1 モジュール別信頼スコア

各モジュールの出力に `confidence`（0.0〜1.0）を含める。

### 5.2 総合信頼スコア

MediaContext 全体の `confidence` はモジュール信頼度の加重平均:

```
confidence = Σ(モジュール信頼度 × 重み) / Σ(重み)
```

デフォルト重み:

| モジュール | 重み |
|-----------|------|
| scene | 1.0 |
| transcription | 1.5 |
| visual | 1.0 |
| color | 0.5 |
| acoustic | 0.8 |
| summary | 2.0 |

### 5.3 品質ゲート

| 判定 | 閾値 | アクション |
|------|------|----------|
| **合格** | confidence >= 0.7 | コンテキストを受理 |
| **要レビュー** | 0.4 <= confidence < 0.7 | 人間のレビューを要求 |
| **不合格** | confidence < 0.4 | 再分析（別モジュール/設定で） |

---

## 6. 分析エージェント（ContextAnalyzer）

### 6.1 役割

実装は専用の分析エージェント（ContextAnalyzer）を提供すべきである。メインのAIアシスタントとは別に、メディア分析だけを担当する専門エージェント。

```
ユーザー: 「この動画を分析して」
  ↓
ContextAnalyzer エージェント
  ├── コスト見積もり → ユーザーに表示
  ├── ユーザーが承認
  ├── モジュール群をディスパッチ（並列実行）
  ├── 結果を集約
  ├── 品質ゲートを通過
  ├── MediaContext を保存
  └── コンテキストグラフの更新をトリガー（対応する場合）
```

### 6.2 再分析

品質ゲート不合格、またはユーザーが再分析を要求した場合:

1. ContextAnalyzer が低信頼度のモジュールを特定
2. **該当モジュールのみ**再実行（パイプライン全体ではない）
3. 新しい結果を既存の MediaContext にマージ
4. 総合信頼スコアを再計算

---

## 7. コンテキスト供給（RAG統合）

### 7.1 JIT（Just-In-Time）供給

消費者はMediaContextの全体をAIプロンプトに入れるべきではない。タスクに応じて適切なレイヤーを選択する。

```typescript
interface M2CContextSelector {
  /** タスクに応じたコンテキストを選択 */
  select(
    mediaIds: string[],
    task: string,
    maxTokens?: number,
  ): string;
}
```

### 7.2 コンテキスト腐敗（Context Rot）の防止

大量の低品質コンテキストをプロンプトに詰め込むと、AI性能が劣化する（context rot）。M2Cは以下で防止:

- **信頼スコアによるフィルタ**: 低信頼度の結果は除外
- **レイヤー選択**: タスクに不要な情報は含めない
- **コンパクション**: summary モジュールによる圧縮表現の活用

---

## 8. バージョニング

- 仕様バージョンはセマンティックバージョニング: `MAJOR.MINOR.PATCH`
- MediaContext 内の `schemaVersion` でスキーマ変更を追跡
- モジュールは独自にバージョンを宣言
- 後方互換性: 消費者は未知のフィールドを無視すること
