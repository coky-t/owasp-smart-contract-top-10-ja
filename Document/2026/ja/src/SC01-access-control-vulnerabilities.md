## SC01:2026 - アクセス制御の脆弱性 (Access Control Vulnerabilities)

#### 説明

不適切なアクセス制御は、*誰が*、*どのような* 条件 *下で*、*どのような* パラメータ *を用いて*、特権的な動作を実行できるかを厳格に強制していない状況を指します。現代の DeFi システムでは、これは単一の `onlyOwner` 修飾子をはるかに超えています。ガバナンスコントラクト、マルチシグ、ガーディアン、プロキシ管理者、クロスチェーンルーターはすべて、トークンの発行や破棄、リザーブの移動、プールの再構成、コアロジックの一時停止や再開、実装のアップグレードをできる人物の強制に関与しています。これらの信頼境界のいずれかが脆弱であったり一貫性なく適用されていると、攻撃者は特権を持つアクターになりすましたり、信頼できないアドレスを認可済みであるかのようにシステムが処理するようにできる可能性があります。

注目する領域は以下のとおりです。
- **所有権 / 管理者制御** (例: `onlyOwner`、ガバナー、マルチシグ)
- **アップグレードと一時停止のメカニズム** (プロキシ管理者、ガーディアン)
- **資金の移動と会計** (発行/破棄、プール再構成、料金ルーティング)
- **クロスチェーンまたはクロスモジュールの信頼境界** (ブリッジ、Vault ルーター、L2 メッセンジャー)

攻撃者は以下を悪用します。

- 機密性の高い関数での **修飾子やロールチェックの欠落**
- **`msg.sender` の誤った想定** (例: デリゲート呼び出しやメタトランザクション経由)
- コントラクトやプロキシの **保護されていない初期化 / 再初期化**
- モジュール間の **権限の混乱** (例: off-by-one チェック、信頼済みアドレスの誤り)

他の問題 (ロジックバグ、アップグレード可能性の欠陥など) と組み合わせると、アクセス制御の不良はプロトコル全体の侵害につながる可能性があります。

### 事例 (脆弱なコントラクト)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract LiquidityPoolVulnerable {
    address public owner;
    mapping(address => uint256) public balances;

    constructor() {
        owner = msg.sender;
    }

    // Anyone can set a new owner – critical access control bug
    function transferOwnership(address newOwner) external {
        owner = newOwner; // No access control
    }

    // Intended to be called only by the owner to rescue tokens
    function emergencyWithdraw(address to, uint256 amount) external {
        // Missing: require(msg.sender == owner)
        require(balances[address(this)] >= amount, "insufficient");
        balances[address(this)] -= amount;
        balances[to] += amount;
    }
}
```

**問題点:**

- `transferOwnership` は誰でも呼び出し可能であり、任意の乗っ取りが可能となります。
- `emergencyWithdraw` はアクセス制御を欠いており、事実上、任意の呼び出し元にコントラクトの残高を空にする能力を付与しています。

### 事例 (ロールベースのアクセス制御での修正バージョン)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/AccessControl.sol";

contract LiquidityPoolSecure is AccessControl {
    bytes32 public constant GOVERNANCE_ROLE = keccak256("GOVERNANCE_ROLE");
    bytes32 public constant GUARDIAN_ROLE = keccak256("GUARDIAN_ROLE");

    mapping(address => uint256) public balances;

    event EmergencyWithdraw(address indexed to, uint256 amount, address indexed triggeredBy);

    constructor(address governance, address guardian) {
        _grantRole(DEFAULT_ADMIN_ROLE, governance);
        _grantRole(GOVERNANCE_ROLE, governance);
        _grantRole(GUARDIAN_ROLE, guardian);
    }

    function grantGovernance(address newGov) external onlyRole(DEFAULT_ADMIN_ROLE) {
        _grantRole(GOVERNANCE_ROLE, newGov);
    }

    function setGuardian(address newGuardian) external onlyRole(GOVERNANCE_ROLE) {
        _grantRole(GUARDIAN_ROLE, newGuardian);
    }

    // Only governance or designated guardians can trigger emergency withdrawals
    function emergencyWithdraw(address to, uint256 amount)
        external
        onlyRole(GUARDIAN_ROLE)
    {
        require(to != address(0), "invalid to");
        require(balances[address(this)] >= amount, "insufficient");
        balances[address(this)] -= amount;
        balances[to] += amount;

        emit EmergencyWithdraw(to, amount, msg.sender);
    }
}
```

**セキュリティの改善:**

- 明示的な **RBAC**: `DEFAULT_ADMIN_ROLE`, `GOVERNANCE_ROLE`, `GUARDIAN_ROLE`
- 信頼できるロールのみが権限を再構成したり緊急引き落としを実行できます。
- 構成 (ガバナンス) と緊急対応 (ガーディアン) の間を明確に分離しています。

### 2025 ケーススタディ

- **Balancer V2 (2025 年 11 月, 約 1 億 2800 万ドルの損失)**  
  複雑なマルチチェーンプールエコシステムはプール設定と所有権の想定でのアクセス制御の欠陥に悩まされていました。`manageUserBalance` 関数には不適切なアクセス制御がありました。`msg.sender` をユーザーが提供した `op.sender` 値と比較しており、攻撃者は `msg.sender` と一致するように設定して保護をバイパスし、プールコントローラになりすまして不正な WITHDRAW_INTERNAL 操作を実行しました。これは `_upscaleArray` の丸めエラーと連鎖し、流動性を枯渇しました。
  主な教訓:
  - 重要なプール操作は **明示的なロールチェック** とオンチェーンガバナンス **によって保護される** 必要があります。
  - クロスチェーンまたはクロスモジュールの「所有者 (owner)」の概念は、メッセージ発信元から推測するのではなく、**オンチェーンで検証される** 必要があります。
  - [https://www.openzeppelin.com/news/understanding-the-balancer-v2-exploit](https://www.openzeppelin.com/news/understanding-the-balancer-v2-exploit)
  - [https://research.checkpoint.com/2025/how-an-attacker-drained-128m-from-balancer-through-rounding-error-exploitation/](https://research.checkpoint.com/2025/how-an-attacker-drained-128m-from-balancer-through-rounding-error-exploitation/)
  - [https://www.halborn.com/blog/post/explained-the-balancer-hack-november-2025](https://www.halborn.com/blog/post/explained-the-balancer-hack-november-2025)

- **Zoth (2025 年 3 月, 840 万ドルの損失)**  
  中核となる会計および管理機能における不適切な権限チェックにより、攻撃者が不正な資金移動を実行することを可能にしました。攻撃者は Zoth のデプロイヤウォレット (単一の EOA が管理者を制御) を侵害し、USD0PPSubVaultUpgradeable プロキシに悪意のあるアップグレードを実行し、悪意のある実装をデプロイして 840 万ドルを引き落としました。このプロトコルは重要な管理機能を単一の秘密鍵で制御するという脆弱な前提に依存していました。
  主な教訓:
  - アドレスに対する **暗黙の信頼** を避けます (例:「デプロイヤは永久に信頼される」)。
  - 運用、緊急、アップグレードのロールを明確に分離した **ロールベースアクセス制御 (RBAC)** を使用します。
  - [https://blog.solidityscan.com/zoth-hack-analysis-80ba3ac5076b](https://blog.solidityscan.com/zoth-hack-analysis-80ba3ac5076b)
  - [https://www.halborn.com/blog/post/explained-the-zoth-hack-march-2025](https://www.halborn.com/blog/post/explained-the-zoth-hack-march-2025)

- **Cork Protocol (2025 年 5 月, 1100 万～ 1200 万ドルの損失)**  
  Uniswap V4 のフックコールバック (例: `beforeSwap`) は適切なアクセス制御を欠いていました。呼び出し元が信頼できる PoolManager であることを検証していなかったのです。`beforeSwap` 関数は `onlyPoolManager` 修飾子を持ちません。攻撃者は任意のパラメータでフックを直接呼び出し、プロトコルを欺いて導出トークンを付与しています。根本原因はフックエントリポイントの **呼び出し元バリデーションの欠如** でした。
  主な教訓:
  - フックとコールバックのエントリポイントはオンチェーンで明示的に **呼び出し元を検証する** (例: onlyPoolManager) 必要があります。
  - [https://dedaub.com/blog/the-11m-cork-protocol-hack-a-critical-lesson-in-uniswap-v4-hook-security/](https://dedaub.com/blog/the-11m-cork-protocol-hack-a-critical-lesson-in-uniswap-v4-hook-security/)
  - [https://www.coindesk.com/business/2025/05/28/a16z-backed-cork-protocol-suffers-usd12m-smart-contract-exploit](https://www.coindesk.com/business/2025/05/28/a16z-backed-cork-protocol-suffers-usd12m-smart-contract-exploit)
  - [https://www.halborn.com/blog/post/explained-the-cork-protocol-hack-may-2025](https://www.halborn.com/blog/post/explained-the-cork-protocol-hack-may-2025)

### ベストプラクティスと緩和策

堅牢なアクセス制御は、独自のロールシステムではなく、OpenZeppelin の `Ownable` や `AccessControl` といった実績のあるプリミティブを使用することから始まります。特権ロールは少数に留め、明確に文書化し、理想的には EOA ではなく、十分にセキュアなマルチシグまたはガバナンスモジュールによって保持されるべきです。アップグレード可能なコントラクトの初期化ルーチンは初回使用後にロックし、再初期化攻撃を防ぐために `initializer`/`reinitializer` ガードと明示的なバージョン管理を行う必要があります。プロキシとコアコンポーネントのアップグレードパスは厳密に管理され、監視可能であるべきで、すべての特権変更やアップグレードに対してイベントを発行することで、オフチェーン監視が不正使用を迅速に検出できます。最後に、アクセス制御ポリシーはテストに組み込まれ、プロパティをファジングし、可能であれば形式仕様で、「特権を持たないアドレスが資金を流用したり、管理制御を奪取したりできない」といったプロパティを検証すべきです。
