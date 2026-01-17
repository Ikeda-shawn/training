# 付録C：エラーメッセージの読み方

CloudFormation で発生する代表的なエラーとその対処法をまとめました。

## スタック作成時のエラー

### 1. テンプレートの構文エラー

#### エラーメッセージ

```
An error occurred (ValidationError) when calling the CreateStack operation:
Template format error: YAML not well-formed
```

#### 原因

- YAML の構文が間違っている
- インデントが不正
- 全角スペースが混入している

#### 対処法

1. **インデントを確認**
   - スペース2つまたは4つで統一されているか
   - タブ文字が混入していないか

2. **YAML のバリデーションツールを使う**
   ```bash
   aws cloudformation validate-template --template-body file://template.yaml
   ```

3. **エディタの YAML プラグインを使う**
   - VS Code、Sublime Text などの YAML プラグインを導入

---

### 2. リソースの作成に失敗

#### エラーメッセージ

```
Resource creation failed: The subnet CIDR block 10.0.1.0/24 conflicts
with another subnet in VPC vpc-xxx
```

#### 原因

- CIDR ブロックが重複している
- 既存のリソースと競合している

#### 対処法

1. **エラーメッセージを読む**
   - 何が原因で失敗したかが記載されている

2. **スタックイベントを確認**
   ```bash
   aws cloudformation describe-stack-events --stack-name my-stack
   ```

3. **該当するリソースの設定を修正**

---

### 3. パラメータの値が不正

#### エラーメッセージ

```
Parameter validation failed:
parameter value 'production' for parameter name 'EnvironmentName' does not match
pattern AllowedValues: [dev, stg, prod]
```

#### 原因

- パラメータの値が AllowedValues に含まれていない
- AllowedPattern の正規表現にマッチしていない

#### 対処法

1. **パラメータの定義を確認**
   ```yaml
   Parameters:
     EnvironmentName:
       Type: String
       AllowedValues:
         - dev
         - stg
         - prod  # 'production' ではなく 'prod'
   ```

2. **正しい値を指定してスタックを作成**

---

### 4. 必須プロパティが不足

#### エラーメッセージ

```
Template error: instance of AWS::EC2::VPC is missing required
property 'CidrBlock'
```

#### 原因

- 必須のプロパティが定義されていない

#### 対処法

1. **AWS 公式ドキュメントで必須プロパティを確認**
   - https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpc.html

2. **必須プロパティを追加**
   ```yaml
   Resources:
     MyVPC:
       Type: AWS::EC2::VPC
       Properties:
         CidrBlock: 10.0.0.0/16  # 必須プロパティ
   ```

---

## スタック更新時のエラー

### 5. Export が他のスタックで使用されている

#### エラーメッセージ

```
Export my-stack-VPCId cannot be deleted as it is in use by another-stack
```

#### 原因

- Export している値を変更・削除しようとしたが、他のスタックが ImportValue で参照している

#### 対処法

1. **参照している側のスタックを先に削除または更新**
   ```bash
   # 参照している側のスタックを削除
   aws cloudformation delete-stack --stack-name another-stack

   # その後、Export している側のスタックを更新
   aws cloudformation update-stack --stack-name my-stack --template-body file://template.yaml
   ```

2. **または、Export 名を変更せずに値だけを変更**

---

### 6. リソースが置き換えられる

#### エラーメッセージ

```
Requires replacement. Stack update cancelled.
```

#### 原因

- 更新しようとしたプロパティが `Replacement` を必要とする
- リソースが削除→再作成される

#### 対処法

1. **ChangeSet で事前に確認**
   ```bash
   aws cloudformation create-change-set \
     --stack-name my-stack \
     --change-set-name my-changeset \
     --template-body file://template.yaml

   aws cloudformation describe-change-set \
     --stack-name my-stack \
     --change-set-name my-changeset
   ```

2. **Replacement が必要な場合は、新しいリソースを作成してから古いリソースを削除**

---

### 7. 更新がロールバックされる

#### エラーメッセージ

```
UPDATE_ROLLBACK_COMPLETE
```

#### 原因

- スタック更新中にエラーが発生し、以前の状態に戻された

#### 対処法

1. **スタックイベントでエラー原因を確認**
   ```bash
   aws cloudformation describe-stack-events \
     --stack-name my-stack \
     --query 'StackEvents[?ResourceStatus==`UPDATE_FAILED`]'
   ```

2. **エラー原因を修正してから再度更新**

---

## スタック削除時のエラー

### 8. S3 バケットが空でない

#### エラーメッセージ

```
The bucket you tried to delete is not empty
```

#### 原因

- S3 バケットに中身が残っている
- CloudFormation は空でない S3 バケットを削除できない

#### 対処法

1. **S3 バケットを空にする**
   ```bash
   aws s3 rm s3://my-bucket --recursive
   ```

2. **スタックを再度削除**
   ```bash
   aws cloudformation delete-stack --stack-name my-stack
   ```

---

### 9. リソースが手動で削除されている

#### エラーメッセージ

```
Resource VPC vpc-xxx does not exist
```

#### 原因

- CloudFormation が管理しているリソースが手動で削除された

#### 対処法

1. **該当するリソースをスキップして削除を続行**
   ```bash
   aws cloudformation delete-stack \
     --stack-name my-stack \
     --retain-resources MyVPC
   ```

---

## その他のエラー

### 10. IAM 権限エラー

#### エラーメッセージ

```
User is not authorized to perform: cloudformation:CreateStack
```

#### 原因

- IAM ユーザーまたはロールに必要な権限がない

#### 対処法

1. **必要な IAM 権限を付与**
   - `cloudformation:CreateStack`
   - `cloudformation:UpdateStack`
   - `cloudformation:DeleteStack`
   - 作成するリソースに応じた権限（例：`ec2:CreateVpc`）

---

### 11. CAPABILITY_IAM が必要

#### エラーメッセージ

```
Requires capabilities : [CAPABILITY_IAM]
```

#### 原因

- テンプレートに IAM リソース（Role、Policy など）が含まれている
- IAM リソースを作成する場合は、明示的に許可が必要

#### 対処法

1. **`--capabilities` オプションを追加**
   ```bash
   aws cloudformation create-stack \
     --stack-name my-stack \
     --template-body file://template.yaml \
     --capabilities CAPABILITY_IAM
   ```

---

### 12. リソース数の上限

#### エラーメッセージ

```
Template may not exceed maximum size 51200 bytes
```

#### 原因

- テンプレートのサイズが上限（51,200 バイト）を超えている

#### 対処法

1. **テンプレートを分割する**
   - スタックを複数に分ける
   - ネストスタックを使う

2. **S3 にテンプレートをアップロードして使う**
   ```bash
   aws s3 cp template.yaml s3://my-bucket/template.yaml

   aws cloudformation create-stack \
     --stack-name my-stack \
     --template-url https://my-bucket.s3.amazonaws.com/template.yaml
   ```

---

## エラーの調査方法

### 1. スタックイベントを確認

```bash
aws cloudformation describe-stack-events --stack-name my-stack
```

### 2. 失敗したリソースのみ表示

```bash
aws cloudformation describe-stack-events \
  --stack-name my-stack \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'
```

### 3. 最新のエラーメッセージのみ表示

```bash
aws cloudformation describe-stack-events \
  --stack-name my-stack \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]|[0].ResourceStatusReason' \
  --output text
```

### 4. AWS コンソールで確認

AWS コンソールの CloudFormation ページで、スタックの「イベント」タブを確認すると、視覚的にわかりやすいです。

---

## エラーを防ぐためのベストプラクティス

1. **テンプレートを検証してから作成**
   ```bash
   aws cloudformation validate-template --template-body file://template.yaml
   ```

2. **ChangeSet で事前に確認**
   ```bash
   aws cloudformation create-change-set \
     --stack-name my-stack \
     --change-set-name my-changeset \
     --template-body file://template.yaml
   ```

3. **小さく始めて徐々に拡張**
   - 最初はシンプルなテンプレートから始める
   - 動作確認しながら少しずつリソースを追加

4. **テンプレートをバージョン管理**
   - Git などでテンプレートを管理
   - 問題が発生したら以前のバージョンに戻せる

---

**参考**：エラーが発生したら、まず**エラーメッセージをよく読む**ことが重要です。多くの場合、エラーメッセージに原因と対処法が記載されています。
