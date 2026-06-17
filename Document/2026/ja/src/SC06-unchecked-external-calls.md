## SC06:2026 - チェックされていない外部呼び出し (Unchecked External Calls)

#### 説明

チェックされていない外部呼び出しは、スマートコントラクトが別のコントラクトまたはアドレスを呼び出す場合 (`call`, `delegatecall`, `staticcall`, または `transfer`/`send` などの高レベル呼び出しを介して)、呼び出し先の動作、戻り値、再入可能性を **十分に考慮せずに** 呼び出す状況を指します。呼び出し元のコントラクトは、呼び出し先が期待どおりに動作すること (成功を返し、再入せず、任意のロジックを実行しないこと) を暗黙的に信頼しています。この前提が破られた場合、呼び出し元は矛盾した状態に陥ったり、悪用される可能性があります。

これは、外部とのやり取りを行うすべてのコントラクトタイプに影響します。DeFi (トークン転送、DEX スワップ、ボルト預金、フラッシュローンコールバック)、NFT (フック付き転送、マーケットプレイス支払い)、DAO (提案コールデータの実行)、ブリッジ (メッセージリレー、資産転送)、構成可能プロトコル (任意のコールバック、ERC-777/ERC-721/ERC-1155 レシーバーフック、ERC-4626 預金/出金フック) があります。非 EVM チェーンにも、同様のパターンが存在し (例: Move の `vector::borrow`, Solana CPI)、プログラム間呼び出しが再実行したり、予期しない動作をする可能性があります。

注目する領域は以下のとおりです。

- **トークン転送** (ERC-20, ERC-721, ERC-1155) および非標準の戻り値や元に戻す動作
- **コールバックおよびフックインタフェース** (ERC-777 `tokensReceived`, ERC-4626 `afterDeposit`/`beforeWithdraw`, `onFlashLoan`, `onERC721Received`)
- **低レベル呼び出し** (`call`, `delegatecall`, `callcode`) およびガス/ストレージへの影響
- 再入可能性が複数のコントラクトにまたがる **構成可能性フロー** (ボルト呼び出しストラテジー、ストラテジー呼び出し DEX)

攻撃者は以下を悪用します。

- 転送にフックするコールバックやトークンに悪意のあるロジックを実装することによる **再入可能性** (SC08 参照)
- 戻り値が無視され (リターンしない ERC-20 など)、状態が不整合なままである場合の **静かな失敗**
- ユーザーが提供したアドレスやプロトコルで設定可能なアドレスを呼び出す際の **予期しないコード実行**

チェックされていない外部呼び出しは *唯一の* 根本原因となることは稀ですが、再入可能性 (SC08)、ビジネスロジックの悪用 (SC02)、および会計上の不整合の **重大な誘因** となります。

### 事例 (脆弱なチェックされていない呼び出しパターン)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IToken {
    function transfer(address to, uint256 amount) external returns (bool);
}

contract VulnerablePayout {
    IToken public token;

    mapping(address => uint256) public rewards;

    constructor(IToken _token) {
        token = _token;
    }

    function claim() external {
        uint256 amount = rewards[msg.sender];
        require(amount > 0, "no rewards");

        // Vulnerable: does not check return value or reentrancy
        token.transfer(msg.sender, amount);

        // State update after external call
        rewards[msg.sender] = 0;
    }
}
```

**問題点:**

- `transfer` 呼び出しの戻り値は無視されます。転送が失敗した場合、報酬は非ゼロになりますが、ユーザーはトークンを受け取りません。
- 状態は外部呼び出しの **後** に更新され、(トークンが悪意のあるものの場合) 再入の可能性を開きます。

### 事例 (堅牢化された外部呼び出し処理)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface ISafeToken {
    function transfer(address to, uint256 amount) external returns (bool);
}

contract SafePayout {
    ISafeToken public immutable token;
    mapping(address => uint256) public rewards;

    error NoRewards();
    error TransferFailed();

    constructor(ISafeToken _token) {
        token = _token;
    }

    function claim() external {
        uint256 amount = rewards[msg.sender];
        if (amount == 0) revert NoRewards();

        // Move state change *before* external call to mitigate reentrancy on this variable
        rewards[msg.sender] = 0;

        bool ok = token.transfer(msg.sender, amount);
        if (!ok) {
            // revert and restore state if needed (not shown here for brevity)
            revert TransferFailed();
        }
    }
}
```

**セキュリティの改善:**

- 状態が外部呼び出しの **前** に更新され、`rewards` に単純な再入可能性を制限します。
- トークン転送の戻り値がチェックされます。失敗した場合には元に戻り、サイレントな不整合を防ぎます。

> 注: 完全な再入可能性保護については、SC08 を参照し、`ReentrancyGuard`、checks-effects-interactions、プルベースのパターンを検討してください。

### 2025 ケーススタディ

- **GMX (July 2025, $42M loss)**  
  Unsafe external interactions and state updates after external calls allowed attackers to re-enter and manipulate accounting. The `executeDecreaseOrder` function transferred control to an attacker-supplied contract address during the refund process, enabling reentrancy. External call ordering, lack of proper checks, and reliance on assumptions about callee behavior amplified the impact.  
  - [https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f](https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f)
  - [https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025](https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025)

- **Arcadia Finance (July 2025, $3.5M loss)**  
  The `SwapLogic._swapRouter()` and `RebalancerSpot` contracts allowed **arbitrary external calls** to user-supplied router addresses via `swapData` parameters, without validating the callee. The attacker registered a malicious contract as both the router and a whitelisted ArcadiaAccount, then used the router callback to spoof privileged execution contexts and invoke `setAssetManager()` / `flashAction()`. The protocol assumed the router would not have elevated permissions—an assumption not enforced in code. Unchecked callbacks and trust in external callee behavior enabled the drain.  
  - [https://blog.solidityscan.com/arcadia-finance-hack-analysis-a03a722e554d](https://blog.solidityscan.com/arcadia-finance-hack-analysis-a03a722e554d)
  - [https://www.guardrail.ai/blog/arcadia-finance-hack-july-2025](https://www.guardrail.ai/blog/arcadia-finance-hack-july-2025)

### ベストプラクティスと緩和策

- **Treat all external calls as untrusted**:
  - Even “standard” tokens or well-known protocols can be upgraded or replaced.
  - Assume they may re-enter or revert unexpectedly.
- Use the **checks-effects-interactions** pattern:
  - Validate pre-conditions.
  - Update internal state.
  - Only then perform external calls.
- Prefer **pull** over **push** for payments:
  - Allow users to withdraw rather than pushing funds to arbitrary addresses in loops.
- Check return values and handle all failure modes:
  - Use libraries like OpenZeppelin’s `SafeERC20` to wrap token operations.
- Be extremely careful with:
  - Low-level calls (`call`, `delegatecall`, `callcode`)
  - Arbitrary callbacks (e.g., hooks, onERC721Received, onFlashLoan callbacks)
