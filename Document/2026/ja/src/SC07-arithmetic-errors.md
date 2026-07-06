## SC07:2026 - 算術エラー (丸めと精度) (Arithmetic Errors (Rounding & Precision))

#### 説明

算術エラー (丸めと精度の損失) は、スマートコントラクトが、切り捨て、スケーリング、単位変換により誤った結果や悪用可能な結果を生じる、整数ベースの計算を実行する状況です。スマートコントラクトは整数演算に制限されています。除算、固定小数点スケーリング、単位間の変換は精度を損失したり、非対称な丸めをもたらしたり、あるいは未チェックのブロックや非 EVM セマンティクスと組み合わせされる場合には、オーバーフロー/アンダーフロー (SC09 を参照) を引き起こす可能性があります。

これは数値計算を行うあらゆるコントラクタタイプに影響を及ぼします。これには DeFi (シェアのミント/バーン、LP トークン、利息の発生、スワップの出力、AMM 不変量の更新)、イールドボールトおよび ERC-4626 (資産/シェアの変換)、リベーストークン、報酬の分配、NFT/トークンエコノミクスがあります。非 EVM チェーン (Move、Rust ベースなど) では、整数セマンティクスや利用可能な精度は異なりますが、算術演算が経済的結果を左右する場面ではどこでも同様のリスクが存在します。

注目する領域は以下のとおりです。

- **シェアおよび LP トークンの計算** (預け入れ/引き落としの式、丸め方向)
- **利息および報酬の発生** (複利計算、時間加重平均)
- **スワップおよび AMM の計算** (定数積、集中流動性、出力計算)
- **固定小数点およびスケーリング** (1e18 や 1e8 の慣例、トークン間換算)
- **リベースおよび比例配分** (ユーザーごととグローバルの会計)

攻撃者は以下を悪用します。

- **丸めバイアス** (例: 敵対的シーケンスの下で預け入れまたはプロトコルに有利となる丸め)
- フラッシュローン (SC04) や高頻度なやり取りを介した **少額利益の積み重ね**
- 計算式が破綻する **エッジケース** (総供給量ゼロ、最初の預け入れ、極端な比率)
- 操作を通じて蓄積する多段階の計算での **精度の損失**

フラッシュローン (SC04) やビジネスロジックの欠陥 (SC02) と組み合わされると、算術エラーはプロトコルの資金流出となるような悪用につながる可能性があります。

### 事例 (脆弱な株価計算)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract VulnerableShares {
    uint256 public totalAssets;
    uint256 public totalShares;

    mapping(address => uint256) public balanceOf;

    function deposit(uint256 assets) external {
        require(assets > 0, "zero");

        uint256 shares;
        if (totalShares == 0) {
            shares = assets;
        } else {
            // Rounds down and may favor the depositor under certain edge states
            shares = (assets * totalShares) / totalAssets;
        }

        totalAssets += assets;
        totalShares += shares;
        balanceOf[msg.sender] += shares;
    }
}
```

**問題点:**

- 丸めは常に切り捨てとなります。預け入れ/引き落としの敵対的シーケンスの下では、これが悪用される可能性があります。
- エッジケースにおいても総株式価値が一定であることを確保する不変テストがありません。

### 事例 (より堅牢な算術演算と不変条件設計)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SaferShares {
    uint256 public totalAssets;
    uint256 public totalShares;

    mapping(address => uint256) public balanceOf;

    error ZeroAmount();
    error InvalidState();

    function deposit(uint256 assets) external {
        if (assets == 0) revert ZeroAmount();

        uint256 shares;
        if (totalShares == 0 || totalAssets == 0) {
            // Explicitly define initial conditions
            shares = assets;
        } else {
            // Use rounding strategy intentionally (up or down) and test it
            shares = (assets * totalShares + totalAssets - 1) / totalAssets; // round up
        }

        uint256 newTotalAssets = totalAssets + assets;
        if (newTotalAssets < totalAssets) revert InvalidState(); // overflow guard

        totalAssets = newTotalAssets;
        totalShares += shares;
        balanceOf[msg.sender] += shares;
    }
}
```

**セキュリティの改善:**

- 初期状態を明示的に処理し、ゼロ除算を回避します。
- 明確に文書化された丸め戦略 (この例では `round up`) を使用します。
- オーバーフローチェックと、わかりやすさのためのカスタムエラーを追加します。

> 実際のプロトコルでは、このようなロジックを形式的推論と不変条件 (形式検証やプロパティベーステストなど) と組み合わせるべきです。

### 2025 ケーススタディ

- **zkLend (2025 年 2 月, 950 万ドルの損失)**  
  `mint()` 関数の丸めエラー (整数除算による切り捨て) は記録された値と実際の値の間に不整合を引き起こしました。1.5 トークンを焼却すべき引き落としでは 1.0 に切り捨てられ、攻撃者は入出金を繰り返すことで lending_accumulator を人為的に膨張させ、約 950 万ドルを流出しました。
  - [https://blog.solidityscan.com/zklend-hack-analysis-e494cb794f71](https://blog.solidityscan.com/zklend-hack-analysis-e494cb794f71)
  - [https://www.halborn.com/blog/post/explained-the-zklend-hack-february-2025](https://www.halborn.com/blog/post/explained-the-zklend-hack-february-2025)

- **Bunni (2025 年 9 月, 840 万ドルの損失)**  
  引き落とし機能の丸めロジックに精度バグがありました。開発者は遊休残高の切り捨てが「安全」であると想定していましたが、操作を繰り返すことで悪用可能な抜け穴を生じました。攻撃者は流動性の 84.4% を焼却するだけで USDC 残高を 85.7% 減少し、不釣り合いな価値の引き出しを可能にしました。
  - [https://www.halborn.com/blog/post/explained-the-bunni-hack-september-2025](https://www.halborn.com/blog/post/explained-the-bunni-hack-september-2025)
  - [https://blog.bunni.xyz/posts/exploit-post-mortem/](https://blog.bunni.xyz/posts/exploit-post-mortem/)
  - [https://cryptotimes.io/2025/09/05/bunni-reveals-code-flaw-behind-8-4-million-exploit](https://cryptotimes.io/2025/09/05/bunni-reveals-code-flaw-behind-8-4-million-exploit)

### ベストプラクティスと緩和策

- **安全な安全な算術パターンを使用します** (Solidity 0.8 以降にはビルトインのチェックを有しますが、それらに関連するロジックも依然として重要です)。
- **丸め戦略** を明確に文書化してテストします。
  - 丸めをプロトコルに有利にするか、ユーザーに有利にするかを決定します。
  - 繰り返し操作が「無償の価値」を生み出すことができないことを証明します。
- 複雑な演算には **十分にレビューされた算術ライブラリ** を利用します。
  - 固定小数点演算 (例: 1e18 スケーリング)
  - 高精度な累乗や対数
- **不変チェック** を組み込みます:
  - 例: `totalAssets` とユーザー残高の合計、操作後のシャアと価値の一貫性。
- **ファズテストや差分テスト** を使用して、小さい/大きい値や繰り返し操作に関するエッジケースを発見します。
