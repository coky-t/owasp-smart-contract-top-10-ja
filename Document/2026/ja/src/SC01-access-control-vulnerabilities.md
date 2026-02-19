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

- **Balancer V2 (November 2025, ~$128M loss)**  
  A complex multi-chain pool ecosystem suffered from flawed access control in pool configuration and ownership assumptions. The `manageUserBalance` function had improper access controls—it checked `msg.sender` against a user-provided `op.sender` value, which attackers could set to match `msg.sender` and bypass protections, allowing them to masquerade as pool controllers and execute unauthorized WITHDRAW_INTERNAL operations. This was chained with a rounding error in `_upscaleArray` to drain liquidity.  
  Key lessons:
  - Critical pool operations must be **guarded by explicit role checks** and on-chain governance.
  - Any cross-chain or cross-module "owner" concept must be **verified on-chain**, not assumed from message origin.  
  - [https://www.openzeppelin.com/news/understanding-the-balancer-v2-exploit](https://www.openzeppelin.com/news/understanding-the-balancer-v2-exploit)
  - [https://research.checkpoint.com/2025/how-an-attacker-drained-128m-from-balancer-through-rounding-error-exploitation/](https://research.checkpoint.com/2025/how-an-attacker-drained-128m-from-balancer-through-rounding-error-exploitation/)
  - [https://www.halborn.com/blog/post/explained-the-balancer-hack-november-2025](https://www.halborn.com/blog/post/explained-the-balancer-hack-november-2025)

- **Zoth (March 2025, $8.4M loss)**  
  Improper privilege checks around core accounting and administrative functions allowed attackers to perform unauthorized fund movements. The attacker compromised Zoth's deployer wallet (single EOA controlling admin) and performed a malicious upgrade to the USD0PPSubVaultUpgradeable proxy, deploying a malicious implementation to withdraw $8.4M. The protocol relied on brittle assumptions—a single private key controlling critical admin functions.  
  Key lessons:
  - Avoid **implicit trust** in addresses (e.g., "deployer is trusted forever").
  - Use **role-based access control (RBAC)** with clear separation between operational, emergency, and upgrade roles.  
  - [https://blog.solidityscan.com/zoth-hack-analysis-80ba3ac5076b](https://blog.solidityscan.com/zoth-hack-analysis-80ba3ac5076b)
  - [https://www.halborn.com/blog/post/explained-the-zoth-hack-march-2025](https://www.halborn.com/blog/post/explained-the-zoth-hack-march-2025)

- **Cork Protocol (May 2025, $11–12M loss)**  
  The Uniswap V4 hook callbacks (e.g., `beforeSwap`) lacked proper access control—they did not validate that the caller was the trusted PoolManager. The `beforeSwap` function had no `onlyPoolManager` modifier. Attackers called the hook directly with arbitrary parameters, fooling the protocol into crediting them with derivative tokens. The root cause was **missing caller validation** on hook entry points.  
  Key lessons:
  - Hook and callback entry points must **validate the caller** (e.g., onlyPoolManager) explicitly on-chain.  
  - [https://dedaub.com/blog/the-11m-cork-protocol-hack-a-critical-lesson-in-uniswap-v4-hook-security/](https://dedaub.com/blog/the-11m-cork-protocol-hack-a-critical-lesson-in-uniswap-v4-hook-security/)
  - [https://www.coindesk.com/business/2025/05/28/a16z-backed-cork-protocol-suffers-usd12m-smart-contract-exploit](https://www.coindesk.com/business/2025/05/28/a16z-backed-cork-protocol-suffers-usd12m-smart-contract-exploit)
  - [https://www.halborn.com/blog/post/explained-the-cork-protocol-hack-may-2025](https://www.halborn.com/blog/post/explained-the-cork-protocol-hack-may-2025)

### ベストプラクティスと緩和策

Robust access control starts with using battle‑tested primitives such as OpenZeppelin’s `Ownable` and `AccessControl` rather than bespoke role systems. Privileged roles should be few, clearly documented, and ideally held by well‑secured multisigs or governance modules instead of EOAs. Initialization routines for upgradeable contracts must be locked after first use, with `initializer`/`reinitializer` guards and explicit versioning to prevent re‑initialization attacks. Upgrade paths for proxies and core components should be tightly controlled and observable, with events emitted for every privilege change or upgrade so that off‑chain monitoring can quickly detect abuse. Finally, access control policies should be encoded in tests, fuzzing properties, and, where possible, formal specifications, verifying properties such as “no unprivileged address can ever drain funds or seize admin control.”
