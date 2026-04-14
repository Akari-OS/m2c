# Contributing to M2C

M2C (Media to Context Protocol) への貢献を検討してくれてありがとう。
本リポジトリはプロトコル仕様 + 参考 SDK を扱う**仕様リポ**やで。
コア実装や UI ではなく、**仕様・Schema・モジュール定義・相互運用**の議論がメイン。

> M2C は [AKARI-OS](https://github.com/Akari-OS) エコシステムの「意味層」（Layer 2）を担う。
> エコシステム全体の行動規範は [Akari-OS/.github/CONTRIBUTING.md](https://github.com/Akari-OS/.github/blob/main/CONTRIBUTING.md) を参照。

---

## リポジトリの役割

M2C は「メディアから構造化コンテキストへ」を定義するオープン・プロトコル。
このリポは以下を扱う：

| ディレクトリ | 内容 |
|---|---|
| `spec/vX.Y/` | プロトコル仕様本体（protocol / modules / supply / interop + schema.json） |
| `spec/vX.Y/ja/` | 日本語訳 |
| `sdk/` | 参考実装（TypeScript / Python — Phase 6 以降で整備） |
| `examples/` | サンプル MediaContext JSON |

**コードは仕様リポであるため最小限**。実装は下記の関連リポで行う：

- **Pool** — `Akari-OS/pool` — Rust ベースの M2C Analyzer 実装
- **AKARI Video** — `Akari-OS/video` — M2C MediaContext の消費側
- **AMP** — `Akari-OS/amp` — Agent Memory Protocol（M2C と相補）

---

## 貢献の種類

| 貢献の形 | 歓迎度 | 始め方 |
|---|:-:|---|
| 🐛 仕様の誤り / 不明瞭さの報告 | ⭐⭐⭐ | Issue を立てる |
| 💡 プロトコル拡張の提案 | ⭐⭐⭐ | Discussion → Issue → Draft PR |
| 📖 仕様ドキュメントの改善 | ⭐⭐⭐ | typo でも歓迎 |
| 🌍 翻訳（英 ↔ 日 以外の言語） | ⭐⭐ | `spec/vX.Y/<lang>/` を追加 |
| 🧩 新モジュール定義 | ⭐⭐ | `modules.md` の Module Registry に追記 |
| 🛠 SDK 実装 | ⭐⭐ | TypeScript / Python ベースで |
| 🔗 Interop マッピング追加 | ⭐⭐ | IPTC / C2PA / XMP / Schema.org など |

---

## 仕様変更プロセス

M2C は**日付ベースのバージョニング** (`YYYY-MM-DD`) を採用。
互換性を守るため、以下のワークフローに従う：

### 1. 軽微な変更（typo・明確化・誤り修正）

- 直接 `spec/<current-version>/` を編集する PR を送る
- バージョンは上げない

### 2. 後方互換な追加（モジュール追加・拡張フィールド追加）

- 同バージョン内で追記 OK
- `CHANGELOG.md` の `[Unreleased]` に記載
- `extensions` 名前空間や `x-` prefix を優先（コア追加は慎重に）

### 3. 破壊的変更（Schema 変更・モジュール IF 変更）

- 新バージョン `spec/vX.(Y+1)/` を作成
- 旧バージョンは残す（deprecated マーク）
- Discussion → Issue → Draft Spec → レビュー → Accept の流れ

### バージョン履歴

| バージョン | 日付 | ステータス | 主な変更 |
|---|---|---|---|
| [v0.1](./spec/v0.1/protocol.md) | 2026-XX-XX | superseded | 初版。基本的な Analysis / Schema のみ |
| [v0.2](./spec/v0.2/protocol.md) | 2026-04-01 | **Draft（current）** | Intent-Driven Dispatch / 4-Pillar Supply / LAN Distributed / Interop |

---

## PR フロー

### 1. Fork してブランチを切る

```bash
git clone https://github.com/<your-username>/m2c.git
cd m2c
git checkout -b spec/add-xxx-module
```

**ブランチ名**:

- `spec/xxx` — 仕様変更
- `docs/xxx` — ドキュメント修正
- `sdk/xxx` — SDK 実装
- `fix/xxx` — 誤り修正

### 2. 変更を加える

- 既存の記述スタイルに合わせる（英語本文 + 日本語訳は別ファイル）
- `schema.json` を変更したら、対応する `protocol.md` / `modules.md` も更新
- 日本語版（`spec/vX.Y/ja/`）も同時更新（翻訳漏れを防ぐ）

### 3. コミット

**コミットメッセージは日本語**で、以下のプレフィックスを付ける：

| プレフィックス | 用途 |
|---|---|
| `[仕様]` | protocol / schema / modules の変更 |
| `[ドキュメント]` | README / CONTRIBUTING 等の更新 |
| `[SDK]` | SDK コードの変更 |
| `[例]` | examples/ の追加・修正 |
| `[翻訳]` | ja/ 他の翻訳更新 |
| `[バグ修正]` | 仕様の誤り修正 |
| `[リファクタ]` | 本質を変えない整理 |

例:

```
[仕様] v0.2 に Intent-Driven Dispatch を追加

ユーザーの purpose/hints/background を dispatcher が受け取り、
不要なモジュール実行をスキップできるようにした。
```

### 4. PR を送る

PR 本文に以下を含める：

- **何を変えたか** (What)
- **なぜ変えたか** (Why)
- **影響範囲** — schema か protocol か modules か、後方互換性への影響
- 関連 Issue への参照 (`Closes #123`)

### 5. レビュー → マージ

- 破壊的変更は複数のレビューが必要
- 軽微な修正は 1 レビューで OK

---

## コーディング規約（SDK）

SDK コード（`sdk/` 配下）に関する規約：

- **TypeScript 優先**（JS 新規ファイル禁止）
- 変数名・関数名・ファイル名は英語
- コメント・コミットメッセージは日本語
- `any` は原則禁止、型を明示
- 新規依存は最小限に

---

## コミュニケーション

- 🐛 **仕様の誤り**: [Issues](https://github.com/Akari-OS/m2c/issues)
- 💡 **拡張提案・議論**: [Discussions](https://github.com/Akari-OS/m2c/discussions)
- 🔒 **セキュリティ脆弱性**: [SECURITY.md](./SECURITY.md)（※ 現在は Akari-OS/.github 側を参照）

**行動規範**: [Code of Conduct](./CODE_OF_CONDUCT.md) に従うこと。

---

## ライセンス

貢献するコード・仕様・ドキュメントは **Apache 2.0** ライセンスで公開される。
PR を送ることは、このライセンスでの公開に同意したとみなす（CLA なし、DCO なし）。

---

## 感謝

仕様リポなので、**誤字 1 つの修正でも仕様の穴を塞ぐ重要な貢献**になる。
すべての貢献を歓迎する。ありがとう。
