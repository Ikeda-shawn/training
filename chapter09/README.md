# 第9章 演習（CloudFormation テンプレート作成）

本章では、これまで学んだ内容を実践する演習問題を用意しています。

## 演習の進め方

1. **要件を読む**
2. **テンプレートを作成する**
3. **テンプレートを検証する**
4. **（可能であれば）実際にデプロイして動作確認する**

## 演習1：基本的な VPC とサブネットの作成

### 要件

以下の要件を満たす CloudFormation テンプレートを作成してください。

#### リソース要件

- **VPC**
  - CIDR ブロック：`10.0.0.0/16`
  - DNS ホスト名を有効化

- **パブリックサブネット**
  - CIDR ブロック：`10.0.1.0/24`
  - アベイラビリティゾーン：`ap-northeast-1a`

- **プライベートサブネット**
  - CIDR ブロック：`10.0.2.0/24`
  - アベイラビリティゾーン：`ap-northeast-1c`

#### テンプレート要件

- YAML 形式で記述すること
- パラメータとして「環境名」を受け取ること（dev, stg, prod のいずれか）
- すべてのリソースに `Name` タグを付けること（例：`dev-vpc`）
- VPC ID とサブネット ID を Outputs として出力すること

### ヒント

- `AWSTemplateFormatVersion` と `Description` を記述しましょう
- パラメータには `AllowedValues` で選択肢を制限しましょう
- タグには `!Sub` を使って環境名を埋め込みましょう

### 検証コマンド

```bash
# テンプレートの検証
aws cloudformation validate-template --template-body file://exercise1.yaml
```

### 解答例の場所

解答例は `templates/exercises/exercise1-solution.yaml` に用意されています。

まずは自分で考えてから、解答例を確認してください。

---

## 演習2：SecurityGroup の作成

### 要件

演習1で作成した VPC に、以下の SecurityGroup を追加してください。

#### リソース要件

- **Web サーバー用 SecurityGroup**
  - HTTP（ポート 80）を全体に公開
  - HTTPS（ポート 443）を全体に公開
  - SSH（ポート 22）を特定の IP アドレスからのみ許可
    - 許可する IP アドレスはパラメータとして受け取る

#### テンプレート要件

- 演習1のテンプレートを拡張すること
- SSH を許可する IP アドレスをパラメータとして受け取ること
  - パラメータ名：`SSHAllowedCIDR`
  - デフォルト値：`0.0.0.0/0`（開発環境用）
  - 説明：「SSH を許可する CIDR ブロック」
- SecurityGroup ID を Outputs として出力すること

### ヒント

- `SecurityGroupIngress` はリスト形式で複数のルールを定義できます
- `CidrIp` にパラメータを `!Ref` で参照できます

### 検証コマンド

```bash
# テンプレートの検証
aws cloudformation validate-template --template-body file://exercise2.yaml
```

---

## 演習3：スタック分割（ネットワークとアプリケーション）

### 要件

演習2で作成したテンプレートを、以下の2つのスタックに分割してください。

#### ネットワークスタック（network-stack.yaml）

- VPC
- パブリックサブネット
- プライベートサブネット

**Outputs として出力する値**：
- VPC ID（Export あり）
- パブリックサブネット ID（Export あり）
- プライベートサブネット ID（Export あり）

#### アプリケーションスタック（application-stack.yaml）

- SecurityGroup

**Parameters として受け取る値**：
- ネットワークスタックのスタック名

**ImportValue で参照する値**：
- VPC ID

### ヒント

- Export 名には `${AWS::StackName}` を使いましょう
- ImportValue では `Fn::Sub` を使ってスタック名を埋め込みましょう

### 検証コマンド

```bash
# ネットワークスタックの検証
aws cloudformation validate-template --template-body file://network-stack.yaml

# アプリケーションスタックの検証
aws cloudformation validate-template --template-body file://application-stack.yaml
```

### デプロイの順序

```bash
# 1. ネットワークスタックを作成
aws cloudformation create-stack \
  --stack-name dev-network-stack \
  --template-body file://network-stack.yaml \
  --parameters ParameterKey=EnvironmentName,ParameterValue=dev

# 2. アプリケーションスタックを作成
aws cloudformation create-stack \
  --stack-name dev-application-stack \
  --template-body file://application-stack.yaml \
  --parameters ParameterKey=NetworkStackName,ParameterValue=dev-network-stack
```

---

## 演習4：IAM Role の作成（応用）

### 要件

EC2 インスタンスが S3 バケットにアクセスできるように、IAM Role を作成してください。

#### リソース要件

- **IAM Role**
  - EC2 インスタンスがこのロールを引き受けられること
  - S3 の読み取り専用ポリシーがアタッチされていること（`AmazonS3ReadOnlyAccess`）

- **Instance Profile**
  - 上記の IAM Role を含むこと

#### テンプレート要件

- YAML 形式で記述すること
- IAM Role 名を Outputs として出力すること
- Instance Profile の ARN を Outputs として出力すること

### ヒント

- `AssumeRolePolicyDocument` で EC2 サービスがロールを引き受けられるように設定します
- `ManagedPolicyArns` で AWS 管理ポリシーをアタッチできます

### 検証コマンド

```bash
# テンプレートの検証
aws cloudformation validate-template --template-body file://exercise4.yaml

# IAM リソースを作成する場合は --capabilities オプションが必要
aws cloudformation create-stack \
  --stack-name iam-role-stack \
  --template-body file://exercise4.yaml \
  --capabilities CAPABILITY_IAM
```

---

## 演習5：総合演習

### 要件

以下の要件を満たす、本番環境を想定したテンプレートを作成してください。

#### システム要件

- Web アプリケーションをホストする環境
- パブリックサブネットに Web サーバーを配置
- プライベートサブネットにデータベースを配置（RDS は不要、将来的に配置することを想定）

#### リソース要件

1. **VPC**
   - CIDR ブロック：パラメータで指定可能

2. **パブリックサブネット × 2**
   - 異なる AZ に配置
   - CIDR ブロックは VPC の CIDR から自動計算（例：10.0.1.0/24, 10.0.2.0/24）

3. **プライベートサブネット × 2**
   - 異なる AZ に配置
   - CIDR ブロックは VPC の CIDR から自動計算（例：10.0.101.0/24, 10.0.102.0/24）

4. **Internet Gateway**
   - VPC にアタッチ

5. **Route Table（パブリック用）**
   - Internet Gateway へのルートを追加
   - パブリックサブネットに関連付け

6. **SecurityGroup（Web サーバー用）**
   - HTTP/HTTPS を全体に公開

7. **SecurityGroup（データベース用）**
   - Web サーバー用 SecurityGroup からのアクセスのみ許可（MySQL: ポート 3306）

#### テンプレート要件

- スタックは適切に分割すること（ネットワーク層とアプリケーション層）
- すべてのリソースに適切な Name タグを付けること
- 必要な値は Outputs として出力すること
- パラメータは必要最小限にすること

### 評価観点

- **正しく動作するか**
- **可読性**（コメント、論理 ID、インデント）
- **設計意図が読み取れるか**
- **適切にスタックが分割されているか**

---

## 次の章へ

演習が完了したら、[第10章 まとめ](../chapter10/README.md) に進んでください。

本コンテンツで学んだ内容を振り返り、次のステップを確認します。
