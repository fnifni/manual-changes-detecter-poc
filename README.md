# manual-changes-detecter-poc
# 概要
- AWS環境の設定変更を通知するConfig通知(Config Stream)から、CloudTrail Eventを引き当てることで、変更を行ったユーザーを特定する概念実証コードです
- 引き当てた情報をSlackに通知する機能を有しています
- 信頼するarnを登録することで、特定arnが行った変更について、通知を除外する機能を有しています

## Slack通知概要
![slack-nortify2.png](/docs/pixs/slack-nortify2.png)

タイトル
- s3にバックアップしたConfig Stream本文へのハイパーリンク
- アクセスするためには、事前にAWS環境へのログインが必要です

Action
- 引き当てたCloudTrail Eventに記載されているAPI名

ResourceName
- 引き当てたCloudTrail Eventに記載されているResource名

arn
- 引き当てたCloudTrail Eventに記載されているUser Identifyに含まれるarn

EventID
- 引き当てたCloudTrail Eventへのハイパーリンク
- リンク先から、Config Timelineへのアクセスが可能
- アクセスするためには、事前にAWS環境へのログインが必要です

## 利用上の注意
- 本番ワークロードで利用する際は、状況に応じた改修やチューニングを必要とします
- 単一AWSアカウント、単一リージョンでの利用を前提としています

# ユースケース
- 安定稼働している環境に対しての、人為的な変更作業を検出できるようにしたい
- CloudFormationやTerraformなどのツールで利用しているIAM{ユーザー|ロール}以外での構成変更を検出できるようにしたい

# 構成するには
- 構成パラメータは、[こちら](/docs/infra.md)を参照してください

# ざっくり構成
![configuration-diagram.png](/docs/pixs/configuration-diagram.png)
## 処理の概要
1. Config Streamが変更情報をSNSに通知
2. SNSがトピックをSQS(Dispache_SQS)にキューイング
3. Cloudwatch Events(Time-Based)が一分間隔でLambda(Dispacher, Lookupper, SendSlack)を呼び出し
4. Lambda(Dispacher)が、SQS(Dispache_SQS)からメッセージを読み出し
5. Lambda(Dispacher)が、読みだしたメッセージをs3に保存
6. Lambda(Dispacher)が、処理対象の情報が含まれているメッセージを、SQS(Lookup_SQS)にキューイング
7. Lambda(Dispacher)が、読み込んだメッセージを削除
8. Lambda(Lookupper)が、SQS(Lookup_SQS)からメッセージを読み出して、タイムスタンプとリソースタイプをキーにCloudTrailをLookupする
9. Lookupの結果が取得できた場合、Lambda(Lookupper)が、処理対象の情報を選別し、通知に用いる情報を抽出/整形して、SQS(SendSlack_SQS)にキューイング

9'. Lookupの結果が空の場合、Lambda(Lookupper)が、SQS(Lookup_SQS_DL)にメッセージをキューイング
10. Lambda(Lookupper)が、読み込んだメッセージを削除
11. Lambda(SendSlack)が、SQS(SendSlack_SQS)からメッセージを読み出し
12. Lambda(SendSlack)が、SlackのIncomming Webhookに投稿
13. Slackのレスポンスが正常なら、Lambda(SendSlack)が、読み込んだメッセージを削除

### Lookup_SQS_DLの処理
14. Lambda(Lookupper)が、SQS(Lookup_SQS_DL)からメッセージを読み出して、タイムスタンプとリソースタイプをキーにCloudTrailをLookupする。
15. Lookupの結果が取得できた場合、Lambda(Lookupper)が、処理対象の情報を選別し、通知に用いる情報を抽出/整形して、SQS(SendSlack_SQS)にキューイング

15'. Lookupの結果が空の場合、Lambda(Lookupper)が、ログに警告を書き出してメッセージを削除する
16. Lambda(Lookupper)が、読み込んだメッセージを削除

