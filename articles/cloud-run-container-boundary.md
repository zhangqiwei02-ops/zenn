---
title: "Cloud Run を「楽なコンテナ平台」と誤解しないための設計ポイント"
emoji: "🚢"
type: "tech"
topics: ["GCP","CloudRun","GoogleCloud","Architecture","Serverless"]
published: true
---
# Cloud Run は便利だが、放置できるコンテナ基盤ではない

Cloud Run はかなり便利です。コンテナイメージを用意すれば、サーバ管理をほとんど意識せずに HTTP サービスや job を動かせます。GKE ほど重くなく、App Engine よりコンテナの自由度があり、Cloud Build や Artifact Registry と組み合わせればデプロイも素直です。

ただし、Cloud Run を「何も考えなくてよいコンテナ平台」と見ると危険です。cold start、request timeout、concurrency、CPU allocation、VPC egress、Cloud SQL 接続、background task、コスト曲線など、運用に入ってから効いてくる論点があります。

この記事では、Cloud Run を Kubernetes の簡単版として雑に持ち上げるのではなく、どこまで任せられて、どこから自分たちで設計すべきかを整理します。

## 先に結論

先に結論です。Cloud Run が向いているのは、HTTP request / event / job の単位で処理を切り出せて、stateless に近く、コンテナとして配布したいサービスです。

例えば、次のような用途です。

- 小〜中規模の Web API
- webhook 受信処理
- Pub/Sub から起動する非同期処理
- Cloud Scheduler から呼ぶ定期 job
- バッチ寄りだが短時間で終わる Cloud Run Jobs
- GKE を持つほどではない内部ツール

逆に、長時間常駐 worker、WebSocket のような長時間接続、細かい node 制御、複雑な service mesh、強い低レイテンシ要件があるなら、GKE、Compute Engine、または別の構成を比較した方が安全です。

Cloud Run の論点は「コンテナが動くか」ではありません。**そのコンテナの実行モデルが Cloud Run の前提に合っているか**です。

## Cloud Run が強い場面

Cloud Run の強みは、コンテナを素早く本番に近い形で出せることです。開発チームは Dockerfile を用意し、Artifact Registry に push し、Cloud Run service として公開できます。IAM、Secret Manager、Cloud Logging、Cloud Monitoring とも自然につながります。

特に相性がよいのは、request を受けて処理し、状態を外部に逃がせるサービスです。状態は Cloud SQL、Firestore、Cloud Storage、Memorystore などに置き、Cloud Run 自体は stateless に保つ。この形なら、スケールやデプロイがかなり楽になります。

また、Cloud Run Jobs を使えば、HTTP service ではなく一回きりの job としてコンテナを実行できます。短いバッチや定期処理なら、Cloud Scheduler や Workflows と組み合わせることで、GKE の CronJob を持ち込むより軽く済むことがあります。

## Cloud Run を雑に使うと苦しくなる理由

Cloud Run は管理対象のコンテナ基盤ですが、アプリケーション設計まで自動で正しくしてくれるわけではありません。

### cold start と起動時間

Cloud Run は request に応じて instance を増やせますが、起動には時間がかかります。軽いアプリなら問題になりにくいですが、依存パッケージが重い、起動時に DB 接続や外部 API 初期化が多い、コンテナイメージが大きい、といった条件では cold start が目立ちます。

min instances を設定すれば緩和できます。ただし、当然コストは増えます。「サーバレスだからゼロ円に近い」と思っていたのに、低レイテンシ維持のために常時起動へ寄せるなら、コスト前提は変わります。

### request timeout と長時間処理

Cloud Run service は request / response を中心に考えると扱いやすいです。長時間処理を HTTP request にぶら下げると、timeout、retry、重複実行、途中失敗の扱いが面倒になります。

長い処理は Pub/Sub、Cloud Tasks、Workflows、Cloud Run Jobs へ逃がす方が自然です。同期 API で全部終わらせようとすると、ユーザー体験も障害切り分けも悪くなります。

### concurrency とリソース設計

Cloud Run では 1 instance が複数 request を同時に処理できます。concurrency を上げれば instance 数を抑えられる可能性がありますが、CPU、memory、外部接続数、DB connection pool の設計を間違えると逆効果です。

例えば Cloud SQL に接続する API で concurrency を雑に上げると、アプリより先に database connection が詰まることがあります。Cloud Run の設定だけでなく、下流サービスの限界も一緒に見ないといけません。

### VPC egress と閉域接続

社内向け API、Cloud SQL private IP、内部ロードバランサ、オンプレ連携がある場合、VPC connector や egress 設計が効いてきます。Cloud Run 自体は簡単でも、ネットワークを閉じ始めると一気に設計項目が増えます。

Serverless VPC Access connector、Private Service Connect、Cloud NAT、Firewall、DNS。ここを後付けすると、最初の軽さはかなり薄れます。閉域前提なら、最初から network path を図にしておくべきです。

## GKE の代替として見すぎない

Cloud Run は GKE より軽い選択肢ですが、GKE の完全代替ではありません。

| 観点 | Cloud Run | GKE |
|---|---|---|
| 運用負荷 | 低い | 高い |
| コンテナ実行 | 管理対象 | 自由度が高い |
| node 制御 | ほぼ不要・不可 | 可能 |
| service mesh | 限定的 | 柔軟 |
| 長時間常駐 | 苦手寄り | 得意 |
| 細かいネットワーク制御 | 制約あり | 設計自由度が高い |

Cloud Run は「GKE を知らなくても使える Kubernetes」ではありません。むしろ、Kubernetes の運用を持ち込まない代わりに、Cloud Run の実行モデルに寄せるサービスです。ここを取り違えると、GKE でやるべきものを無理に Cloud Run へ押し込みます。

## 選定前のチェックポイント

Cloud Run を選ぶ前に、最低限この質問に答えると判断しやすくなります。

1. stateless に近い設計にできるか
2. request timeout 内に処理を収められるか
3. 長い処理を Pub/Sub、Cloud Tasks、Workflows、Cloud Run Jobs に逃がせるか
4. cold start を許容できるか、または min instances のコストを飲めるか
5. concurrency を上げたとき、Cloud SQL や外部 API が耐えられるか
6. VPC egress、private IP、Cloud NAT、DNS の設計が必要か
7. Cloud Logging / Cloud Monitoring / Error Reporting で追える粒度になっているか
8. GKE が必要なほど細かい制御を求めていないか

この問いに複数詰まるなら、Cloud Run だけで完結させる前提を疑った方がいいです。

## よくある失敗パターン

よくあるのは、background task を HTTP request の裏でそのまま走らせることです。request が終わった後の処理、途中失敗、retry、重複排除が曖昧になります。必要なら Cloud Tasks や Pub/Sub に切り出すべきです。

次に多いのは、Cloud SQL 接続を軽く見ることです。local では動いても、本番で concurrency が上がると connection pool が詰まります。Cloud SQL Auth Proxy、connector、接続数、timeout を含めて設計する必要があります。

そして、network を後から閉じるパターンです。最初は public endpoint で動かし、後から private access、VPC connector、Cloud NAT を足すと、障害切り分けが急に難しくなります。閉域要件があるなら、最初から前提に入れるべきです。

## まとめ

Cloud Run は便利です。小さく速く container service を出すにはかなり強い選択肢です。

ただし、放置できるコンテナ基盤ではありません。cold start、timeout、concurrency、VPC egress、Cloud SQL 接続、background task、observability、cost は自分たちで設計する必要があります。

Cloud Run を選ぶときは、「GKE より楽そう」ではなく、Cloud Run の実行モデルに処理が合っているかで判断した方が安全です。短く、stateless で、request / event / job の単位に切れるなら Cloud Run は強い。長く、重く、常駐し、ネットワーク制約も強いなら、別の基盤を比較するべきです。
