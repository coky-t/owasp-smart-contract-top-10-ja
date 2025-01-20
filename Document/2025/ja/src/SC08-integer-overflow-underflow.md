## SC08:2025 - 整数オーバーフローとアンダーフロー (Integer Overflow and Underflow)

### 説明:
Ethereum Virtual Machine (EVM) は整数のデータ型を固定サイズで定義します。これは整数変数が表現できる数値の範囲が有限であることを意味します。たとえば、"uint8" (8 ビットの符号なし整数、つまり非負) は 0 から 255 までの整数のみを格納できます。255 を超える値を "uint8" に格納しようとすると、オーバーフローになります。同様に、"0" から "1" を引くと 255 になります。これをアンダーフローと呼びます。算術演算が型の最大サイズまたは最小サイズを超えるか下回る場合、オーバーフローまたはアンダーフローが発生します。符号付き整数の場合、結果は少し異なります。値が -128 の int8 から "1" を引くと、127 となります。これは、負の値を表す可能性がある符号付き整数型は、負の最大値に達すると、最初からやり直しになるためです。この動作のわかりやすい二つの例としては、周期的な数学関数 (sin の引数に 2 を加えても値はそのまま) や、移動距離を追跡する自動車の走行距離計 (最大値 999999 を超えると 000000 にリセットされる) があります。

***重要な注意事項:-
Solidity `0.8.0` 以降では、コンパイラは算術演算におけるオーバーフローとアンダーフローのチェックを自動的に処理し、オーバーフローやアンダーフローが発生した場合はトランザクションを元に戻します。
また、Solidity `0.8.0` では `unchecked` キーワードを導入し、開発者がこれらの自動チェックなしで算術演算を実行して、元に戻さずに明示的にオーバーフローを許可できます。これはオーバーフローの心配がない場合や、以前のバージョンの Solidity での算術演算の動作と同様のラップアラウンド動作が必要な場合に、ガスの使用を最適化するために特に有用です。***

### 事例 (脆弱なコントラクト):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.4.17;

contract Solidity_OverflowUnderflow {
    uint8 public balance;

    constructor() public {
        balance = 255; // Maximum value of uint8
    }

    // Increments the balance by a given value
    function increment(uint8 value) public {
        balance += value; // Vulnerable to overflow
    }

    // Decrements the balance by a given value
    function decrement(uint8 value) public {
        balance -= value; // Vulnerable to underflow
    }
}

```
### 影響:
- 攻撃者はこのような脆弱性を悪用して、口座残高やトークン量を人為的に増加させ、正規に所有しているよりも多くの資金を引き落とすことができる可能性があります。
- 攻撃者は意図したコントラクトロジックのフローを改変し、資産の窃取や過剰な数のトークンの生成などの不正行為につながる可能性があります。

### 対策:
- 最も簡単なアプローチは、Solidity コンパイラバージョン 0.8.0 以降を使用して、オーバーフローとアンダーフローのチェックを自動的に処理することです。
- 最新の安全な数学ライブラリを利用します。Ethereum コミュニティにとって、OpenZeppelin は安全なライブラリの作成と監査において素晴らしい仕事をしてくれました。特にその SafeMath ライブラリを使用すると、アンダーフローとオーバーフローの脆弱性を防止できます。add(), sub(), mul() などの関数を提供しており、これらは基本的な算術演算を実行し、オーバーフローやアンダーフローが発生すると自動的に元に戻します。

### 事例 (修正バージョン):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_OverflowUnderflow {
    uint8 public balance;

    constructor() {
        balance = 255; // Maximum value of uint8
    }

    // Increments the balance by a given value
    function increment(uint8 value) public {
        balance += value; // Solidity 0.8.x automatically checks for overflow
    }

    // Decrements the balance by a given value
    function decrement(uint8 value) public {
        require(balance >= value, "Underflow detected");
        balance -= value;
    }
}
```

### 整数オーバーフローとアンダーフロー攻撃の被害を受けたスマートコントラクトの事例:
1. [PoWH Coin Ponzi Scheme](https://etherscan.io/token/0xa7ca36f7273d4d38fc2aec5a454c497f86728a7a#code) : 包括的な [ハック分析](https://blog.solidityscan.com/integer-overflow-and-underflow-in-smart-contracts-9598032b5a99)
2. [Poolz Finance](https://bscscan.com/address/0x8bfaa473a899439d8e07bf86a8c6ce5de42fe54b#code) : 包括的な [ハック分析](https://blog.solidityscan.com/poolz-finance-hack-analysis-still-experiencing-overflow-fcf35ab8a6c5)
