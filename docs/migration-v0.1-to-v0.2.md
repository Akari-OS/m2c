# M2C v0.1 → v0.2 Migration Guide

> **スコープ**: M2C v0.1 を実装・運用しているツール・システムが v0.2 へ対応するための実践ガイド。
> 
> **対象読者**:
> - M2C Analyzer の実装者（Reference Implementation など）
> - M2C を統合したアプリケーション（例: AKARI Pool）
> - M2C の Validator / Schema ツール
> - XMP / IPTC 等の相互運用ツール

---

## §1. 変更サマリ

v0.1 → v0.2 で 4 つの major breaking changes があります。これらは**自動互換できず、人間判断が必要**な項目も含みます。

| 項目 | v0.1 | v0.2 | 互換性 | 優先度 |
|---|---|---|---|---|
| `schemaVersion` フィールド型 | 整数 (`1`) | 文字列・日付 (`"2026-04-01"`) | 完全不互換 | P0 |
| `context` 構造 | 単層（flat JSON） | 4 層 L0–L3（nested） | 非互換 | P0 |
| Namespace URI | `m2c-protocol.org` | `github.com/Akari-OS/m2c` | URI 置換で対応 | P1 |
| Schema `$id` ベース URI | `https://m2c-protocol.org/schemas/vX.Y/` | `https://github.com/Akari-OS/m2c/schema/vX.Y/` | URI 置換で対応 | P1 |

---

## §2. `schemaVersion` フィールドの判定と分岐

最初のステップは受け取った JSON の `schemaVersion` を見て、v0.1 か v0.2 かを自動判定することです。

### 2.1 判定ロジック

#### JavaScript / TypeScript

```javascript
function detectM2CVersion(mediaContextJson) {
  const { schemaVersion } = mediaContextJson;
  
  // v0.1: schemaVersion は整数
  if (typeof schemaVersion === 'number') {
    return 'v0.1';
  }
  
  // v0.2: schemaVersion は ISO 8601 日付文字列 (YYYY-MM-DD)
  if (typeof schemaVersion === 'string' && /^\d{4}-\d{2}-\d{2}$/.test(schemaVersion)) {
    return 'v0.2';
  }
  
  throw new Error(
    `Unknown schemaVersion type or format: ${JSON.stringify(schemaVersion)}. ` +
    `Expected number (v0.1) or ISO 8601 date string (v0.2).`
  );
}

// 使用例
const version = detectM2CVersion(receivedJson);
if (version === 'v0.1') {
  const legacy = parseV01(receivedJson);
  const modern = upgradeV01ToV02(legacy);
  return processMediaContext(modern);
} else {
  return processMediaContext(receivedJson);
}
```

#### Python

```python
import re
from typing import Literal

def detect_m2c_version(media_context_json: dict) -> Literal["v0.1", "v0.2"]:
    """Detect M2C schema version from the JSON object."""
    schema_version = media_context_json.get('schemaVersion')
    
    # v0.1: integer
    if isinstance(schema_version, int):
        return 'v0.1'
    
    # v0.2: ISO 8601 date string
    if isinstance(schema_version, str) and re.match(r'^\d{4}-\d{2}-\d{2}$', schema_version):
        return 'v0.2'
    
    raise ValueError(
        f"Unknown schemaVersion type or format: {schema_version!r}. "
        f"Expected int (v0.1) or ISO 8601 date string 'YYYY-MM-DD' (v0.2)."
    )

# Usage
version = detect_m2c_version(received_json)
if version == 'v0.1':
    legacy = parse_v01(received_json)
    modern = upgrade_v01_to_v02(legacy)
    return process_media_context(modern)
else:
    return process_media_context(received_json)
```

### 2.2 新規デプロイ向けの推奨値

新しいシステムを構築する場合は、以下を推奨します：

```json
{
  "schemaVersion": "2026-04-01",
  "mediaType": "video",
  "summary": "...",
  "context": {
    "L0": { ... },
    "L1": { ... },
    "L2": { ... },
    "L3": { ... }
  }
}
```

---

## §3. `context` フィールド構造の変更

これが v0.1 → v0.2 の最大の変更です。単層の flat JSON が 4 層の nested 構造に変わります。

### 3.1 Before / After（最小例）

#### v0.1 の構造（単層）

```json
{
  "schemaVersion": 1,
  "mediaType": "video",
  "summary": "A person presenting on stage",
  "tags": ["presentation", "person", "stage"],
  "mood": "professional",
  "confidence": 0.85,
  "analyzedModules": ["scene-detector", "mood-analyzer"],
  "precisionLevel": "Standard",
  "analyzedAt": "2026-04-01T12:00:00Z",
  "context": {
    "title": "Presentation on Climate Change",
    "persons": [
      { "id": "person_1", "label": "Speaker", "confidence": 0.9 }
    ],
    "objects": [
      { "label": "microphone", "confidence": 0.8 },
      { "label": "projector screen", "confidence": 0.75 }
    ]
  }
}
```

#### v0.2 の構造（4 層 L0–L3）

```json
{
  "schemaVersion": "2026-04-01",
  "mediaType": "video",
  "summary": "A person presenting on stage",
  "tags": ["presentation", "person", "stage"],
  "moods": ["professional", "engaging"],
  "confidence": 0.85,
  "analyzedModules": ["scene-detector", "mood-analyzer"],
  "precisionLevel": "Standard",
  "analyzedAt": "2026-04-01T12:00:00Z",
  "context": {
    "L0": {
      "purpose": "presentation",
      "mediaClass": "video",
      "estimatedTokens": 50
    },
    "L1": {
      "title": "Presentation on Climate Change",
      "summary": "Professional presentation on climate change mitigation strategies",
      "estimatedTokens": 200
    },
    "L2": {
      "scenes": [
        {
          "timeRange": { "start": 0, "end": 120 },
          "description": "Speaker introduction and overview",
          "persons": [
            { "id": "speaker_1", "role": "presenter", "confidence": 0.92 }
          ],
          "estimatedTokens": 500
        }
      ],
      "estimatedTokens": 1000
    },
    "L3": {
      "fullAnalysis": {
        "persons": [
          {
            "id": "speaker_1",
            "name": "Jane Doe",
            "role": "presenter",
            "appearance": { "timeRange": { "start": 0, "end": 300 } },
            "confidence": 0.92
          }
        ],
        "objects": [
          { "label": "microphone", "confidence": 0.88 },
          { "label": "projector screen", "confidence": 0.85 },
          { "label": "stage", "confidence": 0.90 }
        ]
      },
      "estimatedTokens": 5000
    }
  }
}
```

### 3.2 L0–L3 の説明

| 層 | 目的 | 典型的なコンテンツ | Token 予算 | 用途 |
|---|---|---|---|---|
| **L0** | リファレンス・タグ | purpose, mediaClass, intent | ~50 | フィルタリング・ルーティング |
| **L1** | サマリ・メタデータ | title, summary, main tags | ~200 | 検索結果・サムネイル |
| **L2** | シーン・段落分析 | temporal scenes, structure | ~500–1000 | 詳細検索・推奨ジェネレータ |
| **L3** | フル詳細分析 | すべてのエンティティ・関係・属性 | ~2000–5000 | 詳細な QA・編集・アーカイビング |

Consumer（AI assistant など）は必要な詳細度に応じて L0–L3 のいずれかを選択します。

### 3.3 v0.1 を v0.2 に変換する手順

手動で変換する場合の「変換スケルトン」を示します。**自動化は完全には不可能**で、以下の項目は人間判断が必要です：

```javascript
function upgradeV01ToV02(v01Object) {
  const result = {
    schemaVersion: "2026-04-01",
    mediaType: v01Object.mediaType,
    summary: v01Object.summary,
    tags: v01Object.tags,
    moods: v01Object.mood ? [v01Object.mood] : [],  // v0.1: singular, v0.2: plural array
    confidence: v01Object.confidence,
    analyzedModules: v01Object.analyzedModules,
    precisionLevel: v01Object.precisionLevel,
    analyzedAt: v01Object.analyzedAt,
    context: {
      L0: {
        purpose: detectPurpose(v01Object),  // 手動で決定（後述）
        mediaClass: v01Object.mediaType,
        estimatedTokens: 50
      },
      L1: {
        title: v01Object.context?.title || "",
        summary: v01Object.summary,
        estimatedTokens: 200
      },
      L2: {
        // v0.1 の scenes があれば再構造化
        scenes: (v01Object.context?.scenes || []).map(scene => ({
          timeRange: scene.timeRange || { start: 0, end: 0 },
          description: scene.description || "",
          estimatedTokens: 500
        })),
        estimatedTokens: 1000
      },
      L3: {
        // v0.1 の context の詳細をここに
        fullAnalysis: {
          persons: v01Object.context?.persons || [],
          objects: v01Object.context?.objects || [],
          // ... その他のフィールド
        },
        estimatedTokens: 5000
      }
    }
  };
  
  return result;
}

// 補助関数: purpose の自動判定（完全ではない）
function detectPurpose(v01Object) {
  // v0.1 には purpose フィールドがないため、
  // mediaType や tags から推定するしかない
  if (v01Object.tags?.includes("presentation")) return "presentation";
  if (v01Object.tags?.includes("educational")) return "educational";
  if (v01Object.tags?.includes("entertainment")) return "entertainment";
  return "unknown";  // 手動修正が必要
}
```

### 3.4 自動変換できない項目（手動修正必須）

以下の項目は **v0.1 に存在しないため、自動化では判定できません**。v0.2 への移行時に人間が判断・追加してください：

| フィールド | 層 | 説明 | 対応例 |
|---|---|---|---|
| `purpose` | L0 | このメディアの意図・目的 | v0.1 の `tags` や `mood` から推定、またはアノテーション |
| `estimatedTokens` | L0–L3 | 各層の予想トークン数 | v0.2 spec §5.1 の表を参照して入力 |
| `intent` | L0 | Consumer からの意図（optional） | Runtime に追加（spec/v0.2/supply.md 参照） |
| L2 の `timeRange` 精度 | L2 | シーンの時間範囲（ビデオ用） | scene detector の出力を再チェック |

---

## §4. Namespace URI の置換ルール

既存の実装や設定ファイルに硬コードされた URI を置換します。

### 4.1 置換対象

| 用途 | v0.1 URI | v0.2 URI |
|---|---|---|
| XMP namespace | `https://m2c-protocol.org/ns/v0.1/` | `https://github.com/Akari-OS/m2c/ns/v0.2/` |
| JSON Schema `$id` (v0.1) | `https://m2c-protocol.org/schemas/v0.1/media-context.json` | `https://github.com/Akari-OS/m2c/schema/v0.1/media-context.json` |
| JSON Schema `$id` (v0.2) | N/A | `https://github.com/Akari-OS/m2c/schema/v0.2/media-context.json` |

### 4.2 sed による一括置換の例

```bash
# XMP namespace 置換（すべての .xmp ファイル）
find . -name "*.xmp" -type f -exec sed -i.bak \
  's|https://m2c-protocol\.org/ns/v0\.\([0-9]\)|https://github.com/Akari-OS/m2c/ns/v0.\1|g' \
  {} \;

# JSON/YAML 設定ファイルの schema URI 置換
find . -name "*.json" -o -name "*.yaml" -o -name "*.yml" | xargs sed -i.bak \
  's|https://m2c-protocol\.org/schemas/|https://github.com/Akari-OS/m2c/schema/|g'

# 検証（修正できたか確認）
grep -r "m2c-protocol.org" . 2>/dev/null | grep -v ".bak" | wc -l  # 0 であることを確認
```

### 4.3 手動確認ポイント

置換後、以下をテストしてください：

- ✓ XMP validator が新 URI を解決できるか
- ✓ JSON Schema validator が新 `$id` でスキーマをロードできるか
- ✓ HTTP リダイレクト不在：旧 URI にアクセスしても 404 / redirect されないことを確認

---

## §5. Validator と Schema ツール対応

JSON Schema validator や XMP パーサーが新しい URI を正しく扱うか確認します。

### 5.1 JSON Schema Validator の設定確認

#### ajv (JavaScript)

```javascript
import Ajv from 'ajv';
import fetch from 'node-fetch';

const ajv = new Ajv();

// v0.2 schema をロード
const schemaUrl = 'https://github.com/Akari-OS/m2c/schema/v0.2/media-context.json';
const schemaResponse = await fetch(schemaUrl);
const schema = await schemaResponse.json();

ajv.addSchema(schema);

// v0.2 JSON のバリデーション
const validate = ajv.getSchema('https://github.com/Akari-OS/m2c/schema/v0.2/media-context.json');
const isValid = validate(v02MediaContext);

if (!isValid) {
  console.error('Validation errors:', validate.errors);
}
```

#### jsonschema (Python)

```python
import jsonschema
import requests

# v0.2 schema をダウンロード
schema_url = 'https://github.com/Akari-OS/m2c/schema/v0.2/media-context.json'
schema = requests.get(schema_url).json()

# バリデーション
try:
    jsonschema.validate(v02_media_context, schema)
    print("✓ Valid v0.2 MediaContext")
except jsonschema.ValidationError as e:
    print(f"✗ Validation error: {e.message}")
```

### 5.2 XMP Namespace の確認

XMP ファイルの冒頭に以下を確認：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#"
         xmlns:m2c="https://github.com/Akari-OS/m2c/ns/v0.2/">
  <rdf:Description rdf:about="http://example.com/image.jpg">
    <m2c:purpose>documentation</m2c:purpose>
    <m2c:analyzedModules>scene-detector,text-ocr</m2c:analyzedModules>
  </rdf:Description>
</rdf:RDF>
```

**悪い例**（v0.1 URI の残存）：

```xml
xmlns:m2c="https://m2c-protocol.org/ns/v0.1/"  <!-- ← これは v0.2 では未サポート -->
```

---

## §6. 関連ドキュメント

本ガイド以上の詳細は、以下を参照：

| ドキュメント | 説明 |
|---|---|
| [`../spec/v0.2/protocol.md`](../spec/v0.2/protocol.md) | v0.2 プロトコル仕様（§5: Context Supply） |
| [`../spec/v0.2/schema.json`](../spec/v0.2/schema.json) | v0.2 JSON Schema（validator 用） |
| [`../spec/v0.2/interop.md`](../spec/v0.2/interop.md) | v0.2 相互運用性仕様（XMP / IPTC） |
| [`../CHANGELOG.md`](../CHANGELOG.md) | 完全な変更履歴 |

---

## まとめ（チェックリスト）

v0.1 から v0.2 への移行を完了するには、以下のステップを実施：

- [ ] schemaVersion の型チェックコード（JavaScript / Python）を実装
- [ ] v0.1 JSON が到着した場合の分岐処理を追加
- [ ] `context` を単層から 4 層 L0–L3 へ再構造化（自動 + 手動判定）
- [ ] Namespace URI をすべてのファイルで置換（`.xmp`, `.json`, `.yaml`）
- [ ] JSON Schema validator が新 `$id` を解決できることを確認
- [ ] XMP パーサーが新 namespace URI を認識することを確認
- [ ] 統合テストで v0.1 および v0.2 JSON の両方をパース可能か確認

移行に関する質問や問題がある場合は、[M2C GitHub Issues](https://github.com/Akari-OS/m2c/issues) に報告してください。
