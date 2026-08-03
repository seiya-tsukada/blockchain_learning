# Chapter 06: Coinbase Wallet 接続

> wagmi v2 + viem でウォレットを接続し、チェーンの検証と切替を実装する。Web3 フロントエンドの最初の壁を越える。

!!! warning "この章は執筆中です（Phase 2）"
    本章は現在執筆中です。以下は確定済みのアウトラインです。
    README の14項目フォーマットに従って埋められます。

---

## この章で扱う内容

- wagmi v2 の `WagmiProvider` / `QueryClientProvider` のセットアップ
- Connector の選択（Coinbase Wallet SDK / WalletConnect / Injected）
- `useAccount` / `useConnect` / `useDisconnect` の状態遷移
- **`chainId` の検証**とチェーン切替（`useSwitchChain`）
- Server Component と Client Component の境界（App Router）
- Foundry の `out/*.json` から wagmi cli で TypeScript 型を生成する
- `deployments/<chainId>.json` からコントラクトアドレスを読む
- SSR 時の hydration mismatch を避ける（`useEffect` / `mounted` パターン）
- アドレスの表示（ENS / Basename の解決、短縮表示）

## この章の Security 観点

- `NEXT_PUBLIC_*` に秘密情報を置かない
- 接続中チェーンが Base Sepolia であることを**必ず検証**する
- ユーザーが署名を拒否した場合（`UserRejectedRequestError`）の扱い
- WalletConnect の Project ID の露出範囲

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

- 前: [Chapter 05: Base Sepolia Deploy](./chapter05-base-sepolia-deploy.md)
- 次: [Chapter 07: Deposit dApp](./chapter07-deposit-dapp.md)
