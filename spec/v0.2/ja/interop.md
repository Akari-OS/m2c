# M2C 相互運用仕様 v0.2

> ステータス: ドラフト
> 日付: 2026-04-01

## 1. 概要

M2C (Media to Context) は、既存のメディアメタデータエコシステムにおける空白を埋めるものである。現行の標準規格は AI 生成コンテンツのラベリングや来歴追跡に対応しているが、**AI 分析結果** ── メディアファイルの中身に対する構造化された理解 ── を記述するスキーマは存在しない。

### 1.1 3つの標準規格が描く全体像

```
┌─────────────────────────────────────────────────────────┐
│              AI Media Metadata Ecosystem                 │
│                                                         │
│  ┌───────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │  IPTC 2025.1  │  │    C2PA      │  │     M2C      │ │
│  │               │  │              │  │              │ │
│  │  AI GENERATED │  │ AI PROVENANCE│  │ AI ANALYSIS  │ │
│  │  "Was this    │  │ "Who made    │  │ "What is     │ │
│  │   made by AI?"│  │  this, when, │  │  inside this │ │
│  │               │  │  and how?"   │  │  media?"     │ │
│  └───────┬───────┘  └──────┬───────┘  └──────┬───────┘ │
│          │                 │                  │         │
│          └─────────┬───────┘                  │         │
│                    │                          │         │
│          Existing Standards            NEW: M2C v0.2   │
│          (creation metadata)       (understanding meta) │
└─────────────────────────────────────────────────────────┘
```

### 1.2 補完関係

| 標準規格 | 回答する問い | M2C との関係 |
|----------|-------------|-------------|
| **IPTC 2025.1** | 「これは AI で作成・編集されたか？」 | M2C 分析エンジンのメタデータは `AI System Used` にマッピングできる。M2C の tags/summary は IPTC の記述フィールドに反映できる。 |
| **C2PA** | 「来歴の連鎖はどうなっているか？」 | M2C の分析は C2PA のアクションアサーションとして記録される。MediaContext はカスタムアサーションとして埋め込んでもよい（MAY）。 |
| **Schema.org** | 「検索エンジンはこれをどう理解すべきか？」 | M2C のフィールドは Schema.org の語彙にマッピングされ、構造化データ / JSON-LD として利用できる。 |
| **MPEG-7** | 「マルチメディアコンテンツをどう記述するか？」 | M2C は MPEG-7 記述子に対する AI ネイティブな後継である。概念的なマッピングは存在するが、ランタイム相互運用はない。 |

### 1.3 適合性に関する用語

本文書における "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", "MAY" は [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) の定義に従って解釈される。

---

## 2. XMP カスタム名前空間

### 2.1 名前空間の定義

M2C は、分析結果をメディアファイルに埋め込むための XMP カスタム名前空間を定義する。

| プロパティ | 値 |
|-----------|-----|
| **Namespace URI** | `https://m2c-protocol.org/ns/v0.2/` |
| **推奨プレフィックス** | `m2c` |
| **Schema File** | `https://m2c-protocol.org/schema/v0.2/m2c.xmp` |

### 2.2 XMP プロパティマッピング

以下の表は MediaContext のフィールドから XMP プロパティへのマッピングを示す。実装は M2C データを XMP に書き込む際にこれらのプロパティ名を使用しなければならない（MUST）。

| MediaContext フィールド | XMP プロパティ | XMP 型 | 備考 |
|----------------------|--------------|--------|------|
| `schemaVersion` | `m2c:SchemaVersion` | Integer | 必須（MUST） |
| `mediaType` | `m2c:MediaType` | Closed Choice (`video`, `image`, `audio`) | 必須（MUST） |
| `summary` | `m2c:Summary` | Lang Alt | `xml:lang` による多言語対応 |
| `tags` | `m2c:Tags` | Bag Text | 順序なし集合 |
| `mood` | `m2c:Mood` | Text | |
| `colorPalette` | `m2c:ColorPalette` | Seq Text | 16進カラーコードの順序付きリスト |
| `duration` | `m2c:Duration` | Real | 秒単位 |
| `confidence` | `m2c:Confidence` | Real | 0.0〜1.0 |
| `precisionLevel` | `m2c:PrecisionLevel` | Closed Choice (`quick`, `standard`, `deep`) | |
| `analyzedAt` | `m2c:AnalyzedAt` | Date | ISO 8601 |
| `analyzedModules` | `m2c:AnalyzedModules` | Bag Text | モジュール ID |
| `speakers` | `m2c:Speakers` | Bag Struct | セクション 2.3 参照 |
| `scenes` | `m2c:Scenes` | Seq Struct | セクション 2.3 参照 |

複合フィールド（`transcription`, `acoustic`, `visual`, `style`）はサイズ制約のため XMP に埋め込むべきではない（SHOULD NOT）。これらのフィールドはサイドカー JSON ファイル（セクション 7）に格納しなければならない（MUST）。

### 2.3 構造体定義

**m2c:Speaker**

| フィールド | XMP プロパティ | 型 |
|-----------|--------------|-----|
| `id` | `m2c:SpeakerId` | Text |
| `label` | `m2c:SpeakerLabel` | Text |
| `ratio` | `m2c:SpeakerRatio` | Real |

**m2c:Scene**

| フィールド | XMP プロパティ | 型 |
|-----------|--------------|-----|
| `index` | `m2c:SceneIndex` | Integer |
| `start` | `m2c:SceneStart` | Real |
| `end` | `m2c:SceneEnd` | Real |
| `summary` | `m2c:SceneSummary` | Text |
| `mood` | `m2c:SceneMood` | Text |
| `confidence` | `m2c:SceneConfidence` | Real |

### 2.4 ファイル埋め込みパターン

| フォーマット | 方式 | サポートレベル |
|------------|------|--------------|
| **JPEG** | APP1 マーカー内の XMP パケットに埋め込み | 完全 ── サポート必須（MUST） |
| **PNG** | キーワード `XML:com.adobe.xmp` の `iTXt` チャンクに埋め込み | 完全 ── サポート必須（MUST） |
| **PDF** | XMP メタデータストリームに埋め込み | 完全 ── サポートすべき（SHOULD） |
| **MP4** | `uuid` ボックスに埋め込み（XMP UUID: `BE7ACFCB97A942E89C71999491E3AFAC`） | 部分的 ── セクション 2.5 参照 |
| **WebM** | XMP 仕様で未対応 | なし ── サイドカーのみ |
| **WAV** | `_PMX` チャンクに埋め込み | 部分的 ── サポートすべき（SHOULD） |
| **FLAC** | `VORBIS_COMMENT` ブロックに埋め込み | 部分的 ── サポートしてもよい（MAY） |

### 2.5 MP4 / 動画の制約

動画ファイルへの XMP 埋め込みには既知の制約がある:

1. **非標準の XMP ボックス**: ISO 14496-12（MP4）仕様は標準的な XMP コンテナを定義していない。`uuid` ボックスによるアプローチは Adobe の慣例であり、普遍的にサポートされているわけではない。
2. **ファイルサイズへの影響**: 動画ファイルはサイズが大きいため、XMP の追加による相対的な影響は無視できる。ただし、XMP を挿入するためにコンテナの再 mux が必要になる場合（MAY）、ファイル全体の書き直しが求められる。
3. **ストリーミングとの非互換**: MP4 に埋め込まれた XMP は HTTP ストリーミング（HLS/DASH）中にアクセスできない。
4. **ツール対応**: ExifTool は MP4 内の XMP の読み書きに対応している。FFmpeg はネイティブでは XMP を保持しない。

**推奨**: 動画・音声ファイルについては、サイドカー JSON（セクション 7）を優先すべきであり（SHOULD）、XMP 埋め込みはオプションとして扱う。

### 2.6 例: XMP パケット

```xml
<?xpacket begin="﻿" id="W5M0MpCehiHzreSzNTczkc9d"?>
<x:xmpmeta xmlns:x="adobe:ns:meta/">
  <rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#">
    <rdf:Description
      xmlns:m2c="https://m2c-protocol.org/ns/v0.2/"
      m2c:SchemaVersion="1"
      m2c:MediaType="video"
      m2c:Summary="Cooking vlog. 3 recipes demonstrated in a bright kitchen."
      m2c:Mood="bright, casual"
      m2c:Duration="342.5"
      m2c:Confidence="0.92"
      m2c:PrecisionLevel="standard"
      m2c:AnalyzedAt="2026-04-01T12:00:00Z">
      <m2c:Tags>
        <rdf:Bag>
          <rdf:li>cooking</rdf:li>
          <rdf:li>vlog</rdf:li>
          <rdf:li>recipe</rdf:li>
        </rdf:Bag>
      </m2c:Tags>
      <m2c:ColorPalette>
        <rdf:Seq>
          <rdf:li>#F5E6D3</rdf:li>
          <rdf:li>#2C5F2D</rdf:li>
          <rdf:li>#E8A87C</rdf:li>
        </rdf:Seq>
      </m2c:ColorPalette>
      <m2c:AnalyzedModules>
        <rdf:Bag>
          <rdf:li>scene</rdf:li>
          <rdf:li>transcription</rdf:li>
          <rdf:li>visual</rdf:li>
          <rdf:li>color</rdf:li>
          <rdf:li>summary</rdf:li>
        </rdf:Bag>
      </m2c:AnalyzedModules>
    </rdf:Description>
  </rdf:RDF>
</x:xmpmeta>
<?xpacket end="w"?>
```

---

## 3. IPTC マッピング

### 3.1 IPTC Core / Extension プロパティマッピング

M2C の記述フィールドは IPTC 写真メタデータプロパティにマッピングできる。これにより、M2C で分析されたメディアが IPTC 対応ツール（DAM システム、ストックフォトプラットフォーム、CMS）から検出可能になる。

| MediaContext フィールド | IPTC プロパティ | IPTC スキーマ | マッピング品質 | 備考 |
|----------------------|---------------|-------------|--------------|------|
| `summary` | `Description` (dc:description) | IPTC Core | 直接 | M2C summary → IPTC Description。AI 生成であることを示すため、実装は `[M2C]` プレフィックスを付けるべきである（SHOULD）。 |
| `tags` | `Keywords` (dc:subject) | IPTC Core | 直接 | 1:1 マッピング。 |
| `mood` | — | — | マッピングなし | IPTC に mood フィールドは存在しない。M2C XMP 拡張（セクション 2）を使用する。 |
| `speakers[].label` | `Person Shown in the Image` (Iptc4xmpExt:PersonInImage) | IPTC Extension | 部分的 | 話者に人間が読めるラベルがある場合のみマッピング可能。話者 ID のみでは不十分。 |
| `scenes[].summary` | — | — | マッピングなし | IPTC にシーンレベルの構造は存在しない。M2C XMP を使用する。 |
| `colorPalette` | — | — | マッピングなし | IPTC にカラーメタデータは存在しない。M2C XMP を使用する。 |
| `analyzedAt` | `Date Created` (photoshop:DateCreated) | IPTC Core | 意味的不一致 | `analyzedAt` は分析のタイムスタンプであり、メディアの作成日ではない。実装は既存の `Date Created` を上書きしてはならない（MUST NOT）。代わりに `m2c:AnalyzedAt` を使用する。 |
| `mediaType` | — | — | 暗黙的 | IPTC はメディアタイプをメタデータからではなくファイル形式から判断する。 |

### 3.2 IPTC 2025.1 AI プロパティ

IPTC 2025.1 は AI 生成・AI 編集コンテンツに関するプロパティを導入した。M2C の分析メタデータは以下のようにマッピングされる:

| IPTC 2025.1 プロパティ | M2C マッピング | 備考 |
|-----------------------|--------------|------|
| `digitalsourcetype` | `trainedAlgorithmicMedia` または `compositeSynthetic` | ソースメディアを記述するものであり、M2C の分析ではない。M2C はこのフィールドを設定しない ── コンテンツ制作者の責任である。 |
| `AI System Used` (Iptc4xmpExt:AISystemUsed) | M2C 分析エンジンの識別子 | 実装は M2C のモジュールチェーンを記録すべきである（SHOULD）（例: `M2C/v0.2 scene:pyscenedetect/1.0 summary:llm/gpt-4o`）。 |
| `AI Training Data` | — | M2C の分析はトレーニングに該当しない。対象外。 |

**重要な区別**: IPTC 2025.1 はコンテンツが AI によって**作成または編集**されたかどうかをラベリングする。M2C はコンテンツが AI によって**分析**されたことを記録する。これらは直交する関心事である。人間が撮影した写真（IPTC: `digitalCapture`）に M2C の分析メタデータ（M2C: `analyzedModules: [visual, color, summary]`）が付与されることは十分にあり得る（MAY）。

### 3.3 IPTC Video Metadata Hub

IPTC Video Metadata Hub（VMH）は動画アセットのメタデータを定義する。M2C のフィールドは VMH と以下のように関連する:

| VMH プロパティ | M2C フィールド | 備考 |
|--------------|--------------|------|
| `headline` | `summary`（短縮版） | VMH の headline は短い。M2C summary の最初の一文を使用する。 |
| `description` | `summary` | M2C summary の全文。 |
| `keyword` | `tags` | 直接マッピング。 |
| `videoLength` | `duration` | M2C は秒単位で格納; VMH は `HH:MM:SS:FF` を使用する。変換が必要。 |
| `personHeard` | `speakers[].label` | 部分的 ── ラベルが利用可能な場合のみ。 |
| `personShown` | （`visual` モジュールから） | `visual.objectFrequency["person"]` と `pose` モジュールのデータが必要。 |
| `shotType` | （`visual` モジュールから） | 将来の M2C モジュール（`framing`）で提供予定。v0.1 には含まれない。 |

---

## 4. C2PA 統合

### 4.1 C2PA アクションとしての M2C 分析

メディアファイルが M2C で分析された場合、その分析イベントはファイルの C2PA マニフェスト内に [C2PA Action Assertion](https://c2pa.org/specifications/specifications/2.1/specs/C2PA_Specification.html#_actions) として記録すべきである（SHOULD）。

**アクション定義**:

```json
{
  "action": "m2c.analyzed",
  "softwareAgent": "M2C/v0.2",
  "when": "2026-04-01T12:00:00Z",
  "parameters": {
    "m2c.moduleId": ["scene", "transcription", "visual", "color", "summary"],
    "m2c.precisionLevel": "standard",
    "m2c.confidence": 0.92,
    "m2c.schemaVersion": 1
  }
}
```

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| `m2c.moduleId` | array of strings | MUST | 実行されたモジュールの ID |
| `m2c.precisionLevel` | string | MUST | `quick`, `standard`, または `deep` |
| `m2c.confidence` | number | MUST | 集約信頼度（0.0〜1.0） |
| `m2c.schemaVersion` | integer | MUST | MediaContext のスキーマバージョン |

### 4.2 カスタムアサーション: MediaContext の埋め込み

C2PA マニフェスト内に完全な分析結果が必要なアプリケーション向けに、M2C はカスタムアサーションを定義する:

**アサーションラベル**: `m2c.context`

**アサーション内容**: 完全な MediaContext JSON オブジェクトを、C2PA の要件に従い CBOR エンコードしたもの。

```
Assertion label: "m2c.context"
Content type:    application/cbor
Content:         CBOR-encoded MediaContext JSON
```

実装は、サイズ制約に応じて完全な MediaContext またはサブセット（例: L0 または L1 のみ）を埋め込んでもよい（MAY）。アサーション内の `m2c:PrecisionLevel` は埋め込みデータの深度と一致しなければならない（MUST）。

**サイズの目安**:
- L0（tags + mood）: 約 200 バイト ── 埋め込むべき（SHOULD）
- L1（summary + tags + palette + speakers）: 約 1 KB ── 埋め込むべき（SHOULD）
- L2（+ scenes）: 約 5〜50 KB ── 埋め込んでもよい（MAY）
- L3（完全版）: 約 50 KB〜5 MB ── 埋め込むべきではない（SHOULD NOT）; サイドカー参照を使用する

### 4.3 来歴チェーンの記録

M2C 分析は来歴チェーンに新たなステップを作成する。分析とその後の編集との関係は保持されなければならない（MUST）:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Original    │     │  M2C         │     │  AI-Assisted  │
│  Media       │────→│  Analysis    │────→│  Edit         │
│              │     │              │     │              │
│  c2pa:       │     │  c2pa:       │     │  c2pa:       │
│  created     │     │  m2c.analyzed│     │  edited      │
│              │     │  m2c.context │     │  (uses M2C   │
│              │     │              │     │   context)   │
└──────────────┘     └──────────────┘     └──────────────┘
```

チェーン内の各 C2PA マニフェストは以下を含まなければならない（MUST）:
1. 前のマニフェストへの `parentOf` 関係
2. M2C 分析が行われた際の `m2c.analyzed` アクション
3. M2C コンテキストデータを参照してもよい（MAY）後続の編集アクション

### 4.4 トレーニング / データマイニングアサーション

C2PA はトレーニングおよびデータマイニングの許諾を宣言するアサーションを定義している。M2C の分析結果には特有の考慮事項がある:

| C2PA アサーション | M2C との関係 |
|-----------------|-------------|
| `c2pa.training-mining` | M2C 分析出力（MediaContext）は**派生著作物**である。元のメディアがトレーニングを禁止している場合（`c2pa.training-mining.allowed = false`）、M2C の分析結果も同じ制限を引き継ぐべきである（SHOULD）。 |
| `c2pa.ai_generated` | M2C 分析出力は AI 生成のメタデータであり、AI 生成のメディアではない。実装はこれらのケースを区別すべきである（SHOULD）。 |

**ルール**: M2C モジュールが `c2pa.training-mining` アサーションの `allowed = false` を持つメディアを分析した場合、結果の MediaContext JSON には `rights` フィールドを含めなければならない（MUST）:

```json
{
  "rights": {
    "trainingMiningAllowed": false,
    "source": "c2pa:inherited"
  }
}
```

---

## 5. Schema.org マッピング

### 5.1 MediaContext から Schema.org 語彙へ

M2C のフィールドは [Schema.org](https://schema.org/) の型およびプロパティにマッピングされ、検索エンジンやナレッジグラフ向けの構造化データとして利用できる。

| MediaContext フィールド | Schema.org 型 / プロパティ | 備考 |
|----------------------|--------------------------|------|
| （ルート, `mediaType: "video"`） | `VideoObject` | |
| （ルート, `mediaType: "image"`） | `ImageObject` | |
| （ルート, `mediaType: "audio"`） | `AudioObject` | |
| `summary` | `description` | |
| `duration` | `duration` | ISO 8601 duration に変換（例: `PT5M42S`） |
| `tags` | `keywords` | カンマ区切り文字列として結合 |
| `mood` | `additionalProperty` | `name: "mood"` の `PropertyValue` を使用 |
| `colorPalette` | `additionalProperty` | `name: "colorPalette"` の `PropertyValue` を使用 |
| `speakers[].label` | `actor` → `Person.name` | ラベルが人名の場合のみ |
| `scenes[]` | `hasPart` → `Clip` | 各シーンは `startOffset` と `endOffset` を持つ `Clip` にマッピング |
| `scenes[].summary` | `Clip.description` | |
| `transcription` | `transcript` | 連結したプレーンテキスト |
| `confidence` | `additionalProperty` | `name: "m2cConfidence"` の `PropertyValue` を使用 |
| `analyzedAt` | `dateModified` | 分析が実行された日付 |
| `precisionLevel` | `additionalProperty` | `name: "m2cPrecisionLevel"` の `PropertyValue` を使用 |

### 5.2 JSON-LD の例

```json
{
  "@context": [
    "https://schema.org",
    {
      "m2c": "https://m2c-protocol.org/ns/v0.2/"
    }
  ],
  "@type": "VideoObject",
  "name": "Cooking Vlog — 3 Easy Recipes",
  "description": "Cooking vlog. 3 recipes demonstrated in a bright kitchen.",
  "duration": "PT5M42S",
  "keywords": "cooking, vlog, recipe",
  "dateModified": "2026-04-01T12:00:00Z",
  "actor": [
    {
      "@type": "Person",
      "name": "Host"
    }
  ],
  "hasPart": [
    {
      "@type": "Clip",
      "name": "Scene 1",
      "description": "Opening. Host greets viewers.",
      "startOffset": 0.0,
      "endOffset": 45.2
    },
    {
      "@type": "Clip",
      "name": "Scene 2",
      "description": "Pasta recipe demonstration.",
      "startOffset": 45.2,
      "endOffset": 120.0
    }
  ],
  "additionalProperty": [
    {
      "@type": "PropertyValue",
      "name": "m2cConfidence",
      "value": 0.92
    },
    {
      "@type": "PropertyValue",
      "name": "m2cPrecisionLevel",
      "value": "standard"
    },
    {
      "@type": "PropertyValue",
      "name": "mood",
      "value": "bright, casual"
    }
  ],
  "m2c:schemaVersion": 1,
  "m2c:analyzedModules": ["scene", "transcription", "visual", "color", "summary"]
}
```

### 5.3 HTML への埋め込み

Web 上でメディアを配信する実装は、M2C から導出した Schema.org データをページに埋め込むべきである（SHOULD）:

```html
<script type="application/ld+json">
  { ... JSON-LD from Section 5.2 ... }
</script>
```

これにより、検索エンジンが AI 分析済みメディアをリッチな構造化データ付きでインデックスできるようになる。

---

## 6. MPEG-7 との関係

### 6.1 歴史的背景

MPEG-7（ISO/IEC 15938）は正式名称を "Multimedia Content Description Interface" といい、2001年にマルチメディアコンテンツ記述の決定的なフレームワークとして標準化された。Description Schemes（DS）、Descriptors（D）、および Description Definition Language（DDL）を定義する。

M2C は概念的に類似した領域を占めるが、根本的に異なる:

| 観点 | MPEG-7 | M2C |
|------|--------|-----|
| **時代** | Pre-AI（2001年） | AI ネイティブ（2026年） |
| **記述子** | 手作りの特徴量抽出器 | AI モデルの出力 |
| **スキーマ** | XML ベースの DDL | JSON Schema |
| **信頼度** | 固有の概念なし | 第一級の概念 |
| **モジュール性** | モノリシックな標準 | 独立したプラガブルモジュール |
| **普及度** | 低い（複雑すぎた） | シンプルさを重視した設計 |

### 6.2 概念的マッピング

以下の表は MPEG-7 記述子から M2C への対応を示す。これは MPEG-7 システムからの移行を行う実装者向けのリファレンスとして提供される。ランタイム相互運用は定義されていない。

| MPEG-7 記述子 / DS | M2C での相当物 | 備考 |
|-------------------|--------------|------|
| `ColorLayout` | `color` モジュール → `colorPalette` | MPEG-7 は DCT 係数を使用; M2C は 16進カラー値を使用 |
| `DominantColor` | `color` モジュール → `dominantColor` | |
| `ScalableColor` | `color` モジュール → `palette`（完全版） | |
| `EdgeHistogram` | （相当物なし） | M2C はエッジ特徴ではなく AI キャプショニングに依存 |
| `AudioSpectrumBasis` | `acoustic` モジュール → `bgmSegments` | MPEG-7 は低レベル; M2C はセマンティック |
| `AudioTempoType` | （将来の `rhythm` モジュール） | v0.1 には含まれない |
| `SpokenContent` | `transcription` モジュール | MPEG-7 は単語ラティスを定義; M2C はタイムスタンプ付きセグメントを使用 |
| `FaceRecognition` | `pose` モジュール → `face` | MPEG-7 は固有顔を使用; M2C はディープラーニングによるランドマークを使用 |
| `MovingRegion` | `scene` モジュール → scenes | MPEG-7 は任意の領域を追跡; M2C はシーン境界を追跡 |
| `SemanticBase` | `summary` モジュール | MPEG-7 は形式意味論を定義; M2C は LLM 生成の自然言語を使用 |
| `TextAnnotation` | `summary`, `tags` | |

### 6.3 移行ガイダンス

MPEG-7 から M2C に移行するシステムは以下を行うべきである（SHOULD）:

1. 既存の MPEG-7 記述子を最も近い M2C モジュールの出力にマッピングする
2. より高品質な AI ネイティブの結果を得るため、M2C モジュールでメディアを再分析する
3. MPEG-7 データをアーカイブ用メタデータとして保持する（破棄しない）
4. 移行期間中は両方の表現を維持するため、デュアルストレージパターン（セクション 7）を使用する

---

## 7. デュアルストレージパターン

### 7.1 アーキテクチャ

M2C は分析結果を保存する 2 つの補完的な保存先をサポートする:

```
┌─────────────────────┐     ┌──────────────────────────────┐
│  In-File (XMP)      │     │  Sidecar (JSON)              │
│                     │     │                              │
│  ✓ L0–L1 fields    │     │  ✓ Full MediaContext (L0–L3) │
│  ✓ Travels with    │     │  ✓ No file modification      │
│    the file         │     │  ✓ Unlimited size            │
│  ✗ Size-limited    │     │  ✗ Detachable from media     │
│  ✗ Not all formats │     │  ✗ Requires hash binding     │
└────────┬────────────┘     └──────────────┬───────────────┘
         │                                 │
         └──────────┬──────────────────────┘
                    │
          Content Hash Binding
          (SHA-256 of media file)
```

### 7.2 サイドカーファイルの命名規則

サイドカー JSON ファイルは以下の命名規則に従わなければならない（MUST）:

```
<original-filename>.m2c.json
```

例:
- `cooking-vlog.mp4` → `cooking-vlog.mp4.m2c.json`
- `portrait.jpg` → `portrait.jpg.m2c.json`

サイドカーファイルにはバインディング用の `source` フィールドを含めなければならない（MUST）:

```json
{
  "source": {
    "filename": "cooking-vlog.mp4",
    "sha256": "a1b2c3d4e5f6...64-char-hex-digest",
    "size": 104857600,
    "mimeType": "video/mp4"
  },
  "schemaVersion": 1,
  "mediaType": "video",
  "summary": "...",
  ...
}
```

### 7.3 コンテンツハッシュバインディング

`source.sha256` フィールドは、分析時点の元メディアファイルの SHA-256 ハッシュでなければならない（MUST）。

**検証プロセス**:

1. メディアファイルの SHA-256 を計算する
2. サイドカーの `source.sha256` と比較する
3. 一致: コンテキストはこのファイルに対して有効
4. 不一致: コンテキストは陳腐化 ── 再分析をトリガーすべき（SHOULD）

**エッジケース**:

| シナリオ | 動作 |
|---------|------|
| 分析後にメディアファイルが変更された | ハッシュ不一致 → 陳腐化コンテキスト。再分析のフラグを立てる。 |
| メディアファイルがリネームされた | ハッシュは一致 → コンテキストは有効。`source.filename` を更新する。 |
| メディアファイルがトランスコードされた（例: H.264 → H.265） | ハッシュ不一致 → 再分析が必要。一部のコンテキスト（summary, tags）は引き継いでもよい（MAY）。 |
| メディアファイルのないサイドカー | 孤立コンテキスト。実装は設定可能な期間後にガベージコレクションしてもよい（MAY）。 |

### 7.4 配信時のメタデータ喪失

メディアファイルが配信される際（ソーシャルプラットフォームへのアップロード、メッセージングでの送信、CDN によるトランスコード）、埋め込み XMP メタデータは頻繁にストリップされる。デュアルストレージパターンはこれを緩和する:

| 配信チャネル | XMP 保持？ | 緩和策 |
|------------|-----------|--------|
| ファイル直接転送 | はい | 対応不要 |
| クラウドストレージ（S3, GCS） | はい | 対応不要 |
| ソーシャルメディアへのアップロード | 通常いいえ | ダウンロード後に再分析するか、サーバーサイドでサイドカーを維持する |
| メッセージングアプリ | 通常いいえ | サイドカーファイルをメディアと共に送信する |
| CDN トランスコード | いいえ | オリジンにサイドカーを保存; 配信後に再アタッチする |
| メール添付 | 通常はい | 対応不要 |

**推奨**: 実装はサイドカー JSON を常にプライマリストレージとして維持しなければならない（MUST）。XMP 埋め込みは既存ツールとの相互運用のための利便性レイヤーである。

---

## 8. 互換性テストガイドライン

### 8.1 XMP 読み書きの検証

実装は [ExifTool](https://exiftool.org/) を使用して XMP の相互運用性を検証しなければならない（MUST）。

**M2C XMP の書き込み**:

```bash
exiftool \
  -XMP-m2c:SchemaVersion=1 \
  -XMP-m2c:MediaType=video \
  -XMP-m2c:Summary="Cooking vlog. 3 recipes." \
  -XMP-m2c:Mood="bright, casual" \
  -XMP-m2c:Confidence=0.92 \
  -XMP-m2c:PrecisionLevel=standard \
  -XMP-m2c:AnalyzedAt="2026-04-01T12:00:00Z" \
  cooking-vlog.mp4
```

**M2C XMP の読み取り**:

```bash
exiftool -XMP-m2c:all cooking-vlog.mp4
```

**期待される出力**:

```
Schema Version                  : 1
Media Type                      : video
Summary                         : Cooking vlog. 3 recipes.
Mood                            : bright, casual
Confidence                      : 0.92
Precision Level                 : standard
Analyzed At                     : 2026-04-01T12:00:00Z
```

**ExifTool 設定ファイル**（カスタム名前空間に必要）:

```perl
# m2c.exiftool.config
%Image::ExifTool::UserDefined = (
    'Image::ExifTool::XMP::Main' => {
        m2c => {
            SubDirectory => {
                TagTable => 'Image::ExifTool::UserDefined::m2c',
            },
        },
    },
);

%Image::ExifTool::UserDefined::m2c = (
    GROUPS => { 0 => 'XMP', 1 => 'XMP-m2c', 2 => 'Other' },
    NAMESPACE => { m2c => 'https://m2c-protocol.org/ns/v0.2/' },
    SchemaVersion => { Writable => 'integer' },
    MediaType     => { Writable => 'string' },
    Summary       => { Writable => 'string' },
    Mood          => { Writable => 'string' },
    Confidence    => { Writable => 'rational64s' },
    PrecisionLevel => { Writable => 'string' },
    AnalyzedAt    => { Writable => 'date' },
    Tags          => { Writable => 'string', List => 'Bag' },
    AnalyzedModules => { Writable => 'string', List => 'Bag' },
    ColorPalette  => { Writable => 'string', List => 'Seq' },
);
```

### 8.2 C2PA マニフェストの検証

C2PA マニフェストに M2C データを埋め込む実装は、[c2patool](https://github.com/contentauth/c2patool) を使用して検証しなければならない（MUST）:

**マニフェストの検査**:

```bash
c2patool cooking-vlog.mp4 --output manifest.json
```

**M2C アクションの存在確認**:

```bash
# The manifest.json should contain an action with:
# "action": "m2c.analyzed"
# "parameters" with m2c.moduleId, m2c.confidence, etc.
```

**カスタムアサーションの確認**:

```bash
# Look for assertion with label "m2c.context"
# Decode CBOR content and validate against MediaContext JSON Schema
```

**検証チェックリスト**:

- [ ] `m2c.analyzed` アクションが actions アサーションに存在する
- [ ] 必須パラメータ（`m2c.moduleId`, `m2c.precisionLevel`, `m2c.confidence`, `m2c.schemaVersion`）がすべて存在する
- [ ] `m2c.context` アサーション（存在する場合）が有効な MediaContext JSON にデコードされる
- [ ] デコードされた MediaContext が `schema.json` に対してバリデーションを通過する
- [ ] 来歴チェーンが完全である（親マニフェスト参照が有効）

### 8.3 JSON-LD の検証

M2C データから Schema.org JSON-LD を生成する実装は、出力を検証しなければならない（MUST）:

**Google Rich Results Test の使用**:

1. JSON-LD を HTML ページに埋め込む
2. [Rich Results Test](https://search.google.com/test/rich-results) に送信する
3. `VideoObject` / `ImageObject` / `AudioObject` に対してエラーがないことを確認する

**JSON-LD Playground の使用**:

1. [JSON-LD Playground](https://json-ld.org/playground/) に JSON-LD を貼り付ける
2. 展開形式で `m2c:` プレフィックス付きのすべての用語が解決されることを確認する
3. `@context` が `https://schema.org` と M2C 名前空間の両方を含むことを確認する

**プログラムによる検証**:

```bash
# Using jsonld.js CLI or equivalent
jsonld normalize --format application/n-quads input.jsonld
```

出力には M2C 固有のプロパティに対する未解決のブランクノードが含まれてはならない（MUST NOT）。

### 8.4 サイドカーの整合性検証

```bash
# 1. Compute media file hash
sha256sum cooking-vlog.mp4
# Output: a1b2c3d4e5f6... cooking-vlog.mp4

# 2. Compare with sidecar
jq -r '.source.sha256' cooking-vlog.mp4.m2c.json
# Output: a1b2c3d4e5f6...

# 3. Validate MediaContext against schema
npx ajv validate -s schema.json -d cooking-vlog.mp4.m2c.json
```

### 8.5 標準規格間の整合性

メディアファイルが IPTC メタデータと M2C データの両方を含む場合、実装は整合性を検証すべきである（SHOULD）:

| チェック項目 | 条件 | 重大度 |
|------------|------|--------|
| Keywords の一致 | IPTC `Keywords` ⊇ M2C `tags` | Warning |
| Description の存在 | M2C `summary` が存在する場合、IPTC `Description` が設定されている | Warning |
| AI System の記録 | IPTC `AI System Used` に M2C エンジン情報が含まれている | Warning |
| C2PA アクションの存在 | M2C サイドカーが存在する場合、C2PA マニフェストに `m2c.analyzed` が含まれている | Error |
| ハッシュバインディングの有効性 | サイドカーの `source.sha256` が実際のファイルと一致する | Error |
| スキーマバージョンの整合性 | XMP の `m2c:SchemaVersion` がサイドカーの `schemaVersion` と一致する | Error |

---

## 付録 A: 名前空間の登録

M2C XMP 名前空間（`https://m2c-protocol.org/ns/v0.2/`）は Adobe の XMP 名前空間レジストリには登録されていない。実装はすべての XMP パケットに名前空間宣言を含めなければならない（MUST）。Adobe のレジストリへの登録は M2C が v1.0 安定版に到達した時点で推進する。

## 付録 B: 今後の課題

以下の相互運用トピックは将来のバージョンに先送りされる:

| トピック | 対象バージョン | 説明 |
|---------|--------------|------|
| **Dublin Core マッピング** | v0.3 | IPTC がカバーする範囲を超える DC エレメントへの正式なマッピング |
| **EXIF 統合** | v0.3 | カメラ EXIF データを M2C 分析モジュールの入力として読み取る |
| **EBUCore マッピング** | v0.3 | 欧州放送連合のメタデータ標準へのマッピング |
| **ActivityPub / Fediverse** | v0.4 | ActivityPub の `attachment` 経由で M2C コンテキストを配信 |
| **W3C Web Annotation** | v0.4 | シーンレベルのアノテーションを W3C Web Annotations として表現 |
| **IIIF Presentation API** | v0.4 | M2C のシーンを IIIF Canvas / Annotation 構造にマッピング |
| **Adobe XMP SDK 統合** | v1.0 | Adobe XMP SDK でのネイティブ読み書きサポート |

## 付録 C: リファレンス表 ── 標準規格間のフィールドカバレッジ

| MediaContext フィールド | XMP (M2C) | IPTC | C2PA | Schema.org | MPEG-7 |
|----------------------|-----------|------|------|------------|--------|
| `schemaVersion` | `m2c:SchemaVersion` | — | `m2c.schemaVersion` param | `m2c:schemaVersion` | — |
| `mediaType` | `m2c:MediaType` | （暗黙的） | — | `@type` | — |
| `summary` | `m2c:Summary` | `dc:description` | `m2c.context` assertion | `description` | `TextAnnotation` |
| `tags` | `m2c:Tags` | `dc:subject` | `m2c.context` assertion | `keywords` | `TextAnnotation` |
| `mood` | `m2c:Mood` | — | `m2c.context` assertion | `additionalProperty` | — |
| `colorPalette` | `m2c:ColorPalette` | — | `m2c.context` assertion | `additionalProperty` | `DominantColor` |
| `duration` | `m2c:Duration` | VMH:`videoLength` | — | `duration` | — |
| `speakers` | `m2c:Speakers` | `PersonInImage`（部分的） | `m2c.context` assertion | `actor` | `SpokenContent` |
| `scenes` | `m2c:Scenes` | — | `m2c.context` assertion | `hasPart` → `Clip` | `MovingRegion` |
| `transcription` | （サイドカーのみ） | — | `m2c.context` assertion | `transcript` | `SpokenContent` |
| `acoustic` | （サイドカーのみ） | — | `m2c.context` assertion | — | `AudioSpectrumBasis` |
| `visual` | （サイドカーのみ） | — | `m2c.context` assertion | — | `ColorLayout` |
| `style` | （サイドカーのみ） | — | `m2c.context` assertion | — | — |
| `confidence` | `m2c:Confidence` | — | `m2c.confidence` param | `additionalProperty` | — |
| `analyzedModules` | `m2c:AnalyzedModules` | — | `m2c.moduleId` param | `m2c:analyzedModules` | — |
| `precisionLevel` | `m2c:PrecisionLevel` | — | `m2c.precisionLevel` param | `additionalProperty` | — |
| `analyzedAt` | `m2c:AnalyzedAt` | — | `when` (action) | `dateModified` | — |
