## SC09:2026 - 整数オーバーフローとアンダーフロー (Integer Overflow and Underflow)

#### 説明

整数オーバーフローとアンダーフローは、算術演算がオペランド型の表現可能な範囲外の値を生じる状況を指します。Solidity 0.8 以降では、算術演算は **デフォルトでチェック** され、オーバーフロー/アンダーフローで元に戻ります。しかし、明示的な `unchecked` ブロック、アセンブリ、またはカスタムライブラリではこれらのチェックを無効にできます。非 EVM プラットフォーム (Move, Sui, Solana, Rust ベースのチェーンなど) では、デフォルトのオーバーフローセマンティクスが異なります。あるものは暗黙的にラップし、あるものはアボートします。誤った前提や欠陥のあるカスタムチェックは、ラップされた値、残高の計算ミス、不変数の破綻につながる可能性があります。

これは、算術演算を行うあらゆるタイプのコントラクトに影響を及ぼします。DeFi (プールの不変数、残高、金利、シェア)、NFT (供給、トークン ID)、ブリッジ (金額、シーケンス番号)、大きな数値やユーザーが制御する数値の入力に関連する任意のロジックがあります。特に、オーバーフロー/アンダーフローが経済的不変数 (AMM での k = x * y など) を損なったり、残高操作を可能にする場合、その影響は極めて深刻です。

注目する領域は以下のとおりです。

- **EVM/Solidity** (`unchecked`、アセンブリ、0.8 以前のコードベースの使用)
- **非 EVM チェーン** (Move, Sui, Aptos, Solana, など) およびそれらのデフォルトオーバーフローセマンティクス
- **乗算とべき乗** (大きなオペランドでのオーバーフローのリスクが高い)
- **減算とデクリメント** (減数 > 被減数 の場合にアンダーフロー)
- **キャストと型変換** (uint256 から uint128 へのダウンキャストなど)

攻撃者は以下を悪用します。

- Solidity での **unchecked ブロック**: オーバーフロー/アンダーフローは不可能と想定されるが、エッジケースが存在する場合
- **非 EVM セマンティクス**: 暗黙的なラップや、カスタムチェックがバイパスできる場合
- **大きな入力や細工された入力**: 乗算や加算のチェーンでオーバーフローが発生する場合
- **不変数を破壊する値** (例: 小さな k を生成するオーバーフローによって、単純なチェックを通過する)

### 事例 1: Solidity Pre-0.8 オーバーフロー (EVM)

Solidity バージョン **0.8.0 未満** では、算術オーバーフローやアンダーフローが **静かに** 発生しました。元に戻らず、エラーにもなりません。値はラップアラウンドしました (例: `uint8` 255 + 1 = 0)。Solidity 0.8.0 以降はデフォルトでこれを修正しており、オーバーフロー/アンダーフローは `unchecked` で明示的にラップされていない限り元に戻ります。

**脆弱なもの (Solidity 0.7.x):**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.6;

contract VulnerableToken {
    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) external {
        // UNDERFLOW: if balances[msg.sender] < amount, wraps to huge value
        balances[msg.sender] -= amount;  // Silent underflow!
        balances[to] += amount;          // Silent overflow possible
    }
}
```

**修正済み (オプション A — 0.7.x 向けの SafeMath):**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.6;

import "@openzeppelin/contracts/utils/math/SafeMath.sol";

contract SafeToken {
    using SafeMath for uint256;
    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) external {
        balances[msg.sender] = balances[msg.sender].sub(amount);  // Reverts on underflow
        balances[to] = balances[to].add(amount);                   // Reverts on overflow
    }
}
```

**修正済み (オプション B — Solidity 0.8 以降にアップグレード):**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SafeToken {
    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) external {
        balances[msg.sender] -= amount;  // Reverts on underflow (checked by default)
        balances[to] += amount;          // Reverts on overflow
    }
}
```

**重要な結論:** Solidity 0.8.0 以降ではビルトインのオーバーフロー/アンダーフローチェックを提供しています。強い保証がある場合にのみ `unchecked` を使用します。そうでなければ、チェック付き算術演算を選択します。

---

### 事例 2: Cetus プロトコル — Sui/Move (非 EVM)

**2025 年 5 月 22 日** に、Cetus Protocol (Sui で最大の DEX) は `integer-mate` ライブラリの欠陥のあるオーバーフローチェックによって、約 2 億 2300 万ドルを失いました。**Move** では、加算や乗算はオーバーフローで中断しますが、**左シフト (`<<`) では中断せず**、静かに切り捨てます。このプロトコルは `<< 64` シフトをガードするためにカスタムの `checked_shlw` を使用していましたが、そのガードには誤りがありました。

**根本原因:** `math_u256.move` の `checked_shlw` は誤った閾値を使用していました。上位 64 ビットに非ゼロビットを持つ値 (つまり `n >= 1 << 192`) を拒否すべきでした。そうではなく、その実装は `n > (0xFFFFFFFFFFFFFFFF << 192)` を使用していました。これは誤りであり、2^192 以上の値を通過してしまい、`n << 64` でオーバーフローになります。

**脆弱な `checked_shlw` (integer-mate, Sui Move):**

```move
// integer-mate/sui/sources/math_u256.move
// VULNERABLE: incorrect overflow threshold
public fun checked_shlw(n: u256): (u256, bool) {
    let mask = 0xFFFFFFFFFFFFFFFF << 192;  // WRONG! Produces wrong threshold
    if (n > mask) {
        (0, true)   // Should signal overflow
    } else {
        ((n << 64), false)  // Overflow occurs here for n >= 2^192—Move truncates silently
    }
}
```

**修正済み `checked_shlw`:**

```move
// FIXED: correct overflow check—reject if any bits in top 64 bits
public fun checked_shlw(n: u256): (u256, bool) {
    // Correct: shifting left by 64 overflows if n >= 2^192
    if (n >= 1 << 192) {
        (0, true)   // Overflow—abort path
    } else {
        ((n << 64), false)
    }
}
```

**使用されていた箇所:** `clmm_math.move` 内の CLMM 関数 `get_delta_a` は、流動性ポジションに必要なトークン A を計算していました。それは以下を呼び出していました。

```move
let (numerator, overflowing) = math_u256::checked_shlw(
    full_math_u128::full_mul(liquidity, sqrt_price_diff)
);
assert!(!overflowing);  // Assertion passed incorrectly due to flawed check
```

`liquidity ≈ 2^113` と `sqrt_price_diff ≈ 2^79` では、その積は `≈ 2^192 + ε` となりました。欠陥のある `checked_shlw` はそれを通してしまい、`n << 64` がオーバーフローして切り捨てられて極めて小さい値になりました。それから、プロトコルはトークン A の **1 ユニット** だけで膨大な流動性 (約 10^37 ユニット) を生成するために必要になると計算し、流出を可能にしました。

**攻撃フロー (簡略化):** フラッシュスワップする → 狭いティックでのポジションを開設する → 細工された流動性パラメータで `add_liquidity` を呼び出す → 過小請求 (1 トークン) し、膨大な流動性をクレジットする → 流動性を削除する → プールから流出する → フラッシュスワップを返済する。

---

### 事例 3: Solidity 0.8 以降での `unchecked` (明示的なオプトアウト)

Solidity 0.8 以降であっても、`unchecked` はチェックを無効にします。オーバーフロー/アンダーフローが起こりえないことを証明できる場合にのみ使用します。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract UncheckedExample {
    function bad(uint256 x, uint256 y) external pure returns (uint256) {
        unchecked {
            return x * y;  // Can overflow; no revert
        }
    }

    function good(uint256 x) external pure returns (uint256) {
        unchecked {
            return x - 1;  // Safe only if caller ensures x >= 1
        }
    }
}
```

安全性を証明する形式推論やテストがない限り、ユーザーが制御できる入力や無制限の入力に対して **`unchecked` を避ける** ようにします。

---

### 2025 ケーススタディ: Cetus (2025 年 5 月, 2 億 2300 万ドルの損失)

Cetus Protocol on Sui was exploited via a flawed `checked_shlw` in the shared `integer-mate` library. The function was meant to prevent u256 overflow when shifting left by 64 bits during CLMM liquidity calculations. The overflow check used the wrong threshold (`0xFFFFFFFFFFFFFFFF << 192` instead of `1 << 192`), allowing values ≥ 2^192 to pass. In Move, left shift does not abort on overflow—it truncates. The truncated numerator caused `get_delta_a` to return that only 1 token was required to mint enormous liquidity. Attackers repeated this across pools using flash swaps, draining ~$223M. Key lessons: (1) Move shift operations do not abort on overflow; (2) custom overflow guards must be rigorously verified; (3) shared math libraries are high-risk and need formal analysis.  
- [https://dedaub.com/blog/the-cetus-amm-200m-hack-how-a-flawed-overflow-check-led-to-catastrophic-loss/](https://dedaub.com/blog/the-cetus-amm-200m-hack-how-a-flawed-overflow-check-led-to-catastrophic-loss/)
- [https://www.cyfrin.io/blog/inside-the-223m-cetus-exploit-root-cause-and-impact-analysis](https://www.cyfrin.io/blog/inside-the-223m-cetus-exploit-root-cause-and-impact-analysis)
- [https://www.halborn.com/blog/post/explained-the-cetus-hack-may-2025](https://www.halborn.com/blog/post/explained-the-cetus-hack-may-2025)
- [https://blog.verichains.io/p/cetus-protocol-hacked-analysis](https://blog.verichains.io/p/cetus-protocol-hacked-analysis)

### ベストプラクティスと緩和策

- On Solidity/EVM:
  - **Avoid `unchecked`** arithmetic unless you have strong reasons and tests proving safety.
  - Use explicit checks and custom errors for critical invariants.
  - Favor well-reviewed math libraries (for fixed-point, exponentiation, etc.).
- On non-EVM environments (e.g., Move, Rust-based chains):
  - Understand the language’s **default overflow semantics**.
  - Use safe arithmetic constructs or libraries where available.
  - Add **assertions and invariants** around critical arithmetic.
- Test with **extreme value ranges**:
  - Minimum and maximum values for all numeric types.
  - Fuzz tests that target edge cases (near boundaries where overflow/underflow is likely).
