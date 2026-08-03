# Chapter 12: Strategy Engine

> APY を比較して資金を最適配分する。リバランスの経済性（ガス vs 期待収益）を判断できるようにする。

!!! warning "この章は執筆中です（Phase 3）"
    本章は現在執筆中です。以下は確定済みのアウトラインです。

---

## この章で扱う内容

### オンチェーン: StrategyManager

- 複数 Adapter の登録・削除・配分管理
- `allocate` / `deallocate` / `rebalance` の実装
- **`AccessControl` への移行**（`Ownable` から）
    - `KEEPER_ROLE`: リバランスの実行のみ
    - `GUARDIAN_ROLE`: 緊急停止のみ
    - `DEFAULT_ADMIN_ROLE`: Timelock 経由
- **1プロトコルへの配分上限**（分散によるプロトコルリスクの緩和）
- idle 残高の確保（即時出金に応えるためのバッファ）
- スリッページ / 最小改善幅のガード（`minAprImprovementBps`）
- クールダウン（連続リバランスの防止）

### オフチェーン: 最適化ロジック

- Aave / Morpho の APY 収集（Chapter 09 の Scheduler を拡張）
- **リバランスの経済性判断**

```text
期待利益 = 移動額 × APY差 × 保有期間 / 365
判断: 期待利益 > ガスコスト × 安全係数 なら実行
```

- Keeper の実装（署名鍵の分離、nonce 管理、失敗時のリトライ）
- 「AI が判断する前の、決定的なルールベース最適化」の実装
  → Chapter 13 で AI を上に載せる土台

## この章の Security 観点

- **Keeper が資金を盗めないことの保証**（登録済み Adapter 間の移動のみ）
- 外部アドレスへの送金経路が存在しないことの検証
- APY データの汚染（オラクル的なリスク）
- MEV / サンドイッチ攻撃（リバランスのフロントラン）
- Keeper の鍵漏洩時の被害範囲

## この章で追加される不変条件

```text
K1: rebalance は totalAssets を（ガス以外で）減らさない
K2: KEEPER_ROLE は外部アドレスへ資産を移動できない
K3: 1 Adapter への配分は maxAllocationBps を超えない
K4: idle >= minIdleBps × totalAssets
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

- 前: [Chapter 11: Morpho Adapter](./chapter11-morpho-adapter.md)
- 次: [Chapter 13: AI Agent](./chapter13-ai-agent.md)
