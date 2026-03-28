## SC02:2026 - ビジネスロジックの脆弱性 (Business Logic Vulnerabilities)

#### 説明

ビジネスロジックの脆弱性は、個々の低レベルチェック (型安全性、再入防止、アクセス制御など) が正しいにもかかわらず、スマートコントラクトの *意図した* 経済的または機能的な動作が損なわれる可能性のあるあらゆる状況を指します。これらは、システムのルール、インセンティブ、状態遷移、不変条件がオンチェーンでどのようにモデル化されているかにおける **設計上の欠陥** です。低レベルバグ (オーバーフロー、再入可能性) とは異なり、ビジネスロジックの欠陥はルール自体が安全でない場合に発生します。コードは「指示どおりに動作する」ものの、それは悪用可能な結果を許すことになります。

これは、DeFi (融資、AMM、Vault、利回り戦略)、NFT (発行ロジック、ロイヤリティ、マーケットプレイス機構)、DAO (投票、委譲、提案実行)、ブリッジ (burn/mint の非対称性、流動性ルール)、ゲーミング (報酬分配、公平性)、マルチホップの状態遷移が新たな脆弱性を生み出すクロスチェーン/L2 システムなど、すべてのスマートコントラクトの領域にわたって適用します。

注目する領域は以下のとおりです。

- モジュール間における **不変条件違反** (例: vault ↔ strategy ↔ gauge, collateral ↔ debt, supply ↔ backing)
- **報酬および手数料のロジック** (double-counting, under/over-accrual, wrong beneficiary)
- バイパス可能または一貫性のない適用された **適格性および制限のチェック** (借入上限、発行制限、換金閾値)
- 操作的なアクションシーケンスが不可能または一貫性のない状態に到達できる **パス依存ステートマシン**
- **モジュール間およびチェーン間の前提条件** (例: ブリッジの流動性、L2 ファイナリティ、メッセージの順序付け)

攻撃者は以下のように悪用します。

- **一貫性のない会計処理間での裁定取引** (vault 対 strategy、内部残高と外部残高)
- **操作順序のエッジケース** (不変条件を破る預け入れ/引き落とし/請求シーケンス)
- **適格性のバイパス** (例: 持ち分なしでの報酬の請求、健全なポジションの債務弁済)
- 経済的に不合理な状態を生み出す **パラメータ操作** (金利曲線、手数料、担保係数)

### 事例 (脆弱な融資ロジック)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract VulnerableLending {
    mapping(address => uint256) public collateral;
    mapping(address => uint256) public debt;
    uint256 public collateralFactorBps = 7500; // 75%

    function depositCollateral() external payable {
        collateral[msg.sender] += msg.value;
    }

    // Vulnerable: calculates borrow capacity using the *new* amount, not total
    function borrow(uint256 amount) external {
        uint256 allowed = (amount * collateralFactorBps) / 10_000;
        require(allowed >= amount, "not enough collateral"); // meaningless check

        debt[msg.sender] += amount;
        // send tokens from pool (omitted)
    }
}
```

**問題点:**

- `allowed` は、ユーザーの担保残高ではなく、*借入希望額* から計算されています。
- `allowed >= amount` のチェックは `collateralFactorBps >= 10_000` では常に成り立ち、このように誤用されると単なる恒真式となり、経済的な不変条件を強制できません。

### 事例 (修正: 不変式ベースの借用ロジック)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IPriceOracle {
    function getCollateralPrice() external view returns (uint256); // 1e18
}

contract SaferLending {
    mapping(address => uint256) public collateral;
    mapping(address => uint256) public debt;
    uint256 public collateralFactorBps = 7500; // 75%
    IPriceOracle public oracle;

    constructor(IPriceOracle _oracle) {
        oracle = _oracle;
    }

    function depositCollateral() external payable {
        require(msg.value > 0, "no collateral");
        collateral[msg.sender] += msg.value;
    }

    function _maxBorrow(address user) internal view returns (uint256) {
        uint256 price = oracle.getCollateralPrice(); // e.g. ETH price in USD 1e18
        uint256 collateralUsd = (collateral[user] * price) / 1e18;
        return (collateralUsd * collateralFactorBps) / 10_000;
    }

    function borrow(uint256 amountUsd) external {
        uint256 maxBorrowUsd = _maxBorrow(msg.sender);
        require(debt[msg.sender] + amountUsd <= maxBorrowUsd, "exceeds limit");

        debt[msg.sender] += amountUsd;
        // transfer stablecoin from pool (omitted)
    }
}
```

**セキュリティの改善:**

- 借入限度額は、希望額だけでなく、**担保総額** から導き出しています。
- 経済不変条件: `debt[user] <= maxBorrow(user)` はすべての借入に適用されます。
- 価格オラクルは明示的に統合されています (そして SC03 で堅牢化できます)。

### 2025 ケーススタディ

- **Abracadabra (March 2025, $12.9M loss)**  
  Flawed collateral accounting in GMX V2 CauldronV4 contracts. Attackers used a three-stage method: (1) made a deposit into GMX designed to fail, leaving tokens stuck in OrderAgent; (2) self-liquidated their position so the contract erased the position but failed to remove the associated order and collateral; (3) used the "ghost" collateral to borrow 6,260 ETH (~$12.9M). Economic invariants (collateral ↔ debt) were broken by the deposit-fail and liquidation path.  
  Key lessons:
  - All economic invariants (e.g., minimum collateralization, max LTV) must be **provable and enforced** on every state transition.
  - Introducing new spell/strategy types must go through **formal review** of how they interact with the existing system.  
  - [https://blog.solidityscan.com/abracadabra-hack-analysis-f2efcdee9c05](https://blog.solidityscan.com/abracadabra-hack-analysis-f2efcdee9c05)
  - [https://www.halborn.com/blog/post/explained-the-abracadabra-money-hack-march-2025](https://www.halborn.com/blog/post/explained-the-abracadabra-money-hack-march-2025)
  - [https://www.coindesk.com/business/2025/03/25/abracadabra-drained-of-usd13m-in-exploit-targeting-cauldrons-tied-to-gmx-liquidity-tokens](https://www.coindesk.com/business/2025/03/25/abracadabra-drained-of-usd13m-in-exploit-targeting-cauldrons-tied-to-gmx-liquidity-tokens)
  - [https://threesigma.xyz/blog/exploit/abracadabra-gmx-defi-exploit-explained](https://threesigma.xyz/blog/exploit/abracadabra-gmx-defi-exploit-explained)

- **Yearn Finance (November 2025, $9M loss)**  
  A design flaw in the yETH weighted stableswap pool's fixed-point iteration solver. By performing highly imbalanced add/remove liquidity operations, attackers forced the solver into a divergent regime, causing the product term (Π) to collapse to zero—converting the pool from a hybrid stableswap invariant to a constant-sum curve. This allowed minting ~2.35×10⁵⁶ yETH LP tokens without collateral and draining the pool. The vulnerability was **invariant collapse**, not reentrancy or low-level bugs.  
  Key lessons:
  - Reward and fee distribution paths must be **simulation-tested** across adversarial scenarios.
  - Yield strategies should have **clear, testable invariants** (e.g., no user can claim more rewards than their fair share over time).  
  - [https://defimon.xyz/blog/yearn-yeth-hack-november-2025](https://defimon.xyz/blog/yearn-yeth-hack-november-2025)
  - [https://www.cryptopolitan.com/yearn-finance-begins-clawback-after-9m-hack/](https://www.cryptopolitan.com/yearn-finance-begins-clawback-after-9m-hack/)

### ベストプラクティスと緩和策

- **Model protocol economics explicitly** (e.g., with adversarial simulations / agent-based models) rather than relying on intuition.
- Express **core invariants in code and tests**:
  - “Total value withdrawn cannot exceed total deposits + realized yield”
  - “Rewards distribution is proportional to time-weighted stake”
  - “Liquidations never result in protocol loss under honest oracle data”
- Use **formal verification and property-based fuzzing** for key accounting paths (vaults, strategies, reward distribution).
- **Version and gate new strategies / spells**:
  - Roll out behind caps.
  - Monitor metrics and on-chain invariants before raising limits.
- Ensure **governance and operations teams understand invariants**, not just auditors.
