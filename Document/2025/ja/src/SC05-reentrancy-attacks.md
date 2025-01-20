## SC05:2025 - Reentrancy

### 説明:
再入攻撃は、関数が別のコントラクトへの外部呼び出しを行う際に、それ自体の状態を更新する前に、スマートコントラクトの脆弱性を悪用します。これは、悪意のある可能性がある外部コントラクトが元の関数に再入し、同じ状態を使用して、引き落としなどの特定のアクションを繰り返すことができます。このような攻撃により、攻撃者はコントラクトからすべての資金を流出できる可能性があります。

### 事例 (脆弱なコントラクト): 
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_Reentrancy {
    mapping(address => uint) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint amount = balances[msg.sender];
        require(amount > 0, "Insufficient balance");

        // Vulnerability: Ether is sent before updating the user's balance, allowing reentrancy.
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");

        // Update balance after sending Ether
        balances[msg.sender] = 0;
    }
}
```
### 影響:
- 最も直接的で影響のある結果は資金の流出です。攻撃者は脆弱性を悪用して、権利を超える資金を引き落とし、コントラクトの残高を完全に空にする可能性があります。
- 攻撃者は認可されていない関数呼び出しを引き起こすことができます。これにより、コントラクトや関連するシステム内で意図しないアクションが実行される可能性があります。

### 対策:
- 外部コントラクトを呼び出す前に、必ずすべての状態変更が発生するようにします。つまり、外部コードを呼び出す前に、内部で残高やコードを更新します。
- Open Zepplin の Re-entrancy Guard など、再入可能性を防ぐ関数修飾子を使用します。

### 事例 (修正バージョン):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_Reentrancy {
    mapping(address => uint) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint amount = balances[msg.sender];
        require(amount > 0, "Insufficient balance");

        // Fix: Update the user's balance before sending Ether
        balances[msg.sender] = 0;

        // Then send Ether
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### 再入攻撃の被害を受けたスマートコントラクトの事例:
1. [Rari Capital](https://etherscan.io/address/0xe16db319d9da7ce40b666dd2e365a4b8b3c18217#code) : 包括的な [ハック分析](https://blog.solidityscan.com/rari-capital-re-entrancy-vulnerability-analysis-25df2bbfc803)
2. [Orion Protocol](https://etherscan.io/address/0x98a877bb507f19eb43130b688f522a13885cf604#code) : 包括的な [ハック分析](https://blog.solidityscan.com/orion-protocol-hack-analysis-missing-reentrancy-protection-f9af6995acb3)
