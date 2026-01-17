# 第2章 テンプレートの基本構造

## CloudFormationテンプレートの全体構成

CloudFormationテンプレートは、いくつかのセクションで構成されています。

### 基本的なテンプレート構造

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: テンプレートの説明

Parameters:
  # パラメータの定義

Mappings:
  # マッピングの定義

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
| Parameters | 任意 | リソースのデプロイ時に任意の値を受ける |
| Mappings | 任意 | あらかじめ定義した対応表（マッピング）を保持する |
| Resources | **必須** | AWSリソースを定義する |
| Outputs | 任意 | 作成したリソースの情報を出力する |

他にも `Metadata`、`Conditions`、`Transform` などのセクションがありますが、本コンテンツでは基本的なセクションに絞って説明します。

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

省略されていることも多いですが、コードの保守性を上げる意味合いでも**どのようなリソースを作っているテンプレートなのか**程度のことを書いておくことをお勧めします。

```yaml
Description: VPCとパブリックサブネットを作成するテンプレート
```

## Parameters

外部から値を受け取るための定義です。

例えば、以下のように環境名やシステム名などを定義しておくことが多いです。

```yaml
Parameters:
  EnvironmentName:
    Type: String
    Default: dev
    Description: 環境名（dev, stg, prod）
```

詳細は [第4章 パラメータ設計と可読性](../chapter04/README.md) で説明します。

## Mappings

テンプレート内で使用する**固定的な対応表（マッピング）**を定義します。

リージョンごとの設定値や、環境ごとに異なる定数値などを  
テンプレート内にまとめて定義しておくために使用されます。

```yaml
Mappings:
  RegionMap:
    ap-northeast-1:
      VpcCidr: 10.0.0.0/16
    us-east-1:
      VpcCidr: 10.1.0.0/16
```

Mappingsで定義した値は、Fn::FindInMapを使って参照します。

```yaml
CidrBlock: !FindInMap
  - RegionMap
  - !Ref AWS::Region
  - VpcCidr
```

詳細は [第4章 パラメータ設計と可読性](../chapter04/README.md) で説明します。

## Resources（必須）

AWSリソースを定義します。**テンプレートに必ず含める必要があります**。

```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
```

詳細は [第3章 リソース定義の書き方・読み方](../chapter03/README.md) で説明します。

## Outputs

作成したリソースの情報を出力します。

Outputsで出力定義をしたリソースは別のスタックからでも利用可能となるため、出力できる情報は出力しておくことを推奨します。

```yaml
Outputs:
  VPCId:
    Value: !Ref MyVPC
    Description: VPC の ID
```

詳細は [第5章 Outputs とスタック連携の考え方](../chapter05/README.md) で説明します。

## YAML 記法の基本

CloudFormationテンプレートは、YAML形式で記述することが一般的です。

### インデント

YAMLでは**インデント（字下げ）**が非常に重要です。

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

## CloudFormation特有の組み込み関数

CloudFormationには、テンプレート内で使える組み込み関数があります。

### 組み込み関数の2つの記法

CloudFormationの組み込み関数には、2つの書き方があります。

#### 完全名記法

```yaml
CidrBlock: 
  Ref: VPCCidrBlock
```

#### 短縮記法（`!` を使う）

```yaml
CidrBlock: !Ref VPCCidrBlock
```

本コンテンツでは、**短縮記法**を使用します。

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
      VpcId: !Ref MyVPC  # MyVPCリソースのIDを参照
```

### Fn::Sub（文字列置換）

文字列内に変数を埋め込みます。

```yaml
Description: !Sub '${EnvironmentName}環境のVPC'
``` 

`EnvironmentName`が`dev`の場合、`dev環境のVPC`という文字列になります。

### Fn::Join（文字列結合）

複数の文字列を結合します。

```yaml
Name: !Join
  - '-'
  - - !Ref EnvironmentName
    - 'vpc'
```

`EnvironmentName`が`dev`の場合、`dev-vpc`という文字列になります。

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

`!Ref`がリソースのIDを返すのに対し、`!GetAtt`は特定の属性を返します。

### どの属性が取得できるか？

各リソースで取得できる属性は、AWS公式ドキュメントの「Return values」セクションに記載されています。

例：`AWS::EC2::VPC`の場合
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpc.html

## 実際のテンプレート例

これまでの内容を踏まえた、実際のテンプレート例です。

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 学習用のVPCテンプレート

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
    Description: VPCのID
    Value: !Ref MyVPC

  SubnetId:
    Description: SubnetのID
    Value: !Ref MySubnet
```

## 次の章へ

テンプレートの基本構造が理解できたら、[第3章 リソース定義の書き方・読み方](../chapter03/README.md)に進んでください。

Resources セクションの詳細な書き方を学びます。
