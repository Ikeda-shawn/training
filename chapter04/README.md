# 第4章 パラメータ設計と可読性

## Parameters を使う理由

CloudFormation テンプレートに値を直接書き込む（ハードコード）ことも可能ですが、`Parameters` を使うことで多くのメリットがあります。

### Parameters を使わない例

```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16  # ハードコード
      Tags:
        - Key: Name
          Value: dev-vpc  # ハードコード
```

### Parameters を使う例

```yaml
Parameters:
  EnvironmentName:
    Type: String
    Default: dev

  VPCCidrBlock:
    Type: String
    Default: 10.0.0.0/16

Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VPCCidrBlock
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-vpc'
```

### Parameters を使うメリット

1. **再利用性**：同じテンプレートで dev/stg/prod 環境を作成できる
2. **柔軟性**：デプロイ時に値を変更できる
3. **可読性**：パラメータ名で「何の値か」が明確になる
4. **安全性**：デフォルト値を設定することで、入力ミスを防げる

## Parameters の基本的な書き方

### 最小限のパラメータ定義

```yaml
Parameters:
  EnvironmentName:
    Type: String
```

### 推奨されるパラメータ定義

```yaml
Parameters:
  EnvironmentName:
    Type: String
    Default: dev
    Description: 環境名（dev, stg, prod）
    AllowedValues:
      - dev
      - stg
      - prod
```

### パラメータで指定できる項目

| 項目 | 必須 | 説明 |
|---|---|---|
| Type | 必須 | パラメータのデータ型 |
| Default | 任意 | デフォルト値 |
| Description | 任意（推奨） | パラメータの説明 |
| AllowedValues | 任意 | 許可される値のリスト |
| AllowedPattern | 任意 | 正規表現による制約 |
| MinLength / MaxLength | 任意 | 文字列の長さの制約 |
| MinValue / MaxValue | 任意 | 数値の範囲の制約 |

## パラメータの Type（データ型）

### 基本的な Type

| Type | 説明 | 例 |
|---|---|---|
| String | 文字列 | `dev`, `10.0.0.0/16` |
| Number | 数値 | `100`, `256` |
| CommaDelimitedList | カンマ区切りのリスト | `subnet-1,subnet-2` |

### AWS 固有の Type

CloudFormation には、AWS リソース ID を扱うための特別な Type があります。

| Type | 説明 |
|---|---|
| AWS::EC2::VPC::Id | VPC ID |
| AWS::EC2::Subnet::Id | Subnet ID |
| AWS::EC2::SecurityGroup::Id | SecurityGroup ID |
| AWS::EC2::KeyPair::KeyName | EC2 キーペア名 |
| AWS::EC2::Image::Id | AMI ID |

これらを使うと、AWS コンソールからスタックを作成する際に、**プルダウンメニュー**で選択できるようになります。

```yaml
Parameters:
  VPCId:
    Type: AWS::EC2::VPC::Id
    Description: 既存の VPC を選択してください
```

## デフォルト値・制約の考え方

### Default（デフォルト値）

デフォルト値を設定することで、パラメータの入力を省略できます。

```yaml
Parameters:
  InstanceType:
    Type: String
    Default: t2.micro
    Description: EC2 インスタンスタイプ
```

### AllowedValues（許可される値）

選択肢を限定することで、入力ミスを防ぎます。

```yaml
Parameters:
  EnvironmentName:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - stg
      - prod
```

### AllowedPattern（正規表現による制約）

CIDR ブロックなど、特定の形式を強制したい場合に使います。

```yaml
Parameters:
  VPCCidrBlock:
    Type: String
    Default: 10.0.0.0/16
    AllowedPattern: '^(10|172|192)\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}/\\d{1,2}$'
    ConstraintDescription: 有効な CIDR ブロックを入力してください
```

### MinLength / MaxLength（文字列の長さ制約）

```yaml
Parameters:
  ProjectName:
    Type: String
    MinLength: 3
    MaxLength: 20
    Description: プロジェクト名（3〜20文字）
```

## 環境差分の持たせ方

同じテンプレートで複数の環境を作成する際の設計パターンです。

### パターン1：EnvironmentName パラメータを使う

```yaml
Parameters:
  EnvironmentName:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - stg
      - prod

Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-vpc'
        - Key: Environment
          Value: !Ref EnvironmentName
```

### パターン2：環境ごとにパラメータファイルを用意する

テンプレート自体はパラメータのみを定義し、環境ごとに異なる値を別ファイルで管理します。

```yaml
# template.yaml
Parameters:
  EnvironmentName:
    Type: String
  VPCCidrBlock:
    Type: String
  InstanceType:
    Type: String
```

```json
// parameters-dev.json
[
  {
    "ParameterKey": "EnvironmentName",
    "ParameterValue": "dev"
  },
  {
    "ParameterKey": "VPCCidrBlock",
    "ParameterValue": "10.0.0.0/16"
  },
  {
    "ParameterKey": "InstanceType",
    "ParameterValue": "t2.micro"
  }
]
```

```json
// parameters-prod.json
[
  {
    "ParameterKey": "EnvironmentName",
    "ParameterValue": "prod"
  },
  {
    "ParameterKey": "VPCCidrBlock",
    "ParameterValue": "10.1.0.0/16"
  },
  {
    "ParameterKey": "InstanceType",
    "ParameterValue": "t3.medium"
  }
]
```

デプロイ時にパラメータファイルを指定します。

```bash
# dev 環境
aws cloudformation create-stack \
  --stack-name dev-stack \
  --template-body file://template.yaml \
  --parameters file://parameters-dev.json

# prod 環境
aws cloudformation create-stack \
  --stack-name prod-stack \
  --template-body file://template.yaml \
  --parameters file://parameters-prod.json
```

## ハードコードしてよいもの / 避けるべきもの

### ハードコードしてよいもの

以下のような値は、パラメータ化する必要がありません。

- **AWS サービスの固定値**（ポート番号 443、プロトコル tcp など）
- **構造的に変わらない値**（AZ の数、サブネットの構成など）

```yaml
Resources:
  MySecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      SecurityGroupIngress:
        - IpProtocol: tcp  # ハードコードで OK
          FromPort: 443    # ハードコードで OK
          ToPort: 443      # ハードコードで OK
          CidrIp: 0.0.0.0/0
```

### パラメータ化すべきもの

以下のような値は、パラメータ化を検討すべきです。

- **環境によって変わる値**（環境名、CIDR ブロック、インスタンスタイプなど）
- **将来変更する可能性がある値**（タグ、リソース名など）

```yaml
Parameters:
  InstanceType:
    Type: String
    Default: t2.micro

  VPCCidrBlock:
    Type: String
    Default: 10.0.0.0/16
```

## 可読性を損なわないためのパラメータ設計

### パラメータの数を増やしすぎない

パラメータが多すぎると、テンプレートが複雑になり、かえって読みにくくなります。

**悪い例：パラメータが多すぎる**

```yaml
Parameters:
  VPCCidrBlock:
    Type: String
  Subnet1CidrBlock:
    Type: String
  Subnet2CidrBlock:
    Type: String
  Subnet3CidrBlock:
    Type: String
  Subnet4CidrBlock:
    Type: String
  # ... 20 個のパラメータ
```

**良い例：必要最小限のパラメータ**

```yaml
Parameters:
  EnvironmentName:
    Type: String
  VPCCidrBlock:
    Type: String
    Default: 10.0.0.0/16
```

### パラメータ名はわかりやすく

```yaml
# 悪い例
Parameters:
  P1:
    Type: String

# 良い例
Parameters:
  EnvironmentName:
    Type: String
```

### Description を必ず書く

パラメータには必ず `Description` を付けましょう。

```yaml
Parameters:
  InstanceType:
    Type: String
    Default: t2.micro
    Description: EC2 インスタンスのタイプ（本番環境では t3.medium 以上を推奨）
```

## 次の章へ

パラメータ設計が理解できたら、[第5章 Outputs とスタック連携の考え方](ch05_outputs.md) に進んでください。

作成したリソースを他のスタックから参照する方法を学びます。
