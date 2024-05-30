## 脆弱性: 不適切なアクセス制御 (Vulnerability: Improper Access Control)

### 説明:
アクセス制御の脆弱性とは、認可されていないユーザーがコントラクトのデータや機能にアクセスしたり変更できるセキュリティ上の欠陥です。これらの脆弱性は、コントラクトのコードがユーザーのパーミッションレベルに基づいてアクセスを適切に制限できない場合に発生します。スマートコントラクトのアクセス制御は、トークンの生成、プロポーザルの採択、資金の引き出し、コントラクトの一時停止とアップグレード、所有権の変更など、ガバナンスや重要なロジックに関連することがあります。

### 事例 (HospoWise ハック):
```
function burn(address account, uint256 amount) public { //No proper access control is implemented for the burn function
        _burn(account, amount);
    }
}
```
### 影響:
- 攻撃者はコントラクト内の重要な機能やデータへの認可されていないアクセスを獲得して、その完全性とセキュリティを危険にさらす可能性があります。
- 脆弱性は、コントラクトによって管理されている資金や資産の窃取につながり、ユーザーや利害関係者に重大な金銭的損害を引き起こす可能性があります。

### 対策:
- 初期化関数は認可されたエンティティによってのみ一度だけ排他的に呼び出されるようにします。
- コントラクトに Ownable や RBAC (Role-Based Access Control) などの確立されたアクセス制御パターンを使用してパーミッションを管理し、認可されたユーザーだけが特定の機能にアクセスできるようにします。これは機密性の高い機能に `onlyOwner` やカスタムロールなどの適切なアクセス制御修飾子を追加することで実現できます。

### 不適切なアクセス制御攻撃の被害を受けたスマートコントラクトの事例:
1. [HospoWise ハック](https://etherscan.io/address/0x952aa09109e3ce1a66d41dc806d9024a91dd5684#code) : 包括的な [ハック分析](https://blog.solidityscan.com/access-control-vulnerabilities-in-smart-contracts-a31757f5d707)
2. [LAND NFT ハック](https://bscscan.com/address/0x1a62fe088F46561bE92BB5F6e83266289b94C154#code) : 包括的な [ハック分析](https://blog.solidityscan.com/land-hack-analysis-missing-access-control-66fb9555a3e3)
