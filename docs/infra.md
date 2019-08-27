# AWS環境設定
# 前提条件
本PoC構成は、AWSアカウント毎、各リージョン毎に構成する
# Config
## Recording is on
- Configを有効にする
## Resource types to record
- [*] Record all resources supported in this region
- [*] Include global resources (適宜設定要否を判断すること)
## Amazon S3 bucket
- 適宜設定
- Config Snapshotの格納先

## Amazon SNS topic
- [*] Stream configuration changes and notifications to an Amazon SNS topic.
- トピックは適宜作成すること

## AWS Config role
- 自身で作成したConfigのサービスロール(Config - Customizable)に以下を適用する

### Manged IAM Policy
- AWSConfigRole
- AmazonSNSFullAccess

### Custom IAM Policy
- [Limited-ConfigStream2sqs.json](/code/iam/Limited-ConfigStream2sqs.json)

# SQS
- Dead Letter Queueは適宜設定すること

## Dispatch_SQS
Queue Name

- Dispatch_SQS

Queue Type

- Standard

### Add Permissions 1
Principals

- AWS_ACCOUNT_ID
- 実行環境のAWSアカウントIDを登録(12桁数字, ハイフンを除く)

Actions

- SQS:SendMessage

### Add Permissions 2
[チュートリアルSNSトピックへSQSをサブスクライブする](https://docs.aws.amazon.com/ja_jp/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-subscribe-queue-sns-topic.html) 参照

- SNSの設定後に作業すること

## Lookup_SQS.fifo
Queue Name

- Lookup_SQS.fifo

Queue Type

- FIFO

Delivery Delay

- 5min

Content-Based Deduplication

- Enable

## Slack_SQS
Queue Name

- Slack_SQS

# SNS
- Config設定時に作成したものを利用する

## Details
Name

- config-topic

## Subscriptions
Protocol

- Amazon SQS

Endpoint

- arn:aws:sqs:REGION:AWS_ACCOUNT_ID:Dispatch_SQS

# s3
- 必要に応じて作成
- 本ドキュメントでは、便宜上バケット名を"config-stream-backup"とします

# Config
## Recording is on
- Configを有効にする
## Resource types to record
- [*] Record all resources supported in this region
- [*] Include global resources (適宜設定要否を判断すること)
## Amazon S3 bucket
- 適宜設定
## Amazon SNS topic
- [*] Stream configuration changes and notifications to an Amazon SNS topic.
- トピックは適宜作成すること
## AWS Config role
### Manged IAM Policy
- AWSConfigRole
- AmazonSNSFullAccess
### Custom IAM Policy
- [Limited-ConfigStream2sqs.json](/code/iam/Limited-ConfigStream2sqs.json)

# Lambda
## ConfigStreamDispatcher
Runtime

- python3.6

Code

- [ConfigStreamDispatcher.lambda_function](/code/lambda/ConfigStreamDispatcher.lambda_function)

### Environment variables
- SQSやバケット名などを変更している場合は、適宜置き換えてること

|key|value|
|:---|:---|
|ConfigStream_Queue_Name|Dispatch_SQS|
|ConfigStream_backup_backet|config-stream-backup (適宜変更すること)|
|backet_region|s3バケットの配置リージョンを入力|
|dispatch_Queue_Name|Lookup_SQS.fifo|

### Role Attached Policy
- [Limited-s3-put-configstream-backup.json](/code/iam/Limited-s3-put-configstream-backup.json)
- [Limited-sqs-ConfigStreamDispacher.json](/code/iam/Limited-sqs-ConfigStreamDispacher.json)
- AWSLambdaBasicExecutionRoleは、Lambda作成時に作成されているものを利用する

### Basic settings
Timeout

- 5min

## ConfigStreamLookupper
Runtime

- python3.6

Code

- [ConfigStreamLookupper.lambda_function](/code/lambda/ConfigStreamLookupper.lambda_function)

### Environment variables
- SQSを変更している場合は、適宜置き換えてること

|key|value|
|:---|:---|
|Trust_arn|(記入例) arn:aws:sts::AWS_ACCOUNT_ID:assumed-role/AWSServiceRoleForAWSCloud9/aws-cloud9,arn:aws:sts::AWS_ACCOUNT_ID:assumed-role/AWSServiceRoleForAWSCloud8/aws-cloud8|
|slack_Queue_Name|Slack_SQS|
|exclude_resouce_type|AWS::SSM::ManagedInstanceInventory|
|dispatch_Queue_Name|Lookup_SQS.fifo|

### Role Attached Policy
- [Limited-CloudTrail-lookupevents.json](/code/iam/Limited-CloudTrail-lookupevents.json)
- [Limited-sqs-ConfigStreamLookupper.json](/code/iam/Limited-sqs-ConfigStreamLookupper.json)
- AWSLambdaBasicExecutionRoleは、Lambda作成時に作成されているものを利用する

### Basic settings
Timeout

- 5min

## ConfigStream2slack
Runtime

- python3.6

Code

- [ConfigStream2slack.lambda_function](/code/lambda/ConfigStream2slack.lambda_function)

### Environment variables
- SQS名を変更している場合は、適宜置き換えてること
- Incoming Webhookの設定方法は、[こちら](https://api.slack.com/incoming-webhooks)を参照

|key|value|
|:---|:---|
|CloudTrailRegion|参照するCloudTrailのリージョンを指定|
|slack_Queue_Name|Slack_SQS|
|WebHookUrl|https://hooks.slack.com/services/xxxxx/yyyyy|
|slackChannel|#channel-name|

### Role Attached Policy
- [Limited-sqs-ConfigStream2Slack.json](/code/iam/Limited-sqs-ConfigStream2Slack.json)
- AWSLambdaBasicExecutionRoleは、Lambda作成時に作成されているものを利用する

### Basic settings
Timeout

- 30sec

# Cloudwatch Events
## Event Source
- [*] Schedule
- Fixed rate of "1 Minutes"

## Targets
Lambda function

- ConfigStreamDispatcher
- ConfigStreamLookupper
- ConfigStream2slack
