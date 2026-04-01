# M2C コンテキスト供給プロトコル v0.2

> ステータス: Draft
> 日付: 2026-04-01

## 1. 概要

コンテキスト供給プロトコルは、M2C 分析パイプラインで生成された構造化コンテキストを AI コンシューマに配信する方法を定義する。生の `MediaContext` JSON から、最適なサイズでタスクに適合するコンテキストウィンドウへの経路を標準化する。

プロトコルは4つの独立したステージで構成される:

```
MediaContext JSON
      │
      ▼
┌───────────┐     ┌───────────┐     ┌────────────┐     ┌───────────┐
│   Write    │ ──▶ │  Select   │ ──▶ │  Compress  │ ──▶ │  Isolate  │
│ (Analyze → │     │ (JIT      │     │ (Reduce    │     │ (Partition │
│  Structure)│     │  Supply)  │     │  Tokens)   │     │  Context)  │
└───────────┘     └───────────┘     └────────────┘     └───────────┘
                                                              │
                                                              ▼
                                                      AI Consumer
                                                      (LLM Prompt)
```

### 1.1 設計原則

| 原則 | 説明 |
|------|------|
| **ステージ独立性** | 各ステージは独立したプロセスである。実装は任意のステージを別のプロバイダに差し替えてもよい MAY。 |
| **トークンバジェット対応** | すべてのステージは `maxTokens` 制約を遵守する。コンテキストはコンシューマのバジェット内に収まらなければならない MUST。 |
| **タスク適応型** | 供給パイプラインは下流のタスク種別に応じて動作を適応させる。 |
| **信頼度ゲート** | 低信頼度のコンテキストはコンシューマに到達する前にフィルタリングされる。 |
| **マルチメディア対応** | パイプラインは単一リクエスト内で複数のメディアソースからのコンテキストを処理する。 |

### 1.2 用語

| 用語 | 定義 |
|------|------|
| **MediaContext** | M2C 分析パイプラインが出力する構造化 JSON（v0.1 プロトコル仕様を参照）。 |
| **Context Level** | 含まれる詳細度を決定する粒度ティア（L0〜L3）。 |
| **Token Budget** | AI プロンプト内でコンテキストに割り当てられるトークンの最大数。 |
| **Context Rot** | 古い、無関係な、または低品質なコンテキストによって AI のパフォーマンスが劣化する現象。 |
| **Context Policy** | コンテキストフラグメントのコンシューマへの配信方法を制御するメタデータ。 |
| **Consumer** | 供給パイプラインからコンテキストを受信する任意の AI システム（LLM、エージェント、ツール）。 |

---

## 2. Write ステージ（分析 → 構造化）

Write ステージは、生の分析出力を `MediaContext` JSON スキーマに変換する。このステージは v0.1 プロトコル仕様で定義されており、ここでは完全性のために記載する。

### 2.1 3層構造

コンテキストは3つの粒度レベルで記述される:

```
Frame (最細粒度)
  └── Scene (中粒度)
        └── Media (最粗粒度)
```

| レイヤー | スコープ | 内容 |
|---------|---------|------|
| **Frame** | 単一キーフレーム / 音声セグメント | オブジェクト、キャプション、ランドマーク、単語レベルの文字起こし |
| **Scene** | 連続するシーン（`scene` モジュールで検出） | シーン要約、集約されたオブジェクト、シーンレベルのトランスクリプト、ムード |
| **Media** | メディアファイル全体 | 全体要約、タグ、話者、カラーパレット、スタイルプロファイル |

### 2.2 Write の要件

- モジュールは MediaContext スキーマで定義されたフィールドに結果を書き込まなければならない MUST（`schema.json` を参照）。
- 書き込まれる各フィールドには `confidence` スコア（0.0〜1.0）を含めなければならない MUST。
- モジュールは、集約目的で明示的に設計されている場合（例: `summary`）を除き、他のモジュールが書き込んだフィールドを上書きしてはならない MUST NOT。
- `analyzedAt` タイムスタンプは、Write ステージが完了した時点の UTC 時刻に設定しなければならない MUST。

### 2.3 インクリメンタル Write

実装はインクリメンタルな書き込みをサポートすべきである SHOULD:

1. 新しいモジュールが実行される（例: Quick から Standard 精度へのアップグレード）。
2. モジュールは既存の `MediaContext` に出力を書き込む。
3. `analyzedModules` 配列が更新される。
4. 集約 `confidence` が再計算される。

これにより、追加の深度が必要な場合にパイプライン全体を再実行することを回避できる。

---

## 3. Select ステージ（JIT 供給）

Select ステージは、`MediaContext` の*どの*部分をコンシューマに供給するかを決定する。Just-In-Time（JIT）コンテキスト配信を実装し、必要なものだけを必要な時に供給する。

### 3.1 Context Level

| レベル | 名前 | 内容 | トークン推定値 |
|-------|------|------|--------------|
| **L0** | Tags | `tags`, `mood` | 約20〜50トークン |
| **L1** | Summary | L0 + `summary`, `colorPalette`, `speakers`, `duration` | 約100〜300トークン |
| **L2** | Scenes | L1 + `scenes[].summary`, `scenes[].objects`, `scenes[].mood` | 約500〜2,000トークン |
| **L3** | Full | `transcription`, `acoustic`, `visual`, `style` を含むすべてのフィールド | 約2,000〜10,000+トークン |

トークン推定値は概算であり、メディアの長さとコンテンツの密度によって変動する。

### 3.2 タスク適応型セレクション

セレクタはコンシューマのタスク種別に基づいて適切な Context Level を選択しなければならない MUST:

| タスク種別 | 推奨レベル | 根拠 |
|-----------|-----------|------|
| `search` | L0 (Tags) | フィルタリングとランキングに必要な最小限のトークン |
| `edit_planning` | L1 (Summary) | 編集プランを提案するのに十分なコンテキスト |
| `scene_editing` | L2 (Scenes) | カット判断のためのシーンレベルの詳細 |
| `transcription_work` | L3 (Full) | 単語レベルのトランスクリプトデータが必要 |

実装は追加のタスク種別を定義してもよい MAY。不明なタスク種別はデフォルトで L1 にすべきである SHOULD。

### 3.3 Token Budget の管理

`maxTokens` バジェットが指定された場合:

1. セレクタは `maxTokens` を超える出力を生成してはならない MUST NOT。
2. 推奨レベルがバジェットを超過する場合、セレクタはより低いレベルにダウングレードしなければならない MUST。
3. L0 でさえバジェットを超過する場合、セレクタはタグを切り詰めて最小限のコンテキストを返さなければならない MUST。

### 3.4 マルチメディア Budget 配分

単一リクエスト内で複数のメディアソースのコンテキストを選択する場合、セレクタはソース間でトークンバジェットを配分しなければならない MUST。

**デフォルトの配分戦略** — 関連度に比例:

```
Given:
  - N media sources
  - Total budget: maxTokens
  - Relevance score r[i] for each source (0.0–1.0)

Allocation:
  budget[i] = maxTokens × (r[i] / Σ r[j])
```

関連度スコアが利用できない場合、セレクタは均等に配分すべきである SHOULD:

```
budget[i] = maxTokens / N
```

**例**: 10メディアソース、1,000トークンバジェット、均等な関連度:

- 各ソースに約100トークン → L0 または切り詰められた L1

### 3.5 ContextSelector インターフェース

#### TypeScript

```typescript
interface M2CContextSelector {
  /**
   * Select context for one or more media sources.
   *
   * @param request - The context request parameters
   * @returns Selected context, formatted for AI consumption
   */
  select(request: M2CContextRequest): M2CContextResponse;
}

interface M2CContextRequest {
  /** Media IDs to include */
  mediaIds: string[];

  /** Task type that determines default context level */
  taskType: string;

  /** Maximum tokens for the entire response */
  maxTokens?: number;

  /**
   * Minimum confidence threshold.
   * Context with confidence below this value MUST be excluded.
   * Default: 0.4
   */
  minConfidence?: number;

  /**
   * Explicit level override.
   * When set, bypasses task-adaptive selection.
   */
  level?: 'L0' | 'L1' | 'L2' | 'L3';

  /**
   * Per-media relevance scores (0.0–1.0).
   * Keys are media IDs. Missing entries default to 1.0.
   */
  relevance?: Record<string, number>;
}

interface M2CContextResponse {
  /** Selected context per media source */
  contexts: M2CSelectedContext[];

  /** Total estimated tokens consumed */
  totalTokens: number;

  /** The level actually used (may differ from requested if budget-constrained) */
  effectiveLevel: 'L0' | 'L1' | 'L2' | 'L3';
}

interface M2CSelectedContext {
  /** Media ID */
  mediaId: string;

  /** Context level used for this source */
  level: 'L0' | 'L1' | 'L2' | 'L3';

  /** Estimated tokens for this context */
  tokens: number;

  /** The selected context data */
  data: Partial<MediaContext>;
}
```

#### JSON Schema (Request)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://m2c-protocol.org/schema/v0.2/context-request.json",
  "title": "M2CContextRequest",
  "type": "object",
  "required": ["mediaIds", "taskType"],
  "properties": {
    "mediaIds": {
      "type": "array",
      "items": { "type": "string" },
      "minItems": 1
    },
    "taskType": {
      "type": "string",
      "examples": ["search", "edit_planning", "scene_editing", "transcription_work"]
    },
    "maxTokens": {
      "type": "integer",
      "minimum": 1
    },
    "minConfidence": {
      "type": "number",
      "minimum": 0.0,
      "maximum": 1.0,
      "default": 0.4
    },
    "level": {
      "type": "string",
      "enum": ["L0", "L1", "L2", "L3"]
    },
    "relevance": {
      "type": "object",
      "additionalProperties": {
        "type": "number",
        "minimum": 0.0,
        "maximum": 1.0
      }
    }
  }
}
```

---

## 4. Compress ステージ（トークン削減）

Compress ステージは、情報密度を保ちながらコンテキストサイズを削減する。圧縮は Select の後、Isolate の前に適用される。

### 4.1 圧縮戦略

実装は少なくとも1つの圧縮戦略をサポートしなければならず MUST、4つすべてをサポートすべきである SHOULD:

#### 4.1.1 Summary 圧縮

冗長なコンテンツを簡潔な要約に置き換える。

- **入力**: 詳細なシーン記述、詳細なトランスクリプト
- **出力**: 凝縮された自然言語の要約
- **方式**: LLM ベースの要約または抽出要約

```
Before (480 tokens):
  Scene 3: "The host picks up a red bell pepper from the cutting board,
  washes it under the faucet, places it back, then reaches for a knife
  from the magnetic strip on the wall. They begin slicing the pepper
  into thin strips while explaining the technique..."

After (85 tokens):
  Scene 3: "Host prepares red bell pepper: wash, slice into strips.
  Explains cutting technique."
```

#### 4.1.2 Temporal 圧縮

冗長な時間情報を除去する。

- **入力**: フレーム単位またはセグメント単位のデータ
- **出力**: 統合された時間範囲
- **ルール**:
  - 同一のオブジェクトとアクションを持つ連続フレームは、単一の範囲にマージしなければならない MUST。
  - 0.5秒未満の無音セグメントはドロップすべきである SHOULD。
  - 繰り返される BGM のムードセグメントは統合しなければならない MUST。

```
Before:
  Frame 100 (4.0s): person, kitchen, knife
  Frame 101 (4.04s): person, kitchen, knife
  Frame 102 (4.08s): person, kitchen, knife
  Frame 103 (4.12s): person, kitchen, knife, bell_pepper

After:
  4.0s–4.08s: person, kitchen, knife
  4.12s: person, kitchen, knife, bell_pepper (change detected)
```

#### 4.1.3 Modality フィルタリング

指定タスクに対して情報密度の低いモダリティを除去する。

| タスク種別 | 保持 | 除去 |
|-----------|------|------|
| `search` | tags, mood | acoustic, visual, transcription |
| `edit_planning` | summary, scenes, color | acoustic の詳細、単語レベルのトランスクリプト |
| `scene_editing` | scenes, visual, color | 細粒度の acoustic |
| `transcription_work` | transcription, speakers | visual, color, acoustic |

実装はカスタムタスク種別のモダリティ優先度を定義してもよい MAY。

#### 4.1.4 反復的要約

極めて長いコンテキストを持つメディア（例: 60分以上の動画）の場合:

```
L3 Full Context (10,000+ tokens)
      │
      ▼
Per-Chapter Summary (2,000 tokens)
      │
      ▼
Summary of Summaries (500 tokens)
      │
      ▼
Within Budget ✓
```

各反復では以下を保持しなければならない MUST:
- 主要エンティティ（話者、オブジェクト、ロケーション）
- 時間構造（シーン順序、幕の境界）
- 信頼度スコア（入力の最小値として伝搬）

### 4.2 圧縮メタデータ

圧縮されたコンテキストには、適用された圧縮を示すメタデータを付与しなければならない MUST:

```json
{
  "compression": {
    "strategy": ["summary", "temporal", "modality_filter"],
    "originalTokens": 8500,
    "compressedTokens": 1200,
    "ratio": 0.14,
    "lossLevel": "moderate"
  }
}
```

`lossLevel` フィールドは情報損失の度合いを示す:

| レベル | 説明 |
|-------|------|
| `none` | 情報損失なし（フォーマット変更のみ） |
| `minimal` | 冗長性を除去、すべての重要な事実は保持 |
| `moderate` | 要約済み; 一部の詳細は失われるが構造は保持 |
| `aggressive` | 大幅な要約; 高レベルの情報のみ残存 |

---

## 5. Isolate ステージ（コンテキスト分割）

Isolate ステージはコンテキストをスコープされたフラグメントに分割し、各コンシューマが関連する情報のみを受け取ることを保証する。これにより、モダリティ、時間範囲、エージェントロール間でのコンテキスト汚染を防止する。

### 5.1 分離の3次元

#### 5.1.1 Source/Type 分離

コンテキストはモダリティおよびデータ型ごとに分離可能でなければならない MUST:

| ソース種別 | フィールド |
|-----------|----------|
| `visual` | `scenes[].objects`, `scenes[].keyframePath`, `visual.*` |
| `transcription` | `transcription[]`, `speakers[]` |
| `acoustic` | `acoustic.*` |
| `color` | `colorPalette`, `scenes[].colorPalette` |
| `summary` | `summary`, `tags`, `mood` |
| `style` | `style.*` |

コンシューマが `transcription` コンテキストのみをリクエストした場合、他のソース種別のフィールドを含めてはならない MUST NOT。

#### 5.1.2 Temporal 分離

コンテキストは時間範囲でフィルタリング可能でなければならない MUST:

- コンシューマが特定の時間範囲（例: 30.0秒〜60.0秒）で操作する場合、Isolate ステージは以下を行わなければならない MUST:
  1. リクエストされた範囲にオーバーラップするシーンのみを含める。
  2. 文字起こしセグメントを範囲の境界でトリムする。
  3. 範囲内の音響イベントのみを含める。
  4. それ以前の時間範囲からの累積状態をリセットする。

- Temporal 分離は、シーン間の情報漏れを防ぐために、シーン境界でコンテキストリセットを適用すべきである SHOULD。

#### 5.1.3 Agent Role 分離

コンテキストはコンシューマのロールごとに分割可能でなければならない MUST:

| ロール | 典型的なコンテキスト | 根拠 |
|-------|-------------------|------|
| `planner` | L1 の要約 + シーン構造 | 全体像が必要で、詳細は不要 |
| `worker` | 割り当てられたシーン範囲の L2〜L3 のみ | 狭い範囲で深い詳細が必要 |
| `reviewer` | L1 + 品質メトリクス + 信頼度スコア | 評価が必要で、編集は不要 |
| `user` | L0〜L1 の自然言語要約 | 読みやすい表示 |

実装はロールベースのフィルタリングをサポートしなければならない MUST。コンシューマはコンテキストリクエスト内で自身のロールを宣言すべきである SHOULD。

### 5.2 Context Policy スキーマ

各コンテキストフラグメントには、分離の制約を宣言する `contextPolicy` メタデータオブジェクトを付与してもよい MAY:

```json
{
  "contextPolicy": {
    "audience": ["planner", "worker", "user"],
    "temporalScope": {
      "start": 0,
      "end": 45.2
    },
    "modalities": ["visual", "transcription"],
    "priority": 0.8
  }
}
```

#### フィールド定義

| フィールド | 型 | 説明 |
|-----------|---|------|
| `audience` | `string[]` | このコンテキストを受信すべきロール SHOULD。空配列はすべてのロールを意味する。 |
| `temporalScope` | `object` | このコンテキストがカバーする時間範囲。`start` と `end` は秒単位。 |
| `temporalScope.start` | `number` | 開始時刻（秒、包含）。 |
| `temporalScope.end` | `number` | 終了時刻（秒、包含）。 |
| `modalities` | `string[]` | 含まれるソース種別。有効な値はセクション 5.1.1 を参照。 |
| `priority` | `number` | 優先度の重み（0.0〜1.0）。バジェットが逼迫した場合、優先度の高いフラグメントが保持される。 |

#### JSON Schema (Context Policy)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://m2c-protocol.org/schema/v0.2/context-policy.json",
  "title": "M2CContextPolicy",
  "type": "object",
  "properties": {
    "audience": {
      "type": "array",
      "items": {
        "type": "string",
        "examples": ["planner", "worker", "reviewer", "user"]
      },
      "default": []
    },
    "temporalScope": {
      "type": "object",
      "required": ["start", "end"],
      "properties": {
        "start": { "type": "number", "minimum": 0 },
        "end": { "type": "number", "minimum": 0 }
      }
    },
    "modalities": {
      "type": "array",
      "items": {
        "type": "string",
        "enum": ["visual", "transcription", "acoustic", "color", "summary", "style"]
      }
    },
    "priority": {
      "type": "number",
      "minimum": 0.0,
      "maximum": 1.0,
      "default": 0.5
    }
  }
}
```

### 5.3 分離の強制

Isolate ステージは以下のルールを強制しなければならない MUST:

1. `audience: ["planner"]` を持つコンテキストフラグメントは、`worker` コンシューマに配信してはならない MUST NOT。
2. `temporalScope: { start: 0, end: 30 }` を持つコンテキストフラグメントは、30秒を超えるタイムスタンプのデータを含んではならない MUST NOT。
3. `modalities: ["transcription"]` を持つコンテキストフラグメントは、visual や acoustic のデータを含んではならない MUST NOT。
4. バジェット制約によりフラグメントのドロップが必要な場合、`priority` フィールドの値が低いフラグメントから先にドロップしなければならない MUST。

---

## 6. Context Rot の防止

Context Rot は、古い、無関係な、または低品質なコンテキストが AI のパフォーマンスを劣化させる場合に発生する。供給パイプラインは以下の対策を実装しなければならない MUST:

### 6.1 信頼度スコアによるフィルタリング

- `minConfidence`（デフォルト: 0.4）を下回る信頼度のコンテキストは、Select ステージで除外しなければならない MUST。
- フィールドごとの信頼度を確認すべきである SHOULD: シーンの信頼度が閾値を下回る場合、メディア全体の信頼度が閾値を超えていても、そのシーンは除外しなければならない MUST。
- 実装はデバッグのために除外されたコンテキストをログに記録すべきである SHOULD。

### 6.2 レイヤー選択による Rot 防止

必要以上のコンテキストを供給すること自体が Rot の一形態である — 関連情報がノイズで希釈される。

| 症状 | 原因 | 対策 |
|------|------|------|
| AI が重要なコンテキストを無視する | コンテキストウィンドウが大きすぎる | レベルをダウングレード（L3 → L2 → L1） |
| AI がメディアの内容を幻覚する | 低信頼度のフィールドが存在する | `minConfidence` 閾値を引き上げる |
| AI がソースを混同する | 複数のメディアコンテキストが混在 | Source/Type 分離を適用する |
| AI が時間的整合性を失う | ローカルタスクにフルタイムラインを供給 | Temporal 分離を適用する |

### 6.3 コンパクション

コンテキストが蓄積される長時間セッションの場合:

1. 実装は定期的にコンテキストの再選択と再圧縮を行うべきである SHOULD。
2. 古くなったコンテキスト（例: アクティブなワークスペースにもう存在しないメディアのもの）は退避すべきである SHOULD。
3. コンパクションは最も信頼度の高いコンテキストを優先的に保持しなければならない MUST。

### 6.4 鮮度メタデータ

コンテキストには鮮度情報を付与すべきである SHOULD:

```json
{
  "freshness": {
    "analyzedAt": "2026-04-01T10:30:00Z",
    "suppliedAt": "2026-04-01T10:31:15Z",
    "ttlSeconds": 3600
  }
}
```

- `analyzedAt`: 分析が実行された日時。
- `suppliedAt`: コンテキストが最後にコンシューマに供給された日時。
- `ttlSeconds`: 推奨有効期間。期限切れ後、コンシューマは新鮮なコンテキストをリクエストすべきである SHOULD。

---

## 7. インターフェース定義

### 7.1 M2CContextSelector (TypeScript)

すべてのステージを含む完全なインターフェース:

```typescript
interface M2CContextSelector {
  /**
   * Select, compress, and isolate context for AI consumption.
   *
   * This is the primary entry point. It orchestrates Select → Compress → Isolate.
   *
   * @param request - Context request parameters
   * @returns Context response with selected, compressed, isolated fragments
   */
  select(request: M2CContextRequest): M2CContextResponse;
}

interface M2CContextRequest {
  /** One or more media IDs to retrieve context for */
  mediaIds: string[];

  /** Task type for adaptive level selection */
  taskType: string;

  /** Maximum token budget for the entire response */
  maxTokens?: number;

  /** Minimum confidence threshold (default: 0.4) */
  minConfidence?: number;

  /** Explicit level override (bypasses task-adaptive selection) */
  level?: 'L0' | 'L1' | 'L2' | 'L3';

  /** Per-media relevance scores for budget allocation */
  relevance?: Record<string, number>;

  /** Consumer role for isolation (default: all roles) */
  role?: 'planner' | 'worker' | 'reviewer' | 'user';

  /** Temporal scope filter */
  temporalScope?: {
    start: number;
    end: number;
  };

  /** Modality filter */
  modalities?: ('visual' | 'transcription' | 'acoustic' | 'color' | 'summary' | 'style')[];

  /** Compression preferences */
  compression?: {
    /** Allowed strategies (default: all) */
    strategies?: ('summary' | 'temporal' | 'modality_filter' | 'iterative')[];
    /** Maximum acceptable loss level */
    maxLossLevel?: 'none' | 'minimal' | 'moderate' | 'aggressive';
  };
}

interface M2CContextResponse {
  /** Selected context fragments */
  contexts: M2CSelectedContext[];

  /** Total estimated tokens */
  totalTokens: number;

  /** Effective level after budget adjustment */
  effectiveLevel: 'L0' | 'L1' | 'L2' | 'L3';

  /** Compression metadata (if compression was applied) */
  compression?: {
    strategy: string[];
    originalTokens: number;
    compressedTokens: number;
    ratio: number;
    lossLevel: 'none' | 'minimal' | 'moderate' | 'aggressive';
  };

  /** Freshness metadata */
  freshness: {
    analyzedAt: string;
    suppliedAt: string;
    ttlSeconds: number;
  };
}

interface M2CSelectedContext {
  /** Media ID */
  mediaId: string;

  /** Context level used */
  level: 'L0' | 'L1' | 'L2' | 'L3';

  /** Estimated tokens */
  tokens: number;

  /** Selected and possibly compressed context data */
  data: Partial<MediaContext>;

  /** Context policy for this fragment */
  contextPolicy?: {
    audience: string[];
    temporalScope?: { start: number; end: number };
    modalities: string[];
    priority: number;
  };
}
```

### 7.2 JSON Schema (Response)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://m2c-protocol.org/schema/v0.2/context-response.json",
  "title": "M2CContextResponse",
  "type": "object",
  "required": ["contexts", "totalTokens", "effectiveLevel", "freshness"],
  "properties": {
    "contexts": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["mediaId", "level", "tokens", "data"],
        "properties": {
          "mediaId": { "type": "string" },
          "level": { "type": "string", "enum": ["L0", "L1", "L2", "L3"] },
          "tokens": { "type": "integer", "minimum": 0 },
          "data": { "type": "object" },
          "contextPolicy": {
            "$ref": "https://m2c-protocol.org/schema/v0.2/context-policy.json"
          }
        }
      }
    },
    "totalTokens": {
      "type": "integer",
      "minimum": 0
    },
    "effectiveLevel": {
      "type": "string",
      "enum": ["L0", "L1", "L2", "L3"]
    },
    "compression": {
      "type": "object",
      "properties": {
        "strategy": {
          "type": "array",
          "items": { "type": "string" }
        },
        "originalTokens": { "type": "integer" },
        "compressedTokens": { "type": "integer" },
        "ratio": { "type": "number", "minimum": 0, "maximum": 1 },
        "lossLevel": {
          "type": "string",
          "enum": ["none", "minimal", "moderate", "aggressive"]
        }
      }
    },
    "freshness": {
      "type": "object",
      "required": ["analyzedAt", "suppliedAt", "ttlSeconds"],
      "properties": {
        "analyzedAt": { "type": "string", "format": "date-time" },
        "suppliedAt": { "type": "string", "format": "date-time" },
        "ttlSeconds": { "type": "integer", "minimum": 0 }
      }
    }
  }
}
```

---

## 8. エンドツーエンド例

コンシューマが、編集プランを立てるために3つのメディアソースのコンテキストをリクエストする:

### リクエスト

```json
{
  "mediaIds": ["vid_001", "vid_002", "img_003"],
  "taskType": "edit_planning",
  "maxTokens": 800,
  "minConfidence": 0.5,
  "role": "planner",
  "compression": {
    "maxLossLevel": "moderate"
  }
}
```

### パイプライン実行

```
1. SELECT
   - taskType "edit_planning" → L1 (Summary)
   - Budget: 800 tokens / 3 sources = ~267 each
   - L1 fits within ~267 tokens ✓

2. COMPRESS
   - vid_001 L1: 250 tokens → no compression needed
   - vid_002 L1: 310 tokens → summary compression → 220 tokens
   - img_003 L1: 120 tokens → no compression needed
   - Total: 590 tokens (within budget)

3. ISOLATE
   - Role: planner → include summary, scene structure; exclude raw transcription
   - No temporal filter → full media range
   - Attach contextPolicy to each fragment
```

### レスポンス

```json
{
  "contexts": [
    {
      "mediaId": "vid_001",
      "level": "L1",
      "tokens": 250,
      "data": {
        "summary": "Interview with product designer. Two speakers discuss UX trends.",
        "tags": ["interview", "design", "UX"],
        "mood": "professional, thoughtful",
        "speakers": [
          { "id": "SPEAKER_01", "ratio": 0.6 },
          { "id": "SPEAKER_02", "ratio": 0.4 }
        ],
        "duration": 720,
        "confidence": 0.88
      },
      "contextPolicy": {
        "audience": ["planner"],
        "temporalScope": { "start": 0, "end": 720 },
        "modalities": ["summary"],
        "priority": 0.8
      }
    },
    {
      "mediaId": "vid_002",
      "level": "L1",
      "tokens": 220,
      "data": {
        "summary": "Product demo. Single presenter walks through 5 features.",
        "tags": ["demo", "product", "tutorial"],
        "mood": "energetic, clear",
        "speakers": [{ "id": "SPEAKER_01", "ratio": 1.0 }],
        "duration": 300,
        "confidence": 0.91
      },
      "contextPolicy": {
        "audience": ["planner"],
        "temporalScope": { "start": 0, "end": 300 },
        "modalities": ["summary"],
        "priority": 0.8
      }
    },
    {
      "mediaId": "img_003",
      "level": "L1",
      "tokens": 120,
      "data": {
        "summary": "Product hero shot on gradient background. Clean typography.",
        "tags": ["product", "hero", "marketing"],
        "mood": "minimal, premium",
        "colorPalette": ["#1A1A2E", "#E94560", "#FFFFFF"],
        "duration": 0,
        "confidence": 0.95
      },
      "contextPolicy": {
        "audience": ["planner"],
        "temporalScope": { "start": 0, "end": 0 },
        "modalities": ["summary", "color"],
        "priority": 0.6
      }
    }
  ],
  "totalTokens": 590,
  "effectiveLevel": "L1",
  "compression": {
    "strategy": ["summary"],
    "originalTokens": 680,
    "compressedTokens": 590,
    "ratio": 0.87,
    "lossLevel": "minimal"
  },
  "freshness": {
    "analyzedAt": "2026-04-01T10:30:00Z",
    "suppliedAt": "2026-04-01T10:45:00Z",
    "ttlSeconds": 3600
  }
}
```

---

## 9. v0.1 との関係

本仕様は、v0.1 プロトコル仕様のセクション5（「Context Supply」）を拡張するものである。v0.1 の `M2CContextSelector` インターフェースは、ここで定義される v0.2 インターフェースのサブセットである。

| v0.1 の概念 | v0.2 での拡張 |
|------------|-------------|
| JIT Supply（セクション 5.1） | Select ステージ（セクション 3）にタスク適応型セレクションとマルチメディア Budget 配分を追加 |
| Context Selector インターフェース（セクション 5.2） | 圧縮と分離をサポートする完全な `M2CContextRequest` / `M2CContextResponse` |
| Four Pillars 概要（セクション 4.1） | 各ピラーをフルステージ仕様に拡張（セクション 2〜5） |
| Context Level（L0〜L3） | トークン推定値、レベルダウングレードルール、Budget 強制 |

v0.2 は後方互換性がある: v0.1 の `M2CContextSelector.select()` 呼び出し（`mediaIds`, `task`, `maxTokens` を使用）は有効な v0.2 `M2CContextRequest` である。

---

## 10. 適合性

### 10.1 適合性レベル

| レベル | 要件 |
|-------|------|
| **M2C Supply Basic** | Select ステージ（セクション 3）を L0〜L3 レベルとタスク適応型セレクションで実装しなければならない MUST。 |
| **M2C Supply Standard** | Basic に加え、少なくとも1つの Compress 戦略（セクション 4）と信頼度フィルタリング（セクション 6.1）を実装しなければならない MUST。 |
| **M2C Supply Full** | Standard に加え、4つすべての Compress 戦略、3つすべての Isolate 次元（セクション 5）、および鮮度メタデータ（セクション 6.4）を実装しなければならない MUST。 |

### 10.2 拡張ポイント

実装は以下の方法で本仕様を拡張してもよい MAY:

- Select ステージ用のカスタムタスク種別の追加。
- Compress ステージ用のカスタム圧縮戦略の追加。
- Isolate ステージ用のカスタムオーディエンスロールの追加。
- Source/Type 分離用のカスタムモダリティ種別の追加。

拡張は、上記に列挙された標準のタスク種別、戦略、ロール、およびモダリティについて本仕様で定義された動作と矛盾してはならない MUST NOT。

---

## 11. バージョニング

- 本仕様はセマンティックバージョニングに従う。
- `MediaContext` の `schemaVersion` フィールドは本仕様では変更されない（v0.1 で定義されたままである）。
- 供給プロトコルのバージョンは JSON Schema URI の `$id` で個別に管理される（例: `v0.2/context-request.json`）。
