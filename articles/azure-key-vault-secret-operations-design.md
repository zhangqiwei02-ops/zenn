---
title: "Azure Key Vault を「入れれば安全」で終わらせないための設計整理"
emoji: "🔐"
type: "tech"
topics: ["Azure","Security","KeyVault","ManagedIdentity"]
published: true
---
# Azure Key Vault は「保存先」ではなく運用設計で決まる

Azure Key Vault を使えば、アプリケーションのコードや設定ファイルからパスワード、API キー、証明書を追い出せます。これは間違いなく大きな改善です。

ただし、ここで話を止めると危ないです。Key Vault は「秘密情報を入れる箱」ではありますが、**入れた瞬間に安全になる装置ではありません**。誰が読めるのか、どのアプリがどう取得するのか、更新時に止まらないのか、監査で追えるのか。そこまで決めて初めて、実務で使えるセキュリティ設計になります。

この記事では、Azure Key Vault を Secret 管理の置き場としてではなく、Managed Identity、RBAC、ローテーション、監査ログまで含めた運用設計として整理します。

## 先に結論

先に結論です。Azure Key Vault の価値は、秘密情報を「隠す」ことだけではありません。

本当に見るべきポイントは次の 5 つです。

- アプリケーションに長期シークレットを持たせない
- Managed Identity で取得主体を明確にする
- RBAC または Access Policy で読み取り権限を最小化する
- ローテーション時にアプリが止まらない設計にする
- 監査ログで「誰が、いつ、何にアクセスしたか」を追えるようにする

つまり Key Vault は、単なる保管先ではなく、**秘密情報のライフサイクルを管理するための境界**です。

## よくある誤解：Key Vault に入れたら終わり

一番ありがちな誤解は、接続文字列や API キーを Key Vault に入れた時点で安心してしまうことです。

たとえば、次のような状態ならまだ危険です。

- アプリが Key Vault にアクセスするための Client Secret を設定ファイルに持っている
- すべてのアプリに同じ Key Vault 読み取り権限を渡している
- 本番、検証、開発で同じ Vault や同じ Secret 名を使い回している
- ローテーション手順がなく、Secret が何年も固定されている
- 監査ログを有効にしておらず、漏えい時に追跡できない

これでは「コードから秘密情報を消した」だけです。攻撃面を下げたように見えて、別の場所にリスクを移動しただけ、という雑な状態になりがちです。

## Managed Identity を前提に考える

Azure 上のアプリケーションから Key Vault を読むなら、まず Managed Identity を前提にしたいです。App Service、Azure Functions、Container Apps、Virtual Machine などは Managed Identity を使って Microsoft Entra ID 上の ID を持てます。

この形にすると、アプリケーション側に長期の Client Secret を置かずに Key Vault へアクセスできます。

典型的な流れはこうです。

1. App Service や Azure Functions に System-assigned Managed Identity を有効化する
2. Key Vault 側でその Identity に Secret 読み取り権限を付与する
3. アプリケーションは DefaultAzureCredential などでトークンを取得する
4. Key Vault から必要な Secret を読む

Node.js なら、概念的には次のような形です。

```js
import { DefaultAzureCredential } from "@azure/identity";
import { SecretClient } from "@azure/keyvault-secrets";

const credential = new DefaultAzureCredential();
const client = new SecretClient(
  "https://example-vault.vault.azure.net/",
  credential
);

const secret = await client.getSecret("database-password");
```

ポイントは、ここにアプリ用の固定パスワードや Client Secret が出てこないことです。もちろんローカル開発では Azure CLI 認証などを使うため、本番と開発で認証経路が違う点は整理しておく必要があります。

## 権限設計は「Vault を読める」では粗すぎる

Key Vault の権限設計で雑になりやすいのが、「このアプリは Vault を読める」で終わることです。

本番で見るべき粒度はもう少し細かいです。

| 観点 | 悪い例 | ましな例 |
|---|---|---|
| 権限範囲 | 全アプリに全 Secret 読み取り | アプリごとに必要な Secret だけ読ませる |
| 環境分離 | dev/stg/prod が同じ Vault | 環境ごとに Vault または権限境界を分ける |
| 管理者権限 | 開発者全員が変更可能 | 運用担当とアプリ実行 ID を分離する |
| 監査 | ログ未設定 | Diagnostic settings で Log Analytics へ送る |

Key Vault では Azure RBAC と Vault Access Policy のどちらを使うかも設計論点です。新規構成では Azure RBAC に寄せるケースが増えますが、既存環境や運用ルールによっては Access Policy が残っていることもあります。大事なのは方式名ではなく、**誰に何を許可しているかを説明できること**です。

## Secret と Certificate と Key を混ぜない

Key Vault には Secret、Certificate、Key があります。名前が似ているので全部「秘密情報」として雑に扱われがちですが、用途は違います。

- Secret: DB パスワード、API キー、接続文字列など
- Certificate: TLS 証明書やクライアント証明書
- Key: 暗号化・署名に使う鍵。秘密鍵そのものを外へ出さない設計に向く

たとえば、ただの接続文字列なら Secret でよいことが多いです。一方で署名や暗号化に使う鍵をアプリ側へコピーしてしまうと、Key Vault を使う意味が薄れます。Key Vault Key を使うなら、アプリは鍵そのものではなく、Key Vault の暗号操作を呼び出す形を検討します。

## ローテーションを後回しにしない

Secret 管理で一番面倒なのは登録ではなく更新です。

Key Vault に Secret を置いても、ローテーションできなければ古いパスワードが長く残ります。特にデータベース接続文字列や外部 API キーは、更新時にアプリが止まる設計だと誰も回したがりません。

最低限、次の点は決めておきたいです。

- Secret の有効期限を設定するか
- 更新通知をどこで受けるか
- アプリは Secret を起動時だけ読むのか、一定間隔で再取得するのか
- 旧 Secret と新 Secret を併存できる期間を作るか
- ロールバック時にどの Secret version を使うか

Key Vault は Secret の version を持てます。これは便利ですが、アプリが常に最新 version を読むのか、明示的な version を固定するのかで運用が変わります。無計画に最新だけ読むと、更新直後に想定外の影響が出ることがあります。

## 監査ログを取らない Key Vault は怖い

Key Vault はアクセス制御だけでなく、監査も重要です。漏えい疑いが出たときに、誰が Secret を読んだのか分からない構成はかなり厳しいです。

少なくとも次のイベントは追えるようにしておきたいです。

- Secret の読み取り
- Secret の作成、更新、削除
- 権限変更
- 異常に多いアクセス
- 普段と違う主体からのアクセス

Azure では Diagnostic settings を使って Key Vault のログを Log Analytics Workspace、Event Hub、Storage Account などへ送れます。Microsoft Defender for Cloud や Sentinel と組み合わせるなら、検知と調査の線も引きやすくなります。

## Private Endpoint は万能ではない

セキュリティ要件が強い環境では、Key Vault に Private Endpoint を設定し、Public network access を制限することがあります。これは有効な設計です。

ただし、Private Endpoint を入れると DNS、VNet、接続元の経路が一気に設計対象になります。Azure Functions や App Service から閉域で Key Vault に到達させる場合、VNet Integration、Private DNS Zone、Outbound 経路を正しくそろえる必要があります。

ここを甘く見ると、��プリは動いているのに Secret 取得だけ失敗する、という地味に面倒な障害になります。閉域化は強い選択ですが、運用コストも一緒に持つ判断です。

## 実務でのチェックリスト

Key Vault を本番に入れる前に、最低限このあたりを確認すると事故が減ります。

- アプリケーションに長期 Client Secret を置いていないか
- Managed Identity を使えるサービスでは使っているか
- Secret の読み取り権限がアプリ単位で分かれているか
- 本番と非本番の Vault または権限境界が分離されているか
- Secret の有効期限とローテーション手順があるか
- ローテーション時にアプリがどう再読み込みするか決まっているか
- Diagnostic settings で監査ログを保存しているか
- Private Endpoint を使う場合、DNS と VNet 経路を検証しているか
- Key、Secret、Certificate の用途を混ぜていないか

## まとめ

Azure Key Vault は便利ですが、「保存したから安全」ではありません。

重要なのは、Secret をどこへ置くかではなく、

- 誰が読めるのか
- どう認証するのか
- どう更新するのか
- どう監査するのか
- ネットワーク境界をどこに置くのか

を設計することです。

Key Vault は、コードからパスワードを消すための箱ではなく、秘密情報のライフサイクルを管理するためのサービスです。Managed Identity、RBAC、ローテーション、Diagnostic settings、Private Endpoint まで含めて考えると、ようやく「使っている」ではなく「守れている」に近づきます。