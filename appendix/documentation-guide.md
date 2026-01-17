# 付録B：公式ドキュメントの探し方

CloudFormation を使いこなすには、AWS 公式ドキュメントを効率的に読む力が重要です。

## 公式ドキュメントの構成

### 1. AWS CloudFormation ユーザーガイド

URL: https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/Welcome.html

**内容**：
- CloudFormation の基本概念
- テンプレートの書き方
- スタックの管理方法
- ベストプラクティス

**こんな時に参照**：
- CloudFormation の基本的な使い方を知りたい
- スタックのライフサイクルについて理解したい
- ChangeSet の使い方を知りたい

### 2. リソースタイプリファレンス

URL: https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html

**内容**：
- すべての AWS リソースの定義方法
- 各プロパティの詳細
- 使用例

**こんな時に参照**：
- 新しいリソースを定義したい
- プロパティの詳細を確認したい
- サンプルコードを見たい

### 3. 組み込み関数リファレンス

URL: https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html

**内容**：
- Ref、Sub、Join などの組み込み関数の使い方
- 各関数の構文と例

**こんな時に参照**：
- 文字列の結合方法を知りたい
- リソース間の参照方法を確認したい

## リソースタイプの探し方

### ステップ1：リソースタイプリファレンスを開く

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html

### ステップ2：サービスごとに分類されているので、該当するサービスを探す

例：VPC を作成したい場合
- 「Amazon EC2」のセクションを探す
- `AWS::EC2::VPC` を見つける

### ステップ3：リソースのページを開く

`AWS::EC2::VPC` のリンクをクリックすると、以下の情報が記載されています。

1. **Syntax**：基本的な書き方
2. **Properties**：各プロパティの詳細
3. **Return values**：Ref や GetAtt で取得できる値
4. **Examples**：実際の使用例

## リソースのページの読み方

### 1. Syntax（構文）

テンプレートでの基本的な書き方が記載されています。

例：AWS::EC2::VPC

```yaml
Type: AWS::EC2::VPC
Properties:
  CidrBlock: String
  EnableDnsHostnames: Boolean
  EnableDnsSupport: Boolean
  InstanceTenancy: String
  Tags:
    - Tag
```

### 2. Properties（プロパティ）

各プロパティの詳細が記載されています。

| プロパティ | 説明 | 必須 | 型 | Update requires |
|---|---|---|---|---|
| CidrBlock | VPC の CIDR ブロック | Yes | String | Replacement |
| EnableDnsHostnames | DNS ホスト名を有効化 | No | Boolean | No interruption |

#### Update requires の意味

- **No interruption**：ダウンタイムなしで更新可能
- **Some interruptions**：一時的な中断を伴う更新
- **Replacement**：リソースを置き換える（削除→再作成）

### 3. Return values（戻り値）

Ref や GetAtt で取得できる値が記載されています。

#### Ref を使った場合

`!Ref MyVPC` で何が返るかが記載されています。

例：VPC の場合、VPC ID（`vpc-xxx`）が返ります。

#### Fn::GetAtt を使った場合

`!GetAtt MyVPC.属性名` で取得できる属性が一覧で記載されています。

例：
- `!GetAtt MyVPC.CidrBlock` → CIDR ブロック（`10.0.0.0/16`）
- `!GetAtt MyVPC.DefaultNetworkAcl` → デフォルトネットワーク ACL の ID

### 4. Examples（例）

実際の使用例が記載されています。

最も参考になる部分です。YAML 形式と JSON 形式の両方が記載されていることが多いです。

## よくある検索パターン

### パターン1：「○○ を CloudFormation で作成したい」

1. Google で検索：`CloudFormation [リソース名]`
   - 例：`CloudFormation VPC`

2. AWS 公式ドキュメントのリンクをクリック

3. Examples セクションを確認

### パターン2：「このプロパティは何を意味するのか？」

1. リソースのページを開く

2. Properties セクションで該当するプロパティを探す

3. プロパティの説明とリンク先を確認

### パターン3：「Ref で何が返るのか？」

1. リソースのページを開く

2. Return values セクションの「Ref」を確認

### パターン4：「GetAtt でどんな属性が取得できるのか？」

1. リソースのページを開く

2. Return values セクションの「Fn::GetAtt」を確認

## 英語ドキュメントと日本語ドキュメント

### 日本語ドキュメント

URL: `https://docs.aws.amazon.com/ja_jp/...`

**メリット**：
- 日本語で読める
- 理解しやすい

**デメリット**：
- 更新が英語版より遅れることがある
- 機械翻訳のため、わかりにくい箇所がある

### 英語ドキュメント

URL: `https://docs.aws.amazon.com/...`（`ja_jp` を除く）

**メリット**：
- 最新の情報
- 正確な表現

**デメリット**：
- 英語が苦手だと読みづらい

### 推奨する使い分け

- **基本的には日本語ドキュメントを使う**
- **日本語ドキュメントで理解できない場合は、英語ドキュメントを確認する**
- **最新の機能については、英語ドキュメントを確認する**

## Google 検索のコツ

### 検索キーワードの例

```
CloudFormation VPC
CloudFormation SecurityGroup ingress
CloudFormation Ref
CloudFormation GetAtt VPC
CloudFormation ImportValue
```

### サイト内検索

AWS 公式ドキュメント内のみを検索したい場合：

```
site:docs.aws.amazon.com CloudFormation VPC
```

## よくアクセスするページをブックマークする

以下のページは、頻繁にアクセスすることになるので、ブックマークしておくと便利です。

- [AWS CloudFormation ユーザーガイド](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/Welcome.html)
- [リソースタイプリファレンス](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)
- [組み込み関数リファレンス](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html)
- [AWS::EC2::VPC](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpc.html)
- [AWS::EC2::Subnet](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet.html)
- [AWS::EC2::SecurityGroup](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-securitygroup.html)

---

**参考**：公式ドキュメントは定期的に更新されるため、常に最新の情報を確認するようにしましょう。
