---
title: "NotionからQiita/Zennへ記事を自動公開するパイプラインを作り直して学んだこと"
emoji: "🚢"
type: "tech"
topics: ["automation","Qiita","Zenn"]
published: true
---
この記事では、この春に記事自動公開パイプラインを作り直して学んだことを整理します。

対象は、Notionをソース・オブ・トゥルースにして、Qiita、Zenn、独自サイトへ記事を公開する仕組みです。単なる「APIで投稿しました」ではありません。実際に運用すると、壊れるのは投稿APIそのものより、その前後です。

先に結論を書くと、自動公開で一番大事なのは「記事を投稿する処理」ではなく、次の4つでした。

- どのDBを正とするか
- 何を公開成功とみなすか
- 失敗したときにどこまで戻せるか
- プラットフォームごとの制約を同じルールで扱わないこと

このあたりを雑にすると、cronは毎日元気に動いているのに、記事は1本も出ないという悲しい状態になります。ログだけ健康、成果はゼロ。だいぶ嫌なタイプの自動化です。

## 全体構成

今回のパイプラインは、ざっくり次の流れです。

```text
Notion Database
  -> candidate selector
  -> draft generator
  -> QA checker
  -> publisher
      -> Montopi API
      -> Qiita API
      -> Zenn GitHub repo
  -> URL verifier
  -> Notion status updater
  -> notification
```

クラウドで組むなら、API Gateway + Lambda + S3 + CloudWatch Logs + EventBridge のような構成でも実装できます。今回はローカル/エージェント実行に寄せていますが、設計上の論点はほぼ同じです。

ポイントは、投稿処理���1つの巨大な関数にしないことです。

候補選定、本文生成、QA、公開、URL確認、Notion回填を分けないと、失敗時に「どこまで成功したのか」が分からなくなります。

## Notionをソース・オブ・トゥルースにする

最初に決めるべきなのは、記事の状態をどこで管理するかです。

今回はNotion DBを正にしました。記事ごとに、最低限次のプロパティを持たせます。

| プロパティ | 役割 |
|---|---|
| Title | 管理用タイトル |
| Topic Key | 重複防止用のキー |
| Publish Target Date | 狙う公開日 |
| Publish Order | 候補の優先順位 |
| Qiita Body Final | Qiita向け本文 |
| Zenn Body Final | Zenn向け本文 |
| Qiita URL | Qiita公開後URL |
| Zenn URL | Zenn公開後URL |
| Canonical URL | 独自サイトURL |
| Last Publish Attempt At | 最後に処理した時刻 |
| Article Published At | 公開成功とみなした時刻 |

ここをGitHubのMarkdownだけに寄せると、Qiitaや独自サイトの状態が見えにくくなります。逆にNotionだけに寄せすぎると、ZennのGitHub同期状態が見えなくなります。

なので、Notionには「状態」と「公開URL」を置き、ZennはGitHub repoを実体として扱う、という分担にしまし��。

## 旧DB ID問題：cronは正しく失敗し続ける

一番地味で、一番痛かったのがDB IDのズレです。

古いNotion DBを使っていたcron payloadが残っていて、毎日そのDBを読みに行っていました。Notion APIは object_not_found を返します。処理は前置チェックで止まります。

ここで厄介なのは、cron自体は正常に起動していることです。

```text
cron status: ok
publish result: failed before candidate selection
reason: Notion database object_not_found
```

cronがokでも、公開がokとは限りません。

この区別をログとNotionの結果に残さないと、「自動化は動いているのに、なぜ記事が出ないのか」という調査になります。だいたい不毛です。

対策として、DB IDを設定ファイルに寄せ、実行ログにも必ずDB IDを出すようにしました。

```json
{
  "databaseId": "3457282a-e978-81f9-9869-c622d3616b21",
  "today": "2026-05-23",
  "publishedCount": 0,
  "readyPages": 1
}
```

本番なら、CloudWatch LogsやDatadogに database_id、run_id、target_page_id を構造化ログとして出した方がいいです。人間がgrepする運用は、最初は楽でも後で裏切ります。

## readyページを全部処理しない

自動公開でやりがちな事故が、readyページの一括処理です。

「Publish Ready=true の記事を全部投稿する」という実装は簡単です。でも、これはかなり危険です。

- 同じ日に同テーマの記事が連発される
- Zennの日次上限を超える
- QAの弱い記事まで流れる
- 途中失敗したときにどこまで出たか分からない
- NotionのURL回填が混線する

そのため、通常実行では candidate を1件だけ選ぶようにしました。

```js
const pageToPublish = process.env.PUBLISH_TARGET_PAGE_ID
  ? fetchedPages.filter(p => p.id === process.env.PUBLISH_TARGET_PAGE_ID)
  : fetchedPages.slice(0, 1);
```

手動で特定記事を出したい場合だけ、PUBLISH_TARGET_PAGE_ID を指定します。

これは地味ですが、事故率をかなり下げます。自動化は速いので、失敗も速いです。まとめて壊れる設計にしない方がいいです。

## プラットフォームごとに成功条件を分ける

Montopi、Qiita、Zennでは、成功条件が違います。

| プラットフォーム | 成功条件 |
|---|---|
| Montopi | API投稿後、公開URLが200を返す |
| Qiita | API投稿後、Qiita URLが200を返す |
| Zenn | GitHub repoにpushされ、Zenn公開URLが200を返す |

Zennは特に注意が必要です。GitHubにMarkdownをpushできても、Zennの公開URLがすぐ200になるとは限りません。

つまり、次は成功ではありません。

```text
Zenn repo push succeeded
public URL: 404
```

これは「同期はしたが、公開確認は未完了」です。

CI/CDで言えば、GitHub Actionsのjobが通っただけで、本番ALB配下のURLが200とは限らないのと同じです。デプロイ成功と外形監視成功は別物です。

## Zennの日次上限を共通ルールにしない

Zennには、運用上「1日1本まで公開」という制約を入れました。

ここでやってはいけないのは、Zennの制約をQiitaや独自サイトにそのまま適用することです。

例えば、今日すでにZennで1本公開済みでも、Qiitaと独自サイトにはまだ出せる場合があります。

その場合は次のように分けます。

```text
Montopi: publish
Qiita: publish
Zenn: skip / defer
reason: daily public limit reached
```

実際、この分岐を入れたことで、Zennの制約に引きずられてQiitaまで止まる事故を避けられました。

一方で、Notionには「Zennは未公開」と分かる状態を残す必要があります。全部成功扱いにしてしまうと、後からZenn側だけ回収できません。

## QAは投稿前に止めるためにある

QAチェックは、点���を付けるためではなく、投稿前に止めるためにあります。

今回も、本文が弱い記事は最初にQAで落ちました。

```text
QA FAILED
reason: 技術キーワード密度不足
```

ここで大事なのは、QAを通すためにキーワードを雑に詰め込まないことです。

たとえば、AIエージェントの記憶基盤の記事なら、単に「RDS」「DynamoDB」「BigQuery」と並べるのではなく、どの用途にどのDBが向くのかを書く必要があります。

同じように、この記事ではNotion、Qiita、Zenn、GitHub repo、API Gateway、Lambda、S3、CloudWatch、GitHub Actionsといった要素を、パイプライン設計の中で意味のある場所に置いています。

QAは敵ではありません。雑な記事を外に出さないための、最後の嫌な編集者です。嫌ですが必要です。

## 通知は「送れたか」も記録する

自動公開では、通知も壊れます。

今回も、外部チャットへの事前通知を送ろうとして、セッション指定の都合で失敗するケースがありました。

このとき、やってはいけないのは localhost の内部APIにcurlして無理やり送ることです。実行環境が変わると壊れますし、権限境界も曖昧になります。

通知に失敗したら、公開処理とは別に記録します。

```text
pre-notification: failed
reason: session not found
publish: continued
```

通知失敗と公開失敗は別です。混ぜると運用判断を間違えます。

## 外形チェックまでやって初めて公開成功

投稿APIのレスポンスだけでは足りません。

公開後はURLを実際に確認します。

```bash
curl -L -s -o /dev/null -w '%{http_code}'   https://qiita.com/example/items/xxxx
```

MontopiもQiitaもZennも、最終的には公開URLが200を返すかで判断します。

CloudWatch Syntheticsや外形監視サービスを使うなら、このチェックを定期実行にしてもいいです。記事公開パイプラインでも、考え方はWebサービスのデプロイと同じです。

## 作り直して良かったルール

作り直して、特に効いたルールは次です。

- active DB IDを固定し、旧DBを使わない
- 未指定時はreadyページを1件だけ処理する
- PUBLISH_TARGET_PAGE_IDで手動対象を固定できるようにする
- Zennの日次上限をQiita/Montopiと分ける
- GitHub pushとZenn公開URL確認を別ステータスにする
- QA失敗時は投稿しない
- Last Publish Attempt At と Article Published At を分ける
- cron status ok と publish success を混同しない

どれも派手ではありません。でも、こういう地味なルールがないと、自動化はすぐに「動いているように見える失敗装置」になります。

## まとめ

記事自動公開パイプラインを作り直して分かったのは、投稿APIを叩く処理そのものは本質ではない、ということです。

本当に難しいのは、状態管理、成��判定、失敗時の切り分け、プラットフォーム差分の扱いです。

- Notionをソース・オブ・トゥルースにする
- cronの成功と公開成功を分ける
- readyページを一括処理しない
- Montopi、Qiita、Zennの成功条件を分ける
- Zennの上限を他プラットフォームに波及させない
- URLの外形チェックまで見る
- QAで止める勇気を持つ

このあたりを押さえると、自動公開はかなり安定します。

逆に言うと、ここを決めないままAIで本文生成だけ自動化しても、公開運用はすぐ壊れます。記事を作るAIより、記事を安全に出すための地味な制御の方が、実運用ではずっと効きます。
