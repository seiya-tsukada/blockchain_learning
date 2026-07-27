# Chapter 5: Base Sepoliaへのデプロイ

## Goal
VaultコントラクトをBase Sepoliaへデプロイし、BaseScanで確認する。

## 前提
- Foundryインストール済み
- Coinbase Wallet作成済み
- Base Sepolia RPC取得済み

## .env.example

```text
BASE_RPC_URL=https://sepolia.base.org
PRIVATE_KEY=YOUR_PRIVATE_KEY
ETHERSCAN_API_KEY=YOUR_API_KEY
```

## foundry.toml

```toml
[profile.default]
src="src"
out="out"
libs=["lib"]
solc_version="0.8.24"
```

## Deploy.s.sol

```solidity
pragma solidity ^0.8.24;
import "forge-std/Script.sol";
import "../src/Vault.sol";

contract Deploy is Script {
    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        new Vault(vm.envAddress("USDC_ADDRESS"));
        vm.stopBroadcast();
    }
}
```

## Build

```bash
forge build
```

## Deploy

```bash
forge script script/Deploy.s.sol \
  --rpc-url $BASE_RPC_URL \
  --broadcast
```

## Verify

```bash
forge verify-contract <CONTRACT_ADDRESS> src/Vault.sol:Vault
```

## Commit

```bash
git add .
git commit -m "feat: deploy to base sepolia"
```

## Next
Chapter6ではCoinbase WalletとdAppを接続する。
