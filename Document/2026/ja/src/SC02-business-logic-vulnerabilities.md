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

- **Abracadabra (2025 年 3 月, 1290 万ドルの損失)**  
  GMX V2 CauldronV4 コントラクトにおける担保会計に瑕疵がありました。攻撃者は三段階の手法を用いました。 (1) 失敗するように設計された GMX に預け入れを行い、トークンを OrderAgent に滞留します。 (2) ポジションを自己清算して、コントラクトはポジションを消去しましたが、関連する注文と担保は削除しませんでした。 (3) 「ゴースト」担保を使用して、6,260 ETH (約 1290 万ドル) を借り入れました。預け入れ失敗と清算パスによって経済的不変条件 (担保 ↔ 負債) が破られました。  
  主な教訓:
  - すべての経済的不変条件 (例: 最低担保額、最大 LTV) はすべての状態遷移において **証明可能および強制可能** でなければなりません。
  - 新しいスペル/戦略タイプを導入するには、既存システムとのやり取りについて **正式なレビュー** を行わなければなりません。
  - [https://blog.solidityscan.com/abracadabra-hack-analysis-f2efcdee9c05](https://blog.solidityscan.com/abracadabra-hack-analysis-f2efcdee9c05)
  - [https://www.halborn.com/blog/post/explained-the-abracadabra-money-hack-march-2025](https://www.halborn.com/blog/post/explained-the-abracadabra-money-hack-march-2025)
  - [https://www.coindesk.com/business/2025/03/25/abracadabra-drained-of-usd13m-in-exploit-targeting-cauldrons-tied-to-gmx-liquidity-tokens](https://www.coindesk.com/business/2025/03/25/abracadabra-drained-of-usd13m-in-exploit-targeting-cauldrons-tied-to-gmx-liquidity-tokens)
  - [https://threesigma.xyz/blog/exploit/abracadabra-gmx-defi-exploit-explained](https://threesigma.xyz/blog/exploit/abracadabra-gmx-defi-exploit-explained)

- **Yearn Finance (2025 年 11 月, 900万ドルの損失)**  
  yETH 加重ステーブルスワッププールの固定小数点反復ソルバーに設計上の欠陥がありました。極端に不均衡な流動性の追加/削除操作を行うことで、攻撃者はソルバーを発散状態に陥らせ、積項 (Π) をゼロに崩壊させて、プールをハイブリッドステープルスワップ不変から定和曲線へと変化しました。これは、担保なしで約 2.35×10⁵⁶ yETH LP トークンをミントし、プールを枯渇できました。この脆弱性は、再入可能性や低レベルバグではなく、**不変崩壊** でした。  
  主な教訓:
  - 報酬と手数料の分配経路はあらゆる敵対的シナリオで **シミュレーションテスト** されなければなりません。
  - 利回り戦略は **明確でテスト可能な不変条件** を持つべきです (いかなるユーザーも時間の経過とともに公平な分配額を超える報酬を請求できないなど)。
  - [https://defimon.xyz/blog/yearn-yeth-hack-november-2025](https://defimon.xyz/blog/yearn-yeth-hack-november-2025)
  - [https://www.cryptopolitan.com/yearn-finance-begins-clawback-after-9m-hack/](https://www.cryptopolitan.com/yearn-finance-begins-clawback-after-9m-hack/)

### ベストプラクティスと緩和策

- **プロトコルの経済性を** 直感に頼るのではなく **明示的にモデル化します** (例: 敵対的シミュレーションやエージェントベースのモデルを用いて)。
- **コアとなる不変条件をコードとテストで** 表現します:
  - 「引き落とし総額は預け入れ総額＋実効利回りを超えることはできません」
  - 「報酬の分配は時間加重ステークに比例します」
  - 「清算は信頼できるオラクルデータの下ではプロトコル損失をもたらすことはありません」
- 主要な会計処理経路 (Vault、戦略、報酬分配) には **形式検証とプロパティベースのファジング** を使用します。
- **新しい戦略 / スペルのバージョン管理とゲート設定します**:
  - 上限を設けて展開します。
  - 上限を引き上げる前にメトリクスとオンチェーン不変条件を監視します。
- 監査担当者だけでなく **ガバナンスチームと運用チームが不変条件を理解していること** を確認します。
