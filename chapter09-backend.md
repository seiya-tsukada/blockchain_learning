# Chapter 09: Backend

> FastAPI + PostgreSQL でオンチェーンデータをインデックスし、REST API として提供する。オンチェーンを唯一の真実とする設計。

!!! warning "この章は執筆中です（Phase 2）"
    本章は現在執筆中です。以下は確定済みのアウトラインです。

---

## この章で扱う内容

- SQLAlchemy 2.0 のモデル定義と Alembic によるマイグレーション
- **金額を `NUMERIC(78, 0)` でアトミック単位のまま保存する**（float 禁止）
- Indexer の実装
    - `eth_getLogs` によるイベント取得
    - `last_indexed_block` による再開可能性
    - **reorg（チェーン再編成）への対処**（confirmations の待機）
    - 冪等性の担保（`(txHash, logIndex)` を一意キーに）
- APScheduler による定期実行（15秒ごとのインデックス、5分ごとのスナップショット）
- REST API の設計（`/vault/stats`, `/vault/{address}/history`, `/apy`）
- Pydantic による入出力スキーマと OpenAPI 自動生成
- レート制限とキャッシュ
- 「DB はキャッシュである」ことを守る設計（再インデックスで完全復元できる）

## この章の Security 観点

- SQL インジェクション（ORM の適切な使用）
- アドレスの正規化（checksum / lowercase の統一）
- API の認証が不要な範囲と必要な範囲の切り分け
- RPC プロバイダの障害時の挙動（degraded を返す）

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

- 前: [Chapter 08: Withdraw dApp](./chapter08-withdraw-dapp.md)
- 次: [Chapter 10: Aave Adapter](./chapter10-aave-adapter.md)
