# M2C v0.2 — Schema Files

本ディレクトリに同梱される JSON Schema ファイルの一覧。コード生成（codegen）や ランタイム validation から直接参照できるようスタンドアロンで保持する。

| ファイル | 対象 | 出所（spec 本文） |
|---|---|---|
| `schema.json` | `MediaContext`（コア解析結果オブジェクト） | `protocol.md` §4 MediaContext Schema |
| `capabilities.schema.json` | Provider Capabilities（§6.1） | `protocol.md` §6.1 Provider Capabilities |
| `request.schema.json` | Consumer Request（§6.2） | `protocol.md` §6.2 Consumer Request |

## 未抽出の JSON 断片（v0.2 時点）

以下の ```` ```json ```` ブロックは spec 本文にあるが、現時点では型情報が薄く schema 化の価値が小さいため本ディレクトリに独立ファイルとしては置いていない。必要になったら追加する:

- **progress notification**（§6 末尾）— 長時間解析の進捗通知。実装ごとの自由度を残す方針で、`message-type` + 任意 payload の緩い契約
- **custom `mediaType` 例**（§3）— `x-` prefix を例示するだけで schema 本体ではない

## 追記・更新ルール

- spec 本文の JSON 断片を更新したら、対応する schema ファイルも同一 PR で更新する
- `schemaVersion` は `MediaContext` 内の一意に紐づく識別子（`protocol.md` §4 参照）で、本 spec の `protocolVersion`（§6）とは別軸である点に注意
- codegen（`akari-sdk/packages/sdk-types/`）のパイプライン整備は P3 #5 Phase 2（未着手）で実施予定
