# Chapter 10: Aave Adapter

> Aave v3 へ接続し、同時に **share 会計（ERC-4626）へ移行する**。本書で最も重要な設計転換の章。

!!! warning "この章は執筆中です（Phase 3）"
    本章は現在執筆中です。以下は確定済みのアウトラインです。

---

## この章の位置づけ

Chapter 02〜05 で作った Vault は `mapping(address => uint256)` で残高を持つため、
**利回りを既存の預金者に分配できません**（Chapter 02 の演習 2-3 で確認）。

本章では Aave 連携によって初めて「利回り」が発生するため、
同時に share 会計へ移行します。**この移行が本書の中核**です。

---

## この章で扱う内容

### Part A: share 会計への移行

- `mapping` 会計の限界の再確認（O(n) の分配は不可能）
- **ERC-4626 の設計**: `totalAssets` / `totalSupply` / `convertToShares` / `convertToAssets`
- `deposit` / `mint` / `withdraw` / `redeem` の4関数と `maxXxx` / `previewXxx`
- **丸めの方向**を常にプロトコル有利にする理由（`Math.Rounding`）
- **inflation attack（donation attack）** の成立条件と対策
    - OpenZeppelin `ERC4626` の `_decimalsOffset()` による仮想 share
    - 初期デポジット（dead shares）による緩和
- 旧 Vault からの移行手順（イミュータブル設計における「新デプロイ + 移行」）
- Chapter 04 のテストのうち「壊れてよいもの / 壊してはいけないもの」の区別

### Part B: Aave v3 Adapter

- Aave v3 の `IPool` インターフェース（`supply` / `withdraw`）
- `aToken` の性質（rebasing 的に残高が増える）
- `getReserveData` からの APY 算出（**Ray = 1e27 単位**、`liquidityRate` → APY 変換）
- `IStrategyAdapter` インターフェースの設計
- Adapter を別コントラクトに切る理由（Vault 本体を変更しない）
- `forceApprove` が必要になる場面
- **Fork テストで実物の Aave v3 Pool を叩く**
- Aave 側の停止・上限（supply cap）・凍結への対処

## この章の Security 観点

- share 会計の丸めによる資産流出（1 wei ずつ抜くループ攻撃）
- 空の Vault への最初の預入（first depositor attack）
- 外部プロトコルの `totalAssets` を信用することのリスク
- `totalAssets()` が外部呼び出しを含むことによる reentrancy 経路
- Aave の停止時に出金できなくなるリスクと idle 残高の確保

## この章で追加される不変条件

```text
S1 (Solvency): totalAssets() >= convertToAssets(totalSupply())
S2 (No free lunch): convertToAssets(convertToShares(x)) <= x
S3 (Monotonic share price): 資産の増加以外で1 share の価値は下がらない
S4 (Adapter accounting): Σ adapter.totalAssets() + idle == vault.totalAssets()
```

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

- 前: [Chapter 09: Backend](./chapter09-backend.md)
- 次: [Chapter 11: Morpho Adapter](./chapter11-morpho-adapter.md)
