## SC08:2026 - 再入攻撃 (Reentrancy Attacks)

#### 説明

再入可能性は、スマートコントラクトが外部呼び出し (他のコントラクトやアドレスへの呼び出し) を行う際、最初の呼び出しが完了して状態が完全に更新される前に、呼び出し先が元のコントラクトに **呼び戻す** ことができる状況を指します。呼び出し元がリエントランシーセーフであるように設計されていない場合、そのコールバックは古い状態を観測して悪用する可能性があります。たとえば、呼び出し元の残高以上の引き落とし、報酬の二重計上、複雑な多段階フローにわたる会計処理の操作があります。

これは外部呼び出しを行うあらゆるコントラクタタイプに影響を及ぼします。これには、DeFi (トークン転送、DEX スワップ、vault の預け入れ/引き落とし、フラッシュローンコールバック)、NFT (ERC-721/ERC-1155 レシーバーフックを伴う転送、マーケットプレイスでの支払い)、DAO (外部コントラクトを呼び出す提案の実行)、ブリッジ (メッセージリレー、資産転送)、コンポーザブルプロトコル (ERC-777 フック、ERC-4626 預け入れ/引き落としフック) があります。再入可能性は **シングルファンクション** (同じ関数が再帰的に呼び出される)、**クロスファンクション** (異なる関数へのコールバック)、**クロスコントラクト** (コールバックが複数のコントラクトをまたぐ) があります。非 EVM チェーンでは、プログラム間呼び出しが再帰する可能性のあるいずれにおいても、同様のパターンが存在します。

注目する領域は以下のとおりです。

- **従来の更新前引き落とし** (外部呼び出し後の状態変更)
- **コールバックとフックインタフェース** (ERC-777、ERC-721/1155 レシーバ、ERC-4626、フラッシュローンコールバック)
- **関数間の再入可能性** (関数間の書き込み後読み取りの想定)
- **読み出し専用の再入可能性** (コールバックの中で状態を読み取る関数やオラクルの閲覧)
- **コントラクト間およびマルチモジュール** の再入可能性 (vault → strategy → DEX フロー)

攻撃者は以下を悪用します。

- 転送/フックのコールバック上で再入する **悪意のあるトークンやレシーバ**
- 返済前に攻撃者ロジックを実行する **フラッシュローンコールバック**
- 状態がトランザクション途中のモジュール間で不整合となる **複雑な呼び出しグラフ**

### 事例 (脆弱な再入可能性パターン)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IToken {
    function transfer(address to, uint256 amount) external returns (bool);
}

contract VulnerableVault {
    IToken public immutable token;
    mapping(address => uint256) public balances;

    constructor(IToken _token) {
        token = _token;
    }

    function deposit(uint256 amount) external {
        // Assume token already transferred in for brevity
        balances[msg.sender] += amount;
    }

    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "insufficient");

        // External call before state update – reentrancy window
        token.transfer(msg.sender, amount);

        balances[msg.sender] -= amount;
    }
}
```

**問題点:**

- 悪意のあるトークンやプロキシを使用する攻撃者は `transfer` 呼び出し内から `withdraw` を再入し、変更されていない `balances[msg.sender]` に基づいて繰り返し引き落としできます。

### 事例 (再入安全な引き落とし)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

interface ITokenSafe {
    function transfer(address to, uint256 amount) external returns (bool);
}

contract SafeVault is ReentrancyGuard {
    ITokenSafe public immutable token;
    mapping(address => uint256) public balances;

    error InsufficientBalance();
    error TransferFailed();

    constructor(ITokenSafe _token) {
        token = _token;
    }

    function deposit(uint256 amount) external {
        // Assume token already transferred in for brevity
        balances[msg.sender] += amount;
    }

    function withdraw(uint256 amount) external nonReentrant {
        uint256 bal = balances[msg.sender];
        if (bal < amount) revert InsufficientBalance();

        // Effects first
        balances[msg.sender] = bal - amount;

        // Then interaction
        bool ok = token.transfer(msg.sender, amount);
        if (!ok) revert TransferFailed();
    }
}
```

**セキュリティの改善:**

- OpenZeppelin の `ReentrancyGuard` から `nonReentrant` 修飾子を使用します。
- **checks-effects-interactions** パターンを適用して、再入ウィンドウを最小限に抑えます。
- transfer の戻り値をチェックし、失敗した場合には元に戻します。

### 2025 ケーススタディ

- **GMX (2025 年 7 月, 4200 万ドルの損失)**  
  GMX V1 のコントラクトは `executeDecreaseOrder` の古典的だが洗練された再入ベクトルを介して悪用されました。その関数は攻撃者のスマートコントラクトアドレスをパラメータとして受け入れていましたが、払い戻し処理時にそのアドレスに制御を移された際、攻撃者は再入して、グローバル平均ショート価格、AUM、GLP 評価額を操作しました。外部呼び出し後の状態更新と再入ガードの欠如が流出を可能にしました。この脆弱性は 2022 年に未監査のパッチとして導入されました。
  - [https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f](https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f)
  - [https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025](https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025)
  - [https://blog.verichains.io/p/gmx-42m-exploit-root-cause-analysis](https://blog.verichains.io/p/gmx-42m-exploit-root-cause-analysis)

### ベストプラクティスと緩和策

- Use **`ReentrancyGuard`** or similar reentrancy locks on stateful functions that:
  - Modify balances or internal accounting.
  - Perform external calls that could re-enter the contract.
- Follow **checks-effects-interactions**:
  - Check preconditions.
  - Apply all state changes.
  - Only then call external contracts.
- Treat **ERC-777 hooks, ERC-4626 hooks, and other callbacks** as reentrancy vectors.
- Carefully review:
  - Cross-function interactions (e.g., `deposit` calling `withdraw` internally).
  - Multi-contract systems where reentrancy may occur across modules, not just within a single contract.
- Include reentrancy-focused **fuzzing and unit tests**, especially where external calls are involved.
