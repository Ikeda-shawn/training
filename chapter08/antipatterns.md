# 第8章 よくあるアンチパターン

本章では、CloudFormation テンプレートでよく見られる「避けるべき設計パターン」を紹介します。

## アンチパターン1：すべてを1テンプレートに詰め込む

### 問題

すべてのリソース（VPC、Subnet、EC2、RDS、S3 など）を1つのテンプレートに定義してしまう。

```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    # ...

  MySubnet1:
    Type: AWS::EC2::Subnet
    # ...

  MySubnet2:
    Type: AWS::EC2::Subnet
    # ...

  # ... 100 個のリソース定義が続く

  MyRDS:
    Type: AWS::RDS::DBInstance
    # ...

  MyS3Bucket:
    Type: AWS::S3::Bucket
    # ...
```

### なぜ問題か

1. **変更の影響範囲が大きい**
   - アプリケーションの変更で、ネットワーク層も一緒に更新される
   - 意図しないリソースが変更されるリスクが高い

2. **デプロイに時間がかかる**
   - リソース数が多いと、スタックの作成・更新に時間がかかる

3. **テンプレートが読みにくい**
   - 1000 行を超えるテンプレートは、読むのも修正するのも大変

4. **並行作業ができない**
   - チームで同じテンプレートを編集すると、コンフリクトが発生しやすい

### 改善策

スタックを適切に分割する。

```
network-stack（ネットワーク層）
  ├─ VPC
  ├─ Subnet
  └─ RouteTable

application-stack（アプリケーション層）
  ├─ EC2
  ├─ RDS
  └─ SecurityGroup

storage-stack（ストレージ層）
  └─ S3 Bucket
```

## アンチパターン2：可読性を無視したテンプレート

### 問題

- コメントがない
- 論理 ID がわかりにくい
- インデントが乱れている

```yaml
Resources:
  R1:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
  R2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref R1
      CidrBlock: 10.0.1.0/24
```

### なぜ問題か

1. **他の人が読めない**
   - `R1` が何を意味するのかわからない

2. **自分でも後で読めない**
   - 半年後に見たときに、何をしているのかわからない

3. **メンテナンスが困難**
   - 修正したいリソースを探すのに時間がかかる

### 改善策

わかりやすい論理 ID とコメントを使う。

```yaml
Resources:
  # VPC の定義
  MainVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      Tags:
        - Key: Name
          Value: main-vpc

  # パブリックサブネット
  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MainVPC
      CidrBlock: 10.0.1.0/24
      Tags:
        - Key: Name
          Value: public-subnet
```

## アンチパターン3：Parameters を増やしすぎる

### 問題

あらゆる値をパラメータ化してしまう。

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
  SecurityGroupPort1:
    Type: Number
  SecurityGroupPort2:
    Type: Number
  SecurityGroupPort3:
    Type: Number
  # ... 30 個のパラメータ
```

### なぜ問題か

1. **デプロイ時に入力が大変**
   - 30 個のパラメータを毎回入力するのは現実的ではない

2. **テンプレートが読みにくい**
   - パラメータが多すぎて、何が何だかわからない

3. **柔軟性が高すぎて逆に使いにくい**
   - すべてをカスタマイズできる = デフォルトで動かない

### 改善策

本当に必要なパラメータだけに絞る。

```yaml
Parameters:
  # 環境ごとに変わる可能性が高いもののみパラメータ化
  EnvironmentName:
    Type: String
    Default: dev

  VPCCidrBlock:
    Type: String
    Default: 10.0.0.0/16

Resources:
  MainVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VPCCidrBlock

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MainVPC
      CidrBlock: 10.0.1.0/24  # 固定で問題ない値はハードコード

  MySecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443  # 固定で問題ない値はハードコード
          ToPort: 443
```

## アンチパターン4：Outputs を考慮しない設計

### 問題

他のスタックから参照される可能性があるリソースに、Outputs を設定していない。

```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

# Outputs がない！
```

### なぜ問題か

1. **他のスタックから参照できない**
   - VPC ID を他のスタックで使いたい場合、AWS コンソールで確認して手動でコピーする必要がある

2. **スタック間の連携ができない**
   - スタックを分割するメリットが失われる

### 改善策

他のスタックから参照される可能性があるリソースには、必ず Outputs を設定する。

```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

Outputs:
  VPCId:
    Description: VPC の ID
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${AWS::StackName}-VPCId'

  VPCCidrBlock:
    Description: VPC の CIDR ブロック
    Value: !GetAtt MyVPC.CidrBlock
    Export:
      Name: !Sub '${AWS::StackName}-VPCCidrBlock'
```

## アンチパターン5：CloudFormation に処理をさせようとする

### 問題

CloudFormation でアプリケーションのデプロイやデータベースの初期化などを行おうとする。

```yaml
Resources:
  MyEC2:
    Type: AWS::EC2::Instance
    Properties:
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          # 100 行のシェルスクリプト
          # アプリケーションのインストール
          # データベースの初期化
          # ログ設定
          # 監視設定
          # ...
```

### なぜ問題か

1. **CloudFormation の責務を超えている**
   - CloudFormation はインフラの構築が責務
   - アプリケーションのデプロイは別のツールで行うべき

2. **デバッグが困難**
   - UserData でエラーが発生しても、ログが見づらい
   - 失敗時にスタック全体がロールバックされる

3. **再利用性が低い**
   - アプリケーションの変更のたびにスタックを更新する必要がある

### 改善策

インフラの構築と、アプリケーションのデプロイを分離する。

**CloudFormation の役割**：インフラの構築のみ

```yaml
Resources:
  MyEC2:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0123456789abcdef0
      InstanceType: t2.micro
      SubnetId: !Ref MySubnet
      IamInstanceProfile: !Ref MyInstanceProfile
      Tags:
        - Key: Name
          Value: app-server
```

**アプリケーションのデプロイ**：Ansible、Chef、CodeDeploy などの構成管理ツールを使用

```bash
# インフラ構築
aws cloudformation create-stack --stack-name my-stack --template-body file://template.yaml

# アプリケーションデプロイ
ansible-playbook deploy.yml
```

## まとめ：良いテンプレートの条件

### 1. 適切な粒度で分割されている

- ライフサイクルごとにスタックを分割
- 1 スタックのリソース数は 20〜30 個程度が目安

### 2. 読みやすい

- わかりやすい論理 ID
- 適切なコメント
- 一貫したインデント

### 3. 必要最小限のパラメータ

- 環境ごとに変わる値のみパラメータ化
- デフォルト値を設定

### 4. Outputs で連携可能

- 他のスタックから参照される可能性があるリソースに Outputs を設定
- Export 名にスタック名を含める

### 5. CloudFormation の責務範囲内

- インフラの構築に特化
- アプリケーションのデプロイは別のツールで実施

## 次の章へ

アンチパターンが理解できたら、[第9章 演習（CloudFormation テンプレート作成）](ch09_exercises.md) に進んでください。

実際にテンプレートを作成する演習問題に取り組みます。
