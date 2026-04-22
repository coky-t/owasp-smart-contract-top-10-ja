## SC03:2026 - 価格オラクル操作 (Price Oracle Manipulation)

#### 説明

価格オラクル操作は、スマートコントラクトが **攻撃者によって直接的または間接的に影響を受ける** 可能性のある価格データや評価データに依存している状況を指し、プロトコルが誤った値に基づいて決定を行うことになります。オラクルは信頼の境界であり、コントラクトは受け取る価格が現実世界またはオンチェーンの市場状況を反映していると暗黙的に信頼します。その信頼が、操作、陳腐化、または設定ミスによって侵害されると、プロトコルの動作が歪められます。

これは価格データを使用するすべてのコントラクトタイプに影響します。DeFi の貸借 (担保評価、清算)、AMM と DEX (スポットと TWAP ベースの価格設定)、Yield Vault (NAV 計算、株式評価)、流動性ステーキングとデリバティブ (ETH/ステーク価格フィード)、NFT とトークンの評価 (フロア価格オラクル)、クロスチェーンブリッジ (ミント/バーン比率に基づく資産価格設定) などです。EVM 以外のチェーン (例: Move, Solana) においても、外部価格ソースがオンチェーンロジックにフィードする箇所で、同様のパターンが適用します。

注目する領域は以下のとおりです。

- **DEX ベースのオラクル** (スポット価格、TWAP、幾何平均) およびフラッシュローン、JIT 流動性、集中流動性スキュー
- **オフチェーンとハイブリッドフィード** (Chainlink, Pyth, カスタムリレイヤー) および、鮮度、偏差、マルチソース集約に関する想定
- 基となる価格ソースの **流動性と市場の深さ** (薄いプール vs. 深い市場)
- **クロスチェーンと L2 価格設定** (ファイナリティ遅延、シーケンサーの順序付け、メッセージリレーに想定)

攻撃者は以下のように悪用します。

- 同一ブロックでの大口取引、フラッシュローン、JIT 流動性による **スポット価格の操作**
- 短期間または低流動性期間における **TWAP の操作**
- コントラクトが鮮度維持やフォールバック動作を強制しない場合の **データの陳腐化や滞留**
- 集計ロジックが操作された入力を拒否できない場合の **逸脱と外れ値の処理**

### 事例 (脆弱なオラクル使用)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IPriceFeed {
    function latestAnswer() external view returns (int256);
}

contract VulnerableOracleLending {
    IPriceFeed public priceFeed; // single-point oracle
    mapping(address => uint256) public collateralEth;
    mapping(address => uint256) public debtUsd;

    constructor(IPriceFeed _feed) {
        priceFeed = _feed;
    }

    function depositCollateral() external payable {
        collateralEth[msg.sender] += msg.value;
    }

    function borrow(uint256 amountUsd) external {
        int256 price = priceFeed.latestAnswer(); // no sanity checks, no delay
        require(price > 0, "bad price");

        uint256 collateralUsd = (collateralEth[msg.sender] * uint256(price)) / 1e8;
        // Allows borrowing up to 100% of collateral value – overly generous
        require(collateralUsd >= amountUsd, "insufficient collateral");

        debtUsd[msg.sender] += amountUsd;
        // transfer stablecoin (omitted)
    }
}
```

**問題点:**

- 単一のオラクルソースであり、集計や健全性チェックがありません。
- 上限/下限境界がなく、過去の値との逸脱チェックがありません。
- 経済パラメータ (100% LTV) はわずかな操作ですら利益を生み出します。

### 事例 (堅牢化したオラクル統合)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IAggregatorV3 {
    function latestRoundData()
        external
        view
        returns (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        );
}

contract RobustOracleLending {
    IAggregatorV3 public immutable priceFeed;
    mapping(address => uint256) public collateralEth;
    mapping(address => uint256) public debtUsd;

    uint256 public constant MAX_DELAY = 1 hours;
    uint256 public constant COLLATERAL_FACTOR_BPS = 7500; // 75%

    constructor(IAggregatorV3 _feed) {
        priceFeed = _feed;
    }

    function _getSafePrice() internal view returns (uint256) {
        (, int256 answer, , uint256 updatedAt, uint80 answeredInRound) =
            priceFeed.latestRoundData();
        require(answer > 0, "bad answer");
        require(updatedAt != 0 && block.timestamp - updatedAt <= MAX_DELAY, "stale price");
        require(answeredInRound != 0, "incomplete round");
        return uint256(answer);
    }

    function depositCollateral() external payable {
        require(msg.value > 0, "no collateral");
        collateralEth[msg.sender] += msg.value;
    }

    function borrow(uint256 amountUsd) external {
        uint256 price = _getSafePrice();
        uint256 collateralUsd = (collateralEth[msg.sender] * price) / 1e8;
        uint256 maxBorrow = (collateralUsd * COLLATERAL_FACTOR_BPS) / 10_000;
        require(debtUsd[msg.sender] + amountUsd <= maxBorrow, "exceeds limit");

        debtUsd[msg.sender] += amountUsd;
        // transfer stablecoin (omitted)
    }
}
```

**セキュリティの改善:**

- **ラウンドメタデータ** 付きの価格フィードインタフェースを使用して、古いデータや不完全なデータを拒否します。
- 保守的な **担保係数** と明示的な借入限度額を適用します。
- 価格のフェッチとバリデーションを `_getSafePrice` にカプセル化することで、推論とテストを容易にします。

2025 年には、純粋にオラクルのみでの大規模エクスプロイトは少なくなりましたが、オラクル操作は **マルチベクター攻撃** の一要素となることがよくありました。

### 2025 ケーススタディ

- **NGP Token (2025 年 9 月, 約 200 万ドルの損失)**  
  プロトコルの `getPrice()` 関数はトークン価格の計算に DEX ペア (Uniswap V2/PancakeSwap) の準備金残高のみに依存していました。攻撃者はフラッシュローンを使用して多額の資金をスワップし、準備金を操作してオラクルを人為的に低い値に歪め、購入制限とクールダウン保護をバイパスして約 200 万ドルを流出しました。オラクル操作が **根本原因** でした。
  - [https://blog.solidityscan.com/ngp-token-hack-analysis-414b6ca16d96](https://blog.solidityscan.com/ngp-token-hack-analysis-414b6ca16d96)
  - [https://hacken.io/insights/ngp-hack-explained/](https://hacken.io/insights/ngp-hack-explained/)
  - [https://coincentral.com/flash-loan-exploit-drains-2-million-from-ngp-token-on-bnb-chain/](https://coincentral.com/flash-loan-exploit-drains-2-million-from-ngp-token-on-bnb-chain/)

- **GMX (2025 年 7 月, 4200 万ドルの損失)**  
  主な根本原因は `executeDecreaseOrder` での再入可能性ですが、攻撃フローには併せて **価格フィード操作** を必要になります。攻撃者はビットコインのグローバル平均ショート価格を約 57 倍に引き下げ、その後フラッシュローンを使用して GLP を人為的に低い価格で購入し、高値で償還しました。オラクル/価格設定はエクスプロイトの重要な **実現要因** でした。
  - [https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f](https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f)
  - [https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025](https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025)
  - [https://blog.verichains.io/p/gmx-42m-exploit-root-cause-analysis](https://blog.verichains.io/p/gmx-42m-exploit-root-cause-analysis)

### ベストプラクティスと緩和策

- **Aggregate multiple sources**:
  - Use median/mean of several DEXs / oracles.
  - Reject outliers and anomalous deviations.
- **Time-based defenses**:
  - Use TWAPs over sufficient windows to resist short-lived manipulations.
  - Reject prices older than a maximum staleness threshold.
- **Liquidity-aware design**:
  - Avoid basing core prices on **illiquid pools**.
  - Cap impact of a single pool/feed on global pricing.
- **Fail-safe behavior**:
  - On suspicious or unavailable data, **halt sensitive operations** (borrowing, liquidations).
  - Use circuit breakers and rate limiting on parameter changes.
- **Monitoring & alerting**:
  - Track price deviations between your oracle and reference markets.
  - Set automated alerts for out-of-band movements or stuck oracles.
