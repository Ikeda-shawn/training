# 第4章 パラメータ設計と可読性

## この章で学ぶこと

この章では、CloudFormationテンプレートの**Parameters**と**Mappings**について学びます。

これらを適切に使うことで、同じテンプレートを複数の環境（dev/stg/prod）で再利用できるようになります。

## Parameters を使う理由

CloudFormationテンプレートに値を直接書き込む（ハードコード）ことも可能ですが、`Parameters` を使うことで多くのメリットがあります。

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

この書き方では、環境ごとにテンプレートを用意する必要があります。

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

この書き方であれば、デプロイ時にパラメータを変更するだけで、同じテンプレートを複数の環境で使い回せます。

### Parameters を使うメリット

| メリット | 説明 |
|---|---|
| 再利用性 | 同じテンプレートで dev/stg/prod環境を作成できる |
| 柔軟性 | デプロイ時に値を変更できる |
| 可読性 | パラメータ名で「何の値か」が明確になる |
| 安全性 | デフォルト値やバリデーションを設定することで、入力ミスを防げる |

## Parameters の基本的な書き方

### 最小限のパラメータ定義

```yaml
Parameters:
  EnvironmentName:
    Type: String
```

これだけでもパラメータとして機能しますが、実務では以下のような定義が推奨されます。

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

パラメータで指定できる項目としては、下記の項目があります。

私が普段案件などでCloudFormationを書く際には`Type~AllowedPattern`を書いておくことが多いです。

| 項目 | 必須 | 説明 |
|---|---|---|
| Type | 必須 | パラメータのデータ型 |
| Default | 任意 | デフォルト値（省略時はデプロイ時に入力必須） |
| Description | 任意（推奨） | パラメータの説明 |
| AllowedValues | 任意 | 許可される値のリスト |
| AllowedPattern | 任意 | 正規表現による制約 |
| MinLength / MaxLength | 任意 | 文字列の長さの制約 |
| MinValue / MaxValue | 任意 | 数値の範囲の制約 |
| ConstraintDescription | 任意 | 制約に違反した際のエラーメッセージ |

## パラメータの Type（データ型）

### 基本的な Type

| Type | 説明 | 例 |
|---|---|---|
| String | 文字列 | `dev`, `10.0.0.0/16` |
| Number | 数値 | `100`, `256` |
| CommaDelimitedList | カンマ区切りのリスト | `subnet-1,subnet-2` |

## パラメータの制約

パラメータには様々な制約を設定できます。これにより入力ミスを防ぎ、テンプレートの安全性を高めることができます。

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

ここで設定した値はデプロイ時にコンソール上で指定する必要があります。

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

CIDRブロックなど、特定の形式を強制したい場合に使います。

```yaml
Parameters:
  VPCCidrBlock:
    Type: String
    Default: 10.0.0.0/16
    AllowedPattern: '^(10|172|192)\.\d{1,3}\.\d{1,3}\.\d{1,3}/\d{1,2}$'
    ConstraintDescription: 有効なプライベート CIDR ブロックを入力してください（例：10.0.0.0/16）
```

（※`ConstraintDescription`を設定しておくと、制約に違反した際に分かりやすいエラーメッセージが表示されます）

### MinLength/MaxLength（文字列の長さ制約）

AWSリソースの命名規則にプロジェクト名を指定することも多いと思います。

AWSリソースの命名には文字数の制限があるため、このようにパラメータの文字数に制限をつけておくことも良いと思います。

```yaml
Parameters:
  ProjectName:
    Type: String
    MinLength: 3
    MaxLength: 20
    Description: プロジェクト名（3〜20文字）
```

### MinValue/MaxValue（数値の範囲制約）

```yaml
Parameters:
  InstanceCount:
    Type: Number
    MinValue: 1
    MaxValue: 10
    Default: 2
    Description: 起動するインスタンス数（1〜10）
```

## Mappings の使い方

`Mappings`は、テンプレート内で使用する**固定的な対応表（マッピング）**を定義するセクションです。

第2章で軽く触れましたが、ここで詳しく説明します。

### Mappings の基本構造

```yaml
Mappings:
  マッピング名:
    キー1:
      属性1: 値1
      属性2: 値2
    キー2:
      属性1: 値3
      属性2: 値4
```

### 具体例：リージョンごとのAMI ID

リージョンによって AMI IDが異なる場合に便利です。

```yaml
Mappings:
  RegionAMIMap:
    ap-northeast-1:
      HVM64: ami-0123456789abcdef0
    ap-northeast-3:
      HVM64: ami-0fedcba9876543210
    us-east-1:
      HVM64: ami-0abcdef1234567890
```

### Mappingsの参照方法（Fn::FindInMap）

`!FindInMap`（または`Fn::FindInMap`）を使って、Mappingsで定義した値を参照します。

```yaml
!FindInMap
  - マッピング名
  - 第1キー
  - 第2キー
```

#### 使用例

```yaml
Resources:
  MyEC2:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap
        - RegionAMIMap
        - !Ref AWS::Region  # 現在のリージョンを動的に取得
        - HVM64
      InstanceType: t2.micro
```

この例では、デプロイ先のリージョンに応じて適切な AMI IDが自動的に選択されます。

### 具体例：環境ごとの設定値

環境（dev/stg/prod）ごとに異なる設定値を定義することもできます。

```yaml
Mappings:
  EnvironmentConfig:
    dev:
      InstanceType: t2.micro
      VolumeSize: 20
    stg:
      InstanceType: t3.small
      VolumeSize: 50
    prod:
      InstanceType: t3.medium
      VolumeSize: 100

Parameters:
  EnvironmentName:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - stg
      - prod

Resources:
  MyEC2:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !FindInMap
        - EnvironmentConfig
        - !Ref EnvironmentName
        - InstanceType
      # ...その他の設定
```

### ParametersとMappingsの使い分け

私は`適したケース`のところを基準にこの二つを使い分けることが多いです。

普段からある程度標準化したCloudFormationを使っているので、標準から外れている箇所をMappingsに定義しています。

| 観点 | Parameters | Mappings |
|---|---|---|
| 値の決定タイミング | デプロイ時にユーザーが入力 | テンプレート作成時に定義済み |
| 適したケース | 環境名、プロジェクト名など | リージョンごとのAMI ID、環境ごとの固定設定など |
| 柔軟性 | 高い（任意の値を入力可能） | 低い（事前定義された値のみ） |
| 安全性 | AllowedValuesなどで制約可能 | 定義済みの値しか使えないため安全 |

## ハードコードしてよいもの / 避けるべきもの

### ハードコードしてよいもの

以下のような値は、パラメータ化する必要がありません。

- **AWS サービスの固定値**（ポート番号443、プロトコルtcp など）
- **構造的に変わらない値**（AZの数、サブネットの構成など）

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
- **セキュリティに関わる値**（接続元 IP アドレスなど）

```yaml
Parameters:
  InstanceType:
    Type: String
    Default: t2.micro

  AllowedCidr:
    Type: String
    Description: SSH接続を許可するCIDR
```

## 可読性を高めるパラメータ設計

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

**良い例：必要最小限のパラメータ + Mappingsで管理**

```yaml
Parameters:
  EnvironmentName:
    Type: String
    Default: dev

Mappings:
  SubnetConfig:
    dev:
      Subnet1Cidr: 10.0.1.0/24
      Subnet2Cidr: 10.0.2.0/24
    prod:
      Subnet1Cidr: 10.1.1.0/24
      Subnet2Cidr: 10.1.2.0/24
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

### Descriptionを必ず書く

パラメータには必ず`Description`を付けましょう。

```yaml
Parameters:
  InstanceType:
    Type: String
    Default: t2.micro
    Description: EC2 インスタンスのタイプ（本番環境ではt3.medium以上を推奨）
```

### パラメータの順序を意識する

関連するパラメータはグループ化して並べると、可読性が向上します。

```yaml
Parameters:
  # 環境設定
  EnvironmentName:
    Type: String
    Description: 環境名
  ProjectName:
    Type: String
    Description: プロジェクト名

  # ネットワーク設定
  VPCCidrBlock:
    Type: String
    Description: VPCのCIDRブロック

  # コンピューティング設定
  InstanceType:
    Type: String
    Description: EC2インスタンスタイプ
```

## 次の章へ

パラメータ設計が理解できたら、[第5章 Outputs とスタック連携の考え方](../chapter05/README.md) に進んでください。

作成したリソースを他のスタックから参照する方法を学びます。
