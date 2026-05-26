---
title: "TiDBでAIエージェントの記憶基盤を設計する：RAG・SQL・運用ログの分け方"
emoji: "🧠"
type: "tech"
topics: ["AI","Database","Architecture"]
published: true
---
この記事では、TiDBを使ってAIエージェントの「記憶基盤」を作るときの設計を整理します。

RAGを作る話になると、すぐに「ドキュメントをベクトル化して検索しましょう」で終わりがちです。正直、それだけなら記事にする価値は薄いです。AIエージェントのメモリは、検索インデックスだけではありません。

先に結論を書くと、AIエージェントの記憶は次の5つに分けて設計した方が安全です。

- 生の会話ログ
- 要約された短期メモリ
- ユーザーや業務に紐づく長期メモリ
- RAG用のドキュメント/チャンク
- 検索・回答・評価ログ

TiDBのようなSQLベースのデータ基盤を使う意味は、単にデータを置くことではありません。トランザクション、セカンダリインデックス、JSON列、監査、削除、評価を同じ運用面で扱えることにあります。

AWSで言えば、RDSに業務トランザクションを置き、DynamoDBにセッション状態を置き、BigQueryに評価ログを流すような構成もあります。TiDBで考える場合は、これらをどこまで1つのSQLデータ基盤に寄せるか、どこから検索エンジンや分析基盤に逃がすかを設計するのが本題です。

## 想定するユースケース

例えば、社内ナレッジに答えるAIエージェントを考えます。

ユーザーはSlackやWeb UIから質問します。

- 「この顧客の前回の問い合わせ内容は？」
- 「この障害の暫定対応は何だった？」
- 「私が前に依頼した見積もり条件を覚えている？」

ここで必要なのは、ドキュメント検索だけではありません。過去の会話、確定したメモ、忘れるべき情報、回答の根拠、ユーザーからのフィードバックが必要です。

## RAGとメモリを混ぜない

まず分けるべきなのは、RAGとメモリです。

| 種類 | 主な用途 | 保存するもの | 注意点 |
|---|---|---|---|
| RAG | 外部知識の参照 | 文書チャンク、URL、更新日時 | 古い文書の扱い |
| 短期メモリ | 会話の文脈維持 | 直近会話、要約 | セッション終了後の扱い |
| 長期メモリ | 継続利用の個人化 | ユーザー設定、業務上の事実 | 削除・訂正が必要 |
| 評価ログ | 品質改善 | 検索結果、回答、評価 | 個人情報の混入 |

RAGのチャンクをそのまま「エージェントの記憶」と呼ぶと、後で破綻します。

例えば、ユーザーが「前に話した条件で」と言ったとき、検索すべき対象は社内ドキュメントではなく、過去会話や確定済みメモリです。逆に、制度や仕様の説明では社内ドキュメントのRAGが重要になります。

## 最小スキーマ案

TiDB上に置くなら、最小構成は次のように分けます。

```sql
CREATE TABLE conversation_turns (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  tenant_id VARCHAR(64) NOT NULL,
  user_id VARCHAR(64) NOT NULL,
  session_id VARCHAR(64) NOT NULL,
  role VARCHAR(16) NOT NULL,
  content TEXT NOT NULL,
  created_at DATETIME NOT NULL,
  INDEX idx_session_time (tenant_id, session_id, created_at)
);

CREATE TABLE memory_summaries (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  tenant_id VARCHAR(64) NOT NULL,
  user_id VARCHAR(64) NOT NULL,
  memory_type VARCHAR(32) NOT NULL,
  summary TEXT NOT NULL,
  importance_score DECIMAL(5,2) NOT NULL DEFAULT 0,
  source_turn_ids JSON,
  expires_at DATETIME NULL,
  updated_at DATETIME NOT NULL,
  INDEX idx_user_memory (tenant_id, user_id, memory_type)
);

CREATE TABLE documents (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  tenant_id VARCHAR(64) NOT NULL,
  source_url TEXT,
  title TEXT,
  body TEXT,
  version VARCHAR(64),
  updated_at DATETIME NOT NULL
);

CREATE TABLE retrieval_logs (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  tenant_id VARCHAR(64) NOT NULL,
  user_id VARCHAR(64) NOT NULL,
  query TEXT NOT NULL,
  retrieved_ids JSON,
  answer_id VARCHAR(64),
  latency_ms INT,
  created_at DATETIME NOT NULL,
  INDEX idx_retrieval_time (tenant_id, created_at)
);
```

実際にはベクトル検索や全文検索の構成に応じて追加テーブルが必要になります。ただ、最初から全部を1テーブルに押し込まない方がいいです。

## DB選定の境界を決める

ここを曖昧にすると、AI基盤はすぐに「何でも入っている謎DB」になります。

| 選択肢 | 向いているもの | 向いていないもの |
|---|---|---|
| TiDB / RDS系SQL | 会話ログ、メモリ、権限、評価ログ、トランザクション | 超低レイテンシの一時状態だけを大量に捌く用途 |
| DynamoDB系Key-Value | セッション状態、短TTLの一時データ | 複雑な監査クエリやJOINが多い分析 |
| BigQuery系DWH | 大量の評価ログ分析、週次レポート | ユーザー操作ごとの同期トランザクション |
| 専用ベクトル検索 | 類似検索、rerank前の候補取得 | 記憶の訂正、監査、権限制御の主DB |

TiDBを使うなら、少なくとも「業務上の事実」「ユーザー単位の記憶」「検索・回答ログ」はSQLで追える形にしておきたいです。逆に、巨大な埋め込みベクトルそのものをどう持つかは、利用するTiDB Cloudの機能や外部検索基盤の選定に合わせて決めます。未検証の性能を盛って書く必要はありません。

## 会話ログは消せる形で保存する

AIエージェントのログは便利ですが、危険でもあります。

会話には、個人情報、顧客情報、内部事情、まだ確定していない判断が入ります。だから「全部保存して後でAIに食わせる」は雑です。

最低限、次を設計します。

- tenant_id で組織を分ける
- user_id でアクセス制御できるようにする
- session_id で会話単位を追えるようにする
- 保存期間を決める
- 削除要求に対応できるようにする
- 生ログと要約メモリを分ける
- CloudTrail のような監査ログ発想で、誰が何を参照したかを残す

生ログは一定期間で削除し、必要な情報だけ要約メモリに昇格させる設計が現実的です。

## 長期メモリには重要度と根拠を持たせる

AIエージェントに長期記憶を持たせるなら、「何を覚えるか」より「なぜ覚えたか」を残すべきです。

例えば次のようなメモリです。

```json
{
  "memory_type": "user_preference",
  "summary": "ユーザーは回答に実装例と運用上の注意点を含めることを好む",
  "importance_score": 0.82,
  "source_turn_ids": [1021, 1044, 1088],
  "expires_at": null
}
```

source_turn_ids があると、後で「なぜこのメモリがあるのか」を確認できます。これは監査にも、誤記憶の修正にも効きます。

逆に、根拠のないメモリは危険です。AIが一度勘違いして保存した情報を、次回以降も事実として使い続けます。これがメモリ機能の怖いところです。

## RAGパイプラインは検索前と回答後が重要

RAGは「検索してLLMに渡す」だけでは足りません。

私は次の流れに分けます。

```text
user query
  -> query rewrite
  -> policy check
  -> hybrid retrieval
  -> rerank
  -> answer generation
  -> citation check
  -> feedback logging
```

query rewriteでは、会話文を検索しやすい形にします。

例えば「さっきの料金の話」は、そのまま検索しても弱いです。直近会話から「TiDB Cloudの料金プラン比較」などに補正します。

ただし、補正したクエリも retrieval_logs に残します。あとで回答品質を調べるときに、ユーザーの元質問だけ見ても原因が分からないからです。

## TiDBを使うと嬉しい場面

TiDB CloudやTiDBを使う価値が出やすいのは、AI機能そのものより運用側です。

- 会話ログと業務データをSQLで追える
- 複数テーブルをトランザクションで更新できる
- tenant_id / user_id 単位でアクセス制御を設計しやすい
- retrieval_logs と feedback を分析しやすい
- メモリ削除や訂正をSQLで実装しやすい
- スケールしたときに単一ノードDBより逃げ道がある
- RDS的な業務DBの堅さと、BigQuery的な分析要求の間をどこまで寄せるか判断できる

もちろん、すべての小さなRAGアプリにTiDBが必要とは限りません。PoCならSQLiteやPostgreSQLから始めてもいいです。

ただ、AIエージェントを業務で継続運用するなら、後から欲しくなるのは派手な生成機能ではなく、ログ、監査、削除、評価です。ここを最初からテーブルとして設計しておく意味はあります。

## 忘れる設計を入れる

AIメモリ設計で一番抜けがちなのは、忘れ方です。

記憶は増えるほど良い、というものではありません。

- 古いプロジェクト情報
- 期限切れの価格条件
- すでに修正されたユーザー希望
- 一時的なトラブル対応
- 個人情報を含む会話

これらを永遠に覚えているエージェントは、賢いのではなく危ないです。

memory_summaries には expires_at を持たせます。さらに、重要度が低く、参照されていないメモリは定期的に削除または再要約します。

```sql
DELETE FROM memory_summaries
WHERE expires_at IS NOT NULL
  AND expires_at < NOW();
```

地味ですが、こうい���処理がないメモリ機能は、時間が経つほどノイズ製造機になります。

## 評価ログがないRAGは改善できない

回答が外れたとき、「LLMが悪い」で終わらせると改善できません。

見るべきは分解されたログです。

- query rewrite が悪かったのか
- retrieval が外したのか
- rerank が間違えたのか
- 正しい文書は取れていたが回答生成で崩れたのか
- そもそもドキュメントが古かったのか

retrieval_logs と feedback を残しておけば、後から原因を追えます。

```json
{
  "query": "前回の契約条件は？",
  "rewritten_query": "顧客A 2026年4月 契約条件 支払いサイト",
  "retrieved_ids": ["doc_91", "mem_44", "turn_8301"],
  "answer_id": "ans_20260522_001",
  "feedback": "missing_context"
}
```

この粒度で残っていると、改善はかなり現実的になります。

## まとめ

TiDBでAIエージェントの記憶基盤を作るなら、RAGだけを見ていると足りません。

設計すべきなのは、検索インデックスではなく記憶のライフサイクルです。

- 生ログと要約メモリを分ける
- 長期メモリには重要度、根拠、期限を持たせる
- RAGとユーザー記憶を混ぜない
- retrieval_logs と feedback を残す
- 削除、訂正、監査を最初から設計する
- RDS、DynamoDB、BigQuery、専用検索基盤との境界を決める
- すべてを覚えるのではなく、忘れる仕組みを入れる

TiDBのようなデータ基盤は、AIの回答を派手にするためだけのものではありません。むしろ、AIエージェントを業務で壊さず運用するための土台です。

AI時代のデータ基盤で重要なのは、保存量ではありません。何を根拠に覚え、いつ忘れ、どう検証できるかです。そこまで設計して初めて、AIエージェントのメモリは実用品になります。
