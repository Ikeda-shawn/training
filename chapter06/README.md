# 第6章 スタックのライフサイクル

## この章で学ぶこと

CloudFormationでAWSリソースを管理するとき「スタック」という単位でリソースの作成・更新・削除を行います。この章ではスタックがどのような段階を経て作られ、更新され、削除されるのかその一連の流れを学びます。

この章を理解することで、以下のことができるようになります：

- スタックの作成・更新・削除の手順を理解し、実行できる
- 作成や更新が失敗したときに何が起きるかを理解し、適切に対処できる
- ChangeSet（変更セット）を使って、更新内容を事前に確認できる
- DeletionPolicyを使って、スタック削除時のリソースの扱いを制御できる

## スタックのライフサイクルとは

CloudFormationスタックには以下のような基本的なライフサイクルがあります。

```
作成（CREATE） → 更新（UPDATE） → 更新（UPDATE） → ... → 削除（DELETE）
```

各段階でCloudFormationがどのように動作するかを理解することで、トラブルが発生したときに原因を特定しやすくなります。

## スタック作成（CREATE）

### AWS CLIでスタックを作成する

スタックを作成するには、`create-stack`コマンドを使います。

```bash
# 第4章で学んだParametersを使ったスタック作成例
aws cloudformation create-stack \
  --stack-name my-network-stack \
  --template-body file://network-stack.yaml \
  --parameters ParameterKey=EnvironmentName,ParameterValue=dev
```

**補足**：AWSマネジメントコンソールからもスタックの作成・更新・削除は可能です。コンソールは視覚的でわかりやすいというメリットがありますが、本コンテンツではCLIを使った方法を中心に説明します。実務ではCLIを使った自動化やCI/CDパイプラインへの組み込みが主流であり、コマンドとして記録・共有しやすく再現性が高いためです。

### スタック作成時の処理の流れ

CloudFormationがスタックを作成するとき内部では以下の処理が順番に行われます。

#### ステップ1：テンプレートの検証

まずCloudFormationは提供されたテンプレートが正しいかどうかをチェックします。

- **構文チェック**：YAML（またはJSON）として正しく書かれているか
- **リソース定義の妥当性チェック**：各リソースのTypeやPropertiesが正しく定義されているか

もしテンプレートに問題があれば、この段階でエラーが返されリソースの作成は開始されません。

#### ステップ2：リソースの作成開始

テンプレートが正しければCloudFormationはリソースの作成を開始します。

このときCloudFormationは**依存関係を自動的に解析**して適切な順序でリソースを作成します。

例えば、以下のようなリソースがテンプレートに定義されている場合：

```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

  MySubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC  # VPCを参照している
      CidrBlock: 10.0.1.0/24

  MyEC2:
    Type: AWS::EC2::Instance
    Properties:
      SubnetId: !Ref MySubnet  # Subnetを参照している
      InstanceType: t2.micro
      ImageId: ami-0123456789abcdef0
```

CloudFormationは依存関係を解析し以下の順序でリソースを作成します：

```
1. MyVPC（他のリソースに依存していないので最初に作成）
      ↓
2. MySubnet（MyVPCが必要なので、VPCの後に作成）
      ↓
3. MyEC2（MySubnetが必要なので、Subnetの後に作成）
```

この依存関係の解析は`!Ref`や`!GetAtt`などの組み込み関数を見てCloudFormationが自動的に判断します。

#### ステップ3：ステータスの遷移

スタック作成中スタックのステータス（状態）は以下のように遷移します。

スタック作成中、CloudFormationはリソースを順次作成していき、その進捗状況をステータスとして報告します：

| ステータス | 意味 |
|---|---|
| `CREATE_IN_PROGRESS` | リソースを作成中です |
| `CREATE_COMPLETE` | すべてのリソースの作成が正常に完了しました |
| `CREATE_FAILED` | 作成中にエラーが発生しました（この後ロールバックが始まります） |
| `ROLLBACK_IN_PROGRESS` | エラーが発生したため、作成したリソースを削除しています |
| `ROLLBACK_COMPLETE` | ロールバックが完了しました（スタックは空の状態です） |

スタックのステータスは以下のコマンドで確認できます：

```bash
aws cloudformation describe-stacks --stack-name my-stack
```

### 作成失敗時の挙動（自動ロールバック）

スタック作成中にいずれかのリソースでエラーが発生した場合、CloudFormationは**自動的にロールバック**を行います。

ロールバックとはそれまでに作成したリソースをすべて削除してスタック作成前の状態に戻すことです。

#### 具体例：3つのリソースを作成中に3番目で失敗した場合

```
【作成フェーズ】
VPC作成開始 → VPC作成成功
      ↓
Subnet作成開始 → Subnet作成成功
      ↓
EC2作成開始 → EC2作成失敗（例：指定したAMIが存在しない）
      ↓

【ロールバックフェーズ】（作成と逆順で削除される）
      ↓
EC2削除（失敗したが途中まで作成されていれば削除）
      ↓
Subnet削除
      ↓
VPC削除
      ↓
ロールバック完了（ROLLBACK_COMPLETE）
```

#### ロールバック後のスタックの状態

ロールバックが完了するとスタックは`ROLLBACK_COMPLETE`というステータスになります。

**重要**：`ROLLBACK_COMPLETE`状態のスタックはそのまま再利用することができません。同じ名前でスタックを再作成したい場合はまずこのスタックを削除する必要があります。

```bash
# ロールバック完了状態のスタックを削除
aws cloudformation delete-stack --stack-name my-network-stack

# 削除完了を待つ
aws cloudformation wait stack-delete-complete --stack-name my-network-stack

# テンプレートを修正してから再度作成
aws cloudformation create-stack \
  --stack-name my-network-stack \
  --template-body file://network-stack.yaml \
  --parameters ParameterKey=EnvironmentName,ParameterValue=dev
```

## スタック更新（UPDATE）

### スタック更新の基本

既存のスタックに対してテンプレートやパラメータの変更を行う際には更新を実施します。

更新では変更があったリソースだけが処理され、変更がないリソースには何も行われません。

### AWS CLIでスタックを更新する

スタックを更新するには`update-stack`コマンドを使います。

```bash
# 第4章で学んだParametersの値を変更してスタックを更新
aws cloudformation update-stack \
  --stack-name my-network-stack \
  --template-body file://network-stack.yaml \
  --parameters ParameterKey=EnvironmentName,ParameterValue=stg
```

このコマンドの構文は`create-stack`とほぼ同じです。違いは、既存のスタックに対して変更を適用する点です。

上記の例では、第4章で学んだ`EnvironmentName`パラメータを`dev`から`stg`に変更しています。この変更により、リソースのタグなどが更新されます。

### 更新時のリソースの扱い（Update requires）

リソースのプロパティを変更したとき、CloudFormationがどのように更新を行うかは変更するプロパティによって異なります。

これは**Update requires**と呼ばれ、AWS公式ドキュメントの各リソースのプロパティ説明に記載されています。

#### 更新方法の種類

| 更新方法 | 説明 | 影響 | 例 |
|---|---|---|---|
| **No interruption** | ダウンタイムなしで更新されます | サービスに影響なし | EC2インスタンスのTags変更 |
| **Some interruption** | 一時的な中断を伴って更新されます | 一時的にサービスが停止する可能性あり | EC2インスタンスのインスタンスタイプ変更 |
| **Replacement** | リソースが新しく作り直されます | 古いリソースは削除され、新しいリソースが作成される | EC2インスタンスのAMI変更 |

#### Replacementが発生する場合の注意点

**Replacement**（置き換え）が発生するとリソースの物理ID（実際のAWSリソースの識別子）が変わります。

例えば、EC2インスタンスの場合：
- 置き換え前：`i-0123456789abcdef0`（古いインスタンス）
- 置き換え後：`i-0fedcba9876543210`（新しいインスタンス）

元のリソースが削除され新しいリソースが作成されることを意味します。そのため以下のような影響があります：

- インスタンス内のデータが失われる（EBSボリュームも含む場合がある）
- Elastic IPを関連付けていない場合、IPアドレスが変わる
- 他のサービスからそのリソースを参照している場合、参照先が変わる

#### Update requiresの確認方法

各プロパティのUpdate requiresはAWS公式ドキュメントで確認できます。

例えば`AWS::EC2::Instance`リソースの場合：

1. AWS CloudFormation ドキュメントで「AWS::EC2::Instance」を検索
2. 各プロパティの説明に「Update requires」が記載されています

例：`InstanceType`プロパティの場合
- **Update requires**: Some interruptions

これはインスタンスタイプを変更するとインスタンスが一時的に停止することを意味します。

---

## ChangeSet（変更セット）

### ChangeSetとは

ChangeSet（変更セット）はスタックを更新する前に**変更内容をプレビュー**できる機能です。

通常の`update-stack`コマンドではコマンドを実行すると即座に更新が開始されます。しかし、ChangeSetを使うと以下のような流れで安全に更新を行えます：

```
1. ChangeSetを作成（この時点では何も変更されない）
      ↓
2. ChangeSetの内容を確認（どのリソースがどう変わるかを確認）
      ↓
3. 問題なければChangeSetを実行（実際の更新が始まる）
   問題があればChangeSetを削除（何も変更されない）
```

### なぜChangeSetを使うのか

特に本番環境では意図しない変更を防ぐためにChangeSetの使用が強く推奨されます。

例えば以下のような事故を防げます：

- 「タグを追加しただけのつもりが、インスタンスが置き換えられてしまった」
- 「パラメータを1つ変えただけで、複数のリソースが影響を受けてしまった」

ChangeSetを使えば実際に更新を実行する前にどのリソースが追加・変更・削除されるかを確認できます。

### ChangeSetの使い方

#### ステップ1：ChangeSetを作成する

```bash
# ChangeSetを作成（この時点では何も変更されない）
aws cloudformation create-change-set \
  --stack-name my-network-stack \
  --change-set-name update-to-stg \
  --template-body file://network-stack.yaml \
  --parameters ParameterKey=EnvironmentName,ParameterValue=stg
```

このコマンドを実行するとCloudFormationは現在のスタックの状態と新しいテンプレートを比較し変更内容を計算します。**この時点では、実際のリソースには何も変更は加えられません。**

各オプションの説明：

| オプション | 説明 |
|---|---|
| `--stack-name` | 更新対象のスタック名 |
| `--change-set-name` | 作成するChangeSetの名前（任意の名前を付けられます） |
| `--template-body` | 新しいテンプレートファイル |

#### ステップ2：ChangeSetの内容を確認する

作成したChangeSetの内容を確認するには以下のコマンドを使います：

```bash
aws cloudformation describe-change-set \
  --stack-name my-network-stack \
  --change-set-name update-to-stg
```

出力例：

```json
{
  "ChangeSetName": "my-changeset",
  "StackName": "my-stack",
  "Status": "CREATE_COMPLETE",
  "Changes": [
    {
      "Type": "Resource",
      "ResourceChange": {
        "Action": "Modify",
        "LogicalResourceId": "MyEC2",
        "PhysicalResourceId": "i-0123456789abcdef0",
        "ResourceType": "AWS::EC2::Instance",
        "Replacement": "True",
        "Details": [...]
      }
    }
  ]
}
```

この出力の重要な部分を説明します：

| フィールド | 説明 |
|---|---|
| `Action` | リソースに対する操作。`Add`（追加）、`Modify`（変更）、`Remove`（削除）のいずれか |
| `LogicalResourceId` | テンプレート内でのリソース名 |
| `PhysicalResourceId` | 実際のAWSリソースID（既存リソースの場合） |
| `ResourceType` | リソースの種類 |
| `Replacement` | `True`の場合、リソースが置き換えられる（古いリソースが削除され、新しいリソースが作成される） |

**特に注意すべき点**：`Replacement`が`True`になっている場合はそのリソースが完全に作り直されることを意味します。データが失われる可能性があるため慎重に確認してください。

#### ステップ3：ChangeSetを実行する（または削除する）

ChangeSetの内容を確認して問題なければ実行します：

```bash
aws cloudformation execute-change-set \
  --stack-name my-network-stack \
  --change-set-name update-to-stg
```

このコマンドを実行すると実際のスタック更新が始まります。

もしChangeSetの内容に問題があり、更新を中止したい場合はChangeSetを削除します：

```bash
aws cloudformation delete-change-set \
  --stack-name my-network-stack \
  --change-set-name update-to-stg
```

ChangeSetを削除してもスタックには何も影響しません。

### ChangeSetを使うメリットまとめ

1. **変更内容を事前確認できる**
   - どのリソースが追加・変更・削除されるかがわかります
   - リソースが置き換えられるか（Replacement）どうかがわかります

2. **意図しない変更を防げる**
   - 実行前に確認できるため誤操作を防止できます
   - 想定外の変更があれば実行前に気づけます

3. **本番環境での安全性が向上する**
   - 本番環境では直接`update-stack`を使うのではなく、ChangeSetを使うことを強く推奨します

## 更新失敗時の挙動

### 更新失敗時の自動ロールバック

スタック更新中にエラーが発生した場合CloudFormationは**以前の状態にロールバック**します。

これはスタック作成時のロールバックとは少し異なります。作成時は「すべて削除して空の状態に戻す」のに対し更新時は「更新前の正常な状態に戻す」という動作になります。

### 更新失敗時のステータス遷移

```
更新前の状態（UPDATE_COMPLETE）
      ↓
更新開始（UPDATE_IN_PROGRESS）
      ↓
リソース変更中にエラー発生
      ↓
ロールバック開始（UPDATE_ROLLBACK_IN_PROGRESS）
      ↓
更新前の状態に戻る（UPDATE_ROLLBACK_COMPLETE）
```

`UPDATE_ROLLBACK_COMPLETE`になるとスタックは更新前の正常な状態に戻っています。

### ロールバック後の対処方法

更新が失敗してロールバックが完了したら、以下の手順で対処します。

#### 1. エラー原因を確認する

まず、何が原因で更新が失敗したのかを確認します。スタックイベントを見ると、エラーの詳細が分かります。

```bash
aws cloudformation describe-stack-events --stack-name my-network-stack
```

出力には、各リソースの作成・更新・削除のイベントが時系列で表示されます。`FAILED`と表示されているイベントを探し、`ResourceStatusReason`（失敗理由）を確認してください。

#### 2. テンプレートまたはパラメータを修正する

エラーの原因に応じて、テンプレートやパラメータを修正します。

よくあるエラーの例：
- 存在しないAMI IDを指定した
- パラメータの値が制約に違反している
- IAM権限が不足している
- リソースの制限（クォータ）に達した

#### 3. 再度更新を実行する

修正が完了したら、再度`update-stack`コマンド（またはChangeSet）で更新を実行します。

---

## スタック削除（DELETE）

### スタック削除の基本

スタックを削除すると、そのスタックで管理されているリソースも一緒に削除されます。

これはCloudFormationの大きなメリットの一つで、個別にリソースを削除する必要がなく、スタックを削除するだけで関連するすべてのリソースがクリーンアップされます。

### AWS CLIでスタックを削除する

```bash
aws cloudformation delete-stack --stack-name my-network-stack
```

このコマンドは即座に完了しますが、実際のリソースの削除はバックグラウンドで行われます。

削除完了を待つ場合は、以下のコマンドを使います：

```bash
aws cloudformation wait stack-delete-complete --stack-name my-network-stack
```

### 削除時の処理の流れ

#### ステップ1：リソースの削除開始

CloudFormationは、作成時とは**逆の順序**でリソースを削除します。

なぜ逆順かというと、依存関係があるためです。例えば、サブネット内にEC2インスタンスがある場合、先にサブネットを削除しようとするとエラーになります。

```
作成時：VPC → Subnet → EC2
削除時：EC2 → Subnet → VPC
```

#### ステップ2：ステータスの遷移

| ステータス | 説明 |
|---|---|
| `DELETE_IN_PROGRESS` | リソースを削除中です |
| `DELETE_COMPLETE` | すべてのリソースの削除が完了しました（スタック自体も消えます） |
| `DELETE_FAILED` | 削除中にエラーが発生しました |

**注意**：`DELETE_COMPLETE`になると、スタック自体がAWSから消えます。`describe-stacks`コマンドで確認しても、スタックは表示されなくなります。

### 削除が失敗するケース

スタック削除は必ずしも成功するとは限りません。以下のようなケースで失敗することがあります。

#### ケース1：他のスタックから参照されている（Export/ImportValueの依存関係）

第5章で学んだように、スタックの`Outputs`に`Export`を設定し、他のスタックが`ImportValue`で参照している場合、削除できません。

**具体例**：

第5章の例で、`network-stack`が以下のようにVPC IDをエクスポートしているとします：

```yaml
# network-stack.yaml の Outputs
Outputs:
  VPCId:
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${EnvironmentName}-VPCId'  # 例: dev-VPCId
```

そして、`application-stack`がそれを参照しています：

```yaml
# application-stack.yaml の Resources
Resources:
  MySecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !ImportValue
        Fn::Sub: '${EnvironmentName}-VPCId'  # dev-VPCId を参照
```

この状態で`network-stack`を削除しようとすると、以下のようなエラーが発生します：

```
Export dev-VPCId cannot be deleted as it is in use by application-stack
```

**対処法**：

参照している側のスタック（`application-stack`）を先に削除（または参照を解除）してから、元のスタック（`network-stack`）を削除します。

```bash
# 1. 参照している側のスタック（application-stack）を先に削除
aws cloudformation delete-stack --stack-name my-application-stack
aws cloudformation wait stack-delete-complete --stack-name my-application-stack

# 2. 元のスタック（network-stack）を削除
aws cloudformation delete-stack --stack-name my-network-stack
aws cloudformation wait stack-delete-complete --stack-name my-network-stack
```

このように、**スタックの削除順序は作成順序の逆**になります。第5章で説明したスタック間の依存関係を理解しておくことが重要です。

#### ケース2：S3バケットが空でない

CloudFormationは、**空でないS3バケットを削除できません**。これはデータを誤って削除しないための安全機能です。

エラー例：
```
The bucket you tried to delete is not empty
```

**対処法**：

S3バケットを空にしてから、スタックを削除します。

```bash
# 1. S3バケットの中身をすべて削除
aws s3 rm s3://my-bucket --recursive

# 2. バージョニングが有効な場合は、削除マーカーも削除
aws s3api delete-objects --bucket my-bucket \
  --delete "$(aws s3api list-object-versions --bucket my-bucket --query='{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json)"

# 3. スタックを削除
aws cloudformation delete-stack --stack-name my-network-stack
```

#### ケース3：リソースが手動で削除されている

CloudFormationが管理しているリソースを、CloudFormation以外の方法（マネジメントコンソールやCLIで直接）で削除してしまった場合、スタック削除時にエラーになることがあります。

**対処法**：

`--retain-resources`オプションを使って、問題のリソースをスキップして削除を続行します。

```bash
# 論理ID「MyEC2」を持つリソースをスキップして削除を続行
aws cloudformation delete-stack \
  --stack-name my-network-stack \
  --retain-resources MyEC2
```

このオプションで指定したリソース（この例では`MyEC2`）は、CloudFormationの管理から外されますが、削除処理自体は続行されます。

---

## DeletionPolicy（削除ポリシー）

### DeletionPolicyとは

通常、スタックを削除すると、スタック内のリソースもすべて削除されます。しかし、特定のリソースについては「スタックを削除しても残しておきたい」「削除前にバックアップを取りたい」といったケースがあります。

そのような場合に使うのが**DeletionPolicy**（削除ポリシー）です。

DeletionPolicyをリソースに設定することで、スタック削除時の挙動を制御できます。

### DeletionPolicyの種類

#### Delete（デフォルト）

スタック削除時にリソースも削除されます。これがデフォルトの動作なので、明示的に書く必要はありません。

```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    DeletionPolicy: Delete  # デフォルトなので省略可能
    Properties:
      CidrBlock: 10.0.0.0/16
```

#### Retain

スタック削除時にリソースを**残します**。スタックは削除されますが、そのリソースはAWS上に残り続けます。

```yaml
Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain  # スタック削除後もバケットは残る
    Properties:
      BucketName: my-important-bucket
```

**使用例**：
- 重要なログが保存されているS3バケット
- 削除してはいけないデータが入っているリソース

**注意点**：
- Retainを指定したリソースは、スタック削除後は「CloudFormationの管理外」になります
- 後から手動で削除する必要があります
- 残したリソースのコストは引き続き発生します

#### Snapshot（一部のリソースのみ対応）

スタック削除時に**スナップショット（バックアップ）を作成してから**リソースを削除します。

この設定が使えるのは、スナップショット機能をサポートしている一部のリソースのみです：
- `AWS::RDS::DBInstance`（RDSデータベース）
- `AWS::RDS::DBCluster`（RDSクラスター）
- `AWS::EC2::Volume`（EBSボリューム）
- `AWS::ElastiCache::CacheCluster`（ElastiCacheクラスター）
- `AWS::Neptune::DBCluster`（Neptuneクラスター）
- `AWS::Redshift::Cluster`（Redshiftクラスター）

```yaml
Resources:
  MyRDS:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot  # 削除前に自動でスナップショットを作成
    Properties:
      DBInstanceIdentifier: my-db
      DBInstanceClass: db.t3.micro
      Engine: mysql
      MasterUsername: admin
      MasterUserPassword: password123
      AllocatedStorage: 20
```

**使用例**：
- 本番データベース（誤削除時にデータを復旧できるようにする）
- 重要なデータが保存されているEBSボリューム

**注意点**：

- スナップショット作成には時間がかかるため、削除処理も長くなります
- スナップショットの保存にはコストが発生します（手動で削除するまで残り続けます）

### DeletionPolicyの使用例と推奨設定

| リソースの種類 | 推奨設定 | 理由 |
|---|---|---|
| 本番データベース（RDS） | Snapshot | 誤削除時にスナップショットからデータを復旧できる |
| ログ保存用S3バケット | Retain | スタック削除後も監査ログなどを保持する必要がある |
| 開発環境のリソース | Delete（デフォルト） | 不要になったらクリーンアップしたい |

---

## スタックのステータス一覧

スタックの各ステータスとその意味を一覧にまとめます。

### 作成に関するステータス

| ステータス | 説明 |
|---|---|
| `CREATE_IN_PROGRESS` | スタックのリソースを作成中 |
| `CREATE_COMPLETE` | スタックの作成が正常に完了 |
| `CREATE_FAILED` | スタックの作成が失敗（この後ロールバックが始まる） |

### ロールバックに関するステータス

| ステータス | 説明 |
|---|---|
| `ROLLBACK_IN_PROGRESS` | 作成失敗によるロールバック中 |
| `ROLLBACK_COMPLETE` | ロールバックが完了（スタックは空の状態） |
| `ROLLBACK_FAILED` | ロールバックが失敗 |

### 更新に関するステータス

| ステータス | 説明 |
|---|---|
| `UPDATE_IN_PROGRESS` | スタックを更新中 |
| `UPDATE_COMPLETE` | スタックの更新が正常に完了 |
| `UPDATE_COMPLETE_CLEANUP_IN_PROGRESS` | 更新完了後、古いリソースをクリーンアップ中 |
| `UPDATE_FAILED` | スタックの更新が失敗 |
| `UPDATE_ROLLBACK_IN_PROGRESS` | 更新失敗によるロールバック中 |
| `UPDATE_ROLLBACK_COMPLETE` | 更新のロールバックが完了（更新前の状態に戻った） |
| `UPDATE_ROLLBACK_FAILED` | 更新のロールバックが失敗 |

### 削除に関するステータス

| ステータス | 説明 |
|---|---|
| `DELETE_IN_PROGRESS` | スタックを削除中 |
| `DELETE_COMPLETE` | スタックの削除が完了（スタック自体も消える） |
| `DELETE_FAILED` | スタックの削除が失敗 |

---

## この章のまとめ

この章では、CloudFormationスタックのライフサイクルについて学びました。

### 重要なポイント

1. **スタック作成（CREATE）**
   - CloudFormationは依存関係を自動解析し、適切な順序でリソースを作成する
   - 作成が失敗すると自動でロールバックされ、`ROLLBACK_COMPLETE`状態になる
   - `ROLLBACK_COMPLETE`状態のスタックは削除してから再作成する必要がある

2. **スタック更新（UPDATE）**
   - 変更があったリソースだけが処理される
   - Update requiresによって、更新方法（No interruption / Some interruption / Replacement）が異なる
   - 更新が失敗すると自動で以前の状態にロールバックされる

3. **ChangeSet（変更セット）**
   - 更新内容を事前にプレビューできる機能
   - 本番環境では、直接update-stackするのではなく、ChangeSetを使うことを推奨

4. **スタック削除（DELETE）**
   - リソースは作成時と逆順で削除される
   - 他のスタックから参照されている場合や、S3バケットが空でない場合は削除に失敗する

5. **DeletionPolicy**
   - Delete（デフォルト）：スタック削除時にリソースも削除
   - Retain：スタック削除時にリソースを残す
   - Snapshot：スタック削除時にスナップショットを作成してから削除

---

## 次の章へ

スタックのライフサイクルが理解できたら、[第7章 CloudFormationと運用上の注意点](../chapter07/README.md)に進んでください。

次の章では、運用時に注意すべき「ドリフト」（手動変更によるテンプレートと実環境の差分）などの問題について学びます。
