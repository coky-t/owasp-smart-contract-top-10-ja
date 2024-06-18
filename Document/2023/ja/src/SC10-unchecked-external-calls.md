## 脆弱性: チェックされていない外部呼び出し (Vulnerability: Unchecked External Calls)

### 説明:
チェックされていない外部呼び出しとは、コントラクトが別のコントラクトやアドレスへの外部呼び出しを、その呼び出しの結果を適切にチェックせずに行うセキュリティ上の欠陥を指します。Ethereum では、コントラクトが別のコントラクトを呼び出すと、呼び出されたコントラクトは例外をスローせずにサイレントに失敗する可能性があります。呼び出し元のコントラクトが戻り値をチェックしない場合、呼び出しが成功していなくても、呼び出しが成功したと誤って判断するかもしれません。これにより、コントラクトの状態に不整合が生じ、攻撃者が悪用できる脆弱性が発生する可能性があります。

### 事例:
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.4.24;

contract Proxy {
    address public owner;

    constructor() public {
        owner = msg.sender;
    }

    function forward(address callee, bytes _data) public {
        require(callee.delegatecall(_data));
    }
}
```
### 影響:
- チェックされていない外部呼び出しにより、トランザクションが失敗し、意図された操作が正しく完了しない可能性があります。送金が成功したという誤った判断の下でコントラクトが進行するため、資金の損失につながる可能性があります。さらに、コントラクトの状態が不正確になり、コントラクトがさらなる悪用に脆弱となったり、そのロジックに不整合が生じる可能性もあります。

### 対策:
- transfer() は外部呼び出しが失敗した場合にトランザクションを元に戻すため、可能な限り send() ではなく transfer() を使用します。
- send() や call() 関数の戻り値を常にチェックして、false を返した場合に適切な処理が行われるようにします。

### チェックされていない外部呼び出しの被害を受けたスマートコントラクトの事例:
1. [Punk Protocol ハック](https://github.com/PunkFinance/punk.protocol/blob/master/contracts/models/CompoundModel.sol) : 包括的な [ハック分析](https://blog.solidityscan.com/security-issues-with-delegate-calls-4ae64d775b76)
