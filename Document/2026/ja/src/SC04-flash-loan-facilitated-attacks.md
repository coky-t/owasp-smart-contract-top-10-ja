## SC04:2026 - フラッシュローンを利用した攻撃 (Flash Loan–Facilitated Attacks)

#### 説明

フラッシュローンを活用した攻撃は、攻撃者が **無担保の同一取引内借入** (フラッシュローン) を使用する悪用を指し、基盤となる脆弱性を増幅し、プロトコルを破綻するような利益を生む攻撃です。フラッシュローンは本質的には脆弱なものではなく、正当な DeFi プリミティブですが、攻撃者に単一の取引内で任意の規模の一時的な資金を付与します。*リスクにさらされている資本* や *過去のポジションサイズ* に基づいて信頼性を前提とするプロトコルは、攻撃者が自身の資金をリスクにさらすことなく一時的に巨額の残高を保持できる場合、危険にさらされます。

これは経済的制約や比例的エクスポージャーを前提とするすべてのコントラクトタイプに影響を及ぼします。それには、融資プロトコル (ガバナンス投票権、清算閾値、担保比率)、AMM (株式発行、裁定取引、オラクルスキュー)、イールドボールト (預金/出金/株式会計)、ガバナンス (投票権買収、フラッシュローンによるガバナンス攻撃)、NFT およびトークンの評価 (フロア価格、担保評価)、あるコントラクトの状態が別の流動性によって影響を受けるプロトコル間の構成可能性などがあります。EVM 以外のチェーンでも、同様の概念が存在します (たとえば、単一ブロックまたはバッチ内の一時的な大規模ポジションなど)。

注目する領域は以下のとおりです。

- **ガバナンスと投票** (フラッシュローンによる投票の買収、スナップショット操作)
- **オラクルと価格設定** (借入流動性での DEX/TWAP フィードの操作)
- **シェアと会計ロジック** (丸め処理、境界のある入力を想定した比例計算)
- **清算と担保チェック** (ポジションサイズに依存する閾値)
- **構成可能性** (単一のトランザクション内で呼び出し元の残高またはプール状態を信頼するプロトコル)

攻撃者は以下のようなバッチ処理トランザクションを構築します。

1. フラッシュローン (Aave, dYdX, Uniswap V3 フラッシュ, または同等のもの) を介して多額を借り入れる。
2. 借り入れた資金を使用して、プロトコルの状態、価格、または会計を操作する。
3. 利益を得る (流動性を枯渇する、担保不足のローンを得る、ガバナンスを歪める)。
4. 同じトランザクション内でフラッシュローンを返済し、利益を保持する。

ビジネスロジック (SC02)、オラクル (SC03)、算術演算 (SC07)、またはアクセス制御 (SC01) に根本的な弱点が存在する場合、フラッシュローンは **力を増幅する** 役割を果たし、小さなバグを壊滅的なエクスプロイトへと変貌します。

### 事例 (丸めバグを伴うフラッシュローンの脆弱な使用)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IFlashLender {
    function flashLoan(
        address receiver,
        uint256 amount,
        bytes calldata data
    ) external;
}

contract VulnerablePool {
    uint256 public totalShares;
    uint256 public totalAssets;

    mapping(address => uint256) public sharesOf;

    // Vulnerable: naive share minting with truncation benefit to sender
    function deposit(uint256 assets) external {
        uint256 shares;
        if (totalShares == 0) {
            shares = assets;
        } else {
            shares = (assets * totalShares) / totalAssets;
        }

        totalAssets += assets;
        totalShares += shares;
        sharesOf[msg.sender] += shares;
    }

    // No safeguard against flash-loan boosted deposit/withdraw loops
}
```

**Issues:**

- Rounding always truncates in favor of the protocol, but with repeated flash-loan-driven cycles, a mis-tuned formula or mis-accounted state can be turned into a net gain for an attacker.
- No consideration for **max slippage, caps, or frequency** of operations, making repeated high-volume operations feasible.

### 事例 (緩和した設計考慮)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SaferPool {
    uint256 public totalShares;
    uint256 public totalAssets;
    uint256 public lastUpdateBlock;

    mapping(address => uint256) public sharesOf;

    error TooFrequentInteraction();

    modifier rateLimited() {
        // Simple example: limit high-impact operations to once per block
        if (lastUpdateBlock == block.number) revert TooFrequentInteraction();
        _;
        lastUpdateBlock = block.number;
    }

    function deposit(uint256 assets) external rateLimited {
        require(assets > 0, "zero assets");

        uint256 shares;
        if (totalShares == 0) {
            shares = assets;
        } else {
            // Use rounding that errs in favor of the protocol and is formally analyzed
            shares = (assets * totalShares + totalAssets - 1) / totalAssets;
        }

        totalAssets += assets;
        totalShares += shares;
        sharesOf[msg.sender] += shares;
    }
}
```

**Security Improvements:**

- Introduces **basic rate limiting** to reduce susceptibility to rapid flash-loan loops.
- Uses a more conservative, explicitly documented rounding strategy.
- Encourages formal analysis of share/accounting formulas and invariants.

> Note: Real protocols should use stronger defense-in-depth mechanisms rather than relying solely on per-block limits.

### 2025 ケーススタディ

- **Bunni (September 2025, $8.4M loss)**  
  A rounding error in the withdrawal function was amplified by flash loans. Attackers flash-borrowed 3M USDT, pushed the pool's spot price to extremes (USDC active balance to 28 wei), then executed 44 chained tiny withdrawals exploiting rounding—decreasing USDC balance by 85.7% while only burning 84.4% of liquidity. Flash loans enabled the capital scale required.  
  - [https://www.halborn.com/blog/post/explained-the-bunni-hack-september-2025](https://www.halborn.com/blog/post/explained-the-bunni-hack-september-2025)
  - [https://blog.bunni.xyz/posts/exploit-post-mortem/](https://blog.bunni.xyz/posts/exploit-post-mortem/)
  - [https://cryptorank.io/news/feed/27dc8-bunni-hit-by-8-4m-flash-loan-exploit-rounding-error-blamed](https://cryptorank.io/news/feed/27dc8-bunni-hit-by-8-4m-flash-loan-exploit-rounding-error-blamed)

- **zkLend (February 2025, $9.5M loss)**  
  A rounding error in the `mint()` function (integer division rounding down) allowed attackers to inflate the lending_accumulator via repeated deposits/withdrawals. Flash loans scaled the position—turning small per-iteration precision gains into a ~$9.5M drain. Flash loans were the **force multiplier**.  
  - [https://blog.solidityscan.com/zklend-hack-analysis-e494cb794f71](https://blog.solidityscan.com/zklend-hack-analysis-e494cb794f71)
  - [https://www.halborn.com/blog/post/explained-the-zklend-hack-february-2025](https://www.halborn.com/blog/post/explained-the-zklend-hack-february-2025)
  - [https://zircon.tech/blog/the-9-5m-zklend-hack-another-defi-security-wake-up-call/](https://zircon.tech/blog/the-9-5m-zklend-hack-another-defi-security-wake-up-call/)

### ベストプラクティスと緩和策

- **Assume flash loans exist**: design economic and accounting logic on the assumption that arbitrarily large, transient capital is available to attackers.
- **Rate limit sensitive operations**:
  - Per-block or per-epoch limits on rebasing, rebalancing, or high-impact state transitions.
  - Dynamic fees that increase with the magnitude/frequency of actions.
- **Cap exposure per interaction**:
  - Set maximum slippage, max position sizes, and borrowing caps.
  - Limit how much state can change in a single transaction.
- **Simulate flash loan scenarios**:
  - Include flash-loan-style tests in QA and audits.
  - Use fuzzing to discover profitable multi-call sequences.
- **Combine with strong oracles and logic**:
  - Flash loans are usually the *multiplier*; underlying issues (SC02, SC03, SC07) must be fixed at the root.
