# AWS環境設定
# 前提条件
本PoC構成は、AWSアカウント毎、各リージョン毎に構成する
# SQS
- Dead Letter Queueは適宜設定すること

## ConfigStreamQueue
Queue Name

- ConfigStreamQueue

Queue Type

- Standard

### Add Permissions
Principals

- arn:aws:iam::<aws account id>:root

Actions

- SQS:SendMessage
 
Resource

- arn:aws:sqs:<region>:<aws account id>:ConfigStreamQueue

## ConfigStreamQueue.fifo
Queue Name

- ConfigStreamQueue.fifo

Queue Type

- FIFO

Delivery Delay

- 5min

Content-Based Deduplication

- Enable

## ConfigStream2slack
Queue Name

- ConfigStream2slack

# SNS
## Details
Name

- ConfigStreamTopic

## Subscriptions
Protocol

- Amazon SQS

Endpoint

- arn:aws:sqs:<region>:<aws account id>:ConfigStreamQueue

# s3
- 必要に応じて作成
- 便宜上バケット名は"config-stream-backup"とします

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
SQSやバケット名などを変更している場合は、適宜置き換えてること

|key|value|
|:---|:---|
|ConfigStream_Queue_Name|ConfigStreamQueue|
|ConfigStream_backup_backet|config-stream-backup|
|backet_region|s3バケットの配置リージョンを入力|
|dispatch_Queue_Name|ConfigStreamQueue.fifo|

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
SQSやバケット名などを変更している場合は、適宜置き換えてること

|key|value|
|:---|:---|
|Trust_arn|(記入例) arn:aws:sts::<aws account id>:assumed-role/AWSServiceRoleForAWSCloud9/aws-cloud9,arn:aws:sts::<aws account id>:assumed-role/AWSServiceRoleForAWSCloud8/aws-cloud8|
|slack_Queue_Name|ConfigStream2slack|
|exclude_resouce_type|AWS::SSM::ManagedInstanceInventory|
|dispatch_Queue_Name|ConfigStreamQueue.fifo|

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
|slack_Queue_Name|ConfigStream2slack|
|WebHookUrl|https://hooks.slack.com/services/xxxxx/yyyyy|
|slackChannel|#channel-name|
|SQS_URL|https://sqs.<region>.amazonaws.com/<aws account id>/ConfigStream2slack|

### Role Attached Policy
- [Limited-sqs-ConfigStream2Slack.json](/code/iam/Limited-sqs-ConfigStream2Slack.json)
- AWSLambdaBasicExecutionRoleは、Lambda作成時に作成されているものを利用する

### Basic settings
Timeout

- 30sec
