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
- codegen（`akari-sdk/packages/sdk-types/`）のパイプライン整備は ✅ **2026-04-24 に Phase 2 完了**（akari-sdk `3f5e1ee` + Phase 3 `7555e29`）。上流 spec の `.schema.json` は akari-sdk 側 `schemas/m2c/v0.2/` に vendored copy として取り込まれ、`pnpm codegen` で `packages/sdk-types/src/generated/m2c-v0-2.ts` に反映される。同期手順は `akari-sdk/docs/codegen.md` を参照
