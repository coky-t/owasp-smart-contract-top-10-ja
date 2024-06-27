![OWASP Smart Contract Logo](assets/images/OWASP%20Smart%20Contract.png)

# [OWASP Smart Contract Top 10](https://owasp.org/www-project-smart-contract-top-10/)

## スマートコントラクト Top 10 について

OWASP スマートコントラクト Top 10 はスマートコントラクトで見つかった脆弱性の Top 10 についての洞察を Web3 開発者やセキュリティチームに提供することを意図した標準的な啓発文書です。

これは過去数年間に悪用や発見された脆弱性の Top 10 に対してスマートコントラクトが保護されるようにするための参考資料として機能します。

### Top 10

* SC01:2023 - [再入攻撃 (Reentrancy Attacks)](2023/ja/src/SC01-reentrancy-attacks.md)
* SC02:2023 - [整数オーバーフローとアンダーフロー (Integer Overflow and Underflow)](2023/ja/src/SC02-integer-overflow-underflow.md)
* SC03:2023 - [タイムスタンプの依存性 (Timestamp Dependence)](2023/ja/src/SC03-timestamp-dependence.md)
* SC04:2023 - [アクセス制御の脆弱性 (Access Control Vulnerabilities)](2023/ja/src/SC04-access-control-vulnerabilities.md)
* SC05:2023 - [フロントランニング攻撃 (Front-running Attacks)](2023/ja/src/SC05-front-running-attacks.md)
* SC06:2023 - [サービス拒否攻撃 (Denial of Service (DoS) Attacks)](2023/ja/src/SC06-denial-of-service-attacks.md)
* SC07:2023 - [ロジックエラー (Logic Errors)](2023/ja/src/SC07-logic-errors.md)
* SC08:2023 - [安全でないランダム性 (Insecure Randomness)](2023/ja/src/SC08-insecure-randomness.md)
* SC09:2023 - [ガス制限の脆弱性 (Gas Limit Vulnerabilities)](2023/ja/src/SC09-gas-limit-vulnerabilities.md)
* SC10:2023 - [チェックされていない外部呼び出し (Unchecked External Calls)](2023/ja/src/SC10-unchecked-external-calls.md)

### 概要

| タイトル | 説明 |
| -- | -- |
| SC01 - 再入攻撃 (Reentrancy Attacks) | 再入攻撃は、ある機能が自身の状態を更新する前に別のコントラクトへの外部呼び出しを行う際に、スマートコントラクトの脆弱性を悪用します。これにより、おそらく悪意のある外部コントラクトが元の機能に再入して、同じ状態を使用して引き出しなどの特定のアクションを繰り返すことができます。このような攻撃により、攻撃者はコントラクトからすべての資金を流出させる可能性があります。 |
| SC02 - 整数オーバーフローとアンダーフロー (Integer Overflow and Underflow) | Ethereum 仮想マシン (EVM) は整数に固定サイズのデータ型を定義しており、それによって表現できる値の範囲を制限しています。オーバーフローは、算術演算でデータ型が保持できる最大値を超えたときに発生し、アンダーフローは演算が最小値を下回ると発生します。符号なし整数では、アンダーフローは最大値になり、符号付き整数では、最小値を超えると正の最大値に回り込みます。 |
| SC03 - タイムスタンプの依存性 (Timestamp Dependence) | スマートコントラクトでは時間に敏感な機能に block.timestamp を使用することがよくあります。しかし、マイナーはこのタイムスタンプをわずかに調整できるため、タイミングを操作して不当な利益を獲得できる脆弱性が生じます。 |
| SC04 - アクセス制御の脆弱性 (Access Control Vulnerabilities) | アクセス制御の脆弱性は認可されていないユーザーがコントラクトのデータや機能にアクセスしたり変更できるセキュリティ上の欠陥です。これらの脆弱性は、コントラクトのコードがユーザーのパーミッションに基づいてアクセスを適切に制限できない場合に発生します。 |
| SC05 - フロントランニング攻撃 (Front-running Attacks) | フロントランニングは、悪意のあるアクターが保留中のトランザクションに関する知識を悪用して不当に利益を得る攻撃です。攻撃者はメモリプールを観察し、ターゲットとなるトランザクションよりも先に処理されるように、より高いガス価格の独自のトランザクションを配置します。これは金銭的損失をもたらし、スマートコントラクトの機能を妨害する可能性があります。 |
| SC06 - サービス拒否攻撃 (Denial of Service (DoS) Attacks) | Solidity のサービス拒否 (DoS) 攻撃はスマートコントラクト内の脆弱性を悪用して、ガス、CPU サイクル、ストレージなどの重要なリソースを使い果たします。これらの攻撃は、コントラクトを機能不能にし、意図した動作を妨害して、潜在的に金銭的な損害を引き起こすことを目的としています。 |
| SC07 - ロジックエラー (Logic Errors) | ロジックエラー、つまりビジネスロジック脆弱性とは、コードが意図した動作と一致しない、スマートコントラクトの捉えにくい欠陥です。これらのエラーはコントラクトのロジック内に存在するため検出が難しく、意図しない結果や悪用可能な状態につながる可能性があります。 |
| SC08 - 安全でないランダム性 (Insecure Randomness) | ブロックチェーンネットワーク上のスマートコントラクトで真のランダム性を生成することは、その決定論的な性質ゆえに困難です。乱数であるはずの数字に対する予測可能性や影響力があると、攻撃者はコントラクトを悪用して優位を得て、公平性とセキュリティ対策が損なわれる可能性があります。 |
| SC09 - ガス制限の脆弱性 (Gas Limit Vulnerabilities) | Ethereum などのブロックチェーンプラットフォームのガス制限は、トランザクションごとのスマートコントラクトの計算に制約を課します。ブロックガス制限を超える機能、特に配列などの動的データ構造のループを含むものは、リソース枯渇によるトランザクション失敗のリスクがあり、コントラクト設計の一般的な脆弱性を浮き彫りにしています。 |
| SC10 - チェックされていない外部呼び出し (Unchecked External Calls) | Ethereum のスマートコントラクトでは、外部関数呼び出しの結果を適切に検証しないと、意図しない結果につながる可能性があります。呼び出された関数が失敗し、呼び出したコントラクトがこれをチェックしない場合、成功したと想定して不正に処理を進めて、コントラクトの完全性と機能性が損なわれる可能性があります。 |

## 参加するには
すべてのディスカッションは OWASP Smart Contract Top Ten [GitHub リポジトリ](https://github.com/OWASP/www-project-smart-contract-top-10) で行われます。

コミュニティメンバーの皆様が積極的に参加し、このプロジェクトの向上にご協力いただけることを歓迎します。提案やフィードバックがある場合やリストの改善に協力したい場合には、issue を上げたりプルリクエストを送信して対話を始めることをお勧めします。

[こちら](https://github.com/OWASP/www-project-smart-contract-top-10/blob/main/CONTRIBUTING.md) でコントリビュートガイドラインをご覧ください。

## ライセンス
The OWASP Smart Contract Top 10 document is licensed under the [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/), the Creative Commons
Attribution-ShareAlike 4.0 license. Some rights reserved.

[![license](https://mirrors.creativecommons.org/presskit/buttons/88x31/svg/by-nc-sa.svg)](https://github.com/OWASP/www-project-smart-contract-top-10/blob/8083e976d6d18013dce2d5e6e62f98e632151a09/LICENSE.md)

## プロジェクトリーダー
- [Jinson Varghese Behanan](mailto:jinson@owasp.org) (Twitter: [@JinsonCyberSec](https://twitter.com/JinsonCyberSec))
- [Shashank](mailto:shashank@credshields.com) (Twitter: [@cyberboyIndia](https://x.com/cyberboyIndia))
