# SC04:2025 - 入力バリデーションの欠如 (Lack of Input Validation)

## 説明:
入力バリデーションはスマートコントラクトが有効で期待されるデータのみを処理することを確保します。コントラクトが受信した入力の検証を怠ると、ロジックの操作、不正アクセス、予期しない動作などのセキュリティリスクに不注意にさらされることになります。たとえば、コントラクトがユーザー入力を検証なしで常に有効であると想定している場合、攻撃者はこの信頼を悪用して悪意のあるデータを持ち込むことができます。この入力バリデーションの欠如はスマートコントラクトのセキュリティと信頼性を損ないます。

## 事例 (脆弱なコントラクト):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_LackOfInputValidation {
    mapping(address => uint256) public balances;

    function setBalance(address user, uint256 amount) public {
        // The function allows anyone to set arbitrary balances for any user without validation.
        balances[user] = amount;
    }
}
```
### 影響:
- 攻撃者は入力を操作して資金を流出したり、トークンを盗んだり、その他の金銭的損害を引き起こす可能性があります。
- 不適切な入力によって状態変数が破損し、信頼できなく安全でないコントラクタの動作につながる可能性があります。
- 攻撃者はコントラクトを悪用して不正な取引や操作を実行し、ユーザーとシステム全体に影響を及ぼす可能性があります。

### 対策:
- 入力が期待したタイプに適合していることを確認します。
- 入力が許容範囲内にあることを確認します。
- 認可されたエンティティのみが特定の関数を呼び出せることを確認します。
- アドレス形式や文字列長など、入力の構造を確認します。
- 入力がバリデーションに失敗した場合には、常に実行を停止し、明確なエラーメッセージを表示します。

### 事例 (修正バージョン):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract LackOfInputValidation {
    mapping(address => uint256) public balances;
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Caller is not authorized");
        _;
    }

    function setBalance(address user, uint256 amount) public onlyOwner {
        require(user != address(0), "Invalid address");
        balances[user] = amount;
    }
}
```
### 入力バリデーションの欠如の被害を受けたスマートコントラクトの事例:
1. [Convergence Finance](https://etherscan.io/address/0x2b083beaaC310CC5E190B1d2507038CcB03E7606#code) : 包括的な [ハック分析](https://blog.solidityscan.com/convergence-finance-hack-analysis-12e6acd9ea08)
2. [Socket Gateway](https://etherscan.io/address/0x3a23F943181408EAC424116Af7b7790c94Cb97a5#code) : 包括的な [ハック分析](https://blog.solidityscan.com/socket-gateway-hack-analysis-b0e9567f7d3e)
