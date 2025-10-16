# Terraformの設定ブロック完全ガイド：variable、locals、output

## この章で学ぶこと

Terraformの3つの重要な設定ブロックについて、詳しく学習します。これらのブロックを使いこなすことで、柔軟で保守しやすい設定を作成できるようになります。

**学習内容：**

- variableブロック：外部から値を受け取る
- localsブロック：計算結果や繰り返し使う値を定義
- outputブロック：結果を外部に公開
- それぞれの使い分けと連携方法
- ベストプラクティス
- 実践的な活用例

## 3つのブロックの関係性

### プログラミング言語での例え

```
variableブロック = 関数の引数（パラメータ）
localsブロック   = 関数内のローカル変数
outputブロック   = 関数の戻り値（返り値）
```

### Terraformでの役割

```
┌─────────────────────────────────────────────┐
│  外部からの入力（variable）                     │
│  - terraform apply -var="..."              │
│  - 環境変数                                   │
│  - terraform.tfvars ファイル                 │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│  Terraform設定ファイル                         │
│                                             │
│  variable → locals → resource → output     │
│  （入力）   （計算）  （実行）    （出力）        │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│  外部への出力（output）                         │
│  - コマンドライン表示                           │
│  - 他のTerraform設定から参照可能               │
│  - 自動化ツールへの情報渡し                      │
└─────────────────────────────────────────────┘
```

## variableブロック：外部から値を受け取る

### 基本的な考え方

**例えて言うなら：** レストランのオーダーシート

```
お客さん（ユーザー）→ オーダーシート（variable）→ 料理（リソース作成）

「辛さは？」          「普通」を選択           その辛さで料理
「量は？」            「大盛り」を選択          その量で料理
```

**Terraformでは：**

```hcl
# オーダーシート（variable定義）
variable "instance_type" {
  description = "EC2のサイズを選んでください"
  type        = string
  default     = "t4g.small"  # デフォルト値（何も指定しなければこれ）
}

# 料理を作る（resource作成）
resource "aws_instance" "web" {
  instance_type = var.instance_type  # オーダー通りに作る
  # ...
}
```

### variableブロックの基本構文

#### 構文の全体像

```hcl
variable "変数名" {
  type        = 型の指定
  default     = デフォルト値（省略可能）
  description = "この変数の説明"
  sensitive   = true/false（機密情報かどうか）
  nullable    = true/false（nullを許可するか）

  validation {
    condition     = 検証する条件
    error_message = "エラーメッセージ"
  }
}
```

#### 各部分の説明

##### 1. `variable "変数名"`

```hcl
variable "instance_type" {
  # ...
}
```

**説明：**

- `variable` キーワード：これが変数ブロックであることを示す
- `"変数名"`：この変数を参照する時に使う名前
- 使用時は `var.変数名` で参照（例：`var.instance_type`）

**命名ルール：**

- 英数字とアンダースコア（`_`）のみ使用可能
- 数字で始めることはできない
- 予約語（`source`, `version`, `providers`, `count`, `for_each`, `lifecycle`, `depends_on`, `locals`）は使用不可

**良い例：**

```hcl
variable "instance_type"       # OK
variable "vpc_cidr_block"      # OK
variable "enable_monitoring"   # OK
variable "subnet_count"        # OK
```

**悪い例：**

```hcl
variable "1_instance"    # NG：数字で始まっている
variable "instance-type" # NG：ハイフンは使えない
variable "count"         # NG：予約語
```

##### 2. `type`（型の指定）

```hcl
variable "example" {
  type = string  # ← ここで型を指定
}
```

**説明：**

- この変数がどんな種類の値を受け取るかを指定
- 型を指定することで、間違った値が渡された時にエラーで教えてくれる
- 省略可能だが、**必ず指定することを強く推奨**

**使用できる型：**

**プリミティブ型（基本型）：**

```hcl
# 文字列型
variable "region" {
  type = string  # "ap-northeast-1" のような文字列
}

# 数値型
variable "instance_count" {
  type = number  # 1, 2, 100 のような数値
}

# 真偽値型
variable "enable_backup" {
  type = bool  # true または false
}
```

**コレクション型（複数の値）：**

```hcl
# リスト型（順序付きの配列）
variable "availability_zones" {
  type = list(string)  # ["us-east-1a", "us-east-1b"] のようなリスト
}

# マップ型（キーと値のペア）
variable "tags" {
  type = map(string)  # { Environment = "dev", Project = "app" } のようなマップ
}

# セット型（重複のない集合）
variable "allowed_ips" {
  type = set(string)  # ["10.0.0.1", "10.0.0.2"] のような集合
}
```

**構造型（複雑な構造）：**

```hcl
# オブジェクト型（複数の異なる型を持つ構造）
variable "database_config" {
  type = object({
    engine         = string
    engine_version = string
    port           = number
    backup_enabled = bool
  })
}

# タプル型（固定長の配列）
variable "network_config" {
  type = tuple([string, number, bool])
  # 例：["10.0.0.0/16", 2, true]
}
```

**型を指定しない場合：**

```hcl
variable "flexible_value" {
  # type を指定しない場合、any型として扱われる
  # どんな値でも受け取れるが、推奨されない
}
```

##### 3. `default`（デフォルト値）

```hcl
variable "instance_type" {
  type    = string
  default = "t4g.small"  # ← デフォルト値
}
```

**説明：**

- 値が指定されなかった時に使われる初期値
- `default` を指定すると、その変数は**オプショナル（任意）**になる
- `default` を指定しないと、その変数は**必須**になる

**defaultがある場合（オプショナル）：**

```hcl
variable "instance_type" {
  type    = string
  default = "t4g.small"
}

# 使用時に値を指定しなくてもOK
# 何も指定しない → "t4g.small" が使われる
# terraform apply -var="instance_type=t4g.medium" → "t4g.medium" が使われる
```

**defaultがない場合（必須）：**

```hcl
variable "project_name" {
  type = string
  # default なし
}

# 使用時に必ず値を指定する必要がある
# terraform apply  # ← エラー！値を指定してください
# terraform apply -var="project_name=my-app"  # OK
```

**defaultの値の種類：**

```hcl
# プリミティブ型のdefault
variable "region" {
  type    = string
  default = "ap-northeast-1"
}

# リスト型のdefault
variable "azs" {
  type    = list(string)
  default = ["ap-northeast-1a", "ap-northeast-1c"]
}

# マップ型のdefault
variable "tags" {
  type = map(string)
  default = {
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}

# オブジェクト型のdefault
variable "db_config" {
  type = object({
    engine = string
    port   = number
  })
  default = {
    engine = "mysql"
    port   = 3306
  }
}
```

**重要な制約：**

```hcl
# NG：defaultで他の変数や計算式は使えない
variable "full_name" {
  type    = string
  default = "${var.project_name}-app"  # エラー！
}

# OK：リテラル（固定値）のみ
variable "environment" {
  type    = string
  default = "development"  # OK
}
```

##### 4. `description`（説明）

```hcl
variable "instance_type" {
  type        = string
  description = "EC2インスタンスのタイプ。本番環境ではt4g.medium以上を推奨。"
}
```

**説明：**

- この変数の目的や使い方を説明する文字列
- **必須ではないが、必ず書くことを強く推奨**
- 他の人（または未来の自分）がこのコードを読む時に役立つ

**良い説明の書き方：**

```hcl
# 良い例：目的と使い方が明確
variable "vpc_cidr" {
  description = "VPCのCIDRブロック。10.0.0.0/16 形式で指定。重複しないように注意。"
  type        = string
}

# 良い例：期待される値の範囲を記載
variable "instance_count" {
  description = "作成するEC2インスタンスの数。1〜10の範囲で指定してください。"
  type        = number
}

# 良い例：使用場面を説明
variable "enable_backup" {
  description = "データベースの自動バックアップを有効にする。本番環境では必ずtrueに設定。"
  type        = bool
}
```

**悪い説明の書き方：**

```hcl
# 悪い例：変数名を繰り返しているだけ
variable "instance_type" {
  description = "instance type"
  type        = string
}

# 悪い例：何のための変数か不明
variable "count" {
  description = "count"
  type        = number
}
```

**説明を書く視点：**

- **モジュール利用者の視点**で書く（この変数を使う人のために）
- コード管理者のメモはコメント（`#`）で書く

```hcl
# コード管理者向けのメモ（コメント）
# TODO: 将来的にはt4g.largeもサポートする予定

variable "instance_type" {
  # この説明は利用者向け
  description = "EC2インスタンスのタイプ。現在はt4g.small、t4g.mediumをサポート。"
  type        = string
}
```

##### 5. `sensitive`（機密情報フラグ）

```hcl
variable "database_password" {
  type      = string
  sensitive = true  # ← 機密情報であることを示す
}
```

**説明：**

- この変数が機密情報（パスワード、秘密鍵など）であることを示す
- `true` にすると、Terraformの出力（ログ）でその値が隠される
- デフォルトは `false`

**sensitive = true の効果：**

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

resource "aws_db_instance" "main" {
  password = var.db_password
  # ...
}
```

**terraform plan/applyの出力：**

```
# sensitive = true の場合
Terraform will perform the following actions:

  # aws_db_instance.main will be created
  + resource "aws_db_instance" "main" {
      + password = (sensitive value)  # ← 実際の値は表示されない
    }

# sensitive = false の場合
  + resource "aws_db_instance" "main" {
      + password = "mypassword123"  # ← 実際の値が表示される（危険！）
    }
```

**sensitiveにすべき情報：**

```hcl
# パスワード
variable "database_password" {
  type      = string
  sensitive = true
}

# APIキー
variable "api_key" {
  type      = string
  sensitive = true
}

# アクセストークン
variable "github_token" {
  type      = string
  sensitive = true
}

# 秘密鍵
variable "private_key" {
  type      = string
  sensitive = true
}
```

**重要な注意点：**

```
sensitive = true にしても：
✗ terraform.tfstate ファイルには平文で保存される
✗ 完全に安全ではない
✓ ログに表示されなくなるだけ

本当に安全に保管するには：
✓ AWS Secrets Manager を使う
✓ Parameter Store（SecureString）を使う
✓ 環境変数で渡す（TF_VAR_xxx）
```

##### 6. `nullable`（null許可フラグ）

```hcl
variable "optional_value" {
  type     = string
  nullable = true   # ← nullを許可する（デフォルト）
}

variable "required_value" {
  type     = string
  nullable = false  # ← nullを許可しない
}
```

**説明：**

- 変数の値として `null` を許可するかどうかを指定
- デフォルトは `true`（nullを許可）
- `nullable = false` にすると、null値を明示的に拒否

**nullable = true の場合：**

```hcl
variable "backup_retention_days" {
  type     = number
  default  = 7
  nullable = true
}

# 使用例
# terraform apply  → 7 が使われる（default）
# terraform apply -var="backup_retention_days=null"  → null が使われる（defaultを上書き）
```

**nullable = false の場合：**

```hcl
variable "vpc_id" {
  type     = string
  nullable = false  # nullは許可しない
}

# terraform apply -var="vpc_id=null"  # エラー！nullは許可されていません
```

**nullableの実用例：**

```hcl
variable "kms_key_id" {
  description = "暗号化用のKMSキーID。nullの場合はデフォルトキーを使用。"
  type        = string
  default     = null      # デフォルトはnull
  nullable    = true      # nullを許可
}

resource "aws_ebs_volume" "data" {
  kms_key_id = var.kms_key_id  # nullの場合、デフォルトキーが使われる
  # ...
}
```

##### 7. `validation`（検証ブロック）

```hcl
variable "instance_type" {
  type = string

  validation {
    condition     = contains(["t4g.micro", "t4g.small"], var.instance_type)
    error_message = "instance_typeはt4g.microまたはt4g.smallである必要があります。"
  }
}
```

**説明：**

- 変数の値が特定の条件を満たすかチェックする
- `condition` が `false` になると、`error_message` を表示してエラーになる
- 複数の `validation` ブロックを指定可能

**validationの構造：**

```hcl
variable "変数名" {
  type = 型

  validation {
    condition     = 検証する条件（trueまたはfalseを返す式）
    error_message = "条件が満たされない時のエラーメッセージ"
  }
}
```

**実用例1：許可された値のリストをチェック**

```hcl
variable "environment" {
  type = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "environmentはdev、staging、productionのいずれかを指定してください。"
  }
}
```

**実用例2：値の範囲をチェック**

```hcl
variable "instance_count" {
  type = number

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "instance_countは1から10の範囲で指定してください。"
  }
}
```

**実用例3：文字列のパターンをチェック**

```hcl
variable "ami_id" {
  type = string

  validation {
    condition     = length(var.ami_id) > 4 && substr(var.ami_id, 0, 4) == "ami-"
    error_message = "AMI IDは 'ami-' で始まる必要があります。"
  }
}
```

**実用例4：複数の検証条件**

```hcl
variable "vpc_cidr" {
  type = string

  # 検証1：CIDR形式が正しいか
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "有効なCIDR形式で指定してください（例：10.0.0.0/16）。"
  }

  # 検証2：プライベートIPの範囲か
  validation {
    condition = can(regex("^(10\\.|172\\.(1[6-9]|2[0-9]|3[01])\\.|192\\.168\\.)", var.vpc_cidr))
    error_message = "VPC CIDRはプライベートIP範囲（10.x.x.x、172.16-31.x.x、192.168.x.x）を使用してください。"
  }
}
```

**検証で使える関数：**

```hcl
# contains()：リストに値が含まれるか
validation {
  condition = contains(["a", "b", "c"], var.value)
}

# length()：文字列やリストの長さ
validation {
  condition = length(var.password) >= 8
}

# substr()：部分文字列の取得
validation {
  condition = substr(var.ami_id, 0, 4) == "ami-"
}

# can()：式がエラーにならないかチェック
validation {
  condition = can(cidrhost(var.vpc_cidr, 0))
}

# regex()：正規表現マッチ
validation {
  condition = can(regex("^[a-z0-9-]+$", var.name))
}
```

#### variableブロックの完全な例

```hcl
variable "instance_type" {
  # 説明：この変数の目的を明確に
  description = "EC2インスタンスのタイプ。ARM64アーキテクチャのGravitonインスタンスを使用。"

  # 型：string型を指定
  type = string

  # デフォルト値：指定しない場合はt4g.small
  default = "t4g.small"

  # null許可：nullは許可しない
  nullable = false

  # 機密情報ではない
  sensitive = false

  # 検証：許可されたインスタンスタイプのみ
  validation {
    condition = contains(
      ["t4g.micro", "t4g.small", "t4g.medium", "t4g.large"],
      var.instance_type
    )
    error_message = "instance_typeはt4g.micro、t4g.small、t4g.medium、t4g.largeのいずれかを指定してください。"
  }
}

# この変数の使用例
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type  # ← 変数を参照
  # ...
}
```

### 変数の値を指定する方法

Terraformで変数に値を渡す方法は4つあります。それぞれの方法を詳しく説明します。

#### 方法1：コマンドラインで指定

**基本的な使い方：**

```bash
terraform apply -var="変数名=値"
```

**実例：**

```bash
# 1つの変数を指定
terraform apply -var="instance_type=t4g.medium"

# 複数の変数を指定（-varを繰り返す）
terraform apply -var="instance_type=t4g.medium" -var="environment=production"

# planでも同じように使える
terraform plan -var="instance_type=t4g.large"
```

**コマンドの構造：**

```bash
terraform apply -var="instance_type=t4g.medium"
       ↑       ↑    ↑              ↑
    コマンド  オプション 変数名      値
```

**説明：**

- `-var` オプション：変数に値を渡すためのフラグ
- `"変数名=値"` 形式：ダブルクォートで囲む
- 複数の変数を指定する場合：`-var` を複数回使用

**いつ使うか：**

- 一時的に異なる値を試したい時
- CI/CDパイプラインで動的に値を変更したい時
- デフォルト値を上書きしたい時

**注意点：**

```bash
# スペースを含む値はダブルクォートで囲む
terraform apply -var="project_name=My Web App"  # OK

# リスト型の値を指定
terraform apply -var='availability_zones=["ap-northeast-1a","ap-northeast-1c"]'

# マップ型の値を指定
terraform apply -var='tags={Environment="dev",Project="app"}'
```

**実行例：**

```bash
# main.tfファイルがあるディレクトリで実行
$ terraform apply -var="instance_type=t4g.large"

# Terraformは指定された値を使ってリソースを作成
Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + instance_type = "t4g.large"  # ← 指定した値が使われる
    }
```

#### 方法2：環境変数で指定

**基本的な使い方：**

```bash
export TF_VAR_変数名="値"
```

**実例：**

```bash
# 1つの環境変数を設定
export TF_VAR_instance_type="t4g.medium"

# 複数の環境変数を設定
export TF_VAR_instance_type="t4g.medium"
export TF_VAR_environment="production"
export TF_VAR_aws_region="ap-northeast-1"

# 設定後、terraform applyを実行（-varオプション不要）
terraform apply
```

**TF*VAR*プレフィックスの説明：**

```
TF_VAR_instance_type
  ↑      ↑
  |      └─ Terraformの変数名（variable "instance_type"）
  |
  └─ 必須のプレフィックス（固定）
```

**重要なルール：**

```bash
# OK：TF_VAR_プレフィックスを使用
export TF_VAR_instance_type="t4g.small"

# NG：別の名前は使えない
export INSTANCE_TYPE="t4g.small"          # Terraformは認識しない
export MY_VAR_instance_type="t4g.small"   # Terraformは認識しない
export TF_instance_type="t4g.small"       # Terraformは認識しない

# TF_VAR_ は必須の固定プレフィックス
```

**なぜTF*VAR*プレフィックスが必要か：**

```
理由：
1. 他の環境変数と区別するため
   - PATH、HOME、USERなどのシステム環境変数と衝突を避ける

2. Terraformが明示的に認識するため
   - TF_VAR_で始まる環境変数だけをTerraform変数として扱う

3. 安全性のため
   - 意図しない環境変数がTerraformに渡されることを防ぐ
```

**環境変数の確認方法：**

```bash
# 設定した環境変数を確認
echo $TF_VAR_instance_type
# 出力: t4g.medium

# すべてのTF_VAR_環境変数を表示
env | grep TF_VAR_
# 出力例:
# TF_VAR_instance_type=t4g.medium
# TF_VAR_environment=production
# TF_VAR_aws_region=ap-northeast-1
```

**環境変数の削除方法：**

```bash
# 特定の環境変数を削除
unset TF_VAR_instance_type

# 複数削除
unset TF_VAR_instance_type
unset TF_VAR_environment
```

**いつ使うか：**

- CI/CDパイプラインで秘密情報を渡す時
- 開発者ごとに異なる設定を使いたい時
- セッション中、同じ値を何度も使う時

**セキュリティ上の利点：**

```bash
# パスワードなどの機密情報をコマンド履歴に残さない
export TF_VAR_database_password="secret123"  # コマンド履歴に残る
terraform apply  # パスワードはコマンドに含まれない

# 比較：コマンドラインだと履歴に残る
terraform apply -var="database_password=secret123"  # 危険！履歴に残る
```

**実行例：**

```bash
# ステップ1：環境変数を設定
$ export TF_VAR_instance_type="t4g.large"
$ export TF_VAR_environment="production"

# ステップ2：terraform applyを実行（-varオプション不要）
$ terraform apply

# Terraformは環境変数から値を取得
Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + instance_type = "t4g.large"  # ← TF_VAR_instance_typeから取得
    }
```

**複雑な型の環境変数：**

```bash
# リスト型
export TF_VAR_availability_zones='["ap-northeast-1a","ap-northeast-1c"]'

# マップ型
export TF_VAR_tags='{"Environment":"production","Team":"platform"}'

# 複数行の文字列（ヒアドキュメント使用）
export TF_VAR_user_data=$(cat <<EOF
#!/bin/bash
echo "Hello World"
apt-get update
EOF
)
```

#### 方法3：terraform.tfvarsファイルで指定

**terraform.tfvarsファイルとは：**

- 変数の値を定義する専用ファイル
- プロジェクトのルートディレクトリに配置
- **自動的に読み込まれる**（特別な指定不要）

**ファイル構造の例：**

```
プロジェクトディレクトリ/
├── main.tf                 # メインの設定ファイル
├── variables.tf            # 変数定義ファイル
├── terraform.tfvars        # 変数の値を設定（このファイル）
└── outputs.tf              # 出力定義ファイル
```

**terraform.tfvarsの内容：**

```hcl
# terraform.tfvars ファイル

# シンプルな値
instance_type = "t4g.medium"
aws_region    = "ap-northeast-1"
project_name  = "my-app"
environment   = "development"

# 数値
instance_count = 3

# 真偽値
enable_monitoring = true

# リスト
availability_zones = [
  "ap-northeast-1a",
  "ap-northeast-1c"
]

# マップ
tags = {
  Environment = "development"
  Team        = "platform"
  CostCenter  = "engineering"
}

# コメントも書ける
# enable_backup = true  # コメントアウトされた設定
```

**variables.tfとの対応：**

```hcl
# variables.tf（変数の定義）
variable "instance_type" {
  type        = string
  description = "EC2インスタンスのタイプ"
}

variable "aws_region" {
  type        = string
  description = "AWSリージョン"
}
```

```hcl
# terraform.tfvars（変数の値）
instance_type = "t4g.medium"
aws_region    = "ap-northeast-1"
```

**実行方法：**

```bash
# terraform.tfvarsがあるディレクトリで実行
$ terraform apply

# terraform.tfvarsは自動的に読み込まれる
# 特別なオプション指定は不要
```

**いつ使うか：**

- プロジェクトの標準設定を定義したい時
- チーム全体で同じ設定を共有したい時
- 環境ごとの設定を管理したい時

**重要な注意点：**

```
terraform.tfvarsファイルの扱い：

✓ 開発環境の設定などはGitにコミットしてOK
✗ 本番環境のパスワードなどの機密情報はコミットしない
✓ .gitignoreに追加して除外する

# .gitignore の例
terraform.tfvars
*.tfvars
```

**自動的に読み込まれるファイル名：**

```
以下のファイル名は自動的に読み込まれる：
1. terraform.tfvars
2. terraform.tfvars.json
3. *.auto.tfvars
4. *.auto.tfvars.json
```

#### 方法4：カスタムファイルで指定

**カスタムファイルとは：**

- 任意の名前をつけた変数定義ファイル
- 環境別に複数のファイルを作成できる
- **-var-fileオプションで明示的に指定が必要**

**ファイル構造の例：**

```
プロジェクトディレクトリ/
├── main.tf
├── variables.tf
├── terraform.tfvars          # 共通設定（自動読み込み）
├── development.tfvars        # 開発環境用（手動指定）
├── staging.tfvars            # ステージング環境用（手動指定）
└── production.tfvars         # 本番環境用（手動指定）
```

**各ファイルの内容例：**

```hcl
# terraform.tfvars（共通設定）
project_name = "my-web-app"
aws_region   = "ap-northeast-1"
```

```hcl
# development.tfvars（開発環境）
environment   = "development"
instance_type = "t4g.micro"
instance_count = 1
enable_backup = false
```

```hcl
# staging.tfvars（ステージング環境）
environment   = "staging"
instance_type = "t4g.small"
instance_count = 2
enable_backup = false
```

```hcl
# production.tfvars（本番環境）
environment   = "production"
instance_type = "t4g.medium"
instance_count = 4
enable_backup = true
```

**実行方法：**

```bash
# 開発環境で実行
terraform apply -var-file="development.tfvars"

# ステージング環境で実行
terraform apply -var-file="staging.tfvars"

# 本番環境で実行
terraform apply -var-file="production.tfvars"
```

**複数のファイルを同時に指定：**

```bash
# 共通設定 + 環境別設定
terraform apply -var-file="common.tfvars" -var-file="production.tfvars"

# 読み込み順序：
# 1. terraform.tfvars（自動）
# 2. common.tfvars（-var-fileで指定）
# 3. production.tfvars（-var-fileで指定）
# 後から読み込まれた値が優先される
```

**いつ使うか：**

- 複数の環境（開発、ステージング、本番）を管理する時
- 環境ごとに異なる設定を使い分けたい時
- 設定の切り替えを明示的に行いたい時

### main.tfファイル1つだけの場合

**質問：「main.tf だけしかない場合、どうなるの？」**

#### シナリオ1：defaultがある変数のみ

```hcl
# main.tf
variable "instance_type" {
  type    = string
  default = "t4g.small"  # デフォルト値あり
}

variable "aws_region" {
  type    = string
  default = "ap-northeast-1"  # デフォルト値あり
}

resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = var.instance_type
  # ...
}
```

**この場合の動作：**

```bash
# そのまま実行できる（値の指定不要）
$ terraform apply

# デフォルト値が使われる
# instance_type = "t4g.small"
# aws_region = "ap-northeast-1"
```

**説明：**

- すべての変数にdefaultがある
- terraform.tfvarsファイルがなくても実行可能
- 値を変更したい場合は-varオプションや環境変数を使用

#### シナリオ2：defaultがない変数がある

```hcl
# main.tf
variable "project_name" {
  type = string
  # default なし = 必須変数
}

variable "instance_type" {
  type    = string
  default = "t4g.small"
}

resource "aws_instance" "web" {
  instance_type = var.instance_type
  tags = {
    Name = var.project_name
  }
}
```

**この場合の動作：**

```bash
# そのまま実行するとエラー
$ terraform apply

# エラーメッセージ：
Error: No value for required variable

  on main.tf line 1:
   1: variable "project_name" {

The variable "project_name" is required, but is not set.
```

**解決方法1：コマンドラインで指定**

```bash
terraform apply -var="project_name=my-app"
```

**解決方法2：環境変数で指定**

```bash
export TF_VAR_project_name="my-app"
terraform apply
```

**解決方法3：terraform.tfvarsファイルを作成**

```hcl
# terraform.tfvars を作成
project_name = "my-app"
```

```bash
# そのまま実行できる
terraform apply
```

**解決方法4：対話的に入力**

```bash
$ terraform apply

var.project_name
  Enter a value: my-app  # ← ここで値を入力

# 入力した値で実行される
```

#### main.tfファイル1つだけで完結させる場合のベストプラクティス

**推奨パターン1：すべての変数にdefaultを設定**

```hcl
# main.tf（学習用・小規模プロジェクト向け）

# 変数定義（すべてdefault付き）
variable "project_name" {
  type    = string
  default = "my-web-app"
}

variable "environment" {
  type    = string
  default = "development"
}

variable "instance_type" {
  type    = string
  default = "t4g.small"
}

# リソース定義
resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = var.instance_type

  tags = {
    Name        = "${var.project_name}-${var.environment}"
    Environment = var.environment
  }
}

# 出力定義
output "instance_id" {
  value = aws_instance.web.id
}
```

**メリット：**

- ファイル1つだけで完結
- すぐに実行できる（terraform apply だけでOK）
- 学習や検証に適している

**デメリット：**

- 値を変更する時はファイルを直接編集する必要がある
- 環境別の設定管理が難しい

**推奨パターン2：main.tfと小さいterraform.tfvarsを併用**

```hcl
# main.tf（変数定義は最小限）
variable "project_name" {
  type = string
}

variable "instance_type" {
  type    = string
  default = "t4g.small"
}

resource "aws_instance" "web" {
  instance_type = var.instance_type
  tags = {
    Name = var.project_name
  }
}
```

```hcl
# terraform.tfvars（値の設定）
project_name = "my-web-app"
instance_type = "t4g.medium"  # defaultを上書き
```

**メリット：**

- 設定と定義が分離されている
- 値の変更が簡単（terraform.tfvarsだけ編集）
- コードの再利用性が高い

### 4つの方法の優先順位

Terraformは以下の順序で変数の値を決定します（後から読み込まれる方が優先）：

```
1. 変数のdefault値（最も優先度が低い）
   ↓
2. 環境変数（TF_VAR_xxx）
   ↓
3. terraform.tfvarsファイル
   ↓
4. *.auto.tfvarsファイル（アルファベット順）
   ↓
5. -var-fileで指定したファイル（指定した順）
   ↓
6. -varオプション（最も優先度が高い）
```

**実例：**

```hcl
# variables.tf
variable "instance_type" {
  type    = string
  default = "t4g.micro"  # ← 1. default値
}
```

```bash
# 2. 環境変数を設定
export TF_VAR_instance_type="t4g.small"
```

```hcl
# terraform.tfvars
instance_type = "t4g.medium"  # ← 3. tfvarsファイル
```

```bash
# 実行
terraform apply -var-file="production.tfvars" -var="instance_type=t4g.large"
                          ↑                           ↑
                    4. カスタムファイル              5. コマンドライン

# 最終的に使われる値: "t4g.large"（最も優先度が高い）
```

### それぞれの方法を使い分ける実践例

**シナリオ1：個人の学習・検証**

```hcl
# main.tf だけで完結
variable "instance_type" {
  type    = string
  default = "t4g.small"
}

# 試したい値だけコマンドラインで上書き
```

```bash
terraform apply  # デフォルト値で実行
terraform apply -var="instance_type=t4g.medium"  # 値を変更
```

**シナリオ2：チームでの開発**

```
main.tf              # リソース定義
variables.tf         # 変数定義
terraform.tfvars     # 共通設定（Gitにコミット）
```

```bash
# terraform.tfvarsの値で実行
terraform apply
```

**シナリオ3：マルチ環境管理**

```
main.tf
variables.tf
development.tfvars   # 開発環境設定
staging.tfvars       # ステージング環境設定
production.tfvars    # 本番環境設定
```

```bash
# 環境に応じてファイルを切り替え
terraform apply -var-file="development.tfvars"
terraform apply -var-file="production.tfvars"
```

**シナリオ4：CI/CDパイプライン**

```bash
# GitLab CI、GitHub Actionsなどで環境変数を使用
export TF_VAR_database_password="${DB_PASSWORD}"  # シークレットから取得
export TF_VAR_api_key="${API_KEY}"

terraform apply -var-file="${ENVIRONMENT}.tfvars"
```

### 実例：様々な型の変数定義

```hcl
# シンプルな文字列変数
variable "aws_region" {
  description = "AWSのリージョン"
  type        = string
  default     = "ap-northeast-1"  # 東京リージョン
}

# 数値変数
variable "instance_count" {
  description = "作成するEC2インスタンスの数"
  type        = number
  default     = 1
}

# 真偽値変数
variable "enable_monitoring" {
  description = "モニタリングを有効にするかどうか"
  type        = bool
  default     = true
}

# リスト型（配列）
variable "availability_zones" {
  description = "使用するアベイラビリティゾーンのリスト"
  type        = list(string)
  default     = ["ap-northeast-1a", "ap-northeast-1c"]
}

# マップ型（辞書）
variable "tags" {
  description = "すべてのリソースに付けるタグ"
  type        = map(string)
  default = {
    Environment = "development"
    Project     = "my-project"
  }
}

# オブジェクト型（複雑な構造）
variable "database_config" {
  description = "データベースの設定"
  type = object({
    engine            = string
    engine_version    = string
    instance_class    = string
    allocated_storage = number
  })
  default = {
    engine            = "mysql"
    engine_version    = "8.0"
    instance_class    = "db.t3.micro"
    allocated_storage = 20
  }
}

# 機密情報用の変数
variable "database_password" {
  description = "データベースの管理者パスワード"
  type        = string
  sensitive   = true  # ← ログに表示されない
}
```

## localsブロック：計算結果や繰り返し使う値を定義

### 基本的な考え方

**例えて言うなら：** 料理のレシピ内で使う中間材料

```
材料（variable）→ 下ごしらえ（locals）→ 料理（resource）

玉ねぎ、人参        みじん切り             カレー
                 （中間材料）
```

**Terraformでは：**

```hcl
# 材料（variable）
variable "project_name" {
  type    = string
  default = "my-app"
}

variable "environment" {
  type    = string
  default = "dev"
}

# 下ごしらえ（locals）- 繰り返し使う値を定義
locals {
  # プロジェクト名と環境を組み合わせた共通名
  common_name = "${var.project_name}-${var.environment}"

  # すべてのリソースに付けるタグ
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# 料理（resource）- local値を繰り返し使用
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = merge(
    local.common_tags,
    {
      Name = "${local.common_name}-vpc"
    }
  )
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"

  tags = merge(
    local.common_tags,
    {
      Name = "${local.common_name}-public-subnet"
    }
  )
}
```

### localsブロックの基本構文

#### 構文の全体像

```hcl
locals {
  ローカル値名1 = 計算式や値
  ローカル値名2 = 別の計算式や値
  # 複数定義可能
}

# 別のlocalsブロックも定義可能（すべて同じブロックとして扱われる）
locals {
  ローカル値名3 = さらに別の値
}
```

#### 各部分の説明

##### 1. `locals` キーワード

```hcl
locals {
  # ここにローカル値を定義
}
```

**説明：**

- `locals`（複数形）でブロックを開始
- 複数の値を定義できる
- 1つのファイルに複数の `locals` ブロックを書ける
- 参照時は `local.名前`（**単数形**）を使用

**重要：localsとlocalの違い**

```hcl
# 定義時は locals（複数形）
locals {
  instance_name = "web-server"
}

# 参照時は local（単数形）
resource "aws_instance" "web" {
  tags = {
    Name = local.instance_name  # ← local（単数形）
  }
}
```

##### 2. ローカル値の定義

```hcl
locals {
  simple_value = "文字列"
  number_value = 100
  bool_value   = true
  list_value   = ["a", "b", "c"]
  map_value    = { key = "value" }
}
```

**説明：**

- `名前 = 値` の形式で定義
- `=` の左側がローカル値の名前
- `=` の右側が値（式）

**命名ルール：**

```hcl
# 良い例
locals {
  instance_name       # OK：小文字とアンダースコア
  vpc_cidr_block      # OK
  is_production       # OK：真偽値はis_で始めるとわかりやすい
  total_subnet_count  # OK
}

# 悪い例
locals {
  instanceName      # NG：キャメルケースは避ける（スネークケース推奨）
  vpc-cidr          # NG：ハイフンは使えない
  1_instance        # NG：数字で始められない
}
```

##### 3. ローカル値で使える式

**3-1. 変数の参照**

```hcl
variable "project_name" {
  type = string
}

variable "environment" {
  type = string
}

locals {
  # 変数を参照して新しい値を作る
  full_name = "${var.project_name}-${var.environment}"
}
```

**3-2. 他のローカル値の参照**

```hcl
locals {
  base_name = "my-app"
}

locals {
  # 他のローカル値を参照
  instance_name = "${local.base_name}-instance"
  bucket_name   = "${local.base_name}-bucket"
}
```

**3-3. リソース属性の参照**

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

locals {
  # リソースの属性を参照
  vpc_id = aws_vpc.main.id
}
```

**3-4. データソースの参照**

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  # データソースを参照
  az_count = length(data.aws_availability_zones.available.names)
}
```

**3-5. 計算式**

```hcl
locals {
  # 数値計算
  total_instances = 2 + 3
  half_count      = 10 / 2

  # 文字列結合
  full_path = "/var/www/${var.project_name}"

  # 条件式
  instance_type = var.environment == "production" ? "t4g.large" : "t4g.small"
}
```

**3-6. 関数の使用**

```hcl
locals {
  # length()：長さを取得
  az_count = length(["a", "b", "c"])  # 3

  # concat()：リストを結合
  all_subnets = concat(
    aws_subnet.public[*].id,
    aws_subnet.private[*].id
  )

  # merge()：マップを結合
  all_tags = merge(
    { Environment = var.environment },
    { Project = var.project_name }
  )

  # cidrsubnet()：サブネットCIDRを計算
  subnet_cidr = cidrsubnet("10.0.0.0/16", 8, 1)  # "10.0.1.0/24"
}
```

**3-7. forループ**

```hcl
locals {
  # リストの変換
  subnet_cidrs = [
    for i in range(3) :
    cidrsubnet("10.0.0.0/16", 8, i)
  ]
  # 結果: ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]

  # マップの変換
  uppercase_tags = {
    for k, v in var.tags :
    k => upper(v)
  }
}
```

##### 4. ローカル値の参照方法

```hcl
locals {
  instance_name = "web-server"
  vpc_cidr      = "10.0.0.0/16"
}

# 参照：local.名前（単数形）
resource "aws_instance" "web" {
  tags = {
    Name = local.instance_name  # ← local.で参照
  }
}

resource "aws_vpc" "main" {
  cidr_block = local.vpc_cidr  # ← local.で参照
}
```

**重要：参照時のルール**

```hcl
# OK：local.名前
local.instance_name

# NG：locals.名前（複数形は使えない）
locals.instance_name  # エラー！

# NG：var. をつけない
instance_name  # エラー！どこから来た値か不明
```

##### 5. 複数のlocalsブロック

```hcl
# ブロック1：ネットワーク関連
locals {
  vpc_cidr = "10.0.0.0/16"
  az_count = 2
}

# ブロック2：アプリケーション関連
locals {
  app_name = "${var.project_name}-${var.environment}"
  app_port = 8080
}

# ブロック3：計算された値（上記のlocalsを使用）
locals {
  subnet_cidrs = [
    for i in range(local.az_count) :
    cidrsubnet(local.vpc_cidr, 8, i)
  ]
}

# すべて local.名前 で参照可能
resource "aws_vpc" "main" {
  cidr_block = local.vpc_cidr  # ブロック1から参照
}

resource "aws_instance" "web" {
  tags = {
    Name = local.app_name  # ブロック2から参照
  }
}
```

**説明：**

- Terraformは複数の `locals` ブロックを1つのブロックとして扱う
- どのブロックで定義しても、すべて `local.名前` で参照可能
- 関連する値をまとめて整理するために複数ブロックを使う

##### 6. localsの実行順序

**基本ルール：**

```hcl
# OK：依存関係を正しく記述
locals {
  base_value = 10
  derived_value = local.base_value + 1  # base_valueを参照
}

# OK：別のブロックでも参照可能
locals {
  base_value = 10
}

locals {
  derived_value = local.base_value + 1
}

# NG：循環参照はエラー
locals {
  value_a = local.value_b + 1
  value_b = local.value_a + 1  # エラー：value_aがvalue_bを参照し、value_bがvalue_aを参照
}
```

##### 7. localsで避けるべきこと

**避けるべき例1：単純な値のコピー**

```hcl
# 悪い例：ただ変数をコピーしているだけ
variable "region" {
  type = string
}

locals {
  aws_region = var.region  # 意味がない
}

# 良い例：変数を直接使う
resource "aws_instance" "web" {
  # ...
}
```

**避けるべき例2：過度に複雑な計算**

```hcl
# 悪い例：1つのローカル値に複雑すぎる計算
locals {
  complex_value = merge(
    {
      for k, v in var.tags :
      upper(k) => length(v) > 10 ? substr(v, 0, 10) : v
      if k != "ignore"
    },
    {
      for i in range(var.count) :
      "key_${i}" => format("value_%03d", i * 2 + 1)
    }
  )  # 複雑すぎて理解困難
}

# 良い例：段階的に分割
locals {
  # ステップ1：タグの正規化
  normalized_tags = {
    for k, v in var.tags :
    upper(k) => length(v) > 10 ? substr(v, 0, 10) : v
    if k != "ignore"
  }

  # ステップ2：追加タグの生成
  generated_tags = {
    for i in range(var.count) :
    "key_${i}" => format("value_%03d", i * 2 + 1)
  }

  # ステップ3：結合
  all_tags = merge(
    local.normalized_tags,
    local.generated_tags
  )
}
```

#### localsブロックの完全な例

```hcl
# 変数定義
variable "project_name" {
  type = string
}

variable "environment" {
  type = string
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

# データソース
data "aws_availability_zones" "available" {
  state = "available"
}

# ローカル値定義（ブロック1：基本設定）
locals {
  # プロジェクトと環境を組み合わせた名前
  common_name = "${var.project_name}-${var.environment}"

  # 環境に応じたインスタンスタイプ
  instance_type = var.environment == "production" ? "t4g.medium" : "t4g.small"

  # 利用可能なAZの数
  az_count = length(data.aws_availability_zones.available.names)
}

# ローカル値定義（ブロック2：ネットワーク計算）
locals {
  # 使用するAZのリスト（最初の2つ）
  availability_zones = slice(
    data.aws_availability_zones.available.names,
    0,
    min(local.az_count, 2)  # 最大2つまで
  )

  # パブリックサブネットのCIDR自動計算
  public_subnet_cidrs = [
    for i in range(length(local.availability_zones)) :
    cidrsubnet(var.vpc_cidr, 8, i)
  ]

  # プライベートサブネットのCIDR自動計算
  private_subnet_cidrs = [
    for i in range(length(local.availability_zones)) :
    cidrsubnet(var.vpc_cidr, 8, i + 10)
  ]
}

# ローカル値定義（ブロック3：タグ）
locals {
  # すべてのリソースに付ける共通タグ
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
    CreatedAt   = timestamp()
  }

  # 環境別の追加タグ
  environment_tags = var.environment == "production" ? {
    CostCenter = "production-infra"
    Backup     = "required"
  } : {
    CostCenter = "development"
    Backup     = "optional"
  }

  # すべてのタグを結合
  all_tags = merge(
    local.common_tags,
    local.environment_tags
  )
}

# リソース作成（localsを使用）
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  tags = merge(
    local.all_tags,
    { Name = "${local.common_name}-vpc" }
  )
}

resource "aws_subnet" "public" {
  count = length(local.public_subnet_cidrs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = local.public_subnet_cidrs[count.index]
  availability_zone = local.availability_zones[count.index]

  tags = merge(
    local.all_tags,
    {
      Name = "${local.common_name}-public-${count.index + 1}"
      Type = "Public"
    }
  )
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = local.instance_type
  subnet_id     = aws_subnet.public[0].id

  tags = merge(
    local.all_tags,
    { Name = "${local.common_name}-instance" }
  )
}
```

### variableとlocalsの使い分け

**variableを使う場合：**

```hcl
# ユーザーが変更する可能性がある値
variable "instance_type" {
  type    = string
  default = "t4g.small"
}

# 環境によって変わる値
variable "environment" {
  type = string
}
```

**localsを使う場合：**

```hcl
# 計算によって決まる値
locals {
  instance_name = "${var.project_name}-${var.environment}-instance"
}

# 複雑なロジック
locals {
  subnet_count = var.environment == "production" ? 6 : 2
}

# 繰り返し使う定型的な値
locals {
  common_tags = {
    ManagedBy = "Terraform"
    Team      = "Platform"
  }
}
```

**重要なポイント：**

```
variable = 外部から受け取る値（ユーザーが変更できる）
locals   = 内部で計算する値（計算式やロジックを含む）
```

## outputブロック：結果を外部に公開

### 基本的な考え方

**例えて言うなら：** レストランの会計レシート

```
料理を作る → 完成 → レシート発行
            （結果）（金額、料理名など）

お客さんに渡す重要な情報だけを表示
```

**Terraformでは：**

```hcl
# リソースを作成
resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"
  # ...
}

# 重要な情報を出力
output "instance_public_ip" {
  description = "EC2インスタンスのパブリックIPアドレス"
  value       = aws_instance.web.public_ip
}

output "instance_id" {
  description = "EC2インスタンスのID"
  value       = aws_instance.web.id
}
```

**terraform apply後の表示：**

```
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

instance_id        = "i-0abc123def456789"
instance_public_ip = "18.182.XXX.XXX"
```

### outputブロックの基本構文

#### 構文の全体像

```hcl
output "出力名" {
  description = "この出力の説明"
  value       = 出力する値（式）
  sensitive   = true/false（機密情報かどうか）
  depends_on  = [依存するリソース]

  precondition {
    condition     = 事前条件
    error_message = "エラーメッセージ"
  }
}
```

#### 各部分の説明

##### 1. `output "出力名"`

```hcl
output "instance_public_ip" {
  # ...
}
```

**説明：**

- `output` キーワード：これが出力ブロックであることを示す
- `"出力名"`：この出力を識別する名前
- `terraform output 出力名` コマンドでこの出力だけを表示できる

**命名ルール：**

- 英数字とアンダースコア（`_`）のみ使用可能
- 数字で始めることはできない
- わかりやすく、説明的な名前をつける

**良い例：**

```hcl
output "vpc_id"                  # OK：何のIDか明確
output "instance_public_ip"      # OK：どの属性か明確
output "database_endpoint"       # OK：用途が明確
output "ssh_connection_command"  # OK：使い方が明確
```

**悪い例：**

```hcl
output "id"          # NG：何のIDか不明
output "ip"          # NG：どのリソースのIPか不明
output "output1"     # NG：内容が不明
output "result"      # NG：何の結果か不明
```

##### 2. `value`（出力する値）- 必須

```hcl
output "instance_id" {
  value = aws_instance.web.id  # ← 出力する値
}
```

**説明：**

- **必須の引数**：必ず指定する必要がある
- 任意の有効なTerraform式を指定可能
- リソース属性、変数、ローカル値、計算式など

**valueで使える式の種類：**

**2-1. リソース属性の参照**

```hcl
resource "aws_instance" "web" {
  # ...
}

output "instance_id" {
  value = aws_instance.web.id
}

output "instance_public_ip" {
  value = aws_instance.web.public_ip
}

output "instance_private_ip" {
  value = aws_instance.web.private_ip
}
```

**2-2. 変数の参照**

```hcl
variable "environment" {
  type = string
}

output "current_environment" {
  value = var.environment
}
```

**2-3. ローカル値の参照**

```hcl
locals {
  common_name = "${var.project_name}-${var.environment}"
}

output "resource_name_prefix" {
  value = local.common_name
}
```

**2-4. データソースの参照**

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

output "available_zones" {
  value = data.aws_availability_zones.available.names
}
```

**2-5. 複数の値をリストで出力**

```hcl
resource "aws_subnet" "public" {
  count = 3
  # ...
}

output "all_subnet_ids" {
  value = aws_subnet.public[*].id
  # 結果: ["subnet-xxx", "subnet-yyy", "subnet-zzz"]
}
```

**2-6. マップ（オブジェクト）で出力**

```hcl
output "instance_info" {
  value = {
    id         = aws_instance.web.id
    public_ip  = aws_instance.web.public_ip
    private_ip = aws_instance.web.private_ip
    type       = aws_instance.web.instance_type
  }
}

# 出力結果:
# instance_info = {
#   id         = "i-xxxxx"
#   public_ip  = "18.182.XXX.XXX"
#   private_ip = "10.0.1.5"
#   type       = "t4g.small"
# }
```

**2-7. 計算式や条件式**

```hcl
output "instance_url" {
  value = "https://${aws_instance.web.public_ip}:8080"
}

output "ssh_command" {
  value = aws_instance.web.public_ip != "" ? (
    "ssh -i keypair.pem ec2-user@${aws_instance.web.public_ip}"
  ) : "パブリックIPが割り当てられていません"
}
```

**2-8. forループで変換**

```hcl
output "subnet_info_list" {
  value = [
    for subnet in aws_subnet.public :
    {
      id   = subnet.id
      cidr = subnet.cidr_block
      az   = subnet.availability_zone
    }
  ]
}
```

**2-9. 関数を使用**

```hcl
output "total_subnets" {
  value = length(aws_subnet.public)
}

output "combined_subnet_ids" {
  value = concat(
    aws_subnet.public[*].id,
    aws_subnet.private[*].id
  )
}
```

##### 3. `description`（説明）- 強く推奨

```hcl
output "database_endpoint" {
  description = "データベースの接続エンドポイント。アプリケーション設定で使用してください。"
  value       = aws_db_instance.main.endpoint
}
```

**説明：**

- この出力の目的や使い方を説明する文字列
- **必須ではないが、必ず書くことを強く推奨**
- モジュールの利用者や未来の自分のために書く

**良い説明の書き方：**

```hcl
# 良い例：用途と使い方が明確
output "vpc_id" {
  description = "作成したVPCのID。サブネットやセキュリティグループ作成時に使用。"
  value       = aws_vpc.main.id
}

# 良い例：フォーマットの説明
output "database_connection_string" {
  description = "データベース接続文字列。形式：mysql://endpoint:port/database"
  value       = "mysql://${aws_db_instance.main.endpoint}/${aws_db_instance.main.db_name}"
}

# 良い例：注意事項を含む
output "admin_password" {
  description = "管理者パスワード。初回ログイン後に必ず変更してください。"
  value       = random_password.admin.result
  sensitive   = true
}
```

**悪い説明の書き方：**

```hcl
# 悪い例：出力名を繰り返しているだけ
output "instance_id" {
  description = "instance id"
  value       = aws_instance.web.id
}

# 悪い例：何に使うか不明
output "result" {
  description = "result"
  value       = aws_instance.web.public_ip
}
```

##### 4. `sensitive`（機密情報フラグ）

```hcl
output "database_password" {
  description = "データベースの管理者パスワード"
  value       = random_password.db_password.result
  sensitive   = true  # ← 機密情報であることを示す
}
```

**説明：**

- この出力が機密情報であることを示す
- `true` にすると、`terraform apply` の出力で値が隠される
- デフォルトは `false`

**sensitive = true の効果：**

```hcl
output "db_password" {
  value     = random_password.db_password.result
  sensitive = true
}
```

**terraform apply実行時の表示：**

```
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

db_password = <sensitive>
```

**値を確認する方法：**

```bash
# 機密情報を含む出力を表示
terraform output db_password

# JSON形式で表示
terraform output -json db_password

# すべての出力をJSON形式で表示（機密情報も含む）
terraform output -json
```

**sensitiveにすべき出力：**

```hcl
# パスワード
output "admin_password" {
  value     = random_password.admin.result
  sensitive = true
}

# APIキー
output "api_key" {
  value     = aws_api_gateway_api_key.main.value
  sensitive = true
}

# 秘密鍵
output "private_key_pem" {
  value     = tls_private_key.main.private_key_pem
  sensitive = true
}

# アクセストークン
output "access_token" {
  value     = data.aws_secretsmanager_secret_version.token.secret_string
  sensitive = true
}
```

**重要な注意点：**

```
sensitive = true にしても：
✗ terraform.tfstate ファイルには平文で保存される
✗ 完全に安全ではない
✓ コンソール出力が隠されるだけ

本当に安全に扱うには：
✓ terraform output -json を使って必要な時だけ取得
✓ state ファイルへのアクセスを制限
✓ リモートバックエンドで暗号化
```

##### 5. `depends_on`（依存関係の明示）

```hcl
output "instance_ip_addr" {
  description = "インスタンスのIPアドレス"
  value       = aws_instance.web.private_ip

  depends_on = [
    # セキュリティグループルールが作成されるまで待つ
    aws_security_group_rule.allow_ssh
  ]
}
```

**説明：**

- この出力が依存する他のリソースを明示的に指定
- 通常は自動的に依存関係が解決されるが、明示が必要な場合もある
- リストで複数のリソースを指定可能

**depends_onが必要な場面：**

```hcl
# 例：セキュリティグループルールが完全に設定されてからIPを公開
resource "aws_instance" "web" {
  # ...
}

resource "aws_security_group_rule" "allow_ssh" {
  # ...
}

output "instance_ip" {
  description = "SSH接続可能なインスタンスIP"
  value       = aws_instance.web.public_ip

  # セキュリティグループルールが作成されるまで待つ
  depends_on = [
    aws_security_group_rule.allow_ssh
  ]
}
```

**depends_on使用時の推奨事項：**

```hcl
output "instance_ip_addr" {
  value = aws_instance.web.private_ip

  depends_on = [
    aws_security_group_rule.local_access,
  ]

  # なぜdepends_onが必要か説明するコメントを追加
  # 理由：セキュリティグループルールが作成される前にIPを公開すると、
  #       接続できない状態で情報が表示されるため
}
```

##### 6. `precondition`（事前条件チェック）

```hcl
output "instance_public_ip" {
  description = "EC2インスタンスのパブリックIP"
  value       = aws_instance.web.public_ip

  precondition {
    condition     = aws_instance.web.public_ip != ""
    error_message = "インスタンスにパブリックIPが割り当てられていません。"
  }
}
```

**説明：**

- 出力を表示する前に満たすべき条件を指定
- `condition` が `false` の場合、`error_message` を表示してエラーになる
- 複数の `precondition` ブロックを指定可能

**preconditionの構造：**

```hcl
output "出力名" {
  value = 値

  precondition {
    condition     = チェックする条件（trueまたはfalseを返す式）
    error_message = "条件が満たされない時のエラーメッセージ"
  }
}
```

**実用例1：値が空でないことを確認**

```hcl
output "database_endpoint" {
  description = "データベース接続エンドポイント"
  value       = aws_db_instance.main.endpoint

  precondition {
    condition     = aws_db_instance.main.endpoint != ""
    error_message = "データベースエンドポイントが取得できません。"
  }
}
```

**実用例2：リソースの状態を確認**

```hcl
output "instance_public_ip" {
  description = "EC2インスタンスのパブリックIP"
  value       = aws_instance.web.public_ip

  precondition {
    condition     = aws_instance.web.instance_state == "running"
    error_message = "インスタンスが起動していません。現在の状態：${aws_instance.web.instance_state}"
  }
}
```

**実用例3：セキュリティ設定を確認**

```hcl
output "instance_public_ip" {
  description = "WebサーバーのパブリックIP"
  value       = aws_instance.web.public_ip

  # HTTPまたはHTTPSポートが開いているか確認
  precondition {
    condition = length([
      for rule in aws_security_group.web.ingress :
      rule if rule.to_port == 80 || rule.to_port == 443
    ]) > 0
    error_message = "セキュリティグループにHTTP（ポート80）またはHTTPS（ポート443）のインバウンドルールが必要です。"
  }
}
```

**実用例4：複数の事前条件**

```hcl
output "database_connection_info" {
  description = "データベース接続情報"
  value = {
    endpoint = aws_db_instance.main.endpoint
    port     = aws_db_instance.main.port
    database = aws_db_instance.main.db_name
  }

  # 条件1：データベースが利用可能な状態
  precondition {
    condition     = aws_db_instance.main.status == "available"
    error_message = "データベースが利用可能な状態ではありません。現在の状態：${aws_db_instance.main.status}"
  }

  # 条件2：エンドポイントが取得できている
  precondition {
    condition     = aws_db_instance.main.endpoint != ""
    error_message = "データベースエンドポイントが取得できません。"
  }

  # 条件3：バックアップが有効（本番環境の場合）
  precondition {
    condition = var.environment != "production" || (
      aws_db_instance.main.backup_retention_period > 0
    )
    error_message = "本番環境ではデータベースのバックアップを有効にする必要があります。"
  }
}
```

**preconditionで使える関数：**

```hcl
# length()：リストやマップの長さ
precondition {
  condition = length(aws_subnet.public) >= 2
}

# contains()：リストに値が含まれるか
precondition {
  condition = contains(["running", "pending"], aws_instance.web.instance_state)
}

# can()：式がエラーにならないか
precondition {
  condition = can(aws_instance.web.public_ip)
}

# alltrue()：すべての条件がtrueか
precondition {
  condition = alltrue([
    aws_instance.web.public_ip != "",
    aws_instance.web.instance_state == "running"
  ])
}
```

#### outputブロックの完全な例

```hcl
output "instance_connection_info" {
  # 説明：この出力の目的を明確に
  description = <<-EOT
    EC2インスタンスへの接続情報。
    - public_ip: SSH接続に使用
    - ssh_command: そのままコピー＆ペーストで接続可能
    - instance_id: AWSコンソールでの確認用
  EOT

  # 値：マップ形式で複数の情報をまとめる
  value = {
    instance_id  = aws_instance.web.id
    public_ip    = aws_instance.web.public_ip
    ssh_command  = "ssh -i keypair.pem ec2-user@${aws_instance.web.public_ip}"
    console_url  = "https://console.aws.amazon.com/ec2/v2/home?region=${var.aws_region}#Instances:instanceId=${aws_instance.web.id}"
  }

  # 機密情報ではない
  sensitive = false

  # 依存関係：セキュリティグループルールが作成されてから表示
  depends_on = [
    aws_security_group_rule.allow_ssh
  ]

  # 事前条件1：パブリックIPが割り当てられている
  precondition {
    condition     = aws_instance.web.public_ip != ""
    error_message = "インスタンスにパブリックIPが割り当てられていません。"
  }

  # 事前条件2：インスタンスが起動している
  precondition {
    condition     = aws_instance.web.instance_state == "running"
    error_message = "インスタンスが起動していません。現在の状態：${aws_instance.web.instance_state}"
  }

  # 事前条件3：SSHポートが開いている
  precondition {
    condition = length([
      for rule in aws_security_group.web.ingress :
      rule if rule.to_port == 22
    ]) > 0
    error_message = "セキュリティグループにSSH（ポート22）のインバウンドルールが必要です。"
  }
}
```

### 実例：様々な出力パターン

#### 実例1：基本的な出力

```hcl
# VPC IDの出力
output "vpc_id" {
  description = "作成したVPCのID"
  value       = aws_vpc.main.id
}

# 複数のサブネットIDを出力
output "public_subnet_ids" {
  description = "パブリックサブネットのIDリスト"
  value       = aws_subnet.public[*].id
}

# 計算された値を出力
output "vpc_cidr" {
  description = "VPCのCIDRブロック"
  value       = aws_vpc.main.cidr_block
}
```

#### 実例2：複雑な値の出力

```hcl
# マップ形式で出力
output "instance_info" {
  description = "EC2インスタンスの詳細情報"
  value = {
    id         = aws_instance.web.id
    public_ip  = aws_instance.web.public_ip
    private_ip = aws_instance.web.private_ip
    type       = aws_instance.web.instance_type
  }
}

# リスト形式で出力
output "subnet_cidrs" {
  description = "すべてのサブネットのCIDRブロック"
  value = [
    for subnet in aws_subnet.public :
    subnet.cidr_block
  ]
}

# 条件付き出力
output "ssh_command" {
  description = "SSHコマンド（パブリックIPがある場合のみ）"
  value = aws_instance.web.public_ip != "" ? (
    "ssh -i keypair.pem ec2-user@${aws_instance.web.public_ip}"
  ) : "パブリックIPが割り当てられていません"
}
```

#### 実例3：機密情報の出力

```hcl
# 機密情報を含む出力
output "database_password" {
  description = "データベースの管理者パスワード"
  value       = random_password.db_password.result
  sensitive   = true  # ← これにより通常の出力では表示されない
}

# 使用例
resource "random_password" "db_password" {
  length  = 16
  special = true
}

resource "aws_db_instance" "main" {
  password = random_password.db_password.result
  # ...
}
```

#### 実例4：他のモジュールから参照される出力

```hcl
# ネットワークモジュールの出力
output "network_info" {
  description = "ネットワーク情報（他のモジュールから参照可能）"
  value = {
    vpc_id             = aws_vpc.main.id
    public_subnet_ids  = aws_subnet.public[*].id
    private_subnet_ids = aws_subnet.private[*].id
    security_group_id  = aws_security_group.main.id
  }
}

# この出力を別のモジュールから参照
# module "application" {
#   source = "./modules/application"
#
#   network_info = module.network.network_info
# }
```

#### 実例5：条件チェック付き出力

```hcl
# セキュリティグループの検証
output "instance_public_ip" {
  description = "EC2インスタンスのパブリックIP"
  value       = aws_instance.web.public_ip

  # 事前条件：セキュリティグループにHTTPまたはHTTPSルールが存在すること
  precondition {
    condition = length([
      for rule in aws_security_group.web.ingress :
      rule if rule.to_port == 80 || rule.to_port == 443
    ]) > 0
    error_message = "セキュリティグループにHTTP（ポート80）またはHTTPS（ポート443）のルールが必要です。"
  }
}

# データベース接続情報の検証
output "database_endpoint" {
  description = "データベース接続エンドポイント"
  value       = aws_db_instance.main.endpoint

  # 事前条件：データベースが利用可能な状態であること
  precondition {
    condition     = aws_db_instance.main.status == "available"
    error_message = "データベースが利用可能な状態ではありません。"
  }
}
```

### outputブロックの活用場面

**1. デバッグ・確認用：**

```hcl
# リソースが正しく作成されたか確認
output "debug_info" {
  description = "デバッグ用の情報"
  value = {
    vpc_id         = aws_vpc.main.id
    subnet_count   = length(aws_subnet.public)
    instance_state = aws_instance.web.instance_state
  }
}
```

**2. 接続情報の共有：**

```hcl
# チームメンバーに共有する情報
output "connection_info" {
  description = "サービスへの接続情報"
  value = {
    url          = "https://${aws_instance.web.public_ip}"
    ssh_command  = "ssh -i keypair.pem ec2-user@${aws_instance.web.public_ip}"
    database_url = aws_db_instance.main.endpoint
  }
}
```

**3. 自動化ツールへの情報提供：**

```hcl
# CI/CDパイプラインで使用する情報
output "deployment_info" {
  description = "デプロイメント情報（JSON形式で取得可能）"
  value = {
    instance_ids      = aws_instance.web[*].id
    load_balancer_dns = aws_lb.main.dns_name
    target_group_arn  = aws_lb_target_group.main.arn
  }
}
```

**4. 他のTerraformモジュールへの情報共有：**

```hcl
# 親モジュールに公開する情報
output "module_outputs" {
  description = "このモジュールが公開する情報"
  value = {
    vpc_id            = aws_vpc.main.id
    subnet_ids        = aws_subnet.public[*].id
    security_group_id = aws_security_group.main.id
  }
}
```

## 3つのブロックの連携実例

### 実例1：完全な設定例

```hcl
# =============================================================================
# 変数定義（variable）- 外部から受け取る値
# =============================================================================
variable "project_name" {
  description = "プロジェクト名"
  type        = string
  default     = "my-web-app"
}

variable "environment" {
  description = "環境名（dev, staging, production）"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "environment は dev、staging、production のいずれかである必要があります。"
  }
}

variable "vpc_cidr" {
  description = "VPCのCIDRブロック"
  type        = string
  default     = "10.0.0.0/16"
}

variable "enable_monitoring" {
  description = "CloudWatchモニタリングを有効にするか"
  type        = bool
  default     = true
}

# =============================================================================
# ローカル値定義（locals）- 計算や繰り返し使う値
# =============================================================================
locals {
  # 共通名の生成
  common_name = "${var.project_name}-${var.environment}"

  # 環境に応じたインスタンスタイプ
  instance_type = var.environment == "production" ? "t4g.medium" : "t4g.small"

  # アベイラビリティゾーン
  availability_zones = slice(
    data.aws_availability_zones.available.names,
    0,
    2
  )

  # サブネットCIDRの計算
  public_subnet_cidrs = [
    for i in range(length(local.availability_zones)) :
    cidrsubnet(var.vpc_cidr, 8, i)
  ]

  private_subnet_cidrs = [
    for i in range(length(local.availability_zones)) :
    cidrsubnet(var.vpc_cidr, 8, i + 10)
  ]

  # 共通タグ
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# =============================================================================
# データソース
# =============================================================================
data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-kernel-6.1-arm64"]
  }
}

# =============================================================================
# リソース作成
# =============================================================================
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(
    local.common_tags,
    { Name = "${local.common_name}-vpc" }
  )
}

resource "aws_subnet" "public" {
  count = length(local.public_subnet_cidrs)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = local.public_subnet_cidrs[count.index]
  availability_zone       = local.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(
    local.common_tags,
    {
      Name = "${local.common_name}-public-${count.index + 1}"
      Type = "Public"
    }
  )
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = local.instance_type
  subnet_id     = aws_subnet.public[0].id
  monitoring    = var.enable_monitoring

  tags = merge(
    local.common_tags,
    { Name = "${local.common_name}-web-instance" }
  )
}

# =============================================================================
# 出力（output）- 重要な情報を公開
# =============================================================================
output "vpc_info" {
  description = "VPCの情報"
  value = {
    id         = aws_vpc.main.id
    cidr_block = aws_vpc.main.cidr_block
  }
}

output "subnet_info" {
  description = "サブネットの情報"
  value = {
    public_subnet_ids   = aws_subnet.public[*].id
    public_subnet_cidrs = local.public_subnet_cidrs
  }
}

output "instance_info" {
  description = "EC2インスタンスの情報"
  value = {
    id         = aws_instance.web.id
    public_ip  = aws_instance.web.public_ip
    type       = aws_instance.web.instance_type
    monitoring = aws_instance.web.monitoring
  }
}

output "connection_command" {
  description = "SSH接続コマンド"
  value       = "ssh -i keypair.pem ec2-user@${aws_instance.web.public_ip}"
}

output "environment_summary" {
  description = "環境の概要"
  value = {
    project     = var.project_name
    environment = var.environment
    region      = data.aws_availability_zones.available.id
    azs_used    = local.availability_zones
  }
}
```

### 実例2：マルチ環境対応の設定

```hcl
# =============================================================================
# 変数定義
# =============================================================================
variable "environment" {
  type = string
}

# =============================================================================
# 環境別の設定をlocalsで管理
# =============================================================================
locals {
  # 環境別のインスタンス数
  instance_counts = {
    dev        = 1
    staging    = 2
    production = 4
  }

  # 環境別のインスタンスタイプ
  instance_types = {
    dev        = "t4g.micro"
    staging    = "t4g.small"
    production = "t4g.medium"
  }

  # 環境別のバックアップ設定
  backup_enabled = {
    dev        = false
    staging    = false
    production = true
  }

  # 現在の環境の設定を取得
  instance_count  = local.instance_counts[var.environment]
  instance_type   = local.instance_types[var.environment]
  backup_required = local.backup_enabled[var.environment]
}

# =============================================================================
# リソース作成（環境に応じて自動調整）
# =============================================================================
resource "aws_instance" "app" {
  count = local.instance_count

  ami           = data.aws_ami.amazon_linux.id
  instance_type = local.instance_type

  tags = {
    Name        = "app-${count.index + 1}"
    Environment = var.environment
    Backup      = local.backup_required ? "enabled" : "disabled"
  }
}

# =============================================================================
# 環境情報の出力
# =============================================================================
output "environment_config" {
  description = "現在の環境設定"
  value = {
    environment    = var.environment
    instance_count = local.instance_count
    instance_type  = local.instance_type
    backup_enabled = local.backup_required
    instance_ids   = aws_instance.app[*].id
  }
}
```

## ベストプラクティス

### 1. 変数の命名規則

**良い例：**

```hcl
# 明確で具体的な名前
variable "vpc_cidr_block" {
  type = string
}

variable "ec2_instance_type" {
  type = string
}

variable "enable_vpc_flow_logs" {
  type = bool
}
```

**悪い例：**

```hcl
# 曖昧で意味が分からない名前
variable "cidr" {
  type = string
}

variable "type" {
  type = string
}

variable "enable" {
  type = bool
}
```

### 2. 必ず説明を追加

**良い例：**

```hcl
variable "instance_type" {
  description = "EC2インスタンスのタイプ。本番環境ではt4g.medium以上を推奨。"
  type        = string
  default     = "t4g.small"
}

locals {
  # サブネット数は常にAZ数と一致させる
  subnet_count = length(local.availability_zones)
}

output "database_endpoint" {
  description = "データベースの接続エンドポイント。アプリケーション設定で使用してください。"
  value       = aws_db_instance.main.endpoint
}
```

### 3. デフォルト値の適切な設定

**良い例：**

```hcl
# 一般的な値はデフォルトを設定
variable "aws_region" {
  type    = string
  default = "ap-northeast-1"
}

# 必須の値はデフォルトなし
variable "project_name" {
  description = "プロジェクト名（必須）"
  type        = string
  # default なし
}
```

### 4. バリデーションの活用

**良い例：**

```hcl
# 許可された値のみを受け付ける
variable "environment" {
  type = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment は dev、staging、prod のいずれかを指定してください。"
  }
}

# 値の範囲をチェック
variable "instance_count" {
  type = number

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "instance_count は 1 から 10 の範囲で指定してください。"
  }
}
```

### 5. localsの整理

**良い例：**

```hcl
# 関連する値をグループ化
locals {
  # ネットワーク設定
  vpc_cidr = "10.0.0.0/16"
  az_count = 2
}

locals {
  # アプリケーション設定
  app_name = "${var.project_name}-${var.environment}"
  app_port = 8080
}

locals {
  # 計算された値
  subnet_cidrs = [
    for i in range(local.az_count) :
    cidrsubnet(local.vpc_cidr, 8, i)
  ]
}
```

### 6. sensitiveの適切な使用

**良い例：**

```hcl
# 機密情報は必ずsensitiveに
variable "database_password" {
  type      = string
  sensitive = true
}

output "db_password" {
  value     = var.database_password
  sensitive = true
}
```

**悪い例：**

```hcl
# 機密情報なのにsensitiveがない
variable "database_password" {
  type = string
}

output "db_password" {
  value = var.database_password
}
```

### 7. 出力の整理

**良い例：**

```hcl
# 関連する情報をマップでまとめる
output "network_info" {
  description = "ネットワーク関連の情報"
  value = {
    vpc_id          = aws_vpc.main.id
    public_subnets  = aws_subnet.public[*].id
    private_subnets = aws_subnet.private[*].id
  }
}

# 接続情報は別にまとめる
output "connection_info" {
  description = "接続情報"
  value = {
    instance_ip = aws_instance.web.public_ip
    ssh_command = "ssh ec2-user@${aws_instance.web.public_ip}"
  }
}
```

**悪い例：**

```hcl
# バラバラで整理されていない
output "vpc" {
  value = aws_vpc.main.id
}

output "subnet1" {
  value = aws_subnet.public[0].id
}

output "subnet2" {
  value = aws_subnet.public[1].id
}

output "ip" {
  value = aws_instance.web.public_ip
}
```

### 8. 型の明示

**良い例：**

```hcl
# 型を明示的に指定
variable "instance_count" {
  type    = number
  default = 2
}

variable "availability_zones" {
  type    = list(string)
  default = ["ap-northeast-1a", "ap-northeast-1c"]
}

variable "tags" {
  type = map(string)
  default = {
    Environment = "dev"
  }
}
```

**悪い例：**

```hcl
# 型を指定しない（推奨されない）
variable "instance_count" {
  default = 2
}
```

## よくある間違いと対策

### 間違い1：variableとlocalsの混同

**間違った例：**

```hcl
# 計算結果をvariableで定義（これは間違い）
variable "full_name" {
  type    = string
  default = "${var.project_name}-${var.environment}"  # エラー！
}
```

**正しい例：**

```hcl
# 計算結果はlocalsで定義
locals {
  full_name = "${var.project_name}-${var.environment}"
}
```

### 間違い2：循環参照

**間違った例：**

```hcl
locals {
  value_a = local.value_b + 1
  value_b = local.value_a + 1  # エラー：循環参照
}
```

**正しい例：**

```hcl
locals {
  base_value = 10
  value_a    = local.base_value + 1
  value_b    = local.base_value + 2
}
```

### 間違い3：output値の参照ミス

**間違った例：**

```hcl
# outputの値を同じモジュール内で参照（できない）
output "vpc_id" {
  value = aws_vpc.main.id
}

resource "aws_subnet" "public" {
  vpc_id = output.vpc_id  # エラー！
}
```

**正しい例：**

```hcl
# リソースの属性を直接参照
resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id
}

# outputはモジュール外に公開するため
output "vpc_id" {
  value = aws_vpc.main.id
}
```

### 間違い4：機密情報の露出

**間違った例：**

```hcl
# パスワードがsensitiveでない
variable "db_password" {
  type    = string
  default = "password123"  # ハードコーディングも NG！
}

output "password" {
  value = var.db_password  # ログに表示されてしまう
}
```

**正しい例：**

```hcl
# パスワードはsensitive設定
variable "db_password" {
  type      = string
  sensitive = true
  # defaultなし = 必ず外部から指定
}

output "password" {
  value     = var.db_password
  sensitive = true  # 出力も sensitive に
}
```

### 間違い5：型の不一致

**間違った例：**

```hcl
variable "instance_count" {
  type    = number
  default = "2"  # 文字列を渡している
}
```

**正しい例：**

```hcl
variable "instance_count" {
  type    = number
  default = 2  # 数値で指定
}
```

### 間違い6：locals内での他locals参照順序

**間違いやすい例：**

```hcl
locals {
  # value_b がまだ定義されていない
  value_a = local.value_b + 1  # エラーになることがある
  value_b = 10
}
```

**正しい例：**

```hcl
locals {
  # 依存元を先に定義
  value_b = 10
  value_a = local.value_b + 1
}

# または、別のlocalsブロックに分ける
locals {
  value_b = 10
}

locals {
  value_a = local.value_b + 1
}
```

### 間違い7：outputのpreconditionで誤った条件

**間違った例：**

```hcl
output "instance_ip" {
  value = aws_instance.web.public_ip

  # 誤った条件：instance_stateは文字列
  precondition {
    condition     = aws_instance.web.instance_state  # エラー！
    error_message = "インスタンスが起動していません。"
  }
}
```

**正しい例：**

```hcl
output "instance_ip" {
  value = aws_instance.web.public_ip

  # 正しい条件：明示的に比較
  precondition {
    condition     = aws_instance.web.instance_state == "running"
    error_message = "インスタンスが起動していません。"
  }
}
```

## まとめ

### 3つのブロックの使い分け

```
variable（変数）:
  ✓ 外部から値を受け取る
  ✓ ユーザーが変更可能
  ✓ terraform.tfvarsで値を設定
  ✓ 環境によって変わる値
  ✓ 型指定とバリデーションで安全性を確保

locals（ローカル値）:
  ✓ 内部で計算する
  ✓ 繰り返し使う値を定義
  ✓ 複雑なロジックを整理
  ✓ 読み取り専用
  ✓ DRY原則（Don't Repeat Yourself）を実現

output（出力）:
  ✓ 結果を外部に公開
  ✓ 重要な情報を表示
  ✓ 他のモジュールから参照可能
  ✓ 自動化ツールへの情報提供
  ✓ preconditionで安全性を確保
```

### 重要なポイント

1. **variable は外部からの入力**

- デフォルト値で柔軟性を持たせる
- バリデーションで値を検証
- 機密情報は sensitive に
- 型を必ず明示

2. **locals は内部での計算**

- 繰り返しを避ける
- 複雑なロジックを整理
- 関連する値をグループ化
- 計算の段階を明確に

3. **output は外部への公開**

- 重要な情報のみを出力
- 説明を必ず追加
- 関連情報はまとめる
- 機密情報はsensitiveに
- preconditionで事前チェック

4. **3つのブロックの連携**

- variable → locals → resource → output の流れ
- 依存関係を意識
- 段階的に値を変換・計算
- 各ブロックの役割を明確に

5. **ベストプラクティス**

- 明確な命名規則
- 型の明示
- バリデーションの活用
- 適切なコメント
- 機密情報の保護
- 関連する値のグループ化
