## SC10:2025 - サービス拒否 (Denial Of Service)

### 説明:
Solidity のサービス拒否 (DoS) 攻撃は脆弱性を悪用して、ガス、CPU サイクル、ストレージなどのリソースを使い果たし、スマートコントラクトを使用不能にします。一般的なタイプには、悪意のあるアクターが過剰なガスを必要とするトランザクションを作成するガス枯渇攻撃、コントラクト呼び出しシーケンスを悪用して認可されていない資金にアクセスする再入攻撃、ブロックガスを消費して正当なトランザクションを妨害するブロックガス制限攻撃などがあります。

### 事例 (脆弱なコントラクト):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Solidity_DOS {
    address public king;
    uint256 public balance;

    function claimThrone() external payable {
        require(msg.value > balance, "Need to pay more to become the king");

        //If the current king has a malicious fallback function that reverts, it will prevent the new king from claiming the throne, causing a Denial of Service.
        (bool sent,) = king.call{value: balance}("");
        require(sent, "Failed to send Ether");

        balance = msg.value;
        king = msg.sender;
    }
}
```
### 影響:
- DoS 攻撃が成功すると、スマートコントラクトが応答不能になり、ユーザーが意図したとおりにやり取りできなくなります。これによりコントラクトに依存している重要な操作やサービスが中断される可能性があります。
- DoS 攻撃は、特にスマートコントラクトが資金や資産を管理する分散型アプリケーション (dApps) において、金銭的損失につながる可能性があります。
- DoS 攻撃は、スマートコントラクトとそれに関連するプラットフォームの評判を傷つける可能性があります。ユーザーはプラットフォームのセキュリティと信頼性に対する信頼を失い、ユーザーとビジネス機会の喪失につながる可能性があります。

### 対策:
- スマートコントラクトが、失敗する可能性のある外部呼び出しの非同期処理など、失敗を一貫して処理できるようにし、コントラクトの完全性を維持し、予期しない動作を防止します。
- 外部呼び出し、ループ、トラバーサルに `call` を使用する際は慎重に行い、トランザクションの失敗や予期しないコストにつながる可能性のある過剰なガス消費を避けます。
- コントラクトのパーミッションで単一のロールに過剰な認可を与えないでください。代わりに、パーミッションを合理的に分割し、重要なパーミッションを持つロールにはマルチシグナルウォレットを使用して、秘密鍵の侵害によるパーミッション喪失を防止します。

### 事例 (修正バージョン):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Solidity_DOS {
    address public king;
    uint256 public balance;

    // Use a safer approach to transfer funds, like transfer, which has a fixed gas stipend.
    // This avoids using call and prevents issues with malicious fallback functions.
    function claimThrone() external payable {
        require(msg.value > balance, "Need to pay more to become the king");

        address previousKing = king;
        uint256 previousBalance = balance;

        // Update the state before transferring Ether to prevent reentrancy issues.
        king = msg.sender;
        balance = msg.value;

        // Use transfer instead of call to ensure the transaction doesn't fail due to a malicious fallback.
        payable(previousKing).transfer(previousBalance);
    }
}
```
