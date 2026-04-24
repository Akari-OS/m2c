# M2C プロトコル仕様 v0.2

> ステータス: ドラフト
> 日付: 2026-04-01
> 前バージョン: [v0.1](../../v0.1/protocol.md)
> 関連仕様: AMP v0.1 (github.com/Akari-OS/amp), ACE v0.1 (github.com/Akari-OS/ace)

## 1. 概要

M2C (Media to Context) は、メディアファイルを構造化された AI 向けコンテキストに変換するためのオープンプロトコルである。以下の4つの領域を標準化する:

1. **分析 (Analysis)** — メディアの分析方法（モジュール、ディスパッチ、精度）
2. **スキーマ (Schema)** — 分析結果の構造化方法（MediaContext）
3. **供給 (Supply)** — AI コンシューマへのコンテキスト配信方法（JIT、バジェット、分離）
4. **相互運用性 (Interoperability)** — 既存標準（IPTC、C2PA、XMP）との統合方法

### 1.1 設計原則

| 原則 | 説明 |
|------|------|
| **Context First** | コンテキストなしに AI 操作を行ってはならない。メディアは使用前に分析されなければならない (MUST)。 |
| **Modular** | 各分析機能は独立した置換可能なモジュールである。 |
| **Layered** | コンテキストは効率的な配信のためにレイヤー（L0〜L3）で構造化される。 |
| **Scored** | すべての結果に信頼度スコアが付与される。品質ゲートが不正確なコンテキストを防止する。 |
| **Cost-Aware** | 分析コストは事前に見積もられる。ユーザーが精度レベルを選択できる。 |
| **Secure by Default** | メディアデータに対する認証は REQUIRED（オプションではない）。 |
| **Local-First** | プロトコルは完全なローカル実行をサポートしなければならない (MUST)。クラウドはオプション。 |
| **Standard-Compatible** | IPTC、C2PA、XMP、Schema.org と相互運用可能。 |

### 1.2 用語

| 用語 | 定義 |
|------|------|
| **Media** | あらゆるデジタルファイル: 動画、画像、音声、ドキュメント、3D モデルなど。 |
| **Module** | 独立した分析ユニット（例: シーン検出、文字起こし）。 |
| **Context** | メディアの中身を記述する構造化出力（JSON）。 |
| **Dispatcher** | メディアタイプと精度に基づいてモジュールを選択するオーケストレーター。 |
| **Precision Level** | 分析の深さ（Quick / Standard / Deep / カスタム）。 |
| **Quality Gate** | 結果を受け入れる前の信頼度閾値チェック。 |
| **Consumer** | MediaContext を使用するシステム（AI アシスタント、検索エンジン、エディタなど）。 |
| **Supply** | コンシューマへコンテキストを配信するプロセス。[supply.md](../supply.md) を参照。 |

### 1.3 プロトコルバージョン

M2C は**日付ベースのバージョニング**を採用する: `YYYY-MM-DD`（例: `2026-04-01`）。

バージョンネゴシエーションは MCP のパターンに従う:

1. Consumer がサポートするバージョンを送信
2. Provider は同じバージョン（サポートしている場合）または最新バージョンで応答
3. 互換性がない場合、接続は拒否される

---

## 2. メディアタイプ

M2C はあらゆるデジタルメディアをサポートする。`mediaType` フィールドは拡張可能な列挙型を使用する:

### 2.1 コアタイプ

| タイプ | 説明 | 例 |
|--------|------|-----|
| `video` | 音声を伴う場合がある動画 | .mp4, .mov, .avi, .mkv, .webm |
| `image` | 静止画（ラスター） | .jpg, .png, .gif, .webp, .bmp |
| `audio` | スピーチ、効果音、環境音 | .mp3, .wav, .aac, .flac, .ogg |
| `music` | 音楽作品 | .mp3, .wav, .midi, .flac |
| `document` | テキストベースの文書 | .pdf, .docx, .txt, .md, .html |
| `presentation` | スライドベースのコンテンツ | .pptx, .key, .odp |
| `spreadsheet` | 表形式データ | .xlsx, .csv, .tsv |
| `vector` | ベクターグラフィクス | .svg, .ai, .eps |
| `3d` | 3D モデルとシーン | .gltf, .obj, .fbx, .usdz |
| `archive` | コンテナ（再帰的に分析） | .zip, .tar.gz |
| `data` | 構造化 / 半構造化データ | .json, .xml, .yaml |

### 2.2 カスタムタイプ

実装は `x-` プレフィックスを用いてカスタムタイプを定義してもよい (MAY):

```json
{ "mediaType": "x-figma-design" }
```

カスタムタイプであっても、有効な `MediaContext` 出力を生成しなければならない (MUST)。

---

## 3. 分析パイプライン

### 3.1 フロー

```
1. INPUT:    Media URI + 精度レベル
2. NEGOTIATE: 能力ネゴシエーション（利用可能なモジュール）
3. DISPATCH:  メディアタイプ → モジュールセット選択
4. ESTIMATE:  コスト / 所要時間の見積もり → ユーザー確認
5. EXECUTE:   選択されたモジュールを実行（可能な場合は並列）
6. AGGREGATE: モジュール出力を MediaContext にマージ
7. SCORE:     全体の信頼度を算出
8. GATE:      品質閾値との照合
9. OUTPUT:    MediaContext JSON
```

### 3.2 Intent-Driven Dispatch

Dispatcher は**3つの入力**に基づいてモジュールを選択する: メディアタイプ、精度レベル、そして**ユーザーの意図 (Intent)**。

#### 3.2.1 Intent（オプション）

モジュール選択を最適化するために `M2CIntent` オブジェクトを指定してもよい (MAY):

```typescript
interface M2CIntent {
  /** What the user wants to do with the media */
  purpose?: string;  // e.g., "Extract highlights for short-form video"
  /** Known characteristics that skip unnecessary analysis */
  hints?: {
    hasAudio?: boolean;        // false → skip transcription, acoustic
    hasSpeech?: boolean;       // false → skip speaker diarization
    hasMusic?: boolean;        // false → skip music analysis
    cameraType?: 'static' | 'dynamic' | 'multi';  // static → skip scene detection
    speakerCount?: number;     // 1 → skip diarization
    language?: string;         // Pre-set language for transcription
    genre?: string;            // e.g., "tutorial", "vlog", "interview"
  };
  /** Background context for richer summary generation */
  background?: string;  // e.g., "This is part of a cooking series for beginners"
}
```

Intent が指定された場合:

- Dispatcher は無関係なモジュールをスキップすべきである (SHOULD)（例: `hasAudio: false` → `transcription` と `acoustic` をスキップ）
- `summary` モジュールは `purpose` と `background` を生成されるサマリに反映すべきである (SHOULD)
- コスト見積もりは削減されたモジュールセットを反映すべきである (SHOULD)

#### 3.2.2 デフォルトディスパッチテーブル（Intent なしの場合）

Intent が指定されていない場合、モジュールはメディアタイプと精度に基づいて選択される:

| メディアタイプ | Quick | Standard | Deep |
|---------------|-------|----------|------|
| video | scene, summary | + transcription, color | + visual, acoustic, pose |
| image | visual, summary | + color | + pose |
| audio | transcription, summary | + acoustic | + speaker_diarization |
| music | acoustic, summary | + structure | + harmony |
| document | text_extract, summary | + layout | + table_extract |
| presentation | slide_extract, summary | + visual | + layout |
| vector | visual, summary | + color | + structure |
| 3d | visual, summary | + geometry | + material |

#### 3.2.3 Intent によるディスパッチ変更の例

```
Intent: { purpose: "interview highlight reel", hints: { cameraType: "static", speakerCount: 2 } }
Media: video
→ Skip: scene (static camera), pose
→ Prioritize: transcription (speaker diarization), acoustic (silence detection for cuts)
→ Summary: Incorporates "interview" context for relevant highlight suggestions

Intent: { purpose: "add subtitles", hints: { hasMusic: false } }
Media: video
→ Skip: acoustic, color
→ Prioritize: transcription (large model for accuracy)
→ Summary: Focused on speech content and timing

Intent: { hints: { hasAudio: false } }
Media: video (screen recording)
→ Skip: transcription, acoustic, speaker diarization
→ Focus: visual (UI elements, text on screen), scene
```

カスタムメディアタイプ向けにディスパッチルールを拡張してもよい (MAY)。

### 3.3 実行順序

依存関係がない限り、モジュールは並列実行してもよい (MAY):

```
independent modules ──┐
                      ├→ summary (depends on all)
independent modules ──┘
```

`summary` モジュールは最後に実行しなければならない (MUST)。

### 3.4 精度レベル

| レベル | 用途 | コスト | 所要時間 |
|--------|------|--------|---------|
| **Quick** | ブラウジング、プレビュー、基本的な分類 | 低 | 数秒 |
| **Standard** | 編集、検索、AI 支援ワークフロー | 中 | 2分以内 |
| **Deep** | AI 駆動の編集、スタイル分析、完全な理解 | 高 | 数分 |

実装は文字列としてカスタム精度レベルを定義してもよい (MAY)（例: `"ultrafast"`, `"forensic"`）。

コストと所要時間は `M2CEstimate` で報告される。単位は実装定義。

---

## 4. モジュールインターフェース

### 4.1 共通インターフェース

すべての M2C モジュールは以下のインターフェースを実装しなければならない (MUST):

```typescript
interface M2CModule {
  /** Unique module identifier */
  id: string;
  /** Human-readable name */
  name: string;
  /** Semantic version */
  version: string;
  /** Supported media types */
  supportedTypes: string[];
  /** Dependencies (module IDs that must run first, or "*" for all) */
  dependencies: string[];

  /** Analyze media */
  analyze(input: M2CModuleInput): Promise<M2CModuleOutput>;

  /** Estimate cost before execution */
  estimate(input: M2CModuleInput): Promise<M2CEstimate>;
}

interface M2CModuleInput {
  /** Media URI (file://, https://, s3://, etc.) */
  mediaUri: string;
  /** Media type */
  mediaType: string;
  /** Duration in seconds (0 for non-temporal media) */
  duration: number;
  /** User intent (purpose, hints, background) — passed to all modules */
  intent?: M2CIntent;
  /** Module-specific options */
  options?: Record<string, unknown>;
  /** Outputs from dependency modules */
  dependencyOutputs?: Record<string, M2CModuleOutput>;
}

interface M2CModuleOutput {
  /** Module ID */
  moduleId: string;
  /** Module version */
  moduleVersion: string;
  /** Execution time in milliseconds */
  durationMs: number;
  /** Confidence score (0.0–1.0) */
  confidence: number;
  /** Module-specific structured data */
  data: unknown;
  /** Output artifact URIs (keyframes, extracted text, etc.) */
  artifacts?: M2CArtifact[];
}

interface M2CArtifact {
  /** Artifact URI */
  uri: string;
  /** MIME type */
  mimeType: string;
  /** Human-readable label */
  label?: string;
}

interface M2CEstimate {
  /** Estimated cost (implementation-defined unit) */
  cost: number;
  /** Cost unit label (e.g., "credits", "USD", "tokens") */
  costUnit: string;
  /** Estimated time in milliseconds */
  timeMs: number;
}
```

正式な定義は [schema.json](../schema.json) にある。
上記の TypeScript インターフェースは参考情報である。

### 4.2 MCP 互換性

M2C モジュールは MCP ツールサーバーとして公開してもよい (MAY)。ブリッジ仕様については [interop.md §MCP](../interop.md) を参照。

### 4.3 モジュールレジストリ

`M2CModuleManifest` の仕様とモジュール検出メカニズムについては [modules.md §Registry](../modules.md) を参照。

---

## 5. コンテキストスキーマ (MediaContext)

### 5.1 四つの柱

Write/Select/Compress/Isolate モデルを適用したもの。完全な Context Supply Protocol は [supply.md](../supply.md) を参照。

| 柱 | プロトコル上の構成要素 |
|----|----------------------|
| **Write** | 分析パイプライン → MediaContext JSON |
| **Select** | Context Supply → JIT レイヤー選択 |
| **Compress** | Summary モジュール + 圧縮戦略 |
| **Isolate** | 対象者 / 時間 / モダリティ分離のためのコンテキストポリシー |

### 5.2 3レイヤー構造

```
L0: Tags      — tags, moods                          (~50 tokens)
L1: Summary   — summary, tags, colorPalette, speakers (~200 tokens)
L2: Scenes    — L1 + scenes[].summary, objects        (~500-1000 tokens)
L3: Full      — All fields including transcription     (~2000-5000 tokens)
```

### 5.3 コアスキーマ

```typescript
interface MediaContext {
  /** Schema version (date-based) */
  schemaVersion: string;  // e.g., "2026-04-01"
  /** Media type */
  mediaType: string;

  // ── L0: Tags ──
  tags: string[];
  moods: string[];  // Recommended vocabulary in appendix

  // ── L1: Summary ──
  summary: string;
  colorPalette?: string[];  // Hex codes
  speakers?: SpeakerInfo[];
  duration: number;

  // ── L2: Scenes (temporal media only) ──
  scenes?: SceneContext[];

  // ── L3: Full (modality-isolated) ──
  transcription?: TranscriptionSegment[];
  acoustic?: AcousticContext;
  visual?: VisualContext;

  // ── Extensions ──
  extensions?: Record<string, unknown>;

  // ── Meta ──
  confidence: number;
  analyzedModules: string[];
  precisionLevel: string;
  analyzedAt: string;  // ISO 8601

  // ── Intent (what the user wanted when analyzing) ──
  intent?: M2CIntent;

  // ── Context Policy (for Supply) ──
  contextPolicy?: ContextPolicy;
}
```

`extensions` はドメイン固有データのための自由形式オブジェクトである。実装は名前空間付きのキーを使用すべきである (SHOULD)（例: `"style.cutTempo"`, `"brand.guidelines"`）。

### 5.4 信頼度スコアリング

#### モジュール単位の信頼度

各モジュール出力には `confidence`（0.0〜1.0）が含まれる。算出方法はモジュール定義だが、入力品質を考慮しなければならない (MUST)。

#### 集約信頼度

```
confidence = Σ(module_confidence × module_weight) / Σ(module_weight)
```

デフォルトの重みは [modules.md](../modules.md) で定義されている。

#### 品質ゲート

| ゲート | 閾値 | アクション |
|--------|------|-----------|
| **Pass** | confidence >= 0.7 | コンテキストを受理 |
| **Review** | 0.4 <= confidence < 0.7 | ヒューマンレビューのためコンシューマに通知 |
| **Reject** | confidence < 0.4 | 失敗したモジュールのみ再分析 |

実装はコンシューマが閾値をカスタマイズできるようにすべきである (SHOULD)。

---

## 6. 能力ネゴシエーション

MCP のパターンに従い、M2C は接続時に能力ネゴシエーションを行う。

### 6.1 Provider の能力

```json
{
  "protocolVersion": "2026-04-01",
  "capabilities": {
    "modules": ["scene", "transcription", "visual", "color", "summary"],
    "mediaTypes": ["video", "image", "audio"],
    "precisionLevels": ["quick", "standard", "deep"],
    "maxDuration": 7200,
    "supply": {
      "contextLevels": ["L0", "L1", "L2", "L3"],
      "compression": true,
      "isolation": true
    },
    "extensions": ["style", "brand"]
  }
}
```

### 6.2 Consumer リクエスト

```json
{
  "protocolVersion": "2026-04-01",
  "requested": {
    "modules": ["scene", "transcription", "summary"],
    "precisionLevel": "standard",
    "contextLevel": "L2",
    "maxTokens": 1000
  }
}
```

### 6.3 ネゴシエーションフロー

1. Consumer がリクエストする能力を送信
2. Provider が提供可能な内容を応答
3. `protocolVersion` が一致しない場合、Provider は代替を提示してもよい (MAY)
4. 合意に至らない場合、エラーで接続を拒否

---

## 7. 分析エージェント (ContextAnalyzer)

実装は専用の分析エージェントを提供すべきである (SHOULD):

```
User: "Analyze this video"
  ↓
ContextAnalyzer Agent
  ├── 能力ネゴシエーション
  ├── コスト見積もり → ユーザーへ表示
  ├── ユーザー確認
  ├── モジュールのディスパッチ（可能な場合は並列）
  ├── 結果の集約
  ├── 品質ゲート
  ├── MediaContext の保存
  └── 分析後コールバック（実装定義）
```

### 7.1 再分析

品質ゲートの失敗またはユーザーリクエスト時:

1. 信頼度の低いモジュールを特定
2. 該当モジュールのみを再実行
3. 新しい結果を既存の MediaContext にマージ
4. 集約信頼度を再計算

---

## 8. セキュリティ

### 8.1 原則

メディアファイルにはプライベートまたはセンシティブなコンテンツが含まれることが多い。M2C は以下を義務付ける:

1. **認証 REQUIRED**: すべての M2C エンドポイントは認証を要求しなければならない (MUST)（MCP ではオプションだが、M2C では必須）
2. **同意**: ユーザーは明示的に分析を開始しなければならない (MUST)（同意なしにユーザーのメディアを自動分析してはならない）
3. **データローカリティ**: 実装はメディアデータがデバイスから外に出るかどうかを宣言しなければならない (MUST)
4. **アーティファクトのクリーンアップ**: 一時的なアーティファクト（キーフレームなど）は分析後にクリーンアップすべきである (SHOULD)

### 8.2 トランスポートオプション

M2C は3つのデプロイトポロジーをサポートする:

| トポロジー | 説明 | ユースケース |
|-----------|------|-------------|
| **Local** | モジュールが Consumer と同一マシン上で動作 | デフォルト。デスクトップアプリ、プライバシー重視のメディア |
| **LAN Distributed** | モジュールがローカルネットワーク上の別マシンで動作 | 軽量デバイスで編集しつつ、高性能マシンに重い分析をオフロード |
| **Cloud** | モジュールがリモートサーバー（API）で動作 | ローカルハードウェアが不十分な場合、または特殊なモデルの利用 |

#### Local（デフォルト）

```
[Consumer + Modules] — same process or stdio
```

モジュールは関数呼び出し、stdio、または Unix ソケット経由で起動される。ネットワークトラフィックは発生しない。認証は不要。

#### LAN Distributed

```
[Lightweight PC]              [Powerful PC]
 Editor App     ←── LAN ──→  M2C Analyzer
 (consumer)                    (module server)
```

- Consumer は **mDNS/Bonjour**（ゼロコンフィグ）または手動 IP 設定によって Analyzer ノードを検出する
- 通信はローカルネットワーク上で **HTTP + JSON-RPC** を使用する（MCP Streamable HTTP トランスポートと同様）
- メディアファイルはローカルネットワーク経由で転送するか、共有ファイルシステム（NAS、SMB）からアクセスする
- 認証: 共有シークレットまたはローカル限定トークンを使用すべきである (SHOULD)（完全な OAuth は不要）
- データローカリティ: メディアはローカルネットワーク内に留まる。明示的な同意なしにメディアを外部サーバーへ中継してはならない (MUST NOT)
- Consumer の能力ネゴシエーションにより、どのノードでどのモジュールが利用可能かを検出する

**典型的なセットアップ:**

1. 高性能マシンに M2C Analyzer をインストール
2. Analyzer が mDNS 経由で可用性をブロードキャスト: `_m2c._tcp.local`
3. Consumer が自動検出し表示: 「分析サーバーを検出: Desktop-PC (RTX 4090)」
4. ユーザーがオフロードするモジュールを選択

#### Cloud

```
[Consumer] ←── HTTPS ──→ [Cloud M2C Provider]
```

- HTTPS REQUIRED
- 完全な認証（OAuth 2.1 または API キー）
- メディアは Provider にアップロードされる
- 実装はデータローカリティポリシーを宣言しなければならない (MUST)

### 8.3 トランスポートセキュリティ

- **Cloud**: HTTPS REQUIRED
- **LAN Distributed**: HTTPS RECOMMENDED、信頼されたネットワークでは HTTP も許容
- **Local**: トランスポートセキュリティは不要（stdio、Unix ソケット）
- 認証情報を含む Media URI はログに記録してはならない (MUST NOT)

---

## 9. エラーハンドリング

### 9.1 エラーモデル

JSON-RPC の慣例に従い、2つのレイヤーで構成される:

1. **プロトコルエラー**: 無効なリクエスト、未サポートのバージョン、認証失敗
2. **モジュールエラー**: 分析失敗、タイムアウト、未サポートのメディア形式

### 9.2 エラーコード

| コード | 意味 |
|--------|------|
| `-32600` | 無効なリクエスト |
| `-32601` | 不明なモジュール |
| `-32602` | 無効なパラメータ |
| `-32603` | 内部エラー |
| `-32700` | パースエラー |
| `-33001` | メディアが見つからない（URI に到達不可） |
| `-33002` | 未サポートのメディアタイプ |
| `-33003` | 分析タイムアウト |
| `-33004` | 品質ゲート失敗 |
| `-33005` | コスト上限超過 |

### 9.3 進捗通知

長時間の分析に対して、Provider は進捗通知を送信すべきである (SHOULD):

```json
{
  "moduleId": "transcription",
  "progress": 0.65,
  "total": 1.0,
  "message": "Processing audio segments..."
}
```

---

## 10. バージョニング

- プロトコルバージョン: 日付ベース（`YYYY-MM-DD`）
- スキーマバージョン: `MediaContext.schemaVersion` 内で同じく日付ベース形式
- モジュールバージョン: 独立した semver
- Extension: 独立したバージョニング
- 後方互換性: コンシューマは不明なフィールドを無視しなければならない (MUST)

### 10.1 拡張の原則

MCP の拡張設計に従う:

1. **Optional**: Extension はコア機能に必須であってはならない (MUST NOT)
2. **Additive**: Extension はコアの動作を追加するのみで、削除や変更をしてはならない (MUST NOT)
3. **Composable**: 複数の Extension は独立して動作しなければならない (MUST)
4. **Independently versioned**: 各 Extension は独自のバージョンを持つ

---

## 付録 A: 推奨ムードボキャブラリ

実装は `moods` フィールドに以下の用語を使用すべきである (SHOULD):

**Energy**: `energetic`, `calm`, `intense`, `relaxed`, `dynamic`
**Tone**: `bright`, `dark`, `warm`, `cool`, `neutral`
**Emotion**: `happy`, `sad`, `dramatic`, `romantic`, `mysterious`, `humorous`
**Style**: `professional`, `casual`, `cinematic`, `minimal`, `vintage`, `modern`

`x-` プレフィックスを用いてカスタムムードを追加してもよい (MAY): `"x-nostalgic"`。

---

## 付録 B: 関連仕様

| 仕様 | 関係 |
|------|------|
| [MCP](https://modelcontextprotocol.io) | モジュールトランスポートの互換性 |
| [IPTC 2025.1](https://iptc.org/standards/photo-metadata/) | AI 生成ラベル（補完的） |
| [C2PA](https://c2pa.org/) | コンテンツの来歴証明（補完的） |
| [XMP](https://developer.adobe.com/xmp/docs/) | ファイル埋め込みメタデータキャリア |
| [Schema.org](https://schema.org/) | Web ボキャブラリマッピング |
| [JSON Schema 2020-12](https://json-schema.org/) | スキーマ定義フォーマット |
