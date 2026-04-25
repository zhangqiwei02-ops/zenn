---
title: "Azure Functions を何でも屋にしない判断軸｜長時間処理・冷スタート・VNet 制約"
emoji: "⚙️"
type: "tech"
topics: ["Azure","AzureFunctions","ContainerApps","AppService","Serverless"]
published: true
---
# Azure Functions は万能なバックグラウンド基盤ではない

Azure Functions は便利です。HTTP Trigger、Timer Trigger、Queue Trigger を数分で立ち上げられます。Blob Storage、Queue Storage、Service Bus、Event Grid と組み合わせれば、小さなイベント駆動処理はかなり速く作れます。

ただし、便利だからといって、すべてのバックグラウ��ド処理を Azure Functions に寄せると後で苦しくなります。コールドスタート、実行時間、VNet Integration、Private Endpoint、Durable Functions の複雑さ、Application Insights での追跡など、運用に入ってから効いてくる論点が多いからです。

この記事では、Azure Functions を否定するのではなく、Container Apps や App Service と比較しながら、Functions を選ぶべき境界を整理します。

## 先に結論

先に結論です。Azure Functions が向いているのは、短く、状態を長く持たず、Trigger と処理責務が素直につながるイベント駆動処理です。

例えば、次のような処理です。

- Blob Storage へのファイル到着を受けて変換する
- Queue Trigger で短い非同期ジョブを処理する
- Timer Trigger で軽い定期処理を回す
- Event Grid や Service Bus のイベントを受けて通知する
- HTTP Trigger で薄い webhook を受ける

逆に、長時間バッチ、常駐ワーカー、WebSocket、重い API、複雑なワークフロー、厳しい VNet / Private Endpoint 前提の構成は、最初から Container Apps や App Service も比較した方が安全です。

つまり論点は「Functions で書けるか」ではありません。**その処理を Functions に乗せたとき、運用まで含めて素直か**です。

## Azure Functions が強い場面

Functions が一番気持ちよく使えるのは、小さなイベント駆動処理です。アプリケーション本体から切り出したい処理を、必要なときだけ動かせます。

たとえば、画像アップロード後のサムネイル作成、Service Bus メッセージを受けた通知、軽い日次集計、外部 SaaS からの webhook 受付などです。この程度なら、Function App を小さく保ち、Managed Identity で Storage Account や Key Vault にアクセスさせ、Application Insights でログを見る構成が比較的素直です。

Consumption Plan で小さく始め、コールドスタートやスケール要件が厳しくなれば Premium Plan を検討する、という進め方もできます。ここまでは Azure Functions の得意領域です。

## 何でも Functions に寄せると苦しくなる理由

問題は、最初の成功体験をそのまま横展開することです。「非同期処理なら全部 Queue Trigger でよい」と考えると、少しずつ設計が歪みます。

### 長時間処理は設計が急に重くなる

短いタスクなら Functions は快適ですが、長く走る処理ではタイムアウト、再試行、部分失敗��途中経過の保存、再開設計が必要になります。Durable Functions を使えば orchestration はできますが、複雑さが消えるわけではありません。

Durable Functions では、orchestrator の再実行前提、deterministic 制約、state 永続化、失敗時の再開ポイント、local debug のしづらさを理解する必要があります。ここを軽く見ると、普通の container worker より読みにくい構成になります。

### コールドスタートは無視できない

HTTP Trigger をユーザー向け API に出す場合、cold start は体感に直結します。Premium Plan で緩和できるとはいえ、「serverless だから常に安くて速い」と思って設計すると外します。

依存パッケージが重い、起動時に外部接続が多い、VNet Integration を使う、アクセスがスパイクする。この条件が重なる API は、App Service や Container Apps の方が扱いやすいことがあります。

### ネットワーク制約が強いと抽象化メリットが薄れる

社内システム連携では、VNet Integration、Private Endpoint、Outbound 制御、DNS、Key Vault 参照が地味に効きます。Functions でも対応できますが、サーバレスだから勝手に楽になるわけではありません。

内部 API、Storage Account、Azure SQL、Key Vault をすべて閉域寄りにすると、接続確認と障害切り分けのコストが上がります。この時点で、実行基盤の自由度が高い Container Apps の方が自然な場合があります。

## Container Apps / App Service との比較

Azure Container Apps は、コンテナとして動かしたい処理に向いています。KEDA によるスケール、ジョブ、常駐 worker、runtime の自由度が必要なら、Functions より素直です。

| 観点 | Azure Functions | Azure Container Apps |
|---|---|---|
| 実行モデル | Trigger 中心 | Container / Job / Worker 中心 |
| 開発の軽さ | 高い | 中程度 |
| ランタイム自由度 | 低め | 高い |
| 長時間処理 | 苦手寄り | 対応しやすい |
| 常駐 worker | 向かない | 扱いやすい |
| ネットワーク設計 | 構成次第で複雑 | コンテナ前提で整理しやすい |

App Service は古く見えますが、常時起動の Web API、管理画面、セッションを持つアプリでは今でも強いです。Functions は「小さく分割された trigger 処理」、App Service は「1 つのアプリケーションとして保ちたいもの」、Container Apps は「コンテナ自由度が必要な API / worker / job」と考えると判断しやすくなります。

## 選定前のチェックポイント

Azure Functions を選ぶ前に、最低限この質問に答えるべきです。

1. 1 回の実行は短く終わるか
2. Trigger と処理責務が素直につながっているか
3. retry、idempotency、DLQ、失敗時リカバリを設計できるか
4. VNet、Private Endpoint、Outbound 制御の要求は強すぎないか
5. cold start が UX や SLA を壊さないか
6. 常駐 worker や長時間接続を求めていないか
7. Durable Functions を入れても構成が読めるか
8. Application Insights で追跡したい単位が Function の粒度と合っているか

ここで複数詰まるなら、Functions から入る理由は薄いです。最初から Container Apps や App Service を比較対象に入れた方が、あとで楽です。

## よくある失敗パターン

よくある失敗は、「serverless だから運用が消える」と思うことです。消えません。監視、再試行、DLQ、timeout、設定管理、secret 管理、権限設計は必要です。サーバを見る量が減るだけで、運用責務そのものは残ります。

もう一つは、API も job も orchestration も全部 1 つの Function App に詰めることです。責務が混ざると、deploy 影響範囲、設定の衝突、障害切り分けが面倒になります。Function App は小さく保った方が安全です。

最後に、ネットワークとセキュリティ要件を後から足すパターンです。Managed Identity、Key Vault、Private Endpoint、VNet Integration を後付けすると、最初の軽さが一気に消えます。閉域前提なら、最初からその前提でサービスを選ぶべきです。

## まとめ

Azure Functions は優秀ですが、万能なバックグラウンド基盤ではありません。

本当に見るべきなのは、Trigger ベースで素直に切れる処理か、長時間実行や複雑な編成を含むか、cold start を許容できるか、VNet や Private Endpoint の制約が強いか、コンテナ自由度が必要か、という前提です。

Functions は「小さく、短く、イベント駆動で、責務が明確」な処理には強いです。逆に、長く重く複雑で、ネットワーク要件も厳しい処理まで全部押し込むと、サーバレスのうまみより設計負債が先に増えます。

迷ったときは、Functions を第一候補に固定するのではなく、Functions / Container Apps / App Service のどれが運用コストを一番素直に下げるかで選ぶのが実務的です。
