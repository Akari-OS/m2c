# M2C (Media-to-Context Protocol) Documentation

> **このリポの立ち位置**: Media-to-Context Protocol の**仕様リポ**。メディア（動画・音声・画像・PDF 等）を AI が扱えるコンテキストに変換するためのプロトコル定義。
> **扱う範囲**: spec、Module Manifest、Layer 0-4 モデル、アーキテクチャ設計、SDK インタフェース
> **扱わない範囲**: Analyzer の実装（→ AkariPool の analyzer プラグイン）、UI / Shell 層（→ 各アプリリポ）、戦略判断（→ Hub）
>
> - 🌐 正典: [Akari-OS/.github](https://github.com/Akari-OS)
> - 🏛 Hub（非公開）: `akari-os` — 横断研究・戦略・Master Index
> - 🗺 全リポマップ: `akari-os/MAP.md`

---

## このリポのドキュメント

### docs/ 配下

| ファイル | 内容 |
|---|---|
| [architecture.md](./architecture.md) | M2C アーキテクチャ — Layer 0-4 モデル、Module Manifest、Analyzer 連携 |

### spec（プロトコル仕様）

| パス | 内容 |
|---|---|
| [`../spec/v0.1/`](../spec/v0.1/) | v0.1 — 仕様本体 |
| [`../spec/v0.2/`](../spec/v0.2/) | v0.2 — ドラフト |

### ルート直下のメタドキュメント

| ファイル | 内容 |
|---|---|
| [`../README.md`](../README.md) / [`../README.ja.md`](../README.ja.md) | プロトコル概要 |
| [`../CONTRIBUTING.md`](../CONTRIBUTING.md) | コントリビューションガイド |
| [`../CHANGELOG.md`](../CHANGELOG.md) | 変更履歴 |
| [`../examples/`](../examples/) | サンプル |
| [`../sdk/`](../sdk/) | SDK |

---

## 新規ドキュメントの追加

spec の変更は `spec/vX.Y/` 配下で。設計判断や横断的な解説は `docs/` に置き、このファイルの index も必ず更新すること。
