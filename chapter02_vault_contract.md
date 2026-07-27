# Chapter 2: Vault Contract

## Goal
USDCを預ける基礎となるVaultスマートコントラクトを作成する。

## アーキテクチャ
```text
User -> Vault -> (将来 Aave/Morpho)
```

## ディレクトリ
```text
contracts/
 ├─ src/
 │   └─ Vault.sol
 ├─ test/
 │   └─ Vault.t.sol
 └─ script/
     └─ Deploy.s.sol
```

## Vault.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract Vault {
    IERC20 public immutable asset;
    mapping(address=>uint256) public balance;

    event Deposited(address indexed user,uint256 amount);
    event Withdrawn(address indexed user,uint256 amount);

    constructor(address asset_){
        asset=IERC20(asset_);
    }

    function deposit(uint256 amount) external {
        require(amount>0,"amount");
        asset.transferFrom(msg.sender,address(this),amount);
        balance[msg.sender]+=amount;
        emit Deposited(msg.sender,amount);
    }

    function withdraw(uint256 amount) external {
        require(balance[msg.sender]>=amount,"balance");
        balance[msg.sender]-=amount;
        asset.transfer(msg.sender,amount);
        emit Withdrawn(msg.sender,amount);
    }
}
```

## テスト方針
- Deposit成功
- Withdraw成功
- 残高不足でRevert

## ビルド

```bash
cd contracts
forge install OpenZeppelin/openzeppelin-contracts
forge build
forge test
```

## デプロイ

```bash
forge script script/Deploy.s.sol \
 --rpc-url $BASE_RPC_URL \
 --broadcast
```

## Security
- ReentrancyGuard追加予定
- SafeERC20へ変更予定
- AccessControl追加予定

## Commit

```bash
git add .
git commit -m "feat: implement vault contract"
```
