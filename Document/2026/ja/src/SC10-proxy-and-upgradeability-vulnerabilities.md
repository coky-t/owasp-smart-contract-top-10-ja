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

**問題点:**

- `upgrade` にアクセス制御がなく、任意の呼び出し元が `implementation` を変更できます。
- `newImplementation` のチェックがありません (インタフェースの互換性、非ゼロアドレスなど)。

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

**セキュリティの改善:**

- `upgrade` はコントラクト所有者 (それ自体が堅牢なガバナンスやマルチシグであるべきです) に制限されています。
- 実装アドレスは検証され、アップグレードはイベントを介してログ記録されています。

### 初期化と再初期化のリスク

初期化関数 (`initialize()`, OpenZeppelin の `initializer` 修飾子など) がアップグレード可能なパターンには極めて重要です。よくある落とし穴は以下のとおりです。

- **保護されていない初期化**: 誰でも呼び出しできます。
- **再初期化**: 所有権、設定、状態をリセットできます。
- 初期化ロジック: 意図しない方法でプロキシから **delegatecall を通じて** 到達できます。

基本的な事例:

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

適切な初期化制御なしでプロキシの背後で使用した場合、攻撃者はプロキシを介して `initialize` を呼び出して、自信を所有者として設定し、プロトコルを乗っ取ることができます。

### 2025 ケーススタディ

- **Kinto Protocol (2025 年 7 月, 155 万ドルの損失)**  
  攻撃者は **初期化されていない ERC1967 プロキシコントラクト** を悪用しました。適切に初期化されないままデプロイされたばかりのプロキシコントラクトを特定し、そこに休眠状態のバックドアを含む悪意のある実装で初期化しました。数か月後、攻撃者はバックドアを有効化して、プロキシを悪意のあるコードにアップグレードし、K トークンを直接発行して 155 万ドルを流出しました。脆弱性: **初期化は保護されておらず**、誰でもプロキシ管理者になることが可能でした。

- **未初期化プロキシへのキャンペーン (2025 年, 複数プロトコルにわたり 1000 万ドル超)**  
  複数の EVM チェーンにわたる未初期化 ERC1967 プロキシを対象にした大規模なキャンペーンです。攻撃者は自動スキャンを使用して、正規の開発者が初期化を行う前に新規デプロイされたプロキシを検出し、悪意のある実装で初期化しました。バックドアは数か月休眠状態にあり、監査を逃れました。有効化した際に、攻撃者はプロキシをアップグレードして資金を流出しました。
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
