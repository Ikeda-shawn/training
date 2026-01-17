# 第7章 CloudFormation と運用上の注意点

## 手動変更（ドリフト）の問題

CloudFormation で管理しているリソースを、**AWS コンソールや AWS CLI で直接変更すると、テンプレートと実際のリソースに差分が生まれます**。

この差分を**ドリフト（Drift）**と呼びます。

### ドリフトが発生する例

#### 1. CloudFormation でリソースを作成

```yaml
Resources:
  MySecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web server
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
```

この時点では、テンプレートと実際のリソースが一致しています。

#### 2. AWS コンソールから直接変更

運用担当者が、AWS コンソールから SecurityGroup にポート 443 のルールを追加しました。

```
実際のリソース：
  - Port 80（テンプレートに記載）
  - Port 443（手動で追加）← ドリフト発生

テンプレート：
  - Port 80 のみ
```

#### 3. 問題が発生

後日、CloudFormation でスタックを更新すると、**手動で追加したポート 443 のルールが削除されてしまいます**。

```
スタック更新時：
  テンプレートに記載されていない Port 443 のルールが削除される
```

### なぜドリフトが問題なのか

1. **テンプレートが「真実」ではなくなる**
   - テンプレートを見ても、実際の状態がわからない

2. **スタック更新時に意図しない変更が発生する**
   - 手動で追加した設定が削除される
   - 逆に、手動で削除した設定が復活する

3. **再現性が失われる**
   - 同じテンプレートでデプロイしても、同じ環境が作れない

## Drift Detection の考え方

CloudFormation には、**ドリフトを検出する機能**があります。

### Drift Detection の実行

```bash
aws cloudformation detect-stack-drift --stack-name my-stack
```

### ドリフトの確認

```bash
aws cloudformation describe-stack-resource-drifts --stack-name my-stack
```

出力例：

```json
{
  "StackResourceDrifts": [
    {
      "StackId": "arn:aws:cloudformation:...",
      "LogicalResourceId": "MySecurityGroup",
      "ResourceType": "AWS::EC2::SecurityGroup",
      "StackResourceDriftStatus": "MODIFIED",
      "Timestamp": "2025-01-01T00:00:00Z",
      "PropertyDifferences": [
        {
          "PropertyPath": "/SecurityGroupIngress/1",
          "ExpectedValue": "null",
          "ActualValue": "{\"IpProtocol\":\"tcp\",\"FromPort\":443,...}",
          "DifferenceType": "ADD"
        }
      ]
    }
  ]
}
```

### ドリフトが検出された場合の対処

#### パターン1：テンプレートを修正する

手動で追加した変更を、テンプレートに反映します。

```yaml
Resources:
  MySecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web server
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443  # 追加
          ToPort: 443    # 追加
          CidrIp: 0.0.0.0/0
```

#### パターン2：手動変更を取り消す

実際のリソースを、テンプレートに合わせて戻します。

```bash
# SecurityGroup から手動追加したルールを削除
```

## なぜ「触ってはいけないリソース」が生まれるのか

実務では、「このリソースは CloudFormation で管理しているから、手動で触らないでください」という状況がよく発生します。

### 問題の根本原因

1. **CloudFormation の仕組みが理解されていない**
   - 手動変更がドリフトを引き起こすことを知らない

2. **緊急対応で手動変更してしまう**
   - 本番障害時に、テンプレート修正→デプロイの時間がない
   - 応急処置として AWS コンソールから直接変更

3. **テンプレートへの反映を忘れる**
   - 手動変更後、テンプレートに反映するのを忘れる
   - 次回のデプロイ時に手動変更が消える

### 対策

#### 1. チーム全体で CloudFormation の運用ルールを共有

- **手動変更は禁止**
- **緊急対応後は必ずテンプレートに反映**
- **定期的に Drift Detection を実行**

#### 2. タグで管理対象を明示

```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      Tags:
        - Key: ManagedBy
          Value: CloudFormation
        - Key: StackName
          Value: !Ref AWS::StackName
```

#### 3. IAM ポリシーで手動変更を制限

特定のリソースは、CloudFormation 経由でのみ変更できるように IAM ポリシーで制限します。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "ec2:AuthorizeSecurityGroupIngress",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalArn": "arn:aws:iam::123456789012:role/CloudFormationRole"
        }
      }
    }
  ]
}
```

## CloudFormation を前提とした運用意識

### 「コンソールで確認、テンプレートで変更」の原則

- **AWS コンソール**：リソースの確認、ログの参照
- **CloudFormation**：リソースの作成・変更・削除

### ドキュメントとしてのテンプレート

テンプレートは、インフラの**ドキュメント**でもあります。

- 現在の環境がどうなっているかは、テンプレートを見ればわかる
- 変更履歴は Git のコミットログで追跡できる

### 環境の再現性

CloudFormation を正しく運用していれば、いつでも同じ環境を再現できます。

- 災害復旧（DR）時に同じ環境を別リージョンに構築
- 開発環境を本番環境と同じ構成で作成

### 変更の透明性

テンプレートの変更は、コードレビューや ChangeSet で確認できます。

- 誰が、いつ、何を変更したかが明確
- 意図しない変更を事前に防げる

## 実務での運用例

### ケース1：緊急対応での手動変更

**状況**：本番環境で障害が発生し、SecurityGroup のルールを緊急で追加する必要がある

**対応**：
1. 緊急対応として AWS コンソールから SecurityGroup を変更
2. 障害対応後、Issue または Ticket を作成し、「テンプレートへの反映が必要」と記録
3. 落ち着いたタイミングで、テンプレートに反映し、スタックを更新

### ケース2：定期的なドリフト検出

**運用**：
- 毎週月曜日に、本番スタックの Drift Detection を実行
- ドリフトが検出された場合は、原因を調査し、テンプレートに反映

```bash
# Drift Detection の実行
aws cloudformation detect-stack-drift --stack-name prod-stack

# 結果の確認
aws cloudformation describe-stack-resource-drifts --stack-name prod-stack
```

### ケース3：新メンバーへの教育

**チームに新しいメンバーが加わった場合**：
1. CloudFormation の基本概念を説明
2. 「手動変更は原則禁止」というルールを共有
3. テンプレートの Git リポジトリへのアクセス権を付与
4. 変更フローを説明（テンプレート修正 → レビュー → デプロイ）

## 次の章へ

運用上の注意点が理解できたら、[第8章 よくあるアンチパターン](../chapter08/README.md) に進んでください。

避けるべき設計パターンを学び、より良いテンプレートを書けるようになります。
