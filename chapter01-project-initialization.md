# Chapter 01: プロジェクト初期化

> Monorepo に Foundry / Next.js / FastAPI / PostgreSQL を並べ、3つのプロセスが同時に起動する開発環境を作る。

| 項目 | 内容 |
|---|---|
| 所要時間 | 2〜3時間 |
| 前提 | Git / ターミナルの基本操作 |
| 成果物 | 動作する開発環境（`forge build` / `npm run dev` / `fastapi dev` が通る） |
| 難易度 | ★☆☆ |

---

## 1. Goal

この章を終えたとき、以下がすべて「はい」になります。

- [ ] `forge build` が成功する
- [ ] `forge test` が成功する（サンプルテスト1件）
- [ ] `http://localhost:3000` で Next.js のページが表示される
- [ ] `http://localhost:8000/health` が `{"status":"ok"}` を返す
- [ ] `http://localhost:8000/docs` で OpenAPI ドキュメントが表示される
- [ ] `docker compose ps` で PostgreSQL が healthy になっている
- [ ] `make dev` で 3プロセスが一括起動する
- [ ] `.env` が `.gitignore` されており、`.env.example` だけがコミットされている
- [ ] `git log` に初回コミットが1つある

最後の2つが特に重要です。**秘密情報をコミットしない構造**を最初に作ります。
後から作ると、Git の履歴に鍵が残ります。

---

## 2. 完成イメージ

`make dev` を実行すると、3つのサービスが立ち上がります。

```text
$ make dev
docker compose up -d
[+] Running 2/2
 ✔ Network defi-yield-vault_default   Created
 ✔ Container dyv-postgres             Healthy

>>> contracts: forge build
[⠒] Compiling 24 files with Solc 0.8.24
[⠢] Solc 0.8.24 finished in 1.21s
Compiler run successful!

>>> backend: http://localhost:8000
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Application startup complete.

>>> frontend: http://localhost:3000
  ▲ Next.js 15.x
  - Local:  http://localhost:3000
  ✓ Ready in 1.8s
```

ヘルスチェック:

```text
$ curl -s localhost:8000/health | jq
{
  "status": "ok",
  "service": "defi-yield-vault-backend",
  "chain_id": 84532,
  "database": "connected"
}
```

最終的なディレクトリ:

```text
defi-yield-vault/
├── contracts/          Solidity (Foundry)
├── frontend/           Next.js
├── backend/            FastAPI
├── infra/              docker-compose, DB init
├── docs/               このハンドブック
├── .github/            CI
├── Makefile            開発コマンドの入口
├── .env.example        環境変数のテンプレート
└── README.md
```

---

## 3. Architecture

この章で作るのは「箱」です。中身は次章以降で埋めます。

```mermaid
flowchart TB
    subgraph host["開発マシン"]
        FE["Next.js<br/>:3000"]
        BE["FastAPI<br/>:8000"]
        FO["Foundry<br/>forge / anvil"]
    end
    subgraph docker["Docker Compose"]
        PG[("PostgreSQL<br/>:5432")]
    end
    subgraph remote["外部"]
        RPC["Base Sepolia RPC"]
    end

    BE --> PG
    BE -.->|"次章以降"| RPC
    FE -.->|"次章以降"| BE
    FO -.->|"次章以降"| RPC
```

!!! important "なぜ DB だけ Docker なのか"
    アプリケーション（Next.js / FastAPI）をコンテナに入れると、
    ホットリロードが遅くなり、デバッガの接続も面倒になります。
    一方 PostgreSQL は**バージョンと初期化を固定したい**ミドルウェアです。

    そこで本書は「**ミドルウェアはコンテナ、アプリはホスト**」という
    開発時の使い分けを取ります。本番（Chapter 15）ではアプリもコンテナ化します。

---

## 4. Directory

この章で作成されるファイル（`+` = 新規）:

```text
defi-yield-vault/
+├── .env.example
+├── .gitignore
+├── .editorconfig
+├── Makefile
+├── README.md
+├── contracts/
+│   ├── foundry.toml
+│   ├── remappings.txt
+│   ├── src/Counter.sol            ← 疎通確認用。Ch02 で削除
+│   ├── test/Counter.t.sol
+│   └── script/                     （空。Ch05 で使用）
+├── frontend/
+│   ├── package.json
+│   ├── tsconfig.json
+│   ├── next.config.ts
+│   ├── .env.local.example
+│   └── src/app/{layout.tsx,page.tsx,globals.css}
+├── backend/
+│   ├── pyproject.toml
+│   ├── .env.example
+│   └── app/
+│       ├── __init__.py
+│       ├── main.py
+│       ├── config.py
+│       ├── database.py
+│       └── routers/__init__.py
+│       └── routers/health.py
+├── infra/
+│   ├── docker-compose.yml
+│   └── postgres/init.sql
+└── .github/workflows/ci.yml
```

---

## 5. 実装

### 5.1 前提ツールの導入

必要なツールとバージョン確認方法:

```bash
# Git
git --version          # 2.30 以上

# Node.js（Next.js 用）。fnm または nvm 推奨
node -v                # 20 以上（22 LTS 推奨）

# Python
python3 --version      # 3.11 以上（3.12 推奨）

# Docker
docker -v
docker compose version

# Foundry
curl -L https://foundry.paradigm.xyz | bash
foundryup
forge --version
```

!!! warning "Windows の場合"
    ネイティブ Windows ではなく **WSL2 (Ubuntu)** を使ってください。
    Foundry / uv / Docker はいずれも Unix 系を前提に作られており、
    パス区切りや改行コードで無用な問題に時間を取られます。

`uv`（Python パッケージマネージャ）の導入:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
uv --version
```

??? question "なぜ pip ではなく uv なのか"
    速度と再現性です。詳細は [ADR-006](../00-preface/03-tech-stack.md#adr-006-python-環境は-uv) を参照。
    uv が使えない環境では `python -m venv .venv && pip install -e .` に読み替えてください。

### 5.2 リポジトリの初期化

```bash
mkdir defi-yield-vault && cd defi-yield-vault
git init
git branch -M main
```

**先に `.gitignore` を書きます。** これは順序が重要です。
`.env` を作ってから `.gitignore` を書くと、うっかり `git add .` した瞬間に
秘密鍵が履歴へ入ります。

```bash
cat > .gitignore <<'EOF'
# ---- Secrets（最重要）----
.env
.env.*
!.env.example
!.env.local.example
*.pem
*.key
keystore/

# ---- Foundry ----
contracts/out/
contracts/cache/
contracts/lib/
contracts/broadcast/*/dry-run/

# ---- Node ----
node_modules/
.next/
out/
*.tsbuildinfo
next-env.d.ts

# ---- Python ----
__pycache__/
*.py[cod]
.venv/
.pytest_cache/
.ruff_cache/
.mypy_cache/
.coverage

# ---- OS / Editor ----
.DS_Store
.idea/
.vscode/*
!.vscode/extensions.json
EOF

git add .gitignore
git commit -m "chore: add gitignore before creating any secret files"
```

!!! danger "`.DS_Store` について"
    macOS を使っている場合、`.DS_Store` は必ず ignore してください。
    元のリポジトリにもコミットされていました。実害は小さいものの、
    ディレクトリ構造とファイル名が漏れます。既にコミット済みなら:

    ```bash
    git rm --cached .DS_Store
    git commit -m "chore: remove .DS_Store from tracking"
    ```

### 5.3 contracts（Foundry）

```bash
forge init contracts --no-git
```

`--no-git` を付ける理由: `forge init` はデフォルトで Git リポジトリを作ってしまい、
Monorepo の中に入れ子のリポジトリができてしまいます。

依存ライブラリを導入します。

```bash
cd contracts
forge install OpenZeppelin/openzeppelin-contracts
forge install foundry-rs/forge-std
```

!!! note "`lib/` をコミットするかどうか"
    Foundry の依存は Git submodule として `lib/` に入ります。
    本書は `lib/` を `.gitignore` し、代わりに `.gitmodules` と
    `foundry.lock` に相当する情報（submodule のコミットハッシュ）を追跡します。
    クローン後は次のコマンドで復元します。

    ```bash
    git submodule update --init --recursive
    ```

    チームで完全な再現性を求める場合は `lib/` をコミットする流派もあります。
    どちらでもよいですが、**混在させない**ことが重要です。

`foundry.toml` を編集します。

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
test = "test"
script = "script"

solc = "0.8.24"
optimizer = true
optimizer_runs = 200
via_ir = false

# テストの再現性のため fuzz の seed を固定しない（CI で多様な入力を試す）
[profile.default.fuzz]
runs = 256

[profile.ci.fuzz]
runs = 2000

[profile.default.invariant]
runs = 64
depth = 32
fail_on_revert = false

# ネットワーク設定
[rpc_endpoints]
base = "${BASE_RPC_URL}"
base_sepolia = "${BASE_SEPOLIA_RPC_URL}"

[etherscan]
base = { key = "${BASESCAN_API_KEY}", chain = 8453 }
base_sepolia = { key = "${BASESCAN_API_KEY}", chain = 84532 }

[fmt]
line_length = 110
tab_width = 4
bracket_spacing = false
int_types = "long"
```

各設定の意図:

| 設定 | 理由 |
|---|---|
| `solc = "0.8.24"` | バージョンを固定。コンパイラ差でバイトコードが変わると Verify が失敗する |
| `optimizer_runs = 200` | デプロイコストと実行コストのバランス。Vault は呼び出し回数が多いので実行側を優先 |
| `via_ir = false` | IR パイプラインはコンパイルが遅い。最適化が必要になったら Ch15 で検討 |
| `[profile.ci.fuzz] runs = 2000` | ローカルは速さ、CI は網羅性。プロファイルで切り替える |
| `fail_on_revert = false` | Invariant テストで、revert する呼び出しは「無効な入力」として無視する |
| `[rpc_endpoints]` | `--rpc-url base_sepolia` と名前で指定できる。URL の書き間違いを防ぐ |

remappings を明示します。

```bash
cat > remappings.txt <<'EOF'
@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/
forge-std/=lib/forge-std/src/
EOF
```

疎通確認して、ルートに戻ります。

```bash
forge build
forge test
cd ..
```

### 5.4 frontend（Next.js）

```bash
npx create-next-app@latest frontend \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*" \
  --no-turbopack
```

| フラグ | 理由 |
|---|---|
| `--typescript` | 金額を `bigint` で扱うため型が必須 |
| `--app` | App Router。Server Component と Client Component の境界を学ぶ |
| `--src-dir` | `src/` 配下に集約。設定ファイルと混ざらない |
| `--import-alias "@/*"` | 相対パスの `../../..` を避ける |
| `--no-turbopack` | Web3 ライブラリは Node polyfill を要する場合があり、安定側を選ぶ |

Web3 の依存を追加します（実際に使うのは Chapter 06 から）。

```bash
cd frontend
npm install wagmi viem @tanstack/react-query
npm install @coinbase/wallet-sdk
cd ..
```

!!! note "この時点でインストールする理由"
    Chapter 06 で入れてもよいのですが、**lockfile を最初に確定させる**方が
    後のトラブルが少ないです。特に `viem` と `wagmi` はバージョンの整合が重要です。

### 5.5 backend（FastAPI）

```bash
mkdir backend && cd backend
uv init --name defi-yield-vault-backend --python 3.12
uv add "fastapi[standard]" "sqlalchemy[asyncio]" asyncpg pydantic-settings
uv add --dev pytest pytest-asyncio httpx ruff mypy
```

パッケージ構造を作ります。

```bash
mkdir -p app/routers
touch app/__init__.py app/routers/__init__.py
rm -f main.py hello.py   # uv init が作るサンプルを削除
cd ..
```

!!! important "設定は Pydantic Settings で1か所に集約する"
    `os.environ["FOO"]` を各所に散らすと、環境変数の追加時に漏れます。
    `app/config.py` に `Settings` クラスを1つ置き、**型と必須/任意を宣言**します。
    起動時に検証されるため、「本番で環境変数が未設定」に気づくのがデプロイ時ではなく起動時になります。

### 5.6 infra（Docker Compose）

```bash
mkdir -p infra/postgres
```

`infra/docker-compose.yml` を作ります。

ポイントは3点です。

1. **`healthcheck` を書く** — `depends_on: condition: service_healthy` で
   「DB がまだ起動中なのにアプリが接続して落ちる」を防ぐ
2. **named volume を使う** — bind mount はホスト OS 依存の権限問題を起こす
3. **ポートを 5432 以外にする選択肢を残す** — ホストに既存の PostgreSQL があると衝突する

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: dyv-postgres
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-vault}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-vault}
      POSTGRES_DB: ${POSTGRES_DB:-vault}
      # 日本語ロケールでの照合順序の問題を避ける
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=C"
    ports:
      - "${POSTGRES_PORT:-5432}:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/00-init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-vault} -d ${POSTGRES_DB:-vault}"]
      interval: 5s
      timeout: 5s
      retries: 10
    restart: unless-stopped

volumes:
  pgdata:
```

### 5.7 環境変数

`.env.example` を書きます。**実際の値は入れません。**

```bash
cat > .env.example <<'EOF'
# ===== Chain =====
# Base Sepolia (chainId 84532) / Base Mainnet (chainId 8453)
CHAIN_ID=84532
BASE_SEPOLIA_RPC_URL=https://sepolia.base.org
BASE_RPC_URL=https://mainnet.base.org

# ===== Contracts（Chapter 05 でデプロイ後に埋める）=====
VAULT_ADDRESS=
USDC_ADDRESS=0x036CbD53842c5426634e7929541eC2318f3dCF7e

# ===== Deploy（⚠️ Testnet 専用の鍵のみ）=====
# 本番では PRIVATE_KEY を使わず、keystore または HW ウォレットを使う
PRIVATE_KEY=
BASESCAN_API_KEY=

# ===== Database =====
POSTGRES_USER=vault
POSTGRES_PASSWORD=vault
POSTGRES_DB=vault
POSTGRES_PORT=5432
DATABASE_URL=postgresql+asyncpg://vault:vault@localhost:5432/vault

# ===== Backend =====
BACKEND_HOST=127.0.0.1
BACKEND_PORT=8000
LOG_LEVEL=INFO
CORS_ORIGINS=http://localhost:3000
EOF

cp .env.example .env
```

フロントエンド用は別ファイルにします。Next.js は `NEXT_PUBLIC_` 接頭辞の
変数を**ブラウザにバンドルする**ため、秘密情報を混ぜてはいけません。

```bash
cat > frontend/.env.local.example <<'EOF'
# ⚠️ NEXT_PUBLIC_* はブラウザに露出する。秘密情報を絶対に置かない
NEXT_PUBLIC_CHAIN_ID=84532
NEXT_PUBLIC_VAULT_ADDRESS=
NEXT_PUBLIC_USDC_ADDRESS=0x036CbD53842c5426634e7929541eC2318f3dCF7e
NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID=
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000
EOF

cp frontend/.env.local.example frontend/.env.local
```

!!! danger "秘密鍵の扱い"
    `PRIVATE_KEY` に入れるのは **Testnet 専用に新規作成したアドレス**の鍵だけです。
    普段使いのウォレットの鍵を、たとえ Testnet 用途でも `.env` に置かないでください。
    同じ鍵で Mainnet に資産があれば、`.env` の流出は資産の流出です。

    Chapter 05 では `cast wallet import` を使った keystore 方式も紹介します。

### 5.8 Makefile

コマンドの入口を1つにします。「READMEを読まないと起動できない」状態を避けます。

```bash
# Makefile は tab インデント必須（スペースだと動かない）
```

Makefile の全文は後述の「6. コード全文」に記載します。

### 5.9 疎通確認用の最小 API

`app/routers/health.py` に、DB 接続まで含めたヘルスチェックを置きます。
「起動した」ではなく「**依存先に到達できた**」を確認できることが重要です。

---

## 6. コード全文

### `Makefile`

```makefile
.DEFAULT_GOAL := help
SHELL := /bin/bash

# .env を読み込む（存在すれば）
ifneq (,$(wildcard .env))
	include .env
	export
endif

.PHONY: help
help: ## このヘルプを表示
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) \
		| awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-18s\033[0m %s\n", $$1, $$2}'

# ---------- setup ----------
.PHONY: setup
setup: ## 初回セットアップ（依存インストール）
	cd contracts && forge install
	cd frontend && npm ci || npm install
	cd backend && uv sync
	@test -f .env || cp .env.example .env
	@test -f frontend/.env.local || cp frontend/.env.local.example frontend/.env.local
	@echo "✅ setup done. 次は make dev"

# ---------- infra ----------
.PHONY: up down logs psql
up: ## PostgreSQL を起動
	docker compose -f infra/docker-compose.yml --env-file .env up -d
	@echo "⏳ waiting for postgres..."
	@until docker compose -f infra/docker-compose.yml exec -T postgres pg_isready -U $(POSTGRES_USER) >/dev/null 2>&1; do sleep 1; done
	@echo "✅ postgres healthy"

down: ## PostgreSQL を停止
	docker compose -f infra/docker-compose.yml down

reset-db: ## DB を完全に作り直す（データは消える）
	docker compose -f infra/docker-compose.yml down -v
	$(MAKE) up

logs: ## コンテナのログ
	docker compose -f infra/docker-compose.yml logs -f

psql: ## psql に入る
	docker compose -f infra/docker-compose.yml exec postgres \
		psql -U $(POSTGRES_USER) -d $(POSTGRES_DB)

# ---------- contracts ----------
.PHONY: build test test-gas fmt snapshot coverage anvil
build: ## コントラクトをビルド
	cd contracts && forge build

test: ## コントラクトのテスト
	cd contracts && forge test -vv

test-gas: ## ガスレポート付きテスト
	cd contracts && forge test --gas-report

coverage: ## カバレッジ
	cd contracts && forge coverage --report summary

fmt: ## Solidity フォーマット
	cd contracts && forge fmt

anvil: ## ローカルチェーンを起動
	anvil --chain-id 31337

# ---------- backend ----------
.PHONY: api api-test lint-py
api: ## FastAPI を起動
	cd backend && uv run fastapi dev app/main.py --port $(BACKEND_PORT)

api-test: ## backend のテスト
	cd backend && uv run pytest -q

lint-py: ## Python の lint / format / typecheck
	cd backend && uv run ruff format . && uv run ruff check --fix . && uv run mypy app

# ---------- frontend ----------
.PHONY: web web-build lint-web
web: ## Next.js を起動
	cd frontend && npm run dev

web-build: ## Next.js を本番ビルド
	cd frontend && npm run build

lint-web: ## Next.js の lint
	cd frontend && npm run lint

# ---------- all ----------
.PHONY: dev
dev: up ## DB + backend + frontend を一括起動
	@echo ""
	@echo ">>> backend : http://localhost:$(BACKEND_PORT)/docs"
	@echo ">>> frontend: http://localhost:3000"
	@echo ""
	@trap 'kill 0' EXIT; \
	( cd backend && uv run fastapi dev app/main.py --port $(BACKEND_PORT) ) & \
	( cd frontend && npm run dev ) & \
	wait

.PHONY: ci
ci: ## CI と同じ検査をローカルで走らせる
	cd contracts && forge fmt --check && forge build --sizes && FOUNDRY_PROFILE=ci forge test
	cd backend && uv run ruff check . && uv run mypy app && uv run pytest -q
	cd frontend && npm run lint && npm run build
```

!!! tip "`trap 'kill 0' EXIT` の意味"
    バックグラウンドで起動した子プロセスを、`Ctrl+C` で親と一緒に落とすためのおまじないです。
    これがないと `make dev` を止めても uvicorn と next が残り続けます。

### `contracts/foundry.toml`

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
test = "test"
script = "script"

solc = "0.8.24"
optimizer = true
optimizer_runs = 200
via_ir = false

[profile.default.fuzz]
runs = 256

[profile.ci.fuzz]
runs = 2000

[profile.default.invariant]
runs = 64
depth = 32
fail_on_revert = false

[rpc_endpoints]
base = "${BASE_RPC_URL}"
base_sepolia = "${BASE_SEPOLIA_RPC_URL}"

[etherscan]
base = { key = "${BASESCAN_API_KEY}", chain = 8453 }
base_sepolia = { key = "${BASESCAN_API_KEY}", chain = 84532 }

[fmt]
line_length = 110
tab_width = 4
bracket_spacing = false
int_types = "long"
```

### `infra/docker-compose.yml`

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: dyv-postgres
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-vault}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-vault}
      POSTGRES_DB: ${POSTGRES_DB:-vault}
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=C"
    ports:
      - "${POSTGRES_PORT:-5432}:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/00-init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-vault} -d ${POSTGRES_DB:-vault}"]
      interval: 5s
      timeout: 5s
      retries: 10
    restart: unless-stopped

volumes:
  pgdata:
```

### `infra/postgres/init.sql`

```sql
-- Chapter 09 で本格的なスキーマを作る。ここでは拡張のみ有効化する。
CREATE EXTENSION IF NOT EXISTS "pgcrypto";   -- gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS "btree_gin";  -- 複合インデックス用

-- 疎通確認用
CREATE TABLE IF NOT EXISTS _bootstrap (
    id          integer PRIMARY KEY DEFAULT 1,
    initialized timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT  single_row CHECK (id = 1)
);
INSERT INTO _bootstrap (id) VALUES (1) ON CONFLICT DO NOTHING;
```

### `backend/pyproject.toml`

```toml
[project]
name = "defi-yield-vault-backend"
version = "0.1.0"
description = "Off-chain services for the DeFi Yield Vault"
requires-python = ">=3.12"
dependencies = [
    "fastapi[standard]>=0.115",
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.29",
    "pydantic-settings>=2.4",
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.24",
    "httpx>=0.27",
    "ruff>=0.6",
    "mypy>=1.11",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "A", "C4", "SIM", "RUF"]
ignore = ["A003"]

[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["pydantic.mypy"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

### `backend/app/config.py`

```python
"""アプリケーション設定。

環境変数の読み込みはこのモジュールに集約する。
os.environ を各所で直接読むと、必須変数の漏れに気づけない。
"""

from functools import lru_cache
from typing import Literal

from pydantic import Field, field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        # ルートの .env を読む（backend/ から見て1つ上）
        env_file=("../.env", ".env"),
        env_file_encoding="utf-8",
        extra="ignore",
        case_sensitive=False,
    )

    # ---- chain ----
    chain_id: int = Field(default=84532, description="84532=Base Sepolia, 8453=Base")
    base_sepolia_rpc_url: str = "https://sepolia.base.org"
    base_rpc_url: str = "https://mainnet.base.org"

    # ---- contracts ----
    vault_address: str = ""
    usdc_address: str = ""

    # ---- database ----
    database_url: str = "postgresql+asyncpg://vault:vault@localhost:5432/vault"

    # ---- app ----
    log_level: Literal["DEBUG", "INFO", "WARNING", "ERROR"] = "INFO"
    cors_origins: str = "http://localhost:3000"

    @property
    def rpc_url(self) -> str:
        """chain_id に対応する RPC を返す。分岐をここに閉じ込める。"""
        return self.base_rpc_url if self.chain_id == 8453 else self.base_sepolia_rpc_url

    @property
    def cors_origin_list(self) -> list[str]:
        return [o.strip() for o in self.cors_origins.split(",") if o.strip()]

    @field_validator("chain_id")
    @classmethod
    def _known_chain(cls, v: int) -> int:
        # 想定外のチェーンで起動して気づかない事故を防ぐ
        if v not in (8453, 84532, 31337):
            raise ValueError(f"unsupported chain_id: {v} (expected 8453 / 84532 / 31337)")
        return v


@lru_cache
def get_settings() -> Settings:
    """設定はプロセス内で1度だけ構築する。"""
    return Settings()
```

### `backend/app/database.py`

```python
"""データベース接続。

SQLAlchemy 2.0 の async API を使う。
RPC もDBもI/O待ちが支配的なため、同期版を使うとスループットが出ない。
"""

from collections.abc import AsyncIterator

from sqlalchemy.ext.asyncio import AsyncEngine, AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase

from app.config import get_settings


class Base(DeclarativeBase):
    """全モデルの基底クラス。Chapter 09 でモデルを追加する。"""


_settings = get_settings()

engine: AsyncEngine = create_async_engine(
    _settings.database_url,
    echo=_settings.log_level == "DEBUG",
    pool_pre_ping=True,  # 切断済みコネクションを掴む事故を防ぐ
    pool_size=5,
    max_overflow=10,
)

SessionLocal = async_sessionmaker(engine, expire_on_commit=False, class_=AsyncSession)


async def get_session() -> AsyncIterator[AsyncSession]:
    """FastAPI の依存性注入用。リクエストごとにセッションを払い出す。"""
    async with SessionLocal() as session:
        yield session
```

### `backend/app/routers/health.py`

```python
"""ヘルスチェック。

「プロセスが生きている」だけでなく「依存先に到達できる」を確認する。
Cloud Run / k8s の readiness probe でもこのエンドポイントを使う。
"""

from typing import Annotated, Any

from fastapi import APIRouter, Depends
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

from app.config import Settings, get_settings
from app.database import get_session

router = APIRouter(tags=["system"])


@router.get("/health")
async def health(
    session: Annotated[AsyncSession, Depends(get_session)],
    settings: Annotated[Settings, Depends(get_settings)],
) -> dict[str, Any]:
    """依存先の到達性を含めた状態を返す。"""
    try:
        await session.execute(text("SELECT 1"))
        db_status = "connected"
    except Exception as exc:  # noqa: BLE001 - 状態報告のため広く捕捉する
        db_status = f"error: {type(exc).__name__}"

    return {
        "status": "ok" if db_status == "connected" else "degraded",
        "service": "defi-yield-vault-backend",
        "chain_id": settings.chain_id,
        "database": db_status,
    }


@router.get("/livez")
async def livez() -> dict[str, str]:
    """依存先を見ない生存確認。DB 障害時に再起動ループへ入るのを防ぐ。"""
    return {"status": "alive"}
```

!!! important "`/health` と `/livez` を分ける理由"
    Kubernetes / Cloud Run では **liveness**（プロセスを再起動すべきか）と
    **readiness**（トラフィックを流してよいか）を区別します。
    DB が落ちているときにプロセスを再起動しても直りません。
    liveness に DB チェックを入れると、DB 障害がアプリの再起動ループを引き起こします。

### `backend/app/main.py`

```python
"""FastAPI エントリポイント。"""

import logging
from contextlib import asynccontextmanager
from collections.abc import AsyncIterator

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import get_settings
from app.database import engine
from app.routers import health

settings = get_settings()
logging.basicConfig(level=settings.log_level)
logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    """起動・終了時の処理。"""
    logger.info("starting up: chain_id=%s", settings.chain_id)
    yield
    logger.info("shutting down")
    await engine.dispose()  # コネクションプールを明示的に閉じる


app = FastAPI(
    title="DeFi Yield Vault API",
    version="0.1.0",
    description="Off-chain services for the DeFi Yield Vault",
    lifespan=lifespan,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origin_list,
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["*"],
)

app.include_router(health.router)
```

### `backend/tests/test_health.py`

```python
import pytest
from httpx import ASGITransport, AsyncClient

from app.main import app


@pytest.mark.asyncio
async def test_livez_returns_alive() -> None:
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        res = await client.get("/livez")

    assert res.status_code == 200
    assert res.json() == {"status": "alive"}


@pytest.mark.asyncio
async def test_health_reports_chain_id() -> None:
    """DB が起動していれば connected、していなければ degraded になる。

    どちらでもテストが通るようにし、DB 依存でテストが落ちないようにする。
    """
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        res = await client.get("/health")

    assert res.status_code == 200
    body = res.json()
    assert body["chain_id"] in (8453, 84532, 31337)
    assert body["status"] in ("ok", "degraded")
```

### `.github/workflows/ci.yml`

```yaml
name: ci

on:
  push:
    branches: [main, develop]
  pull_request:

permissions:
  contents: read

jobs:
  contracts:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: contracts
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: foundry-rs/foundry-toolchain@v1

      - name: forge fmt --check
        run: forge fmt --check

      - name: forge build
        run: forge build --sizes

      - name: forge test
        run: forge test -vvv
        env:
          FOUNDRY_PROFILE: ci

  backend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true
      - run: uv sync --all-extras --dev
      - run: uv run ruff check .
      - run: uv run ruff format --check .
      - run: uv run mypy app
      - run: uv run pytest -q

  frontend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: npm
          cache-dependency-path: frontend/package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: npm run build
        env:
          NEXT_PUBLIC_CHAIN_ID: "84532"
          NEXT_PUBLIC_API_BASE_URL: "http://localhost:8000"
```

---

## 7. 実行方法

### 初回セットアップ

```bash
make setup
```

```text
cd contracts && forge install
cd frontend && npm ci || npm install
cd backend && uv sync
✅ setup done. 次は make dev
```

### 一括起動

```bash
make dev
```

別ターミナルで確認:

```bash
curl -s localhost:8000/livez
```

```text
{"status":"alive"}
```

```bash
curl -s localhost:8000/health
```

```text
{"status":"ok","service":"defi-yield-vault-backend","chain_id":84532,"database":"connected"}
```

ブラウザで以下を開きます。

| URL | 内容 |
|---|---|
| <http://localhost:3000> | Next.js の初期ページ |
| <http://localhost:8000/docs> | Swagger UI |
| <http://localhost:8000/redoc> | ReDoc |

### 個別起動

```bash
make up      # DB のみ
make api     # backend のみ
make web     # frontend のみ
make build   # contracts のビルド
```

### DB に入る

```bash
make psql
```

```text
vault=# \dt
              List of relations
 Schema |    Name     | Type  | Owner
--------+-------------+-------+-------
 public | _bootstrap  | table | vault
```

---

## 8. デプロイ方法

**この章では該当なし。**

理由: デプロイ対象のコントラクトがまだ存在しません（Chapter 02 で作成）。
ただし本章で CI（`.github/workflows/ci.yml`）を用意しているため、
リモートへ push した時点で自動検査が走る状態になります。

```bash
git remote add origin git@github.com:<your-account>/defi-yield-vault.git
git push -u origin main
```

GitHub の Actions タブで3ジョブ（contracts / backend / frontend）が
緑になることを確認してください。**この時点で CI を通しておくと、
以降の章で「どこで壊れたか」が特定しやすくなります。**

---

## 9. テスト方法

### 検証観点

| # | 観点 | 方法 |
|---|---|---|
| 1 | Solidity のツールチェーンが動く | `forge build` / `forge test` |
| 2 | Python の型と lint が通る | `ruff` / `mypy` |
| 3 | API が起動し依存先に到達する | `pytest` + `curl /health` |
| 4 | Next.js が本番ビルドできる | `npm run build` |
| 5 | DB が初期化されている | `make psql` → `\dt` |
| 6 | **秘密情報がコミットされていない** | 後述の gitleaks |

### 実行

```bash
make ci
```

これは CI と同じ検査をローカルで走らせます。**push する前に必ず実行**してください。

### 秘密情報の混入チェック

```bash
# gitleaks で履歴全体をスキャン
docker run --rm -v "$(pwd):/repo" zricethezav/gitleaks:latest detect \
  --source=/repo --no-banner --redact
```

```text
    ○
    │╲
    │ ○
    ○ ░
    ░    gitleaks

INF 0 commits scanned.
INF no leaks found
```

`.env` が追跡されていないことも確認します。

```bash
git ls-files | grep -E '^\.env$|^\.env\.' 
# → .env.example のみが表示されるべき
```

```text
.env.example
```

!!! warning "もし `.env` がコミットされていたら"
    履歴から消す必要があります。`git rm --cached` だけでは過去のコミットに残ります。

    ```bash
    # 1. まず鍵をローテートする（これが最優先。履歴の掃除より先）
    # 2. 履歴から除去
    pip install git-filter-repo
    git filter-repo --path .env --invert-paths --force
    ```

    公開リポジトリに push 済みの鍵は**漏洩したものとして扱う**しかありません。

---

## 10. Security

この章で増えた攻撃面と対策:

| 攻撃面 | リスク | 対策（本章で実施） |
|---|---|---|
| `.env` のコミット | 秘密鍵・API キーの流出 | `.gitignore` を最初にコミット、gitleaks でスキャン |
| `NEXT_PUBLIC_*` への機密混入 | ブラウザバンドルに露出 | フロント用 env を別ファイル化、コメントで警告 |
| デフォルトの DB 認証情報 | `vault:vault` のまま本番へ | ローカル限定と明記、本番は Secret Manager（Ch15） |
| ポートの外部公開 | `0.0.0.0` バインドで LAN から到達 | `BACKEND_HOST=127.0.0.1` を既定に |
| 依存パッケージのサプライチェーン | 悪意ある更新 | lockfile をコミット、CI では `npm ci` / `uv sync --frozen` |
| CORS の全許可 | 任意サイトから API 呼び出し | `allow_origins` を明示リストに |

### 秘密鍵管理の階層

```mermaid
flowchart TB
    A["レベル1: .env に平文<br/>Testnet の捨て鍵のみ"] --> B["レベル2: keystore + パスフレーズ<br/>cast wallet import（Ch05）"]
    B --> C["レベル3: ハードウェアウォレット<br/>--ledger / --trezor（Ch15）"]
    C --> D["レベル4: マルチシグ + Timelock<br/>Safe（Ch15）"]
```

本章はレベル1です。**Mainnet に触る Chapter 15 まででレベル3〜4へ上げます。**

### やってはいけないこと

- 普段使いのウォレットの秘密鍵を `.env` に置く
- `.env` を Slack / Discord / issue に貼る
- `PRIVATE_KEY` を CI のログに出力する（`echo $PRIVATE_KEY` は禁止）
- 本番の DB 認証情報を `docker-compose.yml` にハードコードする

---

## 11. 設計レビュー

### 採用: Monorepo

3つのサブプロジェクトを1リポジトリで管理します。理由は
[ADR-010](../00-preface/03-tech-stack.md#adr-010-monorepo) の通りですが、
本章時点での実利は「`make setup` 1発で全体が立ち上がる」ことです。

**却下**: Polyrepo。ABI の同期に git submodule か npm package が必要になり、
章ごとの成果物が3つのリポジトリに散ります。

**トレードオフ**: CI が全体を走ると遅くなります。Chapter 15 で
`paths` フィルタによる部分実行を導入します。

### 採用: アプリはホスト、DB はコンテナ

**却下案A: 全部コンテナ（devcontainer）**
再現性は最高です。しかしホットリロードの遅延、ボリュームマウントの権限問題、
Foundry のビルドキャッシュの扱いで、初学者が詰まるポイントが増えます。

**却下案B: 全部ホスト（PostgreSQL もローカルインストール）**
バージョンが揃わず、`brew` / `apt` の差異でトラブルが出ます。
「DB を作り直す」が難しくなるのも痛い（`make reset-db` が使えない）。

**結論**: 現在の折衷案。ただし Chapter 15 では本番用に全コンテナ化します。

### 採用: `Makefile` をコマンドの単一入口にする

**却下案: `package.json` の scripts に集約**
frontend だけならよいのですが、Python と Solidity のコマンドを
Node のスクリプトから叩くのは不自然です。

**却下案: `just` / `task`**
`Makefile` より書きやすいのですが、追加インストールが必要です。
`make` はほぼ全環境に存在します。

**トレードオフ**: Makefile はタブ必須で、変数展開が独特です。
`$(VAR)` と `$$VAR` の使い分けで初心者が詰まります。

### 採用: 設定を Pydantic Settings に集約

「型付きの設定オブジェクト1つ」にすると、起動時に検証されます。
`chain_id` に想定外の値が入ったら**起動しない**のは重要な安全弁です。
Chapter 15 で「Testnet の設定で Mainnet にデプロイした」事故を防ぎます。

### この章で残した技術的負債

| 負債 | 返済予定 |
|---|---|
| `src/Counter.sol` が残っている | Chapter 02 で削除 |
| DB スキーマが `_bootstrap` のみ | Chapter 09 でマイグレーション導入 |
| DB マイグレーションツール未導入（Alembic） | Chapter 09 |
| フロントは初期ページのまま | Chapter 06 |
| Docker イメージのビルド未定義 | Chapter 15 |
| `PRIVATE_KEY` が平文 | Chapter 05 で keystore、Chapter 15 で HW ウォレット |

---

## 12. Git Commit

この章は4つのコミットに分けます。

```bash
# 1. gitignore（すでに実施済み）
git add .gitignore .editorconfig
git commit -m "chore: add gitignore and editorconfig"

# 2. contracts
git add contracts/ .gitmodules
git commit -m "feat(contracts): initialize foundry project with OpenZeppelin"

# 3. frontend
git add frontend/
git commit -m "feat(frontend): initialize next.js with typescript and tailwind"

# 4. backend + infra
git add backend/ infra/
git commit -m "feat(backend): add fastapi skeleton with health check and postgres"

# 5. 開発体験
git add Makefile .env.example .github/ README.md
git commit -m "chore: add makefile, env template and CI workflow"
```

タグを打っておくと、後から章の境界に戻れます。

```bash
git tag -a ch01 -m "Chapter 01: project initialization"
git push origin main --tags
```

---

## 13. 演習問題

### 演習 1-1 ★ ポート衝突を解決する

すでにホストで PostgreSQL が 5432 で動いていると `make up` が失敗します。
`.env` を編集して 5433 で起動し、backend からも接続できるようにしてください。

??? question "解答方針"
    ```bash
    # .env
    POSTGRES_PORT=5433
    DATABASE_URL=postgresql+asyncpg://vault:vault@localhost:5433/vault
    ```

    `docker-compose.yml` の `"${POSTGRES_PORT:-5432}:5432"` は
    「ホスト側:コンテナ側」なので、コンテナ内は 5432 のままです。
    `DATABASE_URL` はホストから接続するため 5433 に変える必要があります。
    この「ホストとコンテナでポートが違う」構造を理解しておくと、
    Chapter 15 の Cloud Run 設定で迷いません。

### 演習 1-2 ★ `/health` に RPC の到達性を追加する

現在の `/health` は DB のみ確認しています。RPC への到達性も返すようにしてください。

??? question "解答方針"
    `httpx` で `eth_chainId` を JSON-RPC で叩き、`settings.chain_id` と一致するか検証します。

    ```python
    import httpx

    async def _check_rpc(settings: Settings) -> str:
        payload = {"jsonrpc": "2.0", "id": 1, "method": "eth_chainId", "params": []}
        try:
            async with httpx.AsyncClient(timeout=5.0) as client:
                res = await client.post(settings.rpc_url, json=payload)
            chain_id = int(res.json()["result"], 16)
        except Exception as exc:
            return f"error: {type(exc).__name__}"
        if chain_id != settings.chain_id:
            # ⚠️ 設定と実際のチェーンが違う。事故の原因になる
            return f"mismatch: rpc={chain_id} expected={settings.chain_id}"
        return "connected"
    ```

    **`mismatch` を検出することが本質です。** 「Sepolia の設定のまま
    Mainnet の RPC を向いていた」は実際に起こる事故です。

### 演習 1-3 ★★ pre-commit で秘密情報の混入を防ぐ

コミット時に自動で gitleaks が走るようにしてください。

??? question "解答方針"
    ```yaml
    # .pre-commit-config.yaml
    repos:
      - repo: https://github.com/gitleaks/gitleaks
        rev: v8.18.4
        hooks:
          - id: gitleaks
      - repo: https://github.com/astral-sh/ruff-pre-commit
        rev: v0.6.9
        hooks:
          - id: ruff
            args: [--fix]
          - id: ruff-format
      - repo: local
        hooks:
          - id: forge-fmt
            name: forge fmt
            entry: bash -c 'cd contracts && forge fmt --check'
            language: system
            files: '\.sol$'
            pass_filenames: false
    ```

    ```bash
    uv tool install pre-commit
    pre-commit install
    ```

    **注意**: `pre-commit` はローカルでスキップ可能（`--no-verify`）です。
    最終防衛線は CI 側に置く必要があります。

### 演習 1-4 ★★ 3ターミナル運用を tmux で自動化する

`make dev` は1ターミナルにログが混ざります。tmux で3分割し、
それぞれに DB ログ / backend / frontend を表示するスクリプトを書いてください。

??? question "解答方針"
    ```bash
    #!/usr/bin/env bash
    # scripts/dev-tmux.sh
    set -euo pipefail
    SESSION=dyv
    tmux kill-session -t $SESSION 2>/dev/null || true
    tmux new-session -d -s $SESSION -n dev
    tmux send-keys -t $SESSION:dev "make logs" C-m
    tmux split-window -h -t $SESSION:dev
    tmux send-keys -t $SESSION:dev "make api" C-m
    tmux split-window -v -t $SESSION:dev
    tmux send-keys -t $SESSION:dev "make web" C-m
    tmux select-layout -t $SESSION:dev tiled
    tmux attach -t $SESSION
    ```

### 演習 1-5 ★★★ CI のキャッシュを最適化する

現在の CI は毎回すべての依存を取得します。以下を計測し、改善してください。

1. 各ジョブの所要時間を記録する
2. Foundry のビルドキャッシュ（`contracts/out`, `contracts/cache`）を
   `actions/cache` で保存・復元する
3. キャッシュキーの設計（何が変わったら無効化すべきか）を説明する

??? question "解答方針"
    ```yaml
    - name: Cache forge build
      uses: actions/cache@v4
      with:
        path: |
          contracts/out
          contracts/cache
        key: forge-${{ runner.os }}-${{ hashFiles('contracts/src/**/*.sol', 'contracts/foundry.toml', 'contracts/.gitmodules') }}
        restore-keys: forge-${{ runner.os }}-
    ```

    キーに含めるべきもの: ソース、`foundry.toml`（solc バージョン・最適化設定）、
    依存のコミットハッシュ。**`foundry.toml` を含めないと、最適化設定を変えても
    古いバイトコードが再利用されます。** これは Verify の失敗として現れ、
    原因の特定に時間がかかります。

---

## 14. 次章

環境が整いました。次は**資産を預かるコントラクト**を書きます。

[Chapter 02: Vault Contract](./chapter02-vault-contract.md) では、
`deposit` / `withdraw` の最小実装を作ります。

なぜ最初に最小の Vault を作るのか:

1. **状態変数・mapping・event・修飾子**という Solidity の基本要素が
   すべて自然な形で登場する
2. この最小実装が抱える問題（ERC20 の戻り値、reentrancy、利回りの分配不能）が、
   Chapter 03 / 04 / 10 の**動機になる**
3. 「動くものを最短で作り、問題に気づいたら直す」という本書のリズムを作る

!!! note "Chapter 02 の実装は、Chapter 10 で大きく書き換わります"
    これは失敗ではなく、意図した進行です。単純な残高 mapping では
    「利回りを既存の預金者に公平に分配する」ことができません。
    その限界に自分でぶつかってから share 会計を学ぶ方が、
    ERC-4626 の必要性が腹に落ちます。
