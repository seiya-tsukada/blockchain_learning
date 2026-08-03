# Chapter 07: Deposit dApp

> Approve → Deposit の2段トランザクションを、permit による1段フローと併せて実装する。DeFi の UX で最も難しい部分。

!!! warning "この章は執筆中です（Phase 2）"
    本章は現在執筆中です。以下は確定済みのアウトラインです。

---

## この章で扱う内容

- `useReadContract` による `allowance` / `balanceOf` の取得
- `useWriteContract` + `useWaitForTransactionReceipt` の状態管理
- **Approve → Deposit の2段フロー**とその状態遷移の設計
- Chapter 03 の **EIP-2612 permit** を使った1トランザクション化
    - `signTypedData` によるオフチェーン署名
    - `eip712Domain()` からドメインを読む（ハードコードしない）
    - permit 非対応時のフォールバック
- 金額入力の扱い（`parseUnits` / `formatUnits`、`bigint` の徹底）
- 6 decimals の入力バリデーション（小数点以下7桁以上の拒否）
- 「無限 approve」を既定にしない UI 設計と revoke 導線
- 楽観的更新とキャッシュの無効化（`queryClient.invalidateQueries`）
- エラーの分類と日本語化（残高不足 / 承認不足 / ユーザー拒否 / ガス不足 / pause 中）

## この章の Security 観点

- 署名前にユーザーへ「何に署名しているか」を明示する
- `deadline` を短く設定する（30分）
- 署名をサーバーへ送らない
- 表示金額と署名金額が一致していることの保証

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

- 前: [Chapter 06: Coinbase Wallet 接続](./chapter06-coinbase-wallet.md)
- 次: [Chapter 08: Withdraw dApp](./chapter08-withdraw-dapp.md)
