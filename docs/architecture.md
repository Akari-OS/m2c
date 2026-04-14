# M2C × AKARI OS — 統合アーキテクチャ

> **このドキュメントの位置づけ**: M2C プロトコルが AKARI OS エコシステムでどう実装・消費されるかを示す統合ガイド。
> プロトコル仕様は `spec/v0.2/` を参照。本ドキュメントは **M2C と AKARI OS の橋渡し** を担う。
>
> - 🌐 M2C プロトコル仕様: [`../spec/v0.2/protocol.md`](../spec/v0.2/protocol.md)
> - 🏛 AKARI OS Hub: [`Akari-OS`](https://github.com/Akari-OS)

---

## 1. M2C は AKARI OS の「意味層」

AKARI OS は「個人のための AI OS」。その中で M2C は、**メディアに意味を与える Layer 2（意味層）** を担う。

```
┌──────────────────────────────────────────────────────┐
│  Layer 4: Agents                                      │
│    Partner / Writer / Analyst / Planner / ...          │
│                                                        │
│    ↓ MCP tools で Pool / Amp / M2C を呼ぶ              │
├──────────────────────────────────────────────────────┤
│  Layer 3: Shell & Modules                             │
│    AKARI Shell（ランチャ）                              │
│    Video / Writer / Explorer / Voice / ...              │
│                                                        │
│    ↓ Pool に問い合わせ、MediaContext を消費             │
├──────────────────────────────────────────────────────┤
│  Layer 2: 意味層 ← M2C                                 │
│    Analyzer (Pool内) → MediaContext JSON               │
│    スキーマ・モジュール IF・Supply プロトコル           │
│                                                        │
│    ↓ Pool に格納                                        │
├──────────────────────────────────────────────────────┤
│  Layer 1: ストア層 ← Pool                              │
│    SQLite + MCP サーバー                                │
│    items / contexts / relationships                     │
│                                                        │
│    ↓ ファイルシステムから取り込み                        │
├──────────────────────────────────────────────────────┤
│  Layer 0: ファイル                                     │
│    .mp4 / .pdf / .png / .mp3 / ...                     │
└──────────────────────────────────────────────────────┘
```

M2C は「Layer 0（生ファイル）を Layer 1（Pool）に取り込むときに、意味を付与するプロトコル」と見ることもできる。

---

## 2. Pool と M2C の関係

### 2.1 Pool は M2C の Reference Implementation の一つ

[Pool](https://github.com/Akari-OS/pool) は AKARI OS の **汎用 Knowledge Store**。
Rust + SQLite + MCP で構築されており、内部に **Analyzer 群** を持つ。

Pool の Analyzer は **M2C Module 仕様に準拠** する形で設計されている：

| Pool Analyzer | M2C Module 対応 | 生成する MediaContext |
|---|---|---|
| `analyzers/video/` | `scene` + `transcription` + `visual` + `summary` | `mediaType: "video"` |
| `analyzers/audio/` | `transcription` + `acoustic` + `summary` | `mediaType: "audio"` |
| `analyzers/image/` | `visual` + `color` + `summary` | `mediaType: "image"` |
| `analyzers/pdf/` | `text_extract` + `layout` + `summary` | `mediaType: "document"` |
| `analyzers/office/` | `text_extract` + `slide_extract` + `summary` | `mediaType: "presentation"` / `"spreadsheet"` |
| `analyzers/article/` | `text_extract` + `summary` | `mediaType: "document"` |
| `analyzers/code/` | `text_extract` + 構造解析 | `mediaType: "document"` (code subtype) |

### 2.2 フロー：ファイル投入 → MediaContext 生成 → 格納

```
User / Shell Module
  │
  │ 1. Pool に "item add <file>" を送信（MCP or CLI）
  ▼
Pool-Core
  │
  │ 2. ファイル拡張子 → mediaType 判定
  │ 3. 対応する Analyzer を dispatch（M2C §3.2 準拠）
  ▼
Analyzer（Pool の analyzers/*/）
  │
  │ 4. M2C Module IF（§4.1）で analyze() を実行
  │    - M2CIntent があれば反映
  │    - 必要なモジュールだけ並列実行
  │    - confidence を算出
  ▼
Aggregator
  │
  │ 5. 結果をマージ → MediaContext JSON（§5）
  │ 6. Quality Gate（§5.4）
  ▼
Pool-Core (SQLite)
  │
  │ 7. items テーブル + contexts テーブルに格納
  │    - L0/L1/L2/L3 それぞれ別カラム or jsonb で保存
  ▼
Ready for Supply
```

### 2.3 Pool Schema と M2C の対応

Pool の `items` テーブルは M2C MediaContext のラッパー：

```sql
-- Pool schema (v1) の抜粋イメージ
items (
  id UUID,
  media_type TEXT,        -- M2C: mediaType
  uri TEXT,                -- M2C: mediaUri
  analyzed_at TIMESTAMP,   -- M2C: analyzedAt
  confidence REAL,         -- M2C: aggregate confidence
  precision_level TEXT,    -- M2C: precisionLevel
  l0_tags JSON,            -- M2C: L0（tags + moods）
  l1_summary JSON,         -- M2C: L1（summary + palette + speakers）
  l2_scenes JSON,          -- M2C: L2（scenes[]）
  l3_full JSON,            -- M2C: L3（transcription + full visual 等）
  extensions JSON          -- M2C: extensions
)
```

> 正式な Pool schema は `Akari-OS/pool/schema/v1.sql` を参照。

---

## 3. Shell / Modules の M2C 消費パターン

AKARI Shell 配下のモジュール（Video / Writer / Explorer 等）は、**直接 Analyzer を呼ばない**。
必ず **Pool MCP 経由で MediaContext を取得** する。

### 3.1 Context Supply 原則（M2C §5.1）

モジュールは「必要なレベルだけ取る」：

| シナリオ | 要求する Context Level | トークン目安 |
|---|---|---|
| ファイル一覧プレビュー | L0（Tags） | ~50 |
| カード・サムネ表示 | L1（Summary） | ~200 |
| タイムライン・編集画面 | L2（Scenes） | ~500–1000 |
| AI 深堀り・詳細解析 | L3（Full） | ~2000–5000 |

### 3.2 例：AKARI Video が動画を開くとき

```
[User] 動画を Video アプリで開く
  │
  ▼
[Video Module]
  │ 「この動画の L2 コンテキストを、予算 800 トークンで」
  ▼
[Pool MCP] pool_get_context(item_id, level="L2", max_tokens=800)
  │
  │ Pool は既に Analyzer で解析済みの MediaContext を Supply Protocol に従って返す
  │ - Select: L2 scenes[] を選択
  │ - Compress: 予算 800 tokens に収まるよう scene summary を圧縮
  │ - Isolate: transcription は除外（L3 の範囲）
  ▼
[Video Module] scenes[] を受け取り、タイムラインに描画
```

### 3.3 例：Writer Agent が文脈を引く

```
[Partner Agent] がユーザーと対話中
  │ "先週の取材の動画、要点まとめて" という発話を受け取る
  ▼
[Partner]
  │ 1. Pool MCP で検索: pool_search("取材", media_type="video", created_after=-7d)
  │ 2. ヒットした item に対し pool_get_context(..., level="L1") で要約取得
  │ 3. L1 summary をプロンプトに注入して応答生成
```

---

## 4. 7 Agents と MediaContext

AKARI OS の 7 エージェント（Partner / Writer / Analyst / Planner / Designer / Coder / Curator）は、
いずれも **MediaContext を Pool MCP 経由で参照** する。

| Agent | M2C 消費パターン |
|---|---|
| **Partner** | L0–L1 中心。会話中の軽量文脈参照 |
| **Writer** | L1–L2。記事・台本作成時にシーン要約を引く |
| **Analyst** | L3。全文 transcription や詳細データを読む |
| **Planner** | L0–L1 + 関係グラフ。項目間の relationship を辿る |
| **Designer** | L1 + `colorPalette`。既存素材のトーンを揃える |
| **Coder** | L1–L3。code 系 item の構造解析結果を使う |
| **Curator** | L0 中心 + relationship。大量の item を俯瞰する |

**Intent の伝達**: エージェントが Pool に問い合わせる際、`M2CIntent` を送れば、
  Pool は該当 item が未解析なら Intent 反映で再解析（または解析スケジュール）を返せる。

---

## 5. MCP ツール化 — Pool Phase 6 との統合

Pool は Phase 6 で **11 個の MCP ツール** を公開予定（`Akari-OS/pool/crates/pool-mcp/`）。
これらはすべて **M2C を暗黙に経由** する：

| Pool MCP Tool | M2C との関係 |
|---|---|
| `pool_add_item` | ファイル投入 → Analyzer 起動 → M2C MediaContext 生成 |
| `pool_get_item` | item メタ情報（mediaType / analyzedAt / confidence） |
| `pool_get_context` | **M2C MediaContext を Level 指定で返す（最重要）** |
| `pool_search` | L0 tags / L1 summary を全文検索 |
| `pool_list_items` | item 一覧。L0 のみ返す（軽量） |
| `pool_update_item` | メタ情報更新（再解析トリガも含む） |
| `pool_delete_item` | item 削除 + 関連 context 削除 |
| `pool_add_relationship` | item 間の関係（M2C Core には無い、Pool 拡張） |
| `pool_get_relationships` | 関係グラフ取得 |
| `pool_analyze` | **既存 item の再解析（M2C §7.1 Re-Analysis）** |
| `pool_estimate` | 解析コスト見積（M2C §4.1 `estimate()`） |

### 5.1 module manifest と MCP tool schema の対応

M2C は `M2CModuleManifest`（`spec/v0.2/modules.md`）でモジュールを記述する。
Pool 側ではこれを **MCP tool schema に変換** して公開できる：

```
M2CModuleManifest                 MCP Tool Schema
─────────────────                 ───────────────
id: "transcription"         →     name: "m2c_transcription"
supportedTypes: ["video",    →    inputSchema.mediaType.enum
                 "audio"]
analyze(input)              →     tool.call (handler)
estimate(input)             →     tool.call (estimate mode)
dependencies: []            →     annotations.dependencies
```

これにより、**個別 M2C モジュールそのものも MCP サーバーとして公開可能** になる。
Pool 自身を使わず、単体 Analyzer を MCP で配布するパターンに対応できる。

### 5.2 LAN Distributed との組み合わせ

M2C §8.2 の LAN Distributed トポロジと組み合わせると：

```
[ノート PC: Shell + Pool (lightweight)]
     │
     │ MCP over Streamable HTTP
     ▼
[デスクトップ PC: M2C Analyzer Cluster]
     - mDNS 公開: _m2c._tcp.local
     - GPU 重いモデル（WhisperX large / BLIP-2 等）
     - 共有シークレットで認証
     - NAS 経由で素材アクセス
```

この構成で、**ノート PC で編集しながら重い解析はデスクトップに委譲** できる。

---

## 6. AMP との関係（補完）

[AMP (Agent Memory Protocol)](https://github.com/Akari-OS/amp) は「エージェントの記憶」を扱うプロトコル。
M2C と AMP は **補完関係**：

| プロトコル | 扱うもの | 例 |
|---|---|---|
| **M2C** | メディア（ファイル）の **意味** | 動画の構成・PDF の内容 |
| **AMP** | エージェント間の **体験・判断** | 「前回の相談で決めた方針」 |

- M2C の MediaContext は「モノに付与する記憶」
- AMP のメモリは「コトに付与する記憶」
- 両方を Pool に格納できる（Pool は両プロトコルのストア実装）

---

## 7. 実装チェックリスト

M2C を AKARI OS の中で新規に実装するときのチェックリスト：

- [ ] **Analyzer を書く場合**
  - [ ] `M2CModule` IF（`spec/v0.2/protocol.md §4.1`）を満たす
  - [ ] `analyze()` と `estimate()` を両方実装
  - [ ] confidence を必ず返す
  - [ ] `dependencies` を正確に宣言
  - [ ] Pool の `analyzers/<type>/` 規約に沿った配置
- [ ] **Consumer（Shell Module / Agent）を書く場合**
  - [ ] Pool MCP 経由で `pool_get_context(level=...)` を呼ぶ
  - [ ] 必要最小限の level で要求する（L0 で済むなら L3 を要求しない）
  - [ ] `max_tokens` を指定してトークン予算を守る
- [ ] **新モジュールをコアに追加したい場合**
  - [ ] まず `spec/v0.2/modules.md` に Module Registry エントリを追記する PR
  - [ ] `schema.json` の必要箇所を更新
  - [ ] Pool に参考実装を追加（別 PR でも可）

---

## 8. 関連ドキュメント

- M2C プロトコル: [`../spec/v0.2/protocol.md`](../spec/v0.2/protocol.md)
- M2C モジュール: [`../spec/v0.2/modules.md`](../spec/v0.2/modules.md)
- M2C Supply: [`../spec/v0.2/supply.md`](../spec/v0.2/supply.md)
- M2C Interop: [`../spec/v0.2/interop.md`](../spec/v0.2/interop.md)
- Pool アーキテクチャ: [`Akari-OS/pool/docs/design/architecture.md`](https://github.com/Akari-OS/pool/blob/main/docs/design/architecture.md)
- Pool Analyzer Plugin: [`Akari-OS/pool/docs/design/analyzer-plugin.md`](https://github.com/Akari-OS/pool/blob/main/docs/design/analyzer-plugin.md)
- AKARI Video × Pool 統合: [`Akari-OS/pool/docs/design/akari-video-integration.md`](https://github.com/Akari-OS/pool/blob/main/docs/design/akari-video-integration.md)
