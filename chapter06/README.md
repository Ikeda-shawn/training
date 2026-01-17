# 第6章 スタックのライフサイクル

## スタック作成・更新・削除の流れ

CloudFormation スタックには、基本的なライフサイクルがあります。

```
作成 → 更新 → 更新 → ... → 削除
```

各段階で CloudFormation がどのように動作するかを理解することが重要です。

## スタック作成（CREATE）

### AWS CLI での作成

```bash
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=EnvironmentName,ParameterValue=dev
```

### 作成時の流れ

1. **テンプレートの検証**
   - YAML/JSON の構文チェック
   - リソース定義の妥当性チェック

2. **リソースの作成開始**
   - 依存関係を解析し、適切な順序でリソースを作成
   - 例：VPC → Subnet → EC2 の順に作成

3. **ステータスの遷移**
   - `CREATE_IN_PROGRESS`：作成中
   - `CREATE_COMPLETE`：作成完了
   - `CREATE_FAILED`：作成失敗（ロールバック開始）
   - `ROLLBACK_COMPLETE`：ロールバック完了

### 作成失敗時の挙動

スタック作成中にエラーが発生した場合、**自動的にロールバック**されます。

```
VPC 作成成功 → Subnet 作成成功 → EC2 作成失敗
  ↓
ロールバック開始
  ↓
EC2 削除 → Subnet 削除 → VPC 削除
```

ロールバック後、スタックは `ROLLBACK_COMPLETE` 状態になり、削除する必要があります。

## スタック更新（UPDATE）

### AWS CLI での更新

```bash
aws cloudformation update-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=EnvironmentName,ParameterValue=stg
```

### 更新時の挙動

リソースの変更内容によって、更新方法が異なります。

| 更新方法 | 説明 | 例 |
|---|---|---|
| **No interruption** | ダウンタイムなしで更新 | EC2 の Tags 変更 |
| **Some interruption** | 一時的な中断を伴う更新 | EC2 のインスタンスタイプ変更 |
| **Replacement** | リソースを置き換える | EC2 の AMI 変更 |

### Update requires の確認方法

AWS 公式ドキュメントの各プロパティに記載されています。

例：`AWS::EC2::Instance` の `InstanceType` プロパティ
- **Update requires**：Some interruption

## ChangeSet（変更セット）

ChangeSet は、スタックを更新する前に**変更内容をプレビュー**する機能です。

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

出力例：

```json
{
  "Changes": [
    {
      "Type": "Resource",
      "ResourceChange": {
        "Action": "Modify",
        "LogicalResourceId": "MyEC2",
        "PhysicalResourceId": "i-0123456789abcdef0",
        "ResourceType": "AWS::EC2::Instance",
        "Replacement": "True",
        "Details": [...]
      }
    }
  ]
}
```

### ChangeSet の実行

```bash
aws cloudformation execute-change-set \
  --stack-name my-stack \
  --change-set-name my-changeset
```

### ChangeSet を使うメリット

1. **変更内容を事前確認できる**
   - どのリソースが追加・変更・削除されるか
   - リソースが置き換えられるか（Replacement）

2. **意図しない変更を防げる**
   - 実行前に確認できるため、誤操作を防止

3. **本番環境での安全性向上**
   - 特に本番環境では、ChangeSet の使用を推奨

## 更新失敗時に何が起きるか

### 更新失敗時の挙動

スタック更新中にエラーが発生した場合、**以前の状態にロールバック**されます。

```
更新前の状態（安定）
  ↓
更新開始（UPDATE_IN_PROGRESS）
  ↓
リソース変更中にエラー発生
  ↓
ロールバック開始（UPDATE_ROLLBACK_IN_PROGRESS）
  ↓
更新前の状態に戻る（UPDATE_ROLLBACK_COMPLETE）
```

### ロールバック後の対処

1. **エラー原因を確認**
   ```bash
   aws cloudformation describe-stack-events --stack-name my-stack
   ```

2. **テンプレートまたはパラメータを修正**

3. **再度更新を実行**

## スタック削除（DELETE）

### AWS CLI での削除

```bash
aws cloudformation delete-stack --stack-name my-stack
```

### 削除時の流れ

1. **リソースの削除開始**
   - 依存関係の逆順で削除
   - 例：EC2 → Subnet → VPC の順に削除

2. **ステータスの遷移**
   - `DELETE_IN_PROGRESS`：削除中
   - `DELETE_COMPLETE`：削除完了（スタック自体も消える）
   - `DELETE_FAILED`：削除失敗

### 削除失敗の原因

#### 1. 他のリソースから参照されている

```
Error: Export my-stack-VPCId cannot be deleted as it is in use by another-stack
```

**対処法**：参照している側のスタックを先に削除する

#### 2. S3 バケットが空でない

CloudFormation は、**空でない S3 バケットを削除できません**。

**対処法**：
```bash
# S3 バケットを空にする
aws s3 rm s3://my-bucket --recursive

# スタックを削除
aws cloudformation delete-stack --stack-name my-stack
```

#### 3. リソースが手動で削除されている

CloudFormation が管理しているリソースを手動で削除すると、スタック削除時にエラーになります。

**対処法**：スキップして削除を続行
```bash
aws cloudformation delete-stack \
  --stack-name my-stack \
  --retain-resources MyEC2
```

## 削除できないリソースがある場合の考え方

### DeletionPolicy

リソースに `DeletionPolicy` を設定することで、スタック削除時の挙動を制御できます。

#### Delete（デフォルト）

スタック削除時にリソースも削除されます。

```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    DeletionPolicy: Delete  # デフォルト（明示不要）
    Properties:
      CidrBlock: 10.0.0.0/16
```

#### Retain

スタック削除時にリソースを残します。

```yaml
Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain  # スタック削除後も残る
    Properties:
      BucketName: my-important-bucket
```

#### Snapshot（一部のリソースのみ）

スタック削除時にスナップショットを作成してから削除します。

```yaml
Resources:
  MyRDS:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot  # 削除前にスナップショット作成
    Properties:
      DBInstanceIdentifier: my-db
      Engine: mysql
```

### DeletionPolicy を使うケース

- **本番データベース**：Snapshot を使い、誤削除時にデータを復旧できるようにする
- **ログ用 S3 バケット**：Retain を使い、スタック削除後もログを保持する

## スタックのステータス一覧

| ステータス | 説明 |
|---|---|
| CREATE_IN_PROGRESS | スタック作成中 |
| CREATE_COMPLETE | スタック作成完了 |
| CREATE_FAILED | スタック作成失敗 |
| ROLLBACK_IN_PROGRESS | ロールバック中 |
| ROLLBACK_COMPLETE | ロールバック完了 |
| UPDATE_IN_PROGRESS | スタック更新中 |
| UPDATE_COMPLETE | スタック更新完了 |
| UPDATE_ROLLBACK_IN_PROGRESS | 更新のロールバック中 |
| UPDATE_ROLLBACK_COMPLETE | 更新のロールバック完了 |
| DELETE_IN_PROGRESS | スタック削除中 |
| DELETE_COMPLETE | スタック削除完了 |
| DELETE_FAILED | スタック削除失敗 |

## 次の章へ

スタックのライフサイクルが理解できたら、[第7章 CloudFormation と運用上の注意点](../chapter07/README.md) に進んでください。

運用時に注意すべき「ドリフト」などの問題について学びます。
