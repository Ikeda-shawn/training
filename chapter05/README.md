# 第5章 Outputs とスタック連携の考え方

## Outputs の役割

`Outputs` セクションは、スタックが作成したリソースの情報を**外部に公開**するための仕組みです。

### Outputs を使わない場合

リソースの ID を知りたい場合、AWS コンソールや AWS CLI で確認する必要があります。

```bash
# VPC ID を確認
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=my-vpc"
```

### Outputs を使う場合

スタックの出力として、必要な情報を取得できます。

```bash
# スタックの Outputs を確認
aws cloudformation describe-stacks --stack-name my-stack
```

## Outputs の基本的な書き方

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
    Description: 作成された VPC の ID
    Value: !Ref MyVPC

  VPCCidrBlock:
    Description: VPC の CIDR ブロック
    Value: !GetAtt MyVPC.CidrBlock
```

## 他スタックから参照される値とは

Outputs には、`Export` を追加することで、**他のスタックから参照できる**ようになります。

### Export を使わない Outputs

```yaml
Outputs:
  VPCId:
    Description: VPC の ID
    Value: !Ref MyVPC
```

この Outputs は、スタックの情報として表示されますが、他のスタックからは参照できません。

### Export を使う Outputs

```yaml
Outputs:
  VPCId:
    Description: VPC の ID
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${AWS::StackName}-VPCId'
```

`Export` の `Name` は、AWS アカウント内で**一意**である必要があります。

`${AWS::StackName}` を使うことで、スタック名をプレフィックスにし、一意性を確保できます。

## Export / ImportValue の基本

### Export する側（ネットワークスタック）

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
    Description: VPC の ID
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${AWS::StackName}-VPCId'

  SubnetId:
    Description: Subnet の ID
    Value: !Ref MySubnet
    Export:
      Name: !Sub '${AWS::StackName}-SubnetId'
```

### ImportValue する側（アプリケーションスタック）

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

### ImportValue の注意点

- **Export している値を変更・削除すると、それを ImportValue しているスタックに影響が出ます**
- Export している値を変更する場合は、先に ImportValue しているスタックを削除または更新する必要があります

## スタック分割の考え方（概要レベル）

すべてのリソースを1つのスタックに詰め込むのではなく、適切に分割することが推奨されます。

### 分割の基準

#### 1. ライフサイクルが異なるリソース

- **ネットワーク層**：VPC、Subnet（めったに変更しない）
- **アプリケーション層**：EC2、RDS（頻繁に変更する）

#### 2. 責任範囲が異なるリソース

- **インフラチーム**：VPC、Subnet、SecurityGroup
- **アプリケーションチーム**：EC2、Lambda、API Gateway

#### 3. 再利用したいリソース

- **共通ネットワークスタック**：複数のアプリケーションで共有する VPC
- **個別アプリケーションスタック**：アプリケーション固有のリソース

### スタック分割の例

```
network-stack（ネットワーク層）
  ├─ VPC
  ├─ Subnet（Public）
  ├─ Subnet（Private）
  ├─ InternetGateway
  └─ RouteTable

application-stack（アプリケーション層）
  ├─ SecurityGroup
  ├─ EC2 Instance
  └─ RDS Instance
```

### スタック分割のメリット

1. **変更の影響範囲を限定できる**
   - アプリケーションの変更で、ネットワークに影響を与えない

2. **再デプロイが高速**
   - 小さなスタックの方が、デプロイが速い

3. **管理がしやすい**
   - スタックごとに責任範囲が明確になる

### スタック分割のデメリット

1. **複雑性が増す**
   - スタック間の依存関係を管理する必要がある

2. **削除の順序に注意が必要**
   - ImportValue している側を先に削除する必要がある

## 密結合を避けるための設計視点

### 密結合とは

スタック間の依存関係が強すぎて、一方を変更すると他方も変更が必要になる状態です。

#### 悪い例：密結合

```yaml
# ネットワークスタック
Outputs:
  PublicSubnet1Id:
    Value: !Ref PublicSubnet1
    Export:
      Name: PublicSubnet1Id  # スタック名を含まない

  PublicSubnet2Id:
    Value: !Ref PublicSubnet2
    Export:
      Name: PublicSubnet2Id  # スタック名を含まない
```

```yaml
# アプリケーションスタック
Resources:
  MyEC2:
    Type: AWS::EC2::Instance
    Properties:
      SubnetId: !ImportValue PublicSubnet1Id  # ハードコード
```

この設計では、Export 名が固定されているため、複数の環境で同じテンプレートを使えません。

#### 良い例：疎結合

```yaml
# ネットワークスタック
Outputs:
  PublicSubnet1Id:
    Value: !Ref PublicSubnet1
    Export:
      Name: !Sub '${AWS::StackName}-PublicSubnet1Id'  # スタック名を含む
```

```yaml
# アプリケーションスタック
Parameters:
  NetworkStackName:
    Type: String
    Description: ネットワークスタック名

Resources:
  MyEC2:
    Type: AWS::EC2::Instance
    Properties:
      SubnetId: !ImportValue
        Fn::Sub: '${NetworkStackName}-PublicSubnet1Id'  # パラメータで柔軟に
```

この設計では、パラメータで柔軟にスタックを指定できます。

## 実践例：ネットワークスタックとアプリケーションスタックの分離

### network-stack.yaml

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: ネットワークリソーススタック

Parameters:
  EnvironmentName:
    Type: String
    Default: dev

Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-vpc'

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.1.0/24
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-public-subnet'

Outputs:
  VPCId:
    Description: VPC ID
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${AWS::StackName}-VPCId'

  PublicSubnetId:
    Description: Public Subnet ID
    Value: !Ref PublicSubnet
    Export:
      Name: !Sub '${AWS::StackName}-PublicSubnetId'
```

### application-stack.yaml

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: アプリケーションリソーススタック

Parameters:
  NetworkStackName:
    Type: String
    Description: ネットワークスタック名

Resources:
  MySecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: App security group
      VpcId: !ImportValue
        Fn::Sub: '${NetworkStackName}-VPCId'
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

Outputs とスタック連携が理解できたら、[第6章 スタックのライフサイクル](../chapter06/README.md) に進んでください。

スタックの作成・更新・削除の流れを学びます。
