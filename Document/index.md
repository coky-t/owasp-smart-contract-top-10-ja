---

layout: col-sidebar
title: OWASP Smart Contract Top 10
tags: smartcontract
level: 2
type: documentation
pitch: Welcome to the OWASP Top Ten for Smart Contracts

---

![OWASP Smart Contract Logo](assets/images/owasp-sctop10.png)


## スマートコントラクト Top 10 について

**OWASP スマートコントラクト Top 10: 2026** は Web3 開発者とセキュリティチームに、スマートコントラクトで見つかった上位 10 の脆弱性についての洞察を提供することを目的とした、標準認識ドキュメントです。これは **より広範な [OWASP Smart Contract Security (OWASP SCS) イニチアチブ](https://scs.owasp.org/) のサブプロジェクト** です。

これは近年悪用または発見された最も重大な弱点に対してスマートコントラクトが保護されていることを確認するためのリファレンスとして役立ちます。**スマートコントラクト Top 10** は他の OWASP SCS プロジェクトと併用することで包括的なリスクカバレッジを確保できます。

- **OWASP SC Top 10 live site (2026): [https://scs.owasp.org/sctop10/](https://scs.owasp.org/sctop10/)** 
- **OWASP SC Weakness Enumeration (SCWE):** https://scs.owasp.org/SCWE/  
- **OWASP SCS Checklist:** https://scs.owasp.org/checklists/  

Top 10 は以下の用途に使用できます。
- **認識**: スマートコントラクトに影響を及ぼす最も一般的で重大な脆弱性を理解します。
- **予防**: これらの既知の問題に対して保護するためのベストプラクティスを実装します。
- **標準コンプライアンス**: スマートコントラクトの安全な開発と評価を確保するために参照します。

> 注: 現在の **2026** Top 10 は **将来予測** です。その順位付けとカテゴリ定義は **2025 年に収集されたセキュリティインシデントと調査データ** に基づいており、翌年に最も重大になると予想されるリスクを予測するために使用されます。言い換えると、2025 年の侵害および脆弱性データは実証的な基盤を提供し、2026 年のリストはそれらの観察結果が近い将来にどのように予想されるかを反映しています。
>  
> このランキングは、最も一般的に発生して影響のある 10 のスマートコントラクトリスクについて、セキュリティ研究者、監査担当者、開発者、プロトコル所有者、および業界全体での意識を高めることを目的としています。

## 変更内容 (2025-2026)

![OWASP 2025 to 2026 Changes](assets/images/Top10mapping2025-2026.png)


### 2026 Top 10 (将来予測)

* SC01:2026 - [アクセス制御の脆弱性 (Access Control Vulnerabilities)](2026/ja/src/SC01-access-control-vulnerabilities.md)
* SC02:2026 - [ビジネスロジックの脆弱性 (Business Logic Vulnerabilities)](2026/ja/src/SC02-business-logic-vulnerabilities.md)
* SC03:2026 - [価格オラクル操作 (Price Oracle Manipulation)](2026/ja/src/SC03-price-oracle-manipulation.md)
* SC04:2026 - [フラッシュローンを利用した攻撃 (Flash Loan–Facilitated Attacks)](2026/ja/src/SC04-flash-loan-facilitated-attacks.md)
* SC05:2026 - [入力バリデーションの欠如 (Lack of Input Validation)](2026/ja/src/SC05-lack-of-input-validation.md)
* SC06:2026 - [チェックされていない外部呼び出し (Unchecked External Calls)](2026/ja/src/SC06-unchecked-external-calls.md)
* SC07:2026 - [算術エラー (Arithmetic Errors)](2026/ja/src/SC07-arithmetic-errors.md)
* SC08:2026 - [再入攻撃 (Reentrancy Attacks)](2026/ja/src/SC08-reentrancy-attacks.md)
* SC09:2026 - [整数オーバーフローとアンダーフロー (Integer Overflow and Underflow)](2026/ja/src/SC09-integer-overflow-underflow.md)
* SC10:2026 - [プロキシとアップグレード可能性の脆弱性 (Proxy & Upgradeability Vulnerabilities)](2026/ja/src/SC10-proxy-and-upgradeability-vulnerabilities.md)

### 概要

| タイトル | 説明 |
| -- | -- |
| SC01 - Access Control Vulnerabilities | Access control flaws allow unauthorized users or roles to invoke privileged functions or modify critical state, often leading to full protocol compromise when admin, governance, or upgrade paths are exposed. |
| SC02 - Business Logic Vulnerabilities | Design-level flaws in lending, AMM, reward, or governance logic that break intended economic or functional rules, enabling attackers to extract value even when low-level checks appear correct. |
| SC03 - Price Oracle Manipulation | Weak oracles and unsafe price integrations that let attackers skew reference prices, enabling under-collateralized borrowing, unfair liquidations, and mispriced swaps as part of larger exploit chains. |
| SC04 - Flash Loan–Facilitated Attacks | Attacks that use large, uncollateralized flash loans to magnify small bugs (in logic, pricing, or arithmetic) into large drains, by executing complex multi-step sequences in a single transaction. |
| SC05 - Lack of Input Validation | Missing or weak validation of user, admin, or cross-chain inputs that allows unsafe parameters to reach core logic, corrupting state, breaking assumptions, or enabling direct fund loss. |
| SC06 - Unchecked External Calls | Unsafe interactions with external contracts or addresses where failures, reverts, or callbacks are not safely handled, often enabling reentrancy or inconsistent state. |
| SC07 - Arithmetic Errors | Subtle bugs in integer math, scaling, and rounding; especially in share, interest, and AMM calculations; that can be repeatedly exploited to cause precision loss, or siphon value, particularly when paired with flash loans. |
| SC08 - Reentrancy Attacks | Situations where external calls can re-enter vulnerable functions before state is fully updated, allowing repeated withdrawals or state changes from outdated views of contract state. |
| SC09 - Integer Overflow and Underflow | Dangerous arithmetic on platforms or code paths without robust overflow checks, leading to wrapped values, broken invariants, and potential drains of liquidity or mis-accounting. |
| SC10 - Proxy & Upgradeability Vulnerabilities | Misconfigured or weakly governed proxy, initialization, and upgrade mechanisms that let attackers seize control of implementations or reinitialize critical state. |


## データソース

### SolidityScan の Web3HackHub:

OWASP スマートコントラクト Top 10 脆弱性を特定して検証するために、複数の信頼できるソースからの洞察を取り入れ、特に **[SolidityScan の Web3HackHub](https://solidityscan.com/web3hackhub?year=2025) (2025)** に注目しました。このリソースは、ブロックチェーン関連のインシデントの包括的なデータベースを提供し、攻撃ベクトル、金銭的損失、傾向に関する貴重なデータを提供します。

Web3HackHub は 2011 年以降の侵害を記録しており、進化する攻撃方法、巧妙化するエクスプロイト、各インシデントから得られた教訓を分析できます。

![SolidityScan Web3HackHub 2025](assets/images/solidityscan-web3hackhub2025.png)

The complete **methodology, ranking logic, and external data sources** for the 2026 Top 10 are documented on the OWASP SCS site:

- **Methodology & Data for the 2026 Top 10: https://scs.owasp.org/sctop10/data-sources/**

On that page you’ll find:
- How the practitioner survey is designed and how category rankings are aggregated.  
- How 2025 smart‑contract incident data (SolidityScan Web3HackHub, SlowMist, BlockSec, DeFiHackLabs, etc.) is mapped into OWASP categories.  
- Category‑wise loss totals, risk metrics, and the justification for the final ordering.

## ライセンス
The OWASP Smart Contract Top 10 (2026) is [licensed](https://github.com/OWASP/www-project-smart-contract-top-10/blob/main/LICENSE.md) under the [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/), the Creative Commons Attribution-ShareAlike 4.0 license. Some rights reserved.

![license](assets/images/by-nc-sa.png)

## プロジェクトリーダー
- [Jinson Varghese Behanan](mailto:jinson@owasp.org) (Twitter: [@JinsonCyberSec](https://x.com/JinsonCyberSec))
- [Shashank](mailto:shashank@credshields.com) (Twitter: [@cyberboyIndia](https://x.com/cyberboyIndia))
