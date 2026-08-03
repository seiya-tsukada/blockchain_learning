# Chapter 11: Morpho Adapter

> ERC-4626 準拠の Morpho Vault へ接続する。Aave との設計思想の違いから、Adapter パターンの価値を体感する。

!!! warning "この章は執筆中です（Phase 3）"
    本章は現在執筆中です。以下は確定済みのアウトラインです。

---

## この章で扱う内容

- Morpho のアーキテクチャ（Morpho Blue と、その上に載る Vault 層）
- **多くの Morpho Vault は ERC-4626 準拠** → Chapter 10 で学んだ知識がそのまま使える
- Aave との対比

| | Aave v3 | Morpho Vault |
|---|---|---|
| インターフェース | 独自（`supply` / `withdraw`） | **ERC-4626 標準** |
| 受け取るもの | `aToken`（残高が増える） | **share**（価格が上がる） |
| APY の取得 | `getReserveData` の Ray 値 | share 価格の変化から算出 |
| リスク管理 | プロトコル全体で共有 | Vault ごとにキュレーター |

- `MorphoAdapter` の実装（`IERC4626` を呼ぶだけで済む）
- **同じ `IStrategyAdapter` を実装することで Vault 本体が無変更**であることの確認
- APY の算出方法の違い（share 価格の時系列から年率換算）
- キュレーターリスク（誰が Vault のリスクパラメータを決めているか）
- `maxWithdraw` / `maxRedeem` を尊重する実装（流動性不足時の部分出金）
- Fork テストで実物の Morpho Vault を叩く

## この章の Security 観点

- キュレーターへの信頼（中央集権的な要素）
- ERC-4626 準拠を名乗るが挙動が異なる Vault への防御
- `previewWithdraw` と実際の `withdraw` の乖離
- 流動性不足時に出金が部分的にしかできないケース

---

## 章の構成（予定）

| # | セクション | 状態 |
|---|---|---|
| 1 | Goal | ⬜ |
| 2 | 完成イメージ | ⬜ |
| 3 | Architecture | ⬜ |
| 4 | Directory | ⬜ |
| 5 | 実装 | ⬜ |
| 6 | コード全文 | ⬜ |
| 7 | 実行方法 | ⬜ |
| 8 | デプロイ方法 | ⬜ |
| 9 | テスト方法 | ⬜ |
| 10 | Security | ⬜ |
| 11 | 設計レビュー | ⬜ |
| 12 | Git Commit | ⬜ |
| 13 | 演習問題 | ⬜ |
| 14 | 次章 | ⬜ |

---

- 前: [Chapter 10: Aave Adapter](./chapter10-aave-adapter.md)
- 次: [Chapter 12: Strategy Engine](./chapter12-strategy-engine.md)
