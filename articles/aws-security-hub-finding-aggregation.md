---
title: "Security Hub は検知ツールではなく、AWS セキュリティ検出結果の整理役として見る"
emoji: "🛡️"
type: "tech"
topics: ["AWS","securityhub","guardduty","inspector","Security"]
published: true
---
AWS でセキュリティサービスを増やしていくと、検知結果はすぐに散らばります。GuardDuty、Inspector、Macie、Config、それぞれが別々の観点で finding を出します。

先に結論を書くと、Security Hub は単体で攻撃を止めるサービスではありません。この記事では、Security Hub を「AWS セキュリティ検出結果を集約し、運用に乗せるための整理役」として理解するための見方を整理します。

## Security Hub の役割

Security Hub の役割は、複数の AWS セキュリティサービスや外部製品から出る検出結果を、標準化された finding として集めることです。

代表的には次のような情報が集まります。

| 入力元 | 内容 |
|---|---|
| GuardDuty | 脅威検知、不審な API 呼び出し、異常通信 |
| Inspector | EC2、ECR、Lambda などの脆弱性 |
| Macie | S3 上の機密データ検出 |
| Security standards | CIS や AWS Foundational Security Best Practices のチェック |

この時点で分かる通り、Security Hub は「新しい検知ロジックを全部持っているサービス」ではありません。すでに出ている検出結果を、見やすく、扱いやすくするための場所です。

## なぜ集約が必要なのか

小さな検証環境なら、各サービスの画面を見に行く運用でも何とかなります。

ただし、アカウント数が増え、本番環境が増え、検知サービスも増えると、そのやり方はすぐ限界になります。

- どの finding が本当に危険なのか分からない
- 同じリソースに複数の指摘が出る
- 担当チームへ渡す基準が曖昧になる
- 対応済みなのか、例外扱いなのか追えない
- 通知が多すぎて誰も見なくなる

Security Hub は、この混乱を完全に解決してくれるわけではありません。ただ、整理するための共通の置き場にはなります。

## severity だけでは優先順位にならない

Security Hub では finding の severity を見られます。Critical や High を優先するのは自然です。

しかし、実務では severity だけでは足りません。

| 観点 | 例 |
|---|---|
| 重大度 | Critical / High か |
| 露出 | インターネットから到達可能か |
| 業務影響 | 本番、顧客データ、決済系に関係するか |
| 継続性 | 一時的な検出か、長期間放置されているか |

例えば、検証環境の High finding と、本番の公開リソースに関係する Medium finding では、後者を先に見る判断もあり得ます。

Security Hub は、こうした判断材料を集める場所です。判断まで自動で完璧にやってくれる場所ではありません。ここを勘違いすると、画面だけ立派で運用が腐ります。

## EventBridge で運用フローにつなぐ

Security Hub の finding は EventBridge に流せます。これを使うと、通知、Lambda、チケットシステムなどへつなげられます。

```text
GuardDuty / Inspector / Macie
        ↓
   Security Hub
        ↓
   EventBridge
        ↓
SNS / AWS Chatbot / Lambda / チケット管理
```

ただし、すべての finding を通知に流すのはやめたほうがいいです。通知の形をしたノイズ製造機になります。

まずは次のように絞るのが現実的です。

- severity が Critical または High
- workflow status が NEW
- 本番アカウントだけ
- GuardDuty の特定タイプだけ
- Inspector の exploitable な脆弱性だけ

この記事で言いたいのは、Security Hub を入れるなら、画面を見るだけで終わらせず、対応フローまで設計するべきということです。

## 最初に決めるべき運用ルール

Security Hub を有効化する前後で、最低限次を決めておくと後が楽です。

1. どの AWS アカウントを集約対象にするか
2. どの security standards を有効にするか
3. Critical / High を誰が一次確認するか
4. 例外扱いの記録方法
5. finding を RESOLVED / SUPPRESSED にする基準

マルチアカウント構成なら、AWS Organizations と委任管理者アカウントを使った集約も検討します。アカウントごとに人力で見に行く設計は、最初は動いても、規模が少し大きくなるとだいたい崩れます。

## まとめ

Security Hub は、セキュリティ検知の主役というより、検知結果を運用可能な形にまとめるためのハブです。

要点は次の通りです。

- GuardDuty、Inspector、Macie などの finding を集約できる
- 価値は「新しい検知」より「整理」と「優先順位づけ」にある
- severity だけでなく、露出、業務影響、継続性も見る
- EventBridge と組み合わせると通知やチケット化へ流せる
- 有効化より先に、担当者と例外処理ルールを決める

Security Hub は派手なサービスではありません。ですが、検知結果が散らばっている環境では、かなり効きます。セキュリティ運用は、見つけるだけでは終わりません。処理できる形に畳むところまでが設計です。