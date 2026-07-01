---
title: "AWS WAF はどこまで DDoS に効くのか｜Geo match と Rate-based rule の使いどころ"
emoji: "🛡️"
type: "tech"
topics: ["AWS","AWSWAF","Security","DDoS"]
published: true
---
特定の国や地域から、同じような HTTP リクエストが大量に飛んでくる。

この状況でいきなり「DDoS だから Shield」と考えると、少し雑です。攻撃が CloudFront、Application Load Balancer、API Gateway まで到達し、HTTP のパス・ヘッダー・送信元国・リクエスト頻度として観測できているなら、まず設計対象になるのは AWS WAF です。

この記事では、指定国からの HTTP フラッドを題材に、AWS WAF の Geo match と Rate-based rule が効く場面、AWS Shield との役割差、導入時に外しやすいポイントを整理します。

## 先に結論

AWS WAF は、DDoS という言葉全体を雑に受け止めるサービスではありません。

効くのは、CloudFront、Application Load Balancer、API Gateway などに届く HTTP/HTTPS リクエストを条件で評価できる場面です。指定国からの大量アクセス、特定 URI への連続 POST、短時間に増える同一送信元からのリクエストなどは、WAF の設計対象になります。

一方で、L3/L4 の大規模攻撃そのものは AWS Shield の領域です。WAF と Shield は上下関係ではなく、見るレイヤーが違います。

| 観点 | AWS WAF | AWS Shield |
|---|---|---|
| 主な対象 | HTTP/HTTPS リクエスト | DDoS 攻撃全般、特にネットワーク/トランスポート層 |
| 判断材料 | 国、IP、URI、ヘッダー、メソッド、レートなど | トラフィックパターン、攻撃緩和基盤 |
| 典型用途 | L7 の遮断、制限、CAPTCHA/Challenge | DDoS 耐性の土台 |

## Geo match は「国で切る」ための道具

Geo match は、送信元 IP から推定される国・地域を条件にします。

たとえば、日本向けのサービスで、通常ほぼアクセスがない国から急に大量の HTTP リクエストが来た場合、Geo match を使って Count、Block、あるいは他条件との組み合わせを作れます。

ただし、国情報だけで攻撃者と正規ユーザーを完全に分けることはできません。VPN、プロキシ、クラウドプロバイダーの出口 IP を使われると、見かけの国は変わります。

そのため、Geo match は単独の正解というより、絞り込み条件の一つとして見るほうが安全です。

## Rate-based rule は HTTP フラッド対策の中心になりやすい

指定国という条件よりも、実務で効きやすいのは Rate-based rule です。

これは、一定時間内のリクエスト数がしきい値を超えた送信元に対して、Block、Count、CAPTCHA、Challenge などのアクションを適用する考え方です。

たとえば、ログイン API に対して以下のような異常があるとします。

```text
/login への通常アクセス:
  5 分あたり数十〜数百リクエスト

攻撃時:
  特定の送信元群から 5 分あたり数千リクエスト

WAF 側の考え方:
  /login かつ一定レート超過を条件に制限する
```

ここで重要なのは、しきい値を勘で決めないことです。WAF ログ、CloudWatch メトリクス、ALB アクセスログ、CloudFront ロ���を見て、通常時の山を把握してから決めます。

攻撃を止めるつもりで正規ユーザーを止めたら、ただの自爆です。

## CloudFront、ALB、API Gateway のどこで評価するか

AWS WAF は単体で浮かせて使うものではなく、保護対象に関連付けて使います。

| 関連付け先 | 向いている構成 | 設計ポイント |
|---|---|---|
| CloudFront | グローバル公開サイト、複数オリジンの入口 | オリジン直アクセスを塞ぐ |
| ALB | Web アプリケーションの入口 | CloudFront 併用時は責務を分ける |
| API Gateway | API 単位の制御 | 認証、Usage Plan、Lambda 側制御と重複させない |

公開 Web サービスなら、CloudFront に WAF を関連付ける構成がよく使われます。ただし、ALB や S3 オリジンへ直接アクセスできる状態だと、CloudFront 側の WAF を迂回されます。

入口を守るなら、入口を一つに寄せる。地味ですが、ここを外すと WAF ルールの前に構成が負けます。

## Shield に任せるべきところ

AWS Shield Standard は、多くの AWS サービスで標準的な DDoS 保護を提供します。Shield Advanced を使うと、より高度な保護、コスト保護、DDoS Response Team などの追加機能も選択肢になります。

ただし、Shield は「/login だけを絞る」「特定国からの POST だけを見る」「User-Agent とレートを組み合わせる」といった Web リクエスト条件の制御をするものではありません。

その細かい判断は WAF の仕事です。

## 導入時の安全な流れ

WAF ルールは、最初から Block で入れると原因調査が面倒になります。まずは Count で当たり方を確認するのが現実的です。

1. WAF ログを有効化する
2. Geo match や Rate-based rule を Count で作る
3. 正規トラフィックが巻き込まれていないか確認する
4. しきい値と対象パスを調整する
5. Block、CAPTCHA、Challenge へ段階的に切り替える

特に国単位の Block は影響が大きいです。海外ユーザー、出張者、監視サービス、外部連携が巻き込まれることがあります。

## まとめ

AWS WAF は、HTTP として見える攻撃に対して使うサービスです。

- 指定国からのアクセス制御には Geo match が使える
- HTTP フラッドには Rate-based rule が効きやすい
- Shield は DDoS 緩和の基盤で、WAF は L7 条件の制御役
- CloudFront、ALB、API Gateway のどこで評価するかを先に決める
- いきなり Block せず、Count とログで観測してから強める

「攻撃っぽいから Shield」ではなく、「HTTP リクエストとして条件を見られるなら WAF」と考える。これだけで、AWS の入口防御はかなり整理されます。