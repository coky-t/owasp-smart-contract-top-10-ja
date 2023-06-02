---

layout: col-sidebar
title: OWASP Smart Contract Top 10
tags: smartcontract
level: 2
type: 
pitch: Welcome to the OWASP Top Ten for Smart Contracts

---

## スマートコントラクト Top 10 について

OWASP スマートコントラクト Top 10 はスマートコントラクトで見つかった脆弱性の Top 10 についての洞察を Web3 開発者やセキュリティチームに提供することを意図した標準的な啓発文書です。

これは過去数年間に悪用や発見された脆弱性の Top 10 に対してスマートコントラクトが保護されるようにするための参考資料として機能します。

### Top 10

* SC01:2023 - [再入攻撃 (Reentrancy Attacks)](2023/en/src/SC01-reentrancy-attacks.md)
* SC02:2023 - [整数オーバーフローとアンダーフロー (Integer Overflow and Underflow)](2023/en/src/SC02-integer-overflow-underflow.md)
* SC03:2023 - [タイムスタンプの依存性 (Timestamp Dependence)](2023/en/src/SC03-timestamp-dependence.md)
* SC04:2023 - [アクセス制御の脆弱性 (Access Control Vulnerabilities)](2023/en/src/SC04-access-control-vulnerabilities.md)
* SC05:2023 - [フロントランニング攻撃 (Front-running Attacks)](2023/en/src/SC05-front-running-attacks.md)
* SC06:2023 - [サービス拒否攻撃 (Denial of Service (DoS) Attacks)](2023/en/src/SC06-denial-of-service-attacks.md)
* SC07:2023 - [ロジックエラー (Logic Errors)](2023/en/src/SC07-logic-errors.md)
* SC08:2023 - [安全でないランダム性 (Insecure Randomness)](2023/en/src/SC08-insecure-randomness.md)
* SC09:2023 - [ガス制限の脆弱性 (Gas Limit Vulnerabilities)](2023/en/src/SC09-gas-limit-vulnerabilities.md)
* SC10:2023 - [チェックされていない外部呼び出し (Unchecked External Calls)](2023/en/src/SC10-unchecked-external-calls.md)

### 概要

| タイトル | 説明 |
| -- | -- |
| SC01 - 再入攻撃 (Reentrancy Attacks) | これは攻撃者がスマートコントラクト内の機能を繰り返し呼び出すことができる場合に、コントラクトの状態が期待通りに更新されていないことを悪用するものです。これによりコントラクトから資金やその他のリソースが流出する可能性があります。 |
| SC02 - 整数オーバーフローとアンダーフロー (Integer Overflow and Underflow) | これらの脆弱性は数値演算の結果が変数のデータ型の範囲外の値になる場合に発生します。スマートコントラクトでは、これを悪用して残高やその他の重要な値を操作する可能性があります。 |
| SC03 - タイムスタンプの依存性 (Timestamp Dependence) | スマートコントラクトの動作がそれが含まれるブロックのタイムスタンプに依存している場合、操作に対して脆弱である可能性があります。マイナーがぷろっくのタイムスタンプをある程度制御できるためです。 |
| SC04 - アクセス制御の脆弱性 (Access Control Vulnerabilities) | スマートコントラクトがアクセス制御を適切に実装していない場合、重要な機能が露出したままになる可能性があります。これにより認可されていないユーザーがコントラクトの状態を変更したり、資金を引き出すなど、制限されるべきアクションを実行できる可能性があります。 |
| SC05 - フロントランニング攻撃 (Front-running Attacks) | フロントランニングはブロックチェーンシステムに特有の脆弱性です。攻撃者は保留中のトランザクションを観察し、より高いガス料金で独自のトランザクションを発行し、マイナーが最初にそれをブロックチェーンに含めるように動機付けることができます。 |
| SC06 - サービス拒否攻撃 (Denial of Service (DoS) Attacks) | DoS 攻撃はコントラクトを応答不能にしたり、利用不能にすることを目的としています。スマートコントラクトでは、利用可能なガスをすべて消費したり、トランザクションを継続的に失敗させることでこれを実現できます。 |
| SC07 - ロジックエラー (Logic Errors) | スマートコントラクトのコードが貧弱である場合、意図しない動作につながるロジックエラーを含む可能性があります。これは正しくない計算から欠陥のある条件文、さらには管理機能の露出に至るまで、多岐にわたる可能性があります。 |
| SC08 - 安全でないランダム性 (Insecure Randomness) | ブロックチェーンネットワークは本質的に決定論的であるため、スマートコントラクトで真のランダム性を生成することは困難になります。攻撃者が推定される乱数を予測したり影響を与えることができる場合、コントラクトを有利になるように操作できます。 |
| SC09 - ガス制限の脆弱性 (Gas Limit Vulnerabilities) | 各イーサリアムブロックにはガス制限があり、含めることができる操作の数が制限されています。コントラクト内の機能がこの制限を超えるガスを必要とする場合、その機能は実行できなくなり、コントラクトやその資金が凍結する可能性があります。 |
| SC10 - チェックされていない外部呼び出し (Unchecked External Calls) | コントラクトが外部機能を呼び出す際、呼び出しの結果を適切にチェックしない可能性があります。外部呼び出しを失敗しても、元のコントラクトがこれをチェックしない場合、呼び出しが成功したものとみなして実行を継続し、意図しない結果を招く可能性があります。 |

## ライセンス
The OWASP Smart Contract Top 10 document is licensed under the [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/), the Creative Commons
Attribution-ShareAlike 4.0 license. Some rights reserved.

## プロジェクトリーダー
- [Jinson Varghese Behanan](mailto:jinson@owasp.org) (Twitter: [@JinsonCyberSec](https://twitter.com/JinsonCyberSec))
