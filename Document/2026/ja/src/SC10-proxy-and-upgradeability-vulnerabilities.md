## SC10:2026 - プロキシとアップグレード可能性の脆弱性 (Proxy & Upgradeability Vulnerabilities)

#### 説明

プロキシとアップグレード可能性の脆弱性は、スマートコントラクトがアップグレード可能なアーキテクチャ (プロキシ、ビーコン、または実装入れ替えパターン) を使用し、そのアップグレードパス、初期化、または管理者制御の設計や設定に不備がある状況を指します。アップグレード可能なコントラクトは、**プロキシ** (状態を保持し、呼び出しを委譲する) と **実装** (ロジックを含む) を分離しています。アップグレード可能性が十分に保護されていない場合、攻撃者はプロキシ管理者やアップグレードロールをハイジャックして悪意のある実装をデプロイしたり、コントラクトを再初期化して所有権を奪取したり、初期化や移行の段階での重要なチェックをバイパスできる可能性があります。

これはアップグレード可能性を使用するすべてのコントラクトタイプに影響を及ぼします。DeFi (レンディング、Vault、DEX)、NFT (コレクション、マーケットプレイス)、DAO (ガバナンス、トレジャリー)、ブリッジ (メッセンジャー、資産コントラクト)、L2/クロスチェーンシステムがあります。よくあるパターンには、Transparent Proxy、UUPS (EIP-1822)、Beacon Proxy、およびカスタムルーター実装設計を含みます。非 EVM チェーンでは、類似のアップグレードメカニズム (Move モジュール、Solana プログラムアップグレードなど) が存在し、同様の信頼性や初期化のリスクを伴います。

注目する領域は以下のとおりです。

- **アップグレードと管理者ロール** (実装、ストレージレイアウトの互換性を変更できる)
- **初期化と再初期化** (保護されていない `initialize`、`initializer` ガードの欠如、ストレージの衝突)
- **プロキシの委譲** (`delegatecall` コンテキスト、`msg.sender`/`msg.value` の伝播)
- **ストレージレイアウト** (プロキシと実装の間のスロット衝突、追記専用ストレージ)
- **タイムロックとガバナンス** (アップグレードプロセス、ロールバック機能)

攻撃者は以下を悪用します。

- **保護されていないアップグレード機能**: 任意の呼び出し元がプロキシを悪意のある実装に指し示すようにできます
- **再初期化**: 所有権、設定、アクセス制御をリセットします
- **delegatecall を通じた初期化**: 攻撃者が制御するパラメータを用いて実装を初期化できます
- プロキシと実装の間での **ストレージの衝突**: 上書きにつながります

これらの問題はアクセス制御 (SC01) と重複する部分も多いのですが、プロキシやアップグレードメカニズムのメカニズムがシステム全体に及ぶ影響によって、個別に注意する必要があります。

### 事例 (脆弱でアップグレード可能なプロキシ管理者)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract VulnerableProxyAdmin {
    address public admin;
    address public implementation;

    constructor(address _implementation) {
        // Critical: no way to set custom admin; implicitly trusts deployer logic
        admin = msg.sender;
        implementation = _implementation;
    }

    function upgrade(address newImplementation) external {
        // Missing: access control (only admin) and sanity checks
        implementation = newImplementation;
    }
}
```

**Issues:**

- No access control on `upgrade`; any caller can change `implementation`.
- No checks on `newImplementation` (e.g., interface compatibility, non-zero address).

### 事例 (より安全なアップグレード可能性パターン)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";

contract SafeProxyAdmin is Ownable {
    address public implementation;

    event Upgraded(address indexed newImplementation);

    error InvalidImplementation();

    constructor(address _implementation) {
        _setImplementation(_implementation);
        _transferOwnership(msg.sender);
    }

    function _setImplementation(address _implementation) internal {
        if (_implementation == address(0)) revert InvalidImplementation();
        implementation = _implementation;
    }

    function upgrade(address newImplementation) external onlyOwner {
        _setImplementation(newImplementation);
        emit Upgraded(newImplementation);
    }
}
```

**Security Improvements:**

- `upgrade` is restricted to the contract owner (which should itself be a robust governance or multisig).
- Implementation addresses are validated and upgrades are logged via events.

### 初期化と再初期化のリスク

Initialization functions (e.g., `initialize()`, `initializer` modifiers in OpenZeppelin) are critical in upgradeable patterns. Common pitfalls:

- **Unprotected initializers** that anyone can call.
- **Re-initialization** that can reset ownership, configuration, or state.
- Initialization logic that can be reached **through delegatecalls** from proxies in unintended ways.

Basic example:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract VulnerableLogic {
    address public owner;

    // Missing initializer guard
    function initialize(address _owner) external {
        owner = _owner;
    }
}
```

If used behind a proxy without proper initialization control, attackers can call `initialize` via the proxy and set themselves as owner, taking over the protocol.

### 2025 ケーススタディ

- **Kinto Protocol (July 2025, $1.55M loss)**  
  Attackers exploited **uninitialized ERC1967 proxy contracts**. They detected freshly deployed proxy contracts that had not been properly initialized, then initialized them with malicious implementations containing dormant backdoors. Months later, the attacker activated the backdoor, upgraded the proxy to malicious code, and minted K tokens directly to drain $1.55M. The vulnerability: **unprotected initialization** allowing anyone to become the proxy admin.  

- **Uninitialized Proxy Campaign (2025, $10M+ across protocols)**  
  A broader campaign targeted uninitialized ERC1967 proxies across multiple EVM chains. Attackers used automated scanning to detect newly deployed proxies before legitimate developers could initialize them, then initialized with malicious implementations. The backdoors lay dormant for months, evading audits. When activated, attackers could upgrade proxies and drain funds.  
  - [https://audita.io/blog-articles/the-proxy-hack-uninitialized-contracts-costing-defi-10m-in-losses](https://audita.io/blog-articles/the-proxy-hack-uninitialized-contracts-costing-defi-10m-in-losses)
  - [https://medium.com/mamori-finance/post-mortem-k-proxy-hack-our-path-forward-c2c3809882c6](https://medium.com/mamori-finance/post-mortem-k-proxy-hack-our-path-forward-c2c3809882c6)

### ベストプラクティスと緩和策

- **Use well-established proxy patterns and libraries** (e.g., OpenZeppelin UUPS/transparent proxies) instead of bespoke designs.
- Protect upgrade and admin roles with **robust governance / multisigs**; never leave them on EOAs without strong operational controls.
- Apply **initializer guards**:
  - Use `initializer` and `reinitializer` modifiers correctly.
  - Lock implementation contracts once deployed to prevent direct initialization.
- Require **timelocks and multi-step processes** for upgrades:
  - Announce upgrade proposals.
  - Allow time for review/monitoring before execution.
- Maintain comprehensive **upgrade runbooks** and checklists, including:
  - Testing of migrations.
  - Verification of new implementation code and storage layout.
  - On-chain simulation of upgrade steps where possible.
