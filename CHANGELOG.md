# Changelog

All notable changes to M2C (Media-to-Context Protocol) will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/) for the SDK
and **date-based versioning** (`YYYY-MM-DD`) for the protocol spec.

---

## [Unreleased]

### Added

- `docs/architecture.md` — AKARI OS エコシステムとの統合アーキテクチャドキュメント
- `CONTRIBUTING.md` — 仕様変更プロセス / PR フロー / コミット規約
- `CODE_OF_CONDUCT.md` — Contributor Covenant v2.1 準拠
- `CHANGELOG.md` — 変更履歴の追跡開始

---

## 2026-04-22 — Schema URI Migration (Breaking)

### Changed (BREAKING)
- Schema `$id` and namespace URIs migrated from `https://m2c-protocol.org/` to `https://github.com/Akari-OS/m2c/` (the dedicated domain was not secured; canonical source is now the GitHub repository).
- Affected: `spec/v0.2/schema.json`, `interop.md`, `supply.md`, `modules.md`, and Japanese translations.

### Migration
- Validators pinning to the old `$id` must update to the new URI:
  - Old: `https://m2c-protocol.org/schemas/v0.2/media-context.json`
  - New: `https://github.com/Akari-OS/m2c/schema/v0.2/media-context.json`
- XMP / interop namespace URI also changed:
  - Old: `https://m2c-protocol.org/ns/v0.2/`
  - New: `https://github.com/Akari-OS/m2c/ns/v0.2/`
- Backwards compatibility: Old URIs will NOT be redirected.

---

> **Note — v0.2 schema artifacts:** Schema types referenced across `spec/v0.2/` documents
> (`module-manifest.json`, `context-request.json`, `context-policy.json`,
> `context-response.json`, `m2c.xmp`) are inlined in the spec documents;
> standalone JSON artifact files are planned for v0.2.1.

## [0.2.0-draft] — 2026-04-01

プロトコルの本格化。v0.1 のコア設計はそのままに、**Intent-Driven Dispatch**、
**4-Pillar Context Supply**、**LAN Distributed** トポロジ、**相互運用仕様**を追加。

### Added

- **Intent-Driven Dispatch** (`protocol.md §3.2`)
  - `M2CIntent` オブジェクト（`purpose` / `hints` / `background`）を追加
  - dispatcher が intent を受け取り、不要なモジュールを自動スキップ
  - summary モジュールが intent を反映してより文脈に沿った要約を生成
- **Context Supply Protocol** (`supply.md` — 新規)
  - 4 Pillars（Write / Select / Compress / Isolate）を正式に定義
  - JIT ロード・トークン予算・モダリティ分離を仕様化
- **Interoperability Spec** (`interop.md` — 新規)
  - IPTC 2025.1 / C2PA / XMP / Schema.org とのマッピング規約
  - MCP ツール経由でのモジュール公開方法
- **LAN Distributed Topology** (`protocol.md §8.2`)
  - mDNS/Bonjour (`_m2c._tcp.local`) での自動発見
  - 共有シークレット / ローカルトークン認証
  - NAS / SMB 経由のメディア共有パターン
- **Multi-Layer Context** (`protocol.md §5.2`)
  - L0 (Tags ~50 tokens) / L1 (Summary ~200) / L2 (Scenes ~500-1000) / L3 (Full ~2000-5000) の 4 層構造
- **Extended Media Types** (`protocol.md §2`)
  - `music` / `presentation` / `spreadsheet` / `vector` / `3d` / `archive` / `data` を追加
  - `x-` prefix によるカスタムタイプのサポート
- **Module Registry** (`modules.md` — 大幅拡張)
  - `M2CModuleManifest` の仕様化
  - dispatch weight / default ordering の明文化
- **Quality Gates** (`protocol.md §5.4`)
  - Pass (≥0.7) / Review (0.4–0.7) / Reject (<0.4) の閾値導入
  - 低 confidence モジュールのみを再実行する Re-Analysis フロー

### Changed

- **Protocol Versioning** — セマンティックバージョニングから**日付ベース** (`YYYY-MM-DD`) に変更（MCP パターンに準拠）
- **Security Default** — 認証を「推奨」から「REQUIRED」に格上げ（Local トポロジ除く）
- **Module Interface** — `analyze()` に加え `estimate()` を必須化（コスト透明性）
- **Confidence Scoring** — 加重平均に変更。各モジュールに default weight を定義

### Deprecated

- v0.1 の単層 `context` 構造 — v0.2 の L0–L3 構造に置き換え（v0.1 は後方互換維持）

---

## [0.1.0] — 2026-04-01

初回リリース（Draft）。

### Added

- 基本プロトコル定義（`spec/v0.1/protocol.md`）
- モジュール IF とレジストリ（`spec/v0.1/modules.md`）
- MediaContext JSON の基本構造
- 対応メディアタイプ: `video` / `image` / `audio` / `document`
- Precision Levels（Quick / Standard / Deep）
- Capability Negotiation（MCP 互換パターン）
- 日本語翻訳（`spec/v0.1/ja/`）

---

[Unreleased]: https://github.com/Akari-OS/m2c/compare/v0.2.0-draft...HEAD
[0.2.0-draft]: https://github.com/Akari-OS/m2c/releases/tag/v0.2.0-draft
[0.1.0]: https://github.com/Akari-OS/m2c/releases/tag/v0.1.0
