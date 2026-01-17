# 第2章 テンプレートの基本構造

## CloudFormation テンプレートの全体構成

CloudFormation テンプレートは、いくつかのセクションで構成されています。

### 基本的なテンプレート構造

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: テンプレートの説明

Parameters:
  # パラメータの定義

Resources:
  # リソースの定義（必須）

Outputs:
  # 出力値の定義
```

### 各セクションの役割

| セクション | 必須/任意 | 役割 |
|---|---|---|
| AWSTemplateFormatVersion | 任意（推奨） | テンプレートのバージョンを指定 |
| Description | 任意（推奨） | テンプレートの説明 |
| Parameters | 任意 | 外部から値を受け取る |
| Resources | **必須** | AWS リソースを定義する |
| Outputs | 任意 | 作成したリソースの情報を出力する |

他にも `Metadata`、`Mappings`、`Conditions`、`Transform` などのセクションがありますが、本コンテンツでは基本的なセクションに絞って説明します。

## AWSTemplateFormatVersion

テンプレートのバージョンを指定します。

```yaml
AWSTemplateFormatVersion: '2010-09-09'
```

現在、指定できるバージョンは `2010-09-09` のみです。

- 省略可能ですが、明示的に記述することが推奨されます
- シングルクォートで囲むのは、YAML の日付型として解釈されるのを防ぐためです

## Description

テンプレートの説明を記述します。

```yaml
Description: VPC とパブリックサブネットを作成するテンプレート
```

- 省略可能ですが、テンプレートの目的を明確にするために記述することが推奨されます
- AWS コンソールでスタックを確認する際に表示されます

## Parameters

外部から値を受け取るための定義です。

```yaml
Parameters:
  EnvironmentName:
    Type: String
    Default: dev
    Description: 環境名（dev, stg, prod）
```

詳細は [第4章 パラメータ設計と可読性](ch04_parameters.md) で説明します。

## Resources（必須）

AWS リソースを定義します。**テンプレートに必ず含める必要があります**。

```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
```

詳細は [第3章 リソース定義の書き方・読み方](ch03_resource_definition.md) で説明します。

## Outputs

作成したリソースの情報を出力します。

```yaml
Outputs:
  VPCId:
    Value: !Ref MyVPC
    Description: VPC の ID
```

詳細は [第5章 Outputs とスタック連携の考え方](ch05_outputs.md) で説明します。

## YAML 記法の基本

CloudFormation テンプレートは、YAML 形式で記述することが一般的です。

### インデント

YAML では**インデント（字下げ）**が非常に重要です。

- **スペース2つ**でインデントするのが一般的です（スペース4つでも可）
- **タブ文字は使用できません**

#### 正しい例

```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
```

#### 間違った例

```yaml
Resources:
MyVPC:  # インデントが足りない
Type: AWS::EC2::VPC  # インデントが足りない
```

### マップ（Map）

キーと値のペアを表します。

```yaml
キー: 値
```

例：

```yaml
Type: AWS::EC2::VPC
Description: これは説明です
```

### リスト（List）

複数の要素を並べる場合に使います。

```yaml
- 要素1
- 要素2
- 要素3
```

例：

```yaml
SecurityGroupIngress:
  - IpProtocol: tcp
    FromPort: 80
    ToPort: 80
  - IpProtocol: tcp
    FromPort: 443
    ToPort: 443
```

### コメント

`#` 以降はコメントとして扱われます。

```yaml
Resources:
  MyVPC:  # これはコメントです
    Type: AWS::EC2::VPC
    Properties:
      # CIDR ブロックの設定
      CidrBlock: 10.0.0.0/16
```

## CloudFormation 特有の組み込み関数

CloudFormation には、テンプレート内で使える組み込み関数があります。

### 組み込み関数の2つの記法

CloudFormation の組み込み関数には、2つの書き方があります。

#### 完全名記法

```yaml
CidrBlock: !Ref VPCCidrBlock
```

#### 短縮記法（`!` を使う）

```yaml
CidrBlock:
  Ref: VPCCidrBlock
```

本コンテンツでは、より簡潔な**短縮記法**を使用します。

### Ref（参照）

パラメータや他のリソースを参照します。

```yaml
Parameters:
  VPCCidrBlock:
    Type: String
    Default: 10.0.0.0/16

Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VPCCidrBlock  # パラメータを参照
```

リソースを参照する場合：

```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

  MySubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC  # MyVPC リソースの ID を参照
```

### Fn::Sub（文字列置換）

文字列内に変数を埋め込みます。

```yaml
Description: !Sub '${EnvironmentName} 環境の VPC'
```

`EnvironmentName` が `dev` の場合、`dev 環境の VPC` という文字列になります。

### Fn::Join（文字列結合）

複数の文字列を結合します。

```yaml
Name: !Join
  - '-'
  - - !Ref EnvironmentName
    - 'vpc'
```

`EnvironmentName` が `dev` の場合、`dev-vpc` という文字列になります。

### Fn::GetAtt（属性取得）

リソースの属性を取得します。

```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

Outputs:
  VPCCidrBlock:
    Value: !GetAtt MyVPC.CidrBlock
```

`!Ref` がリソースの ID を返すのに対し、`!GetAtt` は特定の属性を返します。

### どの属性が取得できるか？

各リソースで取得できる属性は、AWS 公式ドキュメントの「Return values」セクションに記載されています。

例：`AWS::EC2::VPC` の場合
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpc.html

## 実際のテンプレート例

これまでの内容を踏まえた、実際のテンプレート例です。

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 学習用の VPC テンプレート

Parameters:
  EnvironmentName:
    Type: String
    Default: dev
    Description: 環境名

Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-vpc'

  MySubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: ap-northeast-1a
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-subnet'

Outputs:
  VPCId:
    Description: VPC の ID
    Value: !Ref MyVPC

  SubnetId:
    Description: Subnet の ID
    Value: !Ref MySubnet
```

## 次の章へ

テンプレートの基本構造が理解できたら、[第3章 リソース定義の書き方・読み方](ch03_resource_definition.md) に進んでください。

Resources セクションの詳細な書き方を学びます。
