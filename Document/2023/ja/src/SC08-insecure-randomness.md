## 脆弱性: 安全でないランダム性 (Vulnerability: Insecure Randomness)

### 説明:
乱数生成器は、ギャンブル、ゲームの勝者選択、ランダムシード生成など、アプリケーションに不可欠です。Ethereum では、その決定論的な性質のため、乱数を生成することは困難です。Solidity は真の乱数を生成できないため、擬似乱数要素に依存します。さらに、Solidity での複雑な計算はガスに関してコストがかかります。

*Solidity での安全でない乱数生成メカニズム: 開発者は以下のような block 関連メソッドを使用して、乱数を生成することがよくあります。*
  - block.timestamp: 現在のブロックのタイムスタンプ。
  - blockhash(uint blockNumber): 指定されたブロックのハッシュ (直近の 256 ブロックのみ)。
  - block.difficulty: 現在のブロックの difficulty 。
  - block.number: 現在のブロック番号。
  - block.coinbase: 現在のブロックのマイナーのアドレス。

これらのメソッドは、マイナーが操作して、コントラクトのロジックに影響を与えることができるため、安全ではありません。

### 事例 :
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract InsecureRandomNumber {
    constructor() payable {}

    function guess(uint256 _guess) public {
        uint256 answer = uint256(
            keccak256(
                abi.encodePacked(block.timestamp, block.difficulty, msg.sender) // Using insecure mechanisms for random number generation
            ) 
        );

        if (_guess == answer) {
            (bool sent,) = msg.sender.call{value: 1 ether}("");
            require(sent, "Failed to send Ether");
        }
    }
}
```
### 影響:
- 安全でないランダム性は、ゲーム、くじ、その他乱数生成に依存するあらゆるコントラクトにおいて、攻撃者が不当に優位を得るために悪用される可能性があります。本来ランダムであるはずの結果を予測または操作することで、攻撃者は結果を自分たちに有利に影響を及ぼすことができます。これは不公平な勝利、他の参加者の金銭的損失、スマートコントラクトの完全性と公平性に対する一般的な信頼の欠如につながる可能性があります。

### 対策:
- オラクル (Oraclize) をランダム性の外部ソースとして使用します。オラクルを信頼する際には注意が必要です。複数のオラクルを使用することもできます。
- コミットメントスキームを使用します。commit-reveal アプローチを使用する暗号プリミティブに従うことができます。コインフリップ、ゼロ知識証明、セキュアコンピュテーションにも広く応用しています。例: RANDAO
- Chainlink VRF — 証明可能な公平性と検証可能な乱数生成器 (RNG) で、スマートコントラクトがセキュリティやユーザービリティを損なうことなくランダム値にアクセスできます。
- Signidice アルゴリズム — 暗号署名を使用する二者間のアプリケーションにおける PRNG に適しています。
- ビットコインブロックハッシュ — BTCRelay などのオラクルを使用します。Ethereum と Bitcoin 間の橋渡しとして機能します。Ethereum のコントラクトはエントロピーのソースとして Bitcoin Blockchain から将来のブロックハッシュを要求できます。このアプローチはマイナーのインセンティブ問題に対して安全ではないため、慎重に実装する必要があることに注意すべきです。

### 安全でないランダム性攻撃の被害を受けたスマートコントラクトの事例:
1. [Roast Football ハック](https://bscscan.com/address/0x26f1457f067bf26881f311833391b52ca871a4b5#code) : 包括的な [ハック分析](https://blog.solidityscan.com/roast-football-hack-analysis-e9316170c443)
2. [FFIST ハック](https://bscscan.com/address/0x80121da952a74c06adc1d7f85a237089b57af347#code) : 包括的な [ハック分析](https://blog.solidityscan.com/ffist-hack-analysis-9cb695c0fad9)
