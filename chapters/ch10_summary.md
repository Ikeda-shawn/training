# 第10章 まとめ

## 本コンテンツで身につくこと

本コンテンツを通じて、以下のスキルが身についたはずです。

### 1. CloudFormation の基本概念の理解

- テンプレート、スタック、リソースの関係
- 宣言的なインフラ定義の考え方
- CloudFormation の責務と限界

### 2. テンプレートの読み書き

- YAML 形式でのテンプレート記述
- リソース定義の基本構造
- 組み込み関数（Ref、Sub、GetAtt など）の使い方

### 3. パラメータとOutputs の設計

- 環境差分を吸収するパラメータ設計
- スタック間連携のための Outputs 設計
- Export / ImportValue の使い方

### 4. スタックのライフサイクル管理

- スタックの作成・更新・削除
- ChangeSet による事前確認
- ロールバックの仕組み

### 5. 運用上の注意点

- ドリフトの問題と対策
- 手動変更を避けるべき理由
- Drift Detection の活用

### 6. 良い設計と悪い設計の判断

- アンチパターンの理解
- 可読性を重視したテンプレート作成
- 適切なスタック分割

## あえて扱っていない内容とその理由

本コンテンツでは、以下の内容を**あえて扱っていません**。

### 1. 複雑な組み込み関数

**扱っていない機能**：
- `Fn::ForEach`（ループ処理）
- 複雑な `Conditions`（条件分岐）
- `Fn::Transform`（マクロ）

**理由**：
- 初学者にとって理解が難しい
- 可読性が大きく損なわれる
- まずはシンプルなテンプレートを読み書きできることが重要

### 2. ネストスタック

**扱っていない機能**：
- `AWS::CloudFormation::Stack`
- テンプレートのモジュール化

**理由**：
- 概念の理解に時間がかかる
- 基本的なスタック分割（Export / ImportValue）で十分対応可能

### 3. カスタムリソース

**扱っていない機能**：
- `AWS::CloudFormation::CustomResource`
- Lambda を使った独自のリソース定義

**理由**：
- 高度な内容であり、初学者には不要
- AWS が提供する標準リソースで大半のユースケースに対応可能

### 4. CloudFormation 以外の IaC ツール

**扱っていない内容**：
- AWS CDK（Cloud Development Kit）
- Terraform
- Pulumi

**理由**：
- 本コンテンツは CloudFormation に特化
- CloudFormation の基礎を理解していれば、他のツールへの応用が容易

## CloudFormation を使ううえでの基本的な心構え

### 1. テンプレートはコードである

- バージョン管理システム（Git）で管理する
- コードレビューを行う
- CI/CD パイプラインに組み込む

### 2. 手動変更は避ける

- CloudFormation で管理しているリソースは、CloudFormation 経由で変更する
- 緊急時に手動変更した場合は、必ずテンプレートに反映する

### 3. 可読性を重視する

- わかりやすい論理 ID
- 適切なコメント
- 他の人が読んでも理解できるテンプレートを目指す

### 4. 適切な粒度でスタックを分割する

- すべてを1つのスタックに詰め込まない
- ライフサイクルや責任範囲ごとに分割する

### 5. ChangeSet で変更を確認する

- 特に本番環境では、必ず ChangeSet で事前確認する
- 意図しない変更を防ぐ

## 次のステップ

本コンテンツで CloudFormation の基礎を学んだ後、以下のステップに進むことをおすすめします。

### ステップ1：実際に手を動かす

- 自分の AWS アカウントで、実際にスタックを作成・更新・削除してみる
- 演習問題に取り組む
- エラーが発生したら、ログを見て原因を調べる

### ステップ2：公式ドキュメントを読む

- AWS CloudFormation ユーザーガイド
  - https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/Welcome.html

- リソースタイプリファレンス
  - https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html

### ステップ3：実務で使ってみる

- 既存のインフラを CloudFormation で再構築してみる
- 新しいプロジェクトで、最初から CloudFormation を使う
- チームで運用ルールを決める

### ステップ4：より高度な内容を学ぶ

余裕があれば、以下の内容にも挑戦してみましょう。

#### AWS CDK（Cloud Development Kit）

プログラミング言語（TypeScript、Python など）で CloudFormation テンプレートを生成できるツールです。

- 複雑なロジックが書きやすい
- IDE の補完機能が使える
- CloudFormation の知識が前提となる

#### CloudFormation StackSets

複数のアカウントやリージョンに、同じスタックを一括デプロイできる機能です。

- マルチアカウント環境で便利
- AWS Organizations との連携

#### Infrastructure as Code のベストプラクティス

- テストの自動化（cfn-lint、TaskCat など）
- CI/CD パイプラインの構築
- セキュリティスキャン

## 最後に

CloudFormation は、AWS インフラを「コード」として管理するための強力なツールです。

本コンテンツで学んだ基礎をもとに、実際に手を動かしながら、徐々にスキルを磨いていってください。

**重要なのは、「完璧なテンプレートを書くこと」ではなく、「読みやすく、メンテナンスしやすいテンプレートを書くこと」です。**

最初はシンプルなテンプレートから始め、必要に応じて機能を追加していきましょう。

---

## 参考リンク

### 公式ドキュメント

- [AWS CloudFormation ユーザーガイド](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/Welcome.html)
- [リソースタイプリファレンス](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)
- [組み込み関数リファレンス](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html)

### ツール

- [cfn-lint](https://github.com/aws-cloudformation/cfn-lint)：CloudFormation テンプレートの静的解析ツール
- [TaskCat](https://github.com/aws-ia/taskcat)：CloudFormation テンプレートのテストツール
- [AWS CDK](https://aws.amazon.com/jp/cdk/)：プログラミング言語で CloudFormation を生成

### コミュニティ

- [AWS re:Post](https://repost.aws/)：AWS 公式のQ&Aサイト
- [Stack Overflow](https://stackoverflow.com/questions/tagged/aws-cloudformation)：CloudFormation タグの質問

---

お疲れさまでした。本コンテンツが、あなたの CloudFormation 学習の一助となれば幸いです。
