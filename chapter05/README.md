# 第5章 Outputsとスタック連携の考え方

## Outputsの役割

`Outputs`セクションはスタックによって作成されたリソースの情報を**他のスタックで再利用**するために出力する仕組みです。

### Outputsを使わない場合

リソースのID(インスタンスIDなど)を知りたい場合、コンソールやAWS CLIで確認する必要があります。

```bash
# VPC ID を確認
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=my-vpc"
```

これらを取得して別のスタックを作成する都度値を渡すのは非効率的です。

そこで使うのが`Outputs`セクションです。

## Outputsの基本的な書き方

```yaml
Outputs:
  出力名:
    Description: 説明
    Value: 値
```

### 具体例

```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

Outputs:
  VPCId:
    Description: 作成されたVPCのID
    Value: !Ref MyVPC

  VPCCidrBlock:
    Description: VPCのCIDRブロック
    Value: !GetAtt MyVPC.CidrBlock
```

## 他スタックから参照される値とは

Outputsは、`Export`の有無で他スタックから参照できるかどうかが変わります。

### Exportを使わないOutputs

```yaml
Outputs:
  VPCId:
    Description: VPCのID
    Value: !Ref MyVPC
```

このOutputsはスタックの情報として表示されますが、他のスタックからは参照できません。

他スタックから参照させるためには以下の`Export`を利用します。

### Exportを使うOutputs

```yaml
Outputs:
  VPCId:
    Description: VPCのID
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${AWS::StackName}-VPCId'
```

`Export`の`Name`は同一リージョン・アカウント内で**一意**である必要があります。

## Export/ImportValueの基本

### Exportする側（ネットワークスタック）

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: ネットワークリソース

Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

  MySubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.1.0/24

Outputs:
  VPCId:
    Description: VPCのID
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${AWS::StackName}-VPCId'

  SubnetId:
    Description: SubnetのID
    Value: !Ref MySubnet
    Export:
      Name: !Sub '${AWS::StackName}-SubnetId'
```

### ImportValueする側（アプリケーションスタック）

参照する側ではImportValueという組み込み関数を使って参照していきます。

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: アプリケーションリソース

Parameters:
  NetworkStackName:
    Type: String
    Default: network-stack
    Description: ネットワークスタックの名前

Resources:
  MySecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Application security group
      VpcId: !ImportValue
        Fn::Sub: '${NetworkStackName}-VPCId'
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0123456789abcdef0
      InstanceType: t2.micro
      SubnetId: !ImportValue
        Fn::Sub: '${NetworkStackName}-SubnetId'
      SecurityGroupIds:
        - !Ref MySecurityGroup
```

### ImportValueの注意点

- **Exportしている値を変更・削除するとそれをImportValueしているスタックに影響が出ます**
- Exportしている値を変更する場合、先にImportValueしているスタックを削除または更新する必要があります

上記の注意点より環境作成後にスタックを更新するためには依存関係に注意する必要があります。

## スタック分割の考え方（概要レベル）

ImportValueの注意点の箇所でも記載をしましたが、CFnはスタックの分割レベルがそのまま依存関係につながります。

そのため本項目の考え方はあくまでも参考ですが、注意して理解するようにして下さい。

### 分割の基準

分割の基準は様々ですが、一般的に以下の基準に従って分割されることが多いです。

#### 1. ライフサイクルが異なるリソース

- **ネットワーク層**：VPC、Subnet（一度作成したらめったに変更しない）
- **アプリケーション層**：EC2、RDS（頻繁に変更する）

#### 2. 責任範囲が異なるリソース

- **インフラチーム**：VPC、Subnet、SecurityGroup
- **アプリケーションチーム**：EC2、Lambda、API Gateway

#### 3. 再利用したいリソース

- **共通ネットワークスタック**：複数のアプリケーションで共有するVPC
- **個別アプリケーションスタック**：アプリケーション固有のリソース

### スタック分割の例

```
network-stack（ネットワーク層）
  ├─ VPC
  ├─ Subnet（Public/Private）
  └─ RouteTable

application-stack（アプリケーション層）
  ├─ SecurityGroup
  ├─ EC2 Instance
  └─ RDS Instance
```

### スタック分割のメリット

1. **変更の影響範囲を限定できる**
   - アプリケーションの変更でネットワークに影響を与えることがなくなります

2. **再デプロイが高速**
   - 小さなスタックの方がデプロイが早いです

3. **管理がしやすい**
   - スタックごとに責任範囲が明確になります

### スタック分割のデメリット

1. **複雑性が増す**
   - スタック間の依存関係を管理する必要があります

2. **削除の順序に注意が必要**
   - ImportValueしている側を先に削除する必要があります

## 再利用性を高めるExport/Import設計

### Export名の設計パターン

Export名は同一リージョン・アカウント内で一意である必要があります。

そのため、Export名の設計によって同じテンプレートを複数環境で使えるかどうかが決まります。

#### 推奨：命名規則を決めてExport名を固定する

```yaml
# ネットワークスタック
Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, stg, prod]

Outputs:
  PublicSubnet1Id:
    Value: !Ref PublicSubnet1
    Export:
      Name: !Sub '${Environment}-PublicSubnet1Id'  # 環境名を含める
```

```yaml
# アプリケーションスタック
Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, stg, prod]

Resources:
  MyEC2:
    Type: AWS::EC2::Instance
    Properties:
      SubnetId: !ImportValue
        Fn::Sub: '${Environment}-PublicSubnet1Id'  # 同じ環境名で参照
```

**この設計のメリット:**

- Export名が明確で分かりやすい（`dev-PublicSubnet1Id`、`stg-PublicSubnet1Id`、`prod-PublicSubnet1Id`）
- Import側もシンプルでスタック名を意識する必要がない
- 命名規則が統一されているためどのスタックがどの環境のリソースを参照しているか一目で分かる

**この方式が使えるケース:**

- 同一アカウント内に複数環境を構築する場合
- 環境ごとにAWSアカウントを分けている場合（各アカウント内でExport名は一意なので問題ない）

## 実践例：ネットワークスタックとアプリケーションスタックの分離

### network-stack.yaml

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: ネットワークリソーススタック

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, stg, prod]

Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-vpc'

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.1.0/24
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-subnet'

Outputs:
  VPCId:
    Description: VPC ID
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${Environment}-VPCId'

  PublicSubnetId:
    Description: Public Subnet ID
    Value: !Ref PublicSubnet
    Export:
      Name: !Sub '${Environment}-PublicSubnetId'
```

### application-stack.yaml
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: アプリケーションリソーススタック

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, stg, prod]

Resources:
  MySecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: App security group
      VpcId: !ImportValue
        Fn::Sub: '${Environment}-VPCId'
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

Outputs:
  SecurityGroupId:
    Description: SecurityGroup ID
    Value: !Ref MySecurityGroup
```

## 次の章へ

Outputsとスタック連携が理解できたら[第6章 スタックのライフサイクル](../chapter06/README.md) に進んでください。

スタックの作成・更新・削除の流れを学びます。
