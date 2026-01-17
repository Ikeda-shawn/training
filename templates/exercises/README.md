# 演習用ディレクトリ

このディレクトリは、[第9章 演習](../../chapter09/README.md) で作成するテンプレートの保存場所です。

## 演習の進め方

1. 第9章の各演習問題を読む
2. このディレクトリに、対応するテンプレートを作成する
3. テンプレートを検証する
4. （可能であれば）実際にデプロイして動作確認する

## ファイル名の例

- `exercise1-vpc.yaml`：演習1で作成するテンプレート
- `exercise2-security-group.yaml`：演習2で作成するテンプレート
- `exercise3-network-stack.yaml`：演習3のネットワークスタック
- `exercise3-application-stack.yaml`：演習3のアプリケーションスタック
- `exercise4-iam-role.yaml`：演習4で作成するテンプレート
- `exercise5-network-stack.yaml`：演習5のネットワークスタック
- `exercise5-application-stack.yaml`：演習5のアプリケーションスタック

## 検証コマンド

テンプレートを作成したら、以下のコマンドで検証してください。

```bash
# テンプレートの構文チェック
aws cloudformation validate-template --template-body file://exercise1-vpc.yaml
```

## 注意事項

- 実際に AWS にデプロイする際は、料金が発生する可能性があります
- 不要なリソースは速やかに削除してください
- 本番環境では使用しないでください
