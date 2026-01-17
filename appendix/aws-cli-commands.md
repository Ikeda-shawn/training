# 付録A：AWS CLI コマンド集

CloudFormationをAWS CLIで操作する際によく使うコマンドをまとめました。

## スタックの作成

### 基本的な作成

```bash
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml
```

### パラメータを指定して作成

```bash
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --parameters \
    ParameterKey=EnvironmentName,ParameterValue=dev \
    ParameterKey=VPCCidrBlock,ParameterValue=10.0.0.0/16
```

### パラメータファイルを使って作成

```bash
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --parameters file://parameters.json
```

parameters.json の例：

```json
[
  {
    "ParameterKey": "EnvironmentName",
    "ParameterValue": "dev"
  },
  {
    "ParameterKey": "VPCCidrBlock",
    "ParameterValue": "10.0.0.0/16"
  }
]
```

### IAM リソースを含むスタックの作成

```bash
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_IAM
```

## スタックの更新

### 基本的な更新

```bash
aws cloudformation update-stack \
  --stack-name my-stack \
  --template-body file://template.yaml
```

### パラメータを指定して更新

```bash
aws cloudformation update-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --parameters \
    ParameterKey=EnvironmentName,ParameterValue=stg
```

### 前回と同じパラメータ値を使う

```bash
aws cloudformation update-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --parameters \
    ParameterKey=EnvironmentName,UsePreviousValue=true
```

## ChangeSet の操作

### ChangeSet の作成

```bash
aws cloudformation create-change-set \
  --stack-name my-stack \
  --change-set-name my-changeset \
  --template-body file://template.yaml
```

### ChangeSet の確認

```bash
aws cloudformation describe-change-set \
  --stack-name my-stack \
  --change-set-name my-changeset
```

### ChangeSet の実行

```bash
aws cloudformation execute-change-set \
  --stack-name my-stack \
  --change-set-name my-changeset
```

### ChangeSet の削除

```bash
aws cloudformation delete-change-set \
  --stack-name my-stack \
  --change-set-name my-changeset
```

## スタックの削除

### 基本的な削除

```bash
aws cloudformation delete-stack --stack-name my-stack
```

### 特定のリソースを残して削除

```bash
aws cloudformation delete-stack \
  --stack-name my-stack \
  --retain-resources MyVPC
```

## スタックの確認

### スタック一覧の表示

```bash
aws cloudformation list-stacks
```

### 特定のスタックの詳細表示

```bash
aws cloudformation describe-stacks --stack-name my-stack
```

### スタックのリソース一覧

```bash
aws cloudformation list-stack-resources --stack-name my-stack
```

### スタックのイベント確認

```bash
aws cloudformation describe-stack-events --stack-name my-stack
```

最新10件のイベントのみ表示：

```bash
aws cloudformation describe-stack-events \
  --stack-name my-stack \
  --max-items 10
```

### スタックの Outputs 確認

```bash
aws cloudformation describe-stacks \
  --stack-name my-stack \
  --query 'Stacks[0].Outputs'
```

## テンプレートの検証

### テンプレートの構文チェック

```bash
aws cloudformation validate-template --template-body file://template.yaml
```

## ドリフトの検出

### ドリフト検出の実行

```bash
aws cloudformation detect-stack-drift --stack-name my-stack
```

### ドリフト検出結果の確認

```bash
# ドリフト検出 ID を取得
DRIFT_ID=$(aws cloudformation detect-stack-drift \
  --stack-name my-stack \
  --query 'StackDriftDetectionId' \
  --output text)

# 検出結果の確認
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id $DRIFT_ID

# リソースごとのドリフト詳細
aws cloudformation describe-stack-resource-drifts \
  --stack-name my-stack
```

## Export された値の確認

### Export 一覧の表示

```bash
aws cloudformation list-exports
```

### 特定の Export 値の確認

```bash
aws cloudformation list-exports \
  --query "Exports[?Name=='dev-network-stack-VPCId'].Value" \
  --output text
```

## スタックのステータス確認

### スタック作成/更新の完了を待つ

```bash
# スタック作成完了を待つ
aws cloudformation wait stack-create-complete --stack-name my-stack

# スタック更新完了を待つ
aws cloudformation wait stack-update-complete --stack-name my-stack

# スタック削除完了を待つ
aws cloudformation wait stack-delete-complete --stack-name my-stack
```

## 便利なオプション

### JSON 形式で出力

```bash
aws cloudformation describe-stacks \
  --stack-name my-stack \
  --output json
```

### テーブル形式で出力

```bash
aws cloudformation describe-stacks \
  --stack-name my-stack \
  --output table
```

### 特定のフィールドのみ取得（JMESPath クエリ）

```bash
# スタックのステータスのみ取得
aws cloudformation describe-stacks \
  --stack-name my-stack \
  --query 'Stacks[0].StackStatus' \
  --output text

# Outputs の値のみ取得
aws cloudformation describe-stacks \
  --stack-name my-stack \
  --query 'Stacks[0].Outputs[*].[OutputKey,OutputValue]' \
  --output table
```

## エラー時のトラブルシューティング

### スタック作成失敗時のイベント確認

```bash
aws cloudformation describe-stack-events \
  --stack-name my-stack \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'
```

### 最新のエラーメッセージのみ表示

```bash
aws cloudformation describe-stack-events \
  --stack-name my-stack \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]|[0].ResourceStatusReason' \
  --output text
```

## リージョンの指定

### 特定のリージョンで実行

```bash
aws cloudformation create-stack \
  --region ap-northeast-1 \
  --stack-name my-stack \
  --template-body file://template.yaml
```

### 環境変数で指定

```bash
export AWS_DEFAULT_REGION=ap-northeast-1

aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml
```

## プロファイルの使用

### 特定の AWS プロファイルを使用

```bash
aws cloudformation create-stack \
  --profile my-profile \
  --stack-name my-stack \
  --template-body file://template.yaml
```

---

**参考**：AWS CLI の詳細なドキュメントは [AWS CLI Command Reference - CloudFormation](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/) を参照してください。
