## SC05:2026 - 入力バリデーションの欠如 (Lack of Input Validation)

#### 説明

入力バリデーションの欠如は、スマートコントラクトが外部データ (関数パラメータ、呼び出しデータ、クロスチェーンメッセージ、署名付きペイロード) を処理する際に、データが適切な形式であり、想定した範囲内であり、意図した操作に対して認可されていることを **厳密に適用することのない** 状況を指します。入力が良性であると想定するコントラクトは、システムを安全でない状態に追い込んだり、会計処理を破損したり、意図したチェックをバイパスする不正なデータや悪意のあるデータに身にさらします。

これは、DeFi (手数料 bps、スリッページ、金額、アドレス)、NFT (トークン ID、メタデータ、ロイヤリティ設定)、DAO (提案ペイロード、投票パラメータ)、ブリッジ (メッセージペイロード、宛先チェーン)、任意のコールデータまたはリレーコールを受け入れる汎用的な構成可能コントラクトなど、すべてのコントラクトタイプにわたって適用します。EVM 以外のチェーンにおいても、同じ原則を保持します。ユーザー、他のコントラクト、またはクロスチェーンチャネルからの信頼できない入力は使用前に検証する必要があります。

注目する領域は以下のとおりです。

- **数値パラメータ** (金額、手数料、レート、スリッページ、担保係数) および安全境界
- **アドレス** (ゼロアドレス、コントラクト対 EOA の想定、委譲アドレスまたはプロキシアドレス)
- **オフチェーンおよび署名付きデータ** (署名、有効期限、nonce リプレイ)
- **クロスチェーンおよびブリッジペイロード** (メッセージフォーマット、チェーン ID、送信者検証)
- **管理者およびガバナンス入力** (設定値、アップグレードパラメータ) — 通常は信頼できるものとして扱われますが、設定ミスや悪用される可能性があります

攻撃者は以下を悪用します。

- 不変条件を崩壊する **境界外の値** (例: 手数料 100% 超え、金額ゼロ、最大 uint 値)
- 許可リストをバイパスまたは予期しない動作を発生する **不正な形式のアドレスまたはペイロード**
- nonce/有効期限/チェーン ID が検証されていない場合の **リプレイ攻撃および順序付け攻撃**
- コントラクトが呼び出し元形式を前提としている場合、または中継されたデータを信頼している場合の **構成可能性のエッジケース**

2025 年には、入力バリデーションの問題が *要因* として頻繁に現れました。たとえば、流動性や利息計算を制御するパラメータに安全な範囲を適用できない場合などです。

### 事例 (脆弱なパラメータ処理)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract VulnerableConfig {
    uint256 public feeBps;      // basis points 0–10_000
    uint256 public maxDeposit;  // upper bound for deposits

    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function setConfig(uint256 _feeBps, uint256 _maxDeposit) external {
        // Missing: access control and bounds checks
        feeBps = _feeBps;
        maxDeposit = _maxDeposit;
    }
}
```

**問題点:**

- アクセス制御なし: 誰でも `setConfig` を呼び出すことができます。
- `_feeBps` や `_maxDeposit` のバリデーションなし:
  - `feeBps` が 100% を超えると、手数料ロジックが破綻する可能性があります。
  - `maxDeposit` が安全でない値またはゼロ値に設定されると、プロトコルを中断する可能性があります。

### 事例 (強力なバリデーションを備えた修正バージョン)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SafeConfig {
    uint256 public feeBps;      // 0–1_000 (max 10% fee)
    uint256 public maxDeposit;  // upper bound for deposits

    address public immutable owner;

    error NotOwner();
    error InvalidFee();
    error InvalidMaxDeposit();

    constructor(uint256 initialFeeBps, uint256 initialMaxDeposit) {
        owner = msg.sender;
        _setConfig(initialFeeBps, initialMaxDeposit);
    }

    function _setConfig(uint256 _feeBps, uint256 _maxDeposit) internal {
        if (_feeBps > 1_000) revert InvalidFee();
        if (_maxDeposit == 0) revert InvalidMaxDeposit();
        feeBps = _feeBps;
        maxDeposit = _maxDeposit;
    }

    function setConfig(uint256 _feeBps, uint256 _maxDeposit) external {
        if (msg.sender != owner) revert NotOwner();
        _setConfig(_feeBps, _maxDeposit);
    }
}
```

**セキュリティの改善:**

- 手数料が文書化された安全な範囲内に収まっていることを検証します。
- `maxDeposit` が非ゼロであることを必須とし、設定ミスを防止します。
- 設定変更をコントラクトオーナーに制限します (より高度な RBAC については SC01 を参照)。

### 2025 ケーススタディ

- **Cetus (May 2025, $223M loss)**  
  The primary root cause was a flawed overflow check in `checked_shlw` (see SC09). However, **insufficient input validation** was a contributing factor—the protocol allowed extreme liquidity parameters (e.g., ~2^113) without bounds checks. When combined with the flawed arithmetic, these unvalidated inputs produced dangerous edge cases leading to pool drains.  
  - [https://dedaub.com/blog/the-cetus-amm-200m-hack-how-a-flawed-overflow-check-led-to-catastrophic-loss/](https://dedaub.com/blog/the-cetus-amm-200m-hack-how-a-flawed-overflow-check-led-to-catastrophic-loss/)
  - [https://www.cyfrin.io/blog/inside-the-223m-cetus-exploit-root-cause-and-impact-analysis](https://www.cyfrin.io/blog/inside-the-223m-cetus-exploit-root-cause-and-impact-analysis)
  - [https://www.halborn.com/blog/post/explained-the-cetus-hack-may-2025](https://www.halborn.com/blog/post/explained-the-cetus-hack-may-2025)

- **Ionic Money (February 2025, ~$6.9M loss)**  
  Attackers used **social engineering** to convince the protocol to list a counterfeit LBTC token. Once listed, the protocol accepted it as collateral without validating token authenticity on-chain (e.g., verification that listed collateral contracts are legitimate). Attackers minted 250 fake LBTC and used it to borrow ~$8.6M. *Note: The root cause was partly off-chain (governance/listing process); the on-chain vulnerability was insufficient validation that collateral tokens are genuine before trusting whitelisted addresses.*  
  - [https://www.halborn.com/blog/post/explained-the-ionic-money-hack-february-2025](https://www.halborn.com/blog/post/explained-the-ionic-money-hack-february-2025)
  - [https://rekt.news/ionic-money-rekt](https://rekt.news/ionic-money-rekt)

### ベストプラクティスと緩和策

- **Validate all external inputs**, including:
  - Function parameters (amounts, addresses, configuration values)
  - Off-chain-signed data and calldata payloads
  - Cross-chain messages and bridge payloads
- Enforce **tight invariants**:
  - Ranges for fees, interest rates, leverage, and collateral factors.
  - Non-zero requirements for key addresses and limits.
- Use **custom errors** and explicit checks to keep validation clear and gas-efficient.
- Treat **admin and governance inputs** as untrusted until validated—misconfiguration can be as damaging as explicit exploits.
- Include **negative tests** for invalid inputs (fuzzing, property tests) to ensure unexpected values are rejected.
