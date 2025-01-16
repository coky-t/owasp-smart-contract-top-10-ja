# SC02:2025 - 価格オラクル操作 (Price Oracle Manipulation)

## 説明:
価格オラクル操作は、価格やその他の情報を取得するために外部データフィード (オラクル) に依存するスマートコントラクトの重大な脆弱性です。分散型金融 (DeFi) では、オラクルは資産価格などの現実世界のデータをスマートコントラクトに提供するために使用されます。ただし、オラクルによって提供されるデータが操作されると、コントラクトの動作が不正確になる可能性があります。攻撃者は、オラクルが提供するデータを操作することでオラクルを悪用し、不正な引き落とし、過剰なレバレッジ、さらには流動性プールの枯渇などの壊滅的な結果につながる可能性があります。この種の攻撃を防ぐには、適切なセーフガードとバリデーションのメカニズムが不可欠です。


## 事例 (脆弱なコントラクト):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IPriceFeed {
    function getLatestPrice() external view returns (int);
}

contract PriceOracleManipulation {
    address public owner;
    IPriceFeed public priceFeed;

    constructor(address _priceFeed) {
        owner = msg.sender;
        priceFeed = IPriceFeed(_priceFeed);
    }

    function borrow(uint256 amount) public {
        int price = priceFeed.getLatestPrice();
        require(price > 0, "Price must be positive");

        // Vulnerability: No validation or protection against price manipulation
        uint256 collateralValue = uint256(price) * amount;

        // Borrow logic based on manipulated price
        // If an attacker manipulates the oracle, they could borrow more than they should
    }

    function repay(uint256 amount) public {
        // Repayment logic
    }
}
```

### 影響:
- 攻撃者はオラクルを操作して資産の価格を吊り上げ、本来得られるはずのものよりも多くの資金を借り入れることができます。
- 操作された価格によって担保の誤った評価につながる場合、正当なユーザーが誤った評価による清算に直面する可能性があります。
- オラクルが侵害された場合、攻撃者は操作されたデータを悪用してコントラクトの流用性プールを枯渇させたり、さらにはコントラクトを支払い不能にできます。

### 対策:
- 複数の独立したオラクルからデータを集約して、単一のソースによる操作のリスクを軽減します。
- オラクルから受け取る価格の最小と最大の閾値を設定して、急激な価格変動がコントラクトのロジックに影響を及ぼすことを防ぎます。
- 価格更新の間にタイムロックを導入して、攻撃者に悪用される可能性のある即時変更を防ぎます。
- 信頼できるパーティからの署名を要求するなど、暗号論的証明を使用してオラクルから受け取るデータの真正性を確保します。

### 事例 (修正バージョン):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IPriceFeed {
    function getLatestPrice() external view returns (int);
}

contract PriceOracleManipulation {
    address public owner;
    IPriceFeed public priceFeed;
    int public minPrice = 1000; // Set minimum acceptable price
    int public maxPrice = 2000; // Set maximum acceptable price

    constructor(address _priceFeed) {
        owner = msg.sender;
        priceFeed = IPriceFeed(_priceFeed);
    }

    function borrow(uint256 amount) public {
        int price = priceFeed.getLatestPrice();
        require(price > 0 && price >= minPrice && price <= maxPrice, "Price manipulation detected");

        uint256 collateralValue = uint256(price) * amount;

        // Borrow logic based on valid price
    }

    function repay(uint256 amount) public {
        // Repayment logic
    }
}
```

### 価格オラクル操作攻撃の被害を受けたスマートコントラクトの事例:
1. [Polter Finance ハック分析](https://blog.solidityscan.com/polter-finance-hack-analysis-c5eaa6dcfd40) 
2. [BonqDAO プロトコル](https://polygonscan.com/address/0x4248fd3e2c055a02117eb13de4276170003ca295#code) : 包括的な [ハック分析](https://blog.solidityscan.com/bonqdao-protocol-hack-analysis-oracle-manipulation-8e6978149a66)
