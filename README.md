# blockchain_learning
## DeFi Yield Vault Handbook

---

# 概要

このリポジトリは、**Web3・DeFi・AI Agent** を組み合わせた実践的なハンドブック兼OSSプロジェクトです。

学習目的だけではなく、**実際に運用できるDeFiサービス**を構築することをゴールとしています。

最終的には、

* Smart Contract
* dApp
* Backend
* AI Agent
* x402
* Base Mainnet

までを1つのプロジェクトとして完成させます。

---

# コンセプト

この教材は一般的なWeb3チュートリアルとは異なります。

目的は

> 「DeFiの仕組みを理解する」

ではありません。

目的は

> **「自分自身で収益を生み出せるWeb3サービスを設計・開発・公開できるようになること」**

です。

そのため、

* なぜこの設計なのか
* なぜこの技術を選択するのか
* 実運用では何を考慮するか

まで含めて解説します。

---

# ゴール

このハンドブックを完了すると、

* Solidityが書ける
* Foundryが使える
* ERC20を理解できる
* Vaultを書ける
* BaseへDeployできる
* Coinbase Walletを接続できる
* dAppを書ける
* Aaveへ接続できる
* Morphoへ接続できる
* Yield Optimizerを書ける
* AI Agentを組み込める
* x402を組み込める
* GitHubへ公開できる

状態になることを目標とします。

---

# 最終アーキテクチャ

```text
                     Browser

                        │

                   Next.js dApp

                        │

          Coinbase Wallet / WalletConnect

                        │

                 Vault Smart Contract

                        │

        ┌───────────────┼────────────────┐
        │               │                │
      Aave           Morpho         Compound

                        │

                 Yield Strategy

                        │

                 FastAPI Backend

                        │

                  AI Agent Engine

                        │

                    x402 Payment
```

---

# 技術スタック

## Blockchain

* Base
* Base Sepolia
* Ethereum

---

## Smart Contract

* Solidity
* Foundry
* OpenZeppelin

---

## Frontend

* Next.js
* TypeScript
* Tailwind CSS
* wagmi
* viem
* Coinbase Wallet SDK

---

## Backend

* FastAPI
* Python
* SQLAlchemy
* APScheduler

---

## Database

* PostgreSQL

---

## Infrastructure

* Docker
* Docker Compose
* GitHub Actions
* Vercel
* Google Cloud Run

---

## AI

* OpenAI
* Claude
* Gemini

---

# Repository

```text
defi-yield-vault/

contracts/

frontend/

backend/

docs/

infra/

.github/

README.md
```

---

# 開発方針

本プロジェクトは

**Monorepo**

を採用します。

理由

* 学習しやすい
* 管理しやすい
* GitHub公開しやすい
* CI/CDを書きやすい

---

# ブランチ戦略

```text
main

develop

feature/chapter01

feature/chapter02

feature/chapter03
...
```

---

# 学習スタイル

各Chapter終了時には

* 動作する
* Git Commitできる
* GitHubへPushできる

状態になります。

毎回

> Learning → Coding → Testing → Deploy

のサイクルを回します。

---

# Chapter構成

## Chapter 1

### プロジェクト初期化

内容

* Foundry
* Next.js
* FastAPI
* Docker
* GitHub構成
* Docker Compose

成果物

* 開発環境

---

## Chapter 2

### Vault Contract

内容

* Deposit
* Withdraw
* Event
* Mapping

成果物

* Vault.sol

---

## Chapter 3

### ERC20・USDC対応

内容

* SafeERC20
* immutable
* ERC20

成果物

* USDC対応Vault

---

## Chapter 4

### Foundry Test

内容

* Mock Token
* forge test
* Coverage

成果物

* Vault.t.sol

---

## Chapter 5

### Base Sepolia

内容

* Deploy
* Verify
* BaseScan

成果物

* Deploy Script

---

## Chapter 6

### Coinbase Wallet

内容

* Wallet接続
* Base切替
* Address取得

成果物

* Wallet UI

---

## Chapter 7

### Deposit dApp

内容

* Deposit画面
* ERC20 Approve
* Deposit実行

成果物

* Deposit画面

---

## Chapter 8

### Withdraw dApp

内容

* Withdraw画面
* Balance表示
* Withdraw

成果物

* Vault UI完成

---

## Chapter 9

### Backend

内容

* FastAPI
* DB
* REST API

成果物

* Backend

---

## Chapter 10

### Aave Adapter

内容

* Deposit
* Withdraw
* Lending

成果物

* Aave連携

---

## Chapter 11

### Morpho Adapter

内容

* Lending
* Yield取得

成果物

* Morpho連携

---

## Chapter 12

### Strategy Engine

内容

* Yield比較
* 最適化
* Rebalance

成果物

* Yield Optimizer

---

## Chapter 13

### AI Agent

内容

* 利回り分析
* 自動提案
* 通知

成果物

* AI運用

---

## Chapter 14

### x402

内容

* Pay per Use
* Agent Payment

成果物

* マイクロペイメント

---

## Chapter 15

### Production

内容

* GitHub Actions
* CI/CD
* Base Mainnet
* Security

成果物

* 本番公開

---

# 各Chapter共通フォーマット

すべてのChapterは以下の構成で統一します。

1. Goal
2. 完成イメージ
3. Architecture
4. Directory
5. 実装
6. コード全文
7. 実行方法
8. デプロイ方法
9. テスト方法
10. Security
11. 設計レビュー
12. Git Commit
13. 演習問題
14. 次章

---

# 開発ルール

* コードは実際に動作することを前提とする
* サンプルではなく本番品質を目指す
* Chapter終了時には必ず動作確認を行う
* テストコードを必ず作成する
* セキュリティ上の考慮事項を記載する

---

# 将来の拡張

* ERC-4626対応
* Multichain対応
* Arbitrum対応
* Optimism対応
* EigenLayer
* Restaking
* Intentベース取引
* Account Abstraction
* AI Agent連携強化

---

# 最終成果物

このリポジトリを完成させると、

* 約50ページのハンドブック
* GitHub公開可能なOSS
* AI × DeFi × Base の実践プロジェクト
* 将来的な事業化・ポートフォリオの土台

が揃うことを目指します。
