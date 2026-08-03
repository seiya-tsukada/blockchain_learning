# Chapter 3: USDC対応（ERC20対応）

## Goal
VaultをUSDCで安全に利用できるよう改善する。

## この章で学ぶこと
- ERC20の基本
- USDCの扱い
- SafeERC20
- immutable
- Base Sepoliaの設定

## GitHubツリー

```text
contracts/
├── src/
│   ├── Vault.sol
│   └── interfaces/
├── test/
│   └── Vault.t.sol
└── script/
    └── Deploy.s.sol
```

## 改善ポイント

Vault.sol

```solidity
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

using SafeERC20 for IERC20;

IERC20 public immutable asset;

function deposit(uint256 amount) external {
    require(amount > 0, "amount");
    asset.safeTransferFrom(msg.sender, address(this), amount);
    balance[msg.sender] += amount;
}
```

## Base Sepolia

`.env`

```text
BASE_RPC_URL=https://sepolia.base.org
PRIVATE_KEY=...
USDC_ADDRESS=<Base Sepolia USDC>
```

## デプロイ

```bash
forge build
forge test
forge script script/Deploy.s.sol \
  --rpc-url $BASE_RPC_URL \
  --broadcast
```

## 設計レビュー

- transferではなくSafeERC20を採用
- immutableでガス削減
- ERC4626へ将来移行予定

## 演習

1. USDC以外のERC20でも動作確認
2. 0枚Deposit時のRevert確認

## Commit

```bash
git add .
git commit -m "feat: support usdc safely"
```

## Next

Chapter4ではFoundry Testを充実させ、ユニットテストを実装する。
