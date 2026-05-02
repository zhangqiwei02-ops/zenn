---
title: "EC2にS3権限を渡すとき、Access Key配布をやめてIAM Roleで考える"
emoji: "🔐"
type: "tech"
topics: ["AWS","iam","ec2","S3"]
published: true
---
## 結論

EC2 から S3 にアクセスさせるとき、長期 Access Key をインスタンスやアプリケーションに配る設計は避けます。標準形は **IAM Role を EC2 に関連付け、一時認証情報で S3 を呼ぶ** ことです。

ただし、IAM Role だけを見ていると設計を読み間違えます。実際には「EC2 が S3 に何をできるか」と「誰がその Role を EC2 に渡せるか」は別の論点です。後者を制御するのが **iam:PassRole** です。

この記事では、EC2 + S3 の権限設計を、IAM Role / Instance Profile / iam:PassRole に分けて整理します。

## Access Key 配布が危ない理由

EC2 上のアプリケーションに次のような値を置くと、実装は簡単です。

```env
AWS_ACCESS_KEY_ID=AKIAxxxxxxxxxxxx
AWS_SECRET_ACCESS_KEY=xxxxxxxxxxxxxxxx
```

しかし、これは運用上かなり扱いづらい設計です。Access Key は長期認証情報なので、漏れるとそのまま悪用されます。環境変数、ログ、AMI、バックアップ、CI/CD の設定、開発者の手元。置いた場所が増えるほど回収不能になります。

EC2 で AWS SDK を使うなら、Role ベースの一時認証情報を使う方が自然です。SDK は標準の認証プロバイダチェーンで認証情報を探すため、アプリケーションコードに鍵を埋め込む必要はありません。

## IAM Role と Instance Profile の関係

EC2 に S3 権限を渡す構成は、ざっくり次の部品で成り立ちます。

| 部品 | 見るべきポイント |
|---|---|
| IAM Policy | S3 のどの API を、どのリソースに許可するか |
| IAM Role | EC2 が引き受ける権限セット |
| Trust Policy | EC2 サービスがその Role を引き受けられるか |
| Instance Profile | Role を EC2 インスタンスに関連付けるための枠 |

S3 の読み取りだけでよいなら、最初から広い権限を付けない方がいいです。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::example-bucket/*"
    }
  ]
}
```

たとえばログアップロードも必要なら `s3:PutObject` を追加します。バケット一覧が必要なら `s3:ListBucket` も検討します。逆に、必要性を説明できない `AmazonS3FullAccess` は疑っていいです。

## iam:PassRole を別枠で考える

IAM Role に S3 権限を付けても、それを誰でも EC2 に付けられるわけではありません。EC2 を作成・変更する人や自動化ツールには、その Role を EC2 サービスへ���す権限が必要です。

それが `iam:PassRole` です。

この権限は、S3 を操作する権限ではありません。**権限を持った Role をサービスに渡す権限**です。だから雑に広げると危険です。

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::123456789012:role/ec2-s3-readonly-role",
  "Condition": {
    "StringEquals": {
      "iam:PassedToService": "ec2.amazonaws.com"
    }
  }
}
```

この例では、渡せる Role を 1 つに絞り、渡し先サービスも EC2 に限定しています。Role の権限が強いほど、この制御は重要になります。

## 責任を分けるとレビューしやすい

レビューでは、次の 2 つを分けて見ると判断しやすくなります。

| レビュー対象 | 質問 |
|---|---|
| EC2 Role のポリシー | この EC2 は S3 に対して何をしてよいのか |
| iam:PassRole の許可 | 誰が、どの Role を、どのサービスに渡してよいのか |

この分離ができていないと、「S3 は読み取りだけだから安全」と思っていたら、実は強い Role を自由に渡せる、という状態になりがちです。

## 例えばこう確認する

実務では、次のような観点で確認します。

- アプリケーション設定に Access Key が残っていないか
- EC2 Role の Resource が対象バケットや prefix に絞られているか
- `s3:*` や `Resource: *` を説明なしに使っていないか
- `iam:PassRole` の Resource が特定 Role に限定されているか
- `iam:PassedToService` で EC2 向けに制限しているか
- CloudTrail で RunInstances、AssociateIamInstanceProfile、PassRole 相当の操作を追えるか

特に自動化基盤に広い PassRole を渡す場合は注意が必要です。Terraform や CI/CD は便利ですが、便利なものに強い権限を持たせるほど、境界を明確にしておく必要があります。

## まとめ

EC2 に S3 権限を渡す設計では、Access Key を配らず IAM Role を使うのが基本です。そのうえで、Role の中身と iam:PassRole を分けてレビューします。

- EC2 のアプリケーションには長期 Access Key を置かない
- S3 権限は IAM Role に集約する
- Instance Profile 経由で EC2 に関連付ける
- iam:PassRole は対象 Role と渡し先サービスを絞る

IAM は「動くか」だけで見るとすぐ雑になります。誰が、何を、どこへ渡せるのか。そこまで分けて見ると、EC2 と S3 の連携はかなり安全に設計できます。