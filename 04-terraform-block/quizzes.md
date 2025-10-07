# terraformブロック実践演習：randomプロバイダーの追加

## 演習の目的

この演習では、学習したterraformブロックの知識を実際に活用して、新しいプロバイダー（randomプロバイダー）を既存のコードに追加します。

**重要：** この演習では、randomプロバイダーの定義のみを追加します。実際にrandomリソースを作成する必要はありません。

## 事前準備

現在のプロジェクトフォルダ `terraform-aws-practice` で作業を続けます。

## 演習タスク

### タスク1：Terraformレジストリでrandomプロバイダーを調べる

1. https://registry.terraform.io/ にアクセス
2. 検索ボックスに「random」と入力
3. Providersタブを選択
4. 公式のrandomプロバイダーを見つける

**質問1：** randomプロバイダーの完全なsource名は何ですか？

**質問2：** 現在の最新バージョンは何ですか？（2025年10月時点）

### タスク2：required_providersにrandomプロバイダーを追加

現在のterraformブロックを確認してください：

```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

**あなたの課題：** randomプロバイダーをrequired_providersに追加してください。

**ヒント：**

- randomプロバイダーはHashiCorp公式プロバイダーです
- バージョン制約には悲観的制約（~>）を使用してください
- アルファベット順に整理してください

### タスク3：コードを更新する

main.tfファイルのterraformブロック部分を以下の要件に従って修正してください：

**要件：**

1. awsプロバイダーとrandomプロバイダーの両方を含める
2. プロバイダーはアルファベット順に並べる
3. 適切なバージョン制約を使用する
4. コメントを追加して、なぜrandomプロバイダーが必要かを説明する

### タスク4：初期化とプラン確認

1. 変更後、以下のコマンドを実行：

```bash
terraform init
```

2. 新しいプロバイダーがダウンロードされることを確認

3. プランを確認（変更がないことを確認）：

```bash
terraform plan
```

**期待される結果：**

- randomプロバイダーがダウンロードされる
- プランでは「No changes」が表示される（リソース変更なし）

## 解答例

### 質問の解答

**質問1の解答：** `hashicorp/random`

**質問2の解答：** `3.7.2`（2025年10月時点での最新版）

### コード修正例

```hcl
terraform {
  # 最低限必要なTerraformバージョンを指定
  required_version = ">= 1.0"

  # 使用するプロバイダーを明確に定義
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"  # AWS リソース管理用
    }

    random = {
      source  = "hashicorp/random"
      version = "~> 3.7"  # ランダム値生成用（将来使用予定）
    }
  }
}
```

### バージョン制約の説明

```hcl
# randomプロバイダーのバージョン制約オプション

# オプション1：推奨（メジャーバージョン指定）
version = "~> 3.7"  # 3.7.x系列（3.8は除外、4.0も除外）

# オプション2：より保守的（マイナーバージョン固定）
version = "~> 3.7.2"  # 3.7.2以上3.7.x未満

# オプション3：より柔軟（メジャーバージョンのみ制約）
version = "~> 3.0"  # 3.x.x系列全体
```

**推奨：** `"~> 3.7"`を使用（新しい機能とバグ修正を受け取りつつ、破壊的変更は避ける）

### 実行結果の確認

1. `terraform init`実行後の出力例：

```
Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 6.0"...
- Finding hashicorp/random versions matching "~> 3.7"...
- Installing hashicorp/aws v6.x.x...
- Installing hashicorp/random v3.7.2...
- Installed hashicorp/aws v6.x.x (signed by HashiCorp)
- Installed hashicorp/random v3.7.2 (signed by HashiCorp)

Terraform has been successfully initialized!
```

2. `terraform plan`実行後の出力例：

```
No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure with your configuration and found no differences, so no changes are needed.
```

## 演習のポイント解説

### なぜrandomプロバイダーを追加するのか？

1. **一意な名前の生成**：AWSリソース名の重複を避ける
2. **パスワード生成**：データベースの初期パスワードなど
3. **ランダムな値が必要な設定**：セキュリティグループルールなど

### randomプロバイダー3.7系列の特徴

**2025年10月時点での3.7.2バージョンの主な機能：**

- 暗号学的に安全なランダム値生成
- 文字列、数値、UUIDなど多様なランダム値タイプ
- Terraform状態での値の永続化
- 高性能なランダム値生成アルゴリズム

### プロバイダー追加のベストプラクティス

```hcl
# 良い例：整理されたプロバイダー定義
terraform {
  required_version = ">= 1.0"

  required_providers {
    # アルファベット順に整理
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }

    random = {
      source  = "hashicorp/random"
      version = "~> 3.7"  # 2025年10月時点の推奨
    }
  }
}

# 悪い例：順序が不適切、コメントなし
terraform {
  required_providers {
    random = {
      source = "hashicorp/random"
      # version指定なし（非推奨）
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

### プロバイダー追加時の注意点

1. **初期化が必要**：新しいプロバイダーを追加したら`terraform init`実行
2. **バージョン固定**：本番環境では適切なバージョン制約を設定
3. **不要なプロバイダーは追加しない**：使用しないプロバイダーはコードを複雑にする

## 追加演習（オプション）

時間に余裕がある場合は、以下も試してみてください：

### 演習A：他のプロバイダーを調査

Terraformレジストリで以下のプロバイダーを調査し、source名とバージョンを確認してください：

1. `tls`プロバイダー（証明書関連）
2. `archive`プロバイダー（ファイル圧縮）
3. `null`プロバイダー（特殊操作用）

### 演習B：バージョン制約の実験

randomプロバイダーで異なるバージョン制約を試してみてください：

```hcl
# パターン1：最新の機能を活用
version = ">= 3.7"

# パターン2：より厳密な制御
version = "~> 3.7.0"

# パターン3：範囲指定
version = ">= 3.7, < 4.0"
```

それぞれ`terraform init`を実行して、どのバージョンがインストールされるか確認してください。

### 演習C：バージョン確認コマンド

プロバイダーのバージョンを確認する方法を学習しましょう：

```bash
# インストールされているプロバイダーバージョンを確認
terraform providers

# より詳細な情報を表示
terraform version
```

## よくある問題と解決方法

### 問題1：プロバイダーのダウンロードが失敗する

```bash
Error: Failed to query available provider packages
```

**解決方法：**

1. インターネット接続を確認
2. プロキシ設定があれば適切に設定
3. source名の綴りを再確認

### 問題2：バージョン制約エラー

```bash
Error: Unsupported provider version
```

**解決方法：**

```hcl
# より柔軟な制約に変更
random = {
  source  = "hashicorp/random"
  version = ">= 3.0"  # 制約を緩和
}
```

### 問題3：古いプロバイダーキャッシュ

**解決方法：**

```bash
# プロバイダーキャッシュをクリア
rm -rf .terraform
terraform init
```

## まとめ

この演習で学んだこと：

1. **Terraformレジストリの活用**：最新バージョン情報の確認方法
2. **プロバイダー追加の手順**：required_providersへの適切な追加
3. **バージョン制約の選択**：3.7系列での推奨設定
4. **初期化プロセス**：新しいプロバイダーの正しいダウンロード手順
5. **ベストプラクティス**：整理されたコード構造の維持

### 重要なポイント

- **最新情報の確認**：レジストリで常に最新バージョンを確認
- **適切な制約**：`~> 3.7`で安全な自動更新を実現
- **アルファベット順**：コードの可読性と保守性を向上
- **コメントの重要性**：将来の用途を明確に記載
