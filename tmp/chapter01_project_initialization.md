# DeFi Yield Vault Handbook

## Chapter 1: プロジェクト初期化

### ゴール
- Foundry
- Next.js
- FastAPI
- Docker
- Base Sepolia 開発環境を構築する

## リポジトリ構成

```text
defi-yield-vault/
├── contracts/
├── frontend/
├── backend/
├── docs/
├── infra/
├── .github/
└── README.md
```

## contracts

```text
contracts/
├── src/
├── script/
├── test/
├── lib/
└── foundry.toml
```

インストール

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
forge init contracts
```

## frontend

```bash
npx create-next-app@latest frontend --ts --tailwind --app
cd frontend
npm install wagmi viem @coinbase/wallet-sdk
```

## backend

```bash
mkdir backend
cd backend
uv init
uv add fastapi uvicorn sqlalchemy psycopg[binary]
```

## Docker

docker-compose.yml

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: vault
      POSTGRES_PASSWORD: vault
      POSTGRES_DB: vault
    ports:
      - "5432:5432"
```

起動

```bash
docker compose up -d
```

## .env

```text
BASE_RPC_URL=
PRIVATE_KEY=
USDC_ADDRESS=
NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID=
```

## 動作確認

```bash
forge build
cd frontend && npm run dev
cd backend && uv run fastapi dev
```

## Git Commit

```bash
git add .
git commit -m "feat: initialize project"
```

## 次章

Vaultコントラクトを実装し、Deposit / Withdraw を作成する。
