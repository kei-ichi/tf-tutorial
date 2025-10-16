# Terraformのファイル・フォルダ構成完全ガイド

## この章で学ぶこと

Terraformプロジェクトのファイルとフォルダの整理方法を学習します。適切な構成により、コードの保守性、可読性、チーム開発の効率が大幅に向上します。

**学習内容：**

- 基本的なファイル構成
- 各ファイルの役割と命名規則
- プロジェクト規模別の構成パターン
- モジュール構成の概要
- ベストプラクティス
- 実践的なリファクタリング例

## なぜファイルを分割するのか

### 問題：すべてが1つのファイル

**現在のコード（main.tfのみ）：**

```
terraform-practice/
└── main.tf (500行以上)
    ├── terraform設定
    ├── provider設定
    ├── 変数定義
    ├── データソース
    ├── locals定義
    ├── リソース定義
    └── 出力定義
```

**問題点：**

```
1. コードが長すぎて読みにくい
   - 目的のコードを探すのに時間がかかる
   - スクロールが大変

2. 変更が困難
   - どこに何があるか分からない
   - 修正時に他の部分を誤って変更するリスク

3. チーム開発で衝突しやすい
   - 複数人が同じファイルを編集
   - Gitのコンフリクトが頻発

4. 再利用が困難
   - 他のプロジェクトに一部だけ使いたい
   - コピー＆ペーストになる
```

### 解決策：ファイルを分割

**例えて言うなら：** 本の章立て

```
本を書く時：
間違い：すべてを1つの章に書く
   - 読みにくい、探しにくい

正しい：章ごとに分ける
   - 第1章：はじめに
   - 第2章：基礎知識
   - 第3章：実践
   - 読みやすい、探しやすい
```

**Terraformでも同じ：**

```
terraform-practice/
├── versions.tf      # バージョン設定
├── backend.tf       # 状態管理
├── providers.tf     # プロバイダー設定
├── variables.tf     # 変数定義
├── data.tf          # データソース
├── main.tf          # リソース定義
└── outputs.tf       # 出力定義
```

**メリット：**

- 目的のコードがすぐ見つかる
- 変更が安全にできる
- チーム開発がスムーズ
- 再利用しやすい

## 基本的なファイル構成

### 最小構成（小規模プロジェクト）

```
terraform-practice/
├── main.tf           # リソース定義
├── variables.tf      # 変数定義
├── outputs.tf        # 出力定義
└── terraform.tfvars  # 変数の値
```

**各ファイルの役割：**

| ファイル名       | 役割           | 必須       |
| ---------------- | -------------- | ---------- |
| main.tf          | リソースの定義 | 必須       |
| variables.tf     | 変数の宣言     | 推奨       |
| outputs.tf       | 出力の定義     | 推奨       |
| terraform.tfvars | 変数の値       | オプション |

### 推奨構成（一般的なプロジェクト）

```
terraform-practice/
├── versions.tf       # Terraformバージョン設定
├── backend.tf        # 状態管理設定
├── providers.tf      # プロバイダー設定
├── variables.tf      # 変数定義
├── data.tf           # データソース定義
├── main.tf           # メインのリソース定義
├── outputs.tf        # 出力定義
├── terraform.tfvars  # 変数の値（開発環境）
└── .gitignore        # Gitで除外するファイル
```

### 完全構成（大規模プロジェクト）

```
terraform-practice/
├── versions.tf
├── backend.tf
├── providers.tf
├── variables.tf
├── data.tf
├── main.tf           # モジュール呼び出しのみ
├── outputs.tf
├── terraform.tfvars
├── environments/
│   ├── dev.tfvars
│   ├── staging.tfvars
│   └── prod.tfvars
├── modules/          # 機能ごとにモジュール化
│   ├── network/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── database/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── .gitignore
└── README.md
```

## 役割別ファイル分割の完全ガイド

### 役割別に分割とは

**基本的な考え方：**

```
会社の組織：
├── 総務部    → 会社全体の設定
├── 人事部    → 人の管理
├── 営業部    → 営業活動
└── 経理部    → お金の管理

Terraformの構成：
├── versions.tf   → バージョン管理
├── backend.tf    → 状態管理の設定
├── providers.tf  → AWSへの接続設定
├── variables.tf  → 変数の定義
├── data.tf       → 既存情報の取得
├── main.tf       → 実際のリソース作成
└── outputs.tf    → 結果の出力
```

**なぜ役割別に分けるのか：**

```
理由1：見つけやすい
  「変数を変更したい」 → variables.tf を開く
  「出力を追加したい」 → outputs.tf を開く

理由2：変更しやすい
  プロバイダー設定だけ変更 → providers.tf だけ編集
  他のファイルに影響しない

理由3：チーム開発しやすい
  Aさん：リソース追加（main.tf）
  Bさん：変数追加（variables.tf）
  同時作業でコンフリクトが起きにくい
```

### 現在のコードを分割してみよう

#### 現在の状態

**現在のmain.tf（すべてが1つのファイル）：**

```hcl
# main.tf（現在：すべてがここに入っている）

terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "3.7.2"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.1"
    }
  }
}

provider "aws" {
  region = var.aws_region
  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project_name
      ManagedBy   = "Terraform"
    }
  }
}

provider "random" {}
provider "tls" {}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-northeast-1"
}

# ... 他の変数定義 ...

data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  availability_zones = slice(data.aws_availability_zones.available.names, 0, 2)
  # ... 他のlocals ...
}

resource "aws_vpc" "main" {
  # ... VPC定義 ...
}

# ... 他のリソース ...

output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

# ... 他の出力 ...
```

**問題点を見てみましょう：**

```
現在のmain.tf の行数: 約500行

内訳：
- terraform ブロック: 20行
- provider ブロック: 15行
- variable ブロック: 50行
- data ブロック: 10行
- locals ブロック: 30行
- resource ブロック: 350行
- output ブロック: 25行

問題：
- スクロールが大変
- 変数を探すのに時間がかかる
- 複数人で編集するとコンフリクト
- どこに何があるか分かりにくい
```

#### 分割後の目標

```
terraform-practice/
├── versions.tf       # 20行（terraform ブロック）
├── backend.tf        # 10行（backend ブロック）※新規追加
├── providers.tf      # 15行（provider ブロック）
├── variables.tf      # 50行（variable ブロック）
├── data.tf           # 40行（data ブロック + locals）
├── main.tf           # 350行（resource ブロック）
├── outputs.tf        # 25行（output ブロック）
└── terraform.tfvars  # 10行（変数の値）

メリット：
- 各ファイルが短い（50行以下が多い）
- 目的のコードがすぐ見つかる
- 役割が明確
- チーム開発がスムーズ
```

## 各ファイルの詳細説明と分割方法

### 1. versions.tf（バージョン管理）

#### 役割

```
versions.tf の役割：
1. Terraformのバージョンを指定
2. 使用するプロバイダーとそのバージョンを指定
3. チーム全体で同じバージョンを使用するための保証

なぜ分離するのか：
- バージョン情報を一箇所で管理
- 更新時に探しやすい
- バージョン管理の方針が明確
```

#### 現在のコードから抜き出す

**現在のmain.tfから、この部分を抜き出します：**

```hcl
# main.tf（現在）

terraform {                        # この部分を
  required_version = ">= 1.0"      # versions.tf に
                                   # 移動します
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "3.7.2"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.1"
    }
  }
}
```

#### 新しいversions.tfファイルを作成

```hcl
# versions.tf（新規作成）

# =============================================================================
# Terraformとプロバイダーのバージョン管理
# =============================================================================

terraform {
  # Terraformの最小バージョン
  # チームメンバー全員がこのバージョン以上を使用する必要がある
  required_version = ">= 1.0"

  # 使用するプロバイダーとそのバージョンを定義
  required_providers {
    # AWSプロバイダー
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"  # 6.x.x系の最新を使用
    }

    # Randomプロバイダー（ランダム値生成用）
    random = {
      source  = "hashicorp/random"
      version = "~> 3.7"
    }

    # TLSプロバイダー（証明書・暗号化用）
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.1"
    }
  }
}
```

#### バージョン指定の意味

```hcl
# バージョン指定の方法

version = "6.0.0"        # 完全一致（6.0.0のみ）
version = ">= 6.0.0"     # 以上（6.0.0以上のすべて）
version = "~> 6.0"       # 悲観的バージョン制約
                         # 6.0.0 <= version < 7.0.0
                         # これが最も推奨される

version = "~> 6.0.0"     # より厳密な悲観的バージョン制約
                         # 6.0.0 <= version < 6.1.0
```

**推奨：悲観的バージョン制約（~>）を使用**

```hcl
# 推奨
version = "~> 6.0"

理由：
1. マイナーバージョンアップは自動で取得
   6.0.0 → 6.1.0 → 6.2.0（自動的に最新）

2. メジャーバージョンアップは防ぐ
   6.x.x → 7.0.0（破壊的変更を防ぐ）

3. セキュリティパッチは自動適用
   6.0.0 → 6.0.1（バグ修正・セキュリティ）
```

### 2. backend.tf（状態管理の設定）

#### 役割

```
backend.tf の役割：
1. terraform.tfstate の保存場所を指定
2. チーム開発での状態共有
3. 状態のロック機構（同時実行の防止）

なぜ分離するのか：
- 重要な設定なので目立たせる
- 環境ごとに異なる可能性がある
- セキュリティ上重要な設定
```

#### backend.tfファイルを作成

```hcl
# backend.tf（新規作成）

# =============================================================================
# バックエンド設定（状態ファイルの保存場所）
# =============================================================================

# 注意：この設定は学習段階ではコメントアウトしておく
# チーム開発を始める時に有効化する

# terraform {
#   backend "s3" {
#     # 状態ファイルを保存するS3バケット
#     bucket = "my-terraform-state-bucket"
#
#     # S3内のファイルパス
#     key    = "terraform-practice/terraform.tfstate"
#
#     # リージョン
#     region = "ap-northeast-1"
#
#     # 状態ロック用のDynamoDBテーブル（同時実行防止）
#     dynamodb_table = "terraform-state-lock"
#
#     # 保存時に暗号化
#     encrypt = true
#   }
# }
```

**ローカル開発（現在）：**

```hcl
# backend.tf（ローカル開発用）

# =============================================================================
# ローカルバックエンド（デフォルト）
# =============================================================================

# バックエンド設定なし = ローカルに terraform.tfstate を保存
# これは学習・開発段階での推奨設定

# ローカルバックエンドの動作：
# - terraform.tfstate がプロジェクトディレクトリに作成される
# - シンプルで理解しやすい
# - 個人開発に最適

# チーム開発に移行する時：
# 1. S3バケットとDynamoDBテーブルを作成
# 2. 上記のS3バックエンド設定のコメントを外す
# 3. terraform init -migrate-state を実行
```

**チーム開発時：**

```hcl
# backend.tf（チーム開発用）

terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state"
    key            = "projects/terraform-practice/terraform.tfstate"
    region         = "ap-northeast-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true

    # アクセス制御
    acl = "private"
  }
}
```

#### なぜbackend.tfを分離するのか

```
理由1：重要性を明確にする
  状態ファイルの管理は最も重要
  専用ファイルにすることで重要性を強調

理由2：環境ごとに異なる可能性
  開発環境：ローカルバックエンド
  本番環境：S3バックエンド

理由3：セキュリティ設定の可視化
  暗号化設定
  アクセス制御
  ロック機構
```

### 3. providers.tf（プロバイダー設定）

#### 役割

```
providers.tf の役割：
1. AWSへの接続設定
2. デフォルトタグの設定
3. 複数プロバイダーの管理（AWS、Random、TLSなど）

なぜ分離するのか：
- 接続設定を一箇所で管理
- デフォルトタグを明確に
- プロバイダーごとの設定が分かりやすい
```

#### 現在のコードから抜き出す

**現在のmain.tfから、この部分を抜き出します：**

```hcl
# main.tf（現在）

provider "aws" {              # この部分を
  region = var.aws_region     # providers.tf に
  default_tags {              # 移動します
    tags = {
      Environment = var.environment
      Project     = var.project_name
      ManagedBy   = "Terraform"
    }
  }
}

provider "random" {}          # これも
provider "tls" {}             # これも
```

#### 新しいproviders.tfファイルを作成

```hcl
# providers.tf（新規作成）

# =============================================================================
# AWSプロバイダー設定
# =============================================================================

provider "aws" {
  # リージョンの指定（変数で柔軟に変更可能）
  region = var.aws_region

  # すべてのAWSリソースに自動的に付与されるタグ
  # これによりタグの付け忘れを防ぐ
  default_tags {
    tags = {
      Environment = var.environment  # 環境名（dev/staging/prod）
      Project     = var.project_name # プロジェクト名
      ManagedBy   = "Terraform"      # Terraform管理であることを明示
    }
  }
}

# =============================================================================
# Randomプロバイダー設定
# =============================================================================

provider "random" {
  # Randomプロバイダーには特別な設定パラメータは不要
  # ランダムな文字列、数値、パスワードなどを生成するために使用
}

# =============================================================================
# TLSプロバイダー設定
# =============================================================================

provider "tls" {
  # TLSプロバイダーには特別な設定パラメータは不要
  # 証明書、秘密鍵、公開鍵などを生成するために使用
}
```

#### default_tagsの重要性

```hcl
# default_tags の効果

# default_tags を使用した場合
provider "aws" {
  default_tags {
    tags = {
      Environment = "development"
      ManagedBy   = "Terraform"
    }
  }
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  # タグを書かなくても...
}

# 実際に作成されるVPC：
# Tags:
#   Environment = "development"  自動的に追加
#   ManagedBy   = "Terraform"    自動的に追加

# default_tags を使用しない場合
provider "aws" {}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  # 毎回書く必要がある（書き忘れのリスク）
  tags = {
    Environment = "development"
    ManagedBy   = "Terraform"
  }
}
```

#### 複数リージョンを使用する場合

```hcl
# providers.tf（複数リージョン）

# デフォルトプロバイダー（東京リージョン）
provider "aws" {
  region = "ap-northeast-1"

  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project_name
      ManagedBy   = "Terraform"
    }
  }
}

# 追加プロバイダー（大阪リージョン）
provider "aws" {
  alias  = "osaka"  # エイリアス（別名）を付ける
  region = "ap-northeast-3"

  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project_name
      ManagedBy   = "Terraform"
      Region      = "osaka"  # 追加のタグ
    }
  }
}

# 使用例
resource "aws_vpc" "tokyo" {
  # デフォルトプロバイダー（東京）を使用
  cidr_block = "10.0.0.0/16"
}

resource "aws_vpc" "osaka" {
  # エイリアスを指定して大阪リージョンを使用
  provider   = aws.osaka
  cidr_block = "10.1.0.0/16"
}
```

### 4. variables.tf（変数定義）

#### 役割

```
variables.tf の役割：
1. すべての変数を定義（型、説明、デフォルト値）
2. バリデーションルールの設定
3. 変数の仕様を文書化

なぜ分離するのか：
- 変数の一覧が一目瞭然
- 新しい変数を追加する場所が明確
- ドキュメントとしての役割
```

#### 現在のコードから抜き出す

**現在のmain.tfから、この部分を抜き出します：**

```hcl
# main.tf（現在）

variable "aws_region" {           # これらの
  description = "AWS region"      # variable ブロックを
  type        = string            # すべて
  default     = "ap-northeast-1"  # variables.tf に
}                                 # 移動します

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "development"
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "terraform-practice"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.88.0.0/16"
}
```

#### 新しいvariables.tfファイルを作成

```hcl
# variables.tf（新規作成）

# =============================================================================
# 基本設定
# =============================================================================

variable "aws_region" {
  description = "AWSリージョン。リソースを作成する地理的な場所を指定します。"
  type        = string
  default     = "ap-northeast-1"  # 東京リージョン
}

variable "environment" {
  description = "環境名。development、staging、productionのいずれかを指定してください。"
  type        = string
  default     = "development"

  # バリデーション：許可された値のみ受け付ける
  validation {
    condition = contains(
      ["development", "staging", "production"],
      var.environment
    )
    error_message = "environmentはdevelopment、staging、productionのいずれかである必要があります。"
  }
}

variable "project_name" {
  description = "プロジェクト名。すべてのリソース名の接頭辞として使用されます。"
  type        = string
  default     = "terraform-practice"

  # バリデーション：英数字とハイフンのみ許可
  validation {
    condition     = can(regex("^[a-z0-9-]+$", var.project_name))
    error_message = "project_nameは小文字の英数字とハイフンのみ使用できます。"
  }
}

# =============================================================================
# ネットワーク設定
# =============================================================================

variable "vpc_cidr" {
  description = "VPCのCIDRブロック。プライベートIPアドレスの範囲を指定します。"
  type        = string
  default     = "10.88.0.0/16"

  # バリデーション：有効なCIDR形式か確認
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "有効なCIDR形式を指定してください（例：10.0.0.0/16）。"
  }
}
```

#### 変数のグループ化

```hcl
# variables.tf（グループ化の例）

# =============================================================================
# 基本設定
# =============================================================================

variable "aws_region" { }
variable "environment" { }
variable "project_name" { }

# =============================================================================
# ネットワーク設定
# =============================================================================

variable "vpc_cidr" { }
variable "enable_nat_gateway" { }
variable "single_nat_gateway" { }

# =============================================================================
# コンピュート設定
# =============================================================================

variable "instance_type" { }
variable "instance_count" { }
variable "enable_monitoring" { }

# =============================================================================
# データベース設定
# =============================================================================

variable "db_instance_class" { }
variable "db_allocated_storage" { }
variable "db_engine_version" { }
```

#### 良い変数定義の例

```hcl
# 良い例：説明が詳しく、バリデーションあり

variable "instance_type" {
  description = <<-EOT
    EC2インスタンスのタイプ。
    - t4g.micro: 開発環境向け（最小構成）
    - t4g.small: ステージング環境向け
    - t4g.medium: 本番環境向け（推奨）
    - t4g.large: 高負荷の本番環境向け
  EOT
  type        = string
  default     = "t4g.small"

  validation {
    condition = contains(
      ["t4g.micro", "t4g.small", "t4g.medium", "t4g.large"],
      var.instance_type
    )
    error_message = "許可されたインスタンスタイプを指定してください。"
  }
}

# 悪い例：説明が不十分、バリデーションなし

variable "instance_type" {
  description = "instance type"  # 説明が不十分
  type        = string
  default     = "t4g.small"
  # バリデーションなし
}
```

### 5. data.tf（データソース定義）

#### 役割

```
data.tf の役割：
1. 既存のAWSリソース情報を取得
2. AWSから動的に情報を取得
3. 外部データの参照

なぜ分離するのか：
- データ取得と作成を明確に区別
- 外部依存を可視化
- データソースの一覧が分かりやすい
```

#### 現在のコードから抜き出す

**現在のmain.tfから、この部分を抜き出します：**

```hcl
# main.tf（現在）

data "aws_availability_zones" "available" {  # この data ブロックと
  state = "available"                        # 関連する locals ブロックを
}                                             # data.tf に移動します

locals {
  availability_zones = slice(
    data.aws_availability_zones.available.names,
    0,
    2
  )
  public_subnet_cidrs = ["10.88.1.0/24", "10.88.2.0/24"]
  private_subnet_cidrs = ["10.88.11.0/24", "10.88.12.0/24"]
}
```

#### 新しいdata.tfファイルを作成

```hcl
# data.tf（新規作成）

# =============================================================================
# アベイラビリティゾーン情報
# =============================================================================

data "aws_availability_zones" "available" {
  # 利用可能な（正常に動作している）AZのみを取得
  state = "available"

  # 説明：
  # - アベイラビリティゾーン（AZ）はAWSのデータセンターの場所
  # - リージョンごとに複数のAZが存在
  # - 高可用性のために複数のAZにリソースを分散配置する

  # フィルター（オプション）
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]  # デフォルトで利用可能なAZのみ
  }
}

# =============================================================================
# ローカル値（データソースに関連する計算）
# =============================================================================

locals {
  # 使用するアベイラビリティゾーン（最初の2つ）
  availability_zones = slice(
    data.aws_availability_zones.available.names,
    0,
    2
  )

  # 説明：
  # - slice()関数でリストの一部を取得
  # - 引数：(リスト, 開始位置, 終了位置)
  # - 結果例：["ap-northeast-1a", "ap-northeast-1c"]

  # パブリックサブネットのCIDR（インターネットアクセス可能）
  public_subnet_cidrs = [
    "10.88.1.0/24",   # 1つ目のAZ用
    "10.88.2.0/24"    # 2つ目のAZ用
  ]

  # プライベートサブネットのCIDR（内部通信のみ）
  private_subnet_cidrs = [
    "10.88.11.0/24",  # 1つ目のAZ用
    "10.88.12.0/24"   # 2つ目のAZ用
  ]
}
```

#### localsブロックの配置について

**重要な考え方：**

```
locals ブロックの配置には2つの考え方があります：

方法1：data.tf にまとめる（今回の例）
  - データソースと関連する locals を同じファイルに
  - データの取得と加工が一箇所にまとまる
  - 小〜中規模プロジェクト向け

方法2：使用箇所の近くに配置
  - locals を使うリソースの直前に配置
  - コードの流れが分かりやすい
  - 大規模プロジェクト向け

どちらが正しいかは、プロジェクトの規模とチームの好みによります。
```

**方法1：data.tf にまとめる例（今回採用）：**

```hcl
# data.tf

data "aws_availability_zones" "available" {
  state = "available"
}

# データソースに関連する locals
locals {
  availability_zones   = slice(data.aws_availability_zones.available.names, 0, 2)
  public_subnet_cidrs  = ["10.88.1.0/24", "10.88.2.0/24"]
  private_subnet_cidrs = ["10.88.11.0/24", "10.88.12.0/24"]
}
```

**方法2：使用箇所の近くに配置する例：**

```hcl
# data.tf
data "aws_availability_zones" "available" {
  state = "available"
}

# main.tf
# VPC作成の直前に関連する locals を配置
locals {
  availability_zones = slice(data.aws_availability_zones.available.names, 0, 2)
}

resource "aws_vpc" "main" {
  # ...
}

# サブネット作成の直前に関連する locals を配置
locals {
  public_subnet_cidrs = [
    for i in range(length(local.availability_zones)) :
    cidrsubnet(var.vpc_cidr, 8, i)
  ]
}

resource "aws_subnet" "public" {
  # ...
}
```

**推奨事項：**

```
小規模プロジェクト（今回のケース）：
  - data.tf に locals をまとめる
  - シンプルで理解しやすい

大規模プロジェクト：
  - locals は使用箇所の近くに配置
  - コードの流れが明確になる

選択基準：
  - locals の数が5個以下 → data.tf にまとめる
  - locals の数が5個以上 → 使用箇所の近くに配置
  - チームの慣習に従う
```

#### 他のデータソースの例

```hcl
# data.tf（拡張版）

# =============================================================================
# アベイラビリティゾーン情報
# =============================================================================

data "aws_availability_zones" "available" {
  state = "available"

  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}

locals {
  availability_zones = slice(
    data.aws_availability_zones.available.names,
    0,
    2
  )
  public_subnet_cidrs  = ["10.88.1.0/24", "10.88.2.0/24"]
  private_subnet_cidrs = ["10.88.11.0/24", "10.88.12.0/24"]
}

# =============================================================================
# AMI情報（Amazon Machine Image）
# =============================================================================

data "aws_ami" "amazon_linux_2023" {
  most_recent = true  # 最新のAMIを取得
  owners      = ["amazon"]  # Amazonが公式に提供するAMI

  # Amazon Linux 2023の最新AMIを検索
  filter {
    name   = "name"
    values = ["al2023-ami-*-kernel-6.1-arm64"]
  }

  filter {
    name   = "architecture"
    values = ["arm64"]  # ARM64アーキテクチャ（Graviton）
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]  # Hardware Virtual Machine
  }
}

# 使用例：
# ami = data.aws_ami.amazon_linux_2023.id

# =============================================================================
# 現在のAWSアカウント情報
# =============================================================================

data "aws_caller_identity" "current" {
  # 現在使用しているAWSアカウントの情報を取得
}

# 使用例：
# account_id = data.aws_caller_identity.current.account_id
# arn        = data.aws_caller_identity.current.arn

# =============================================================================
# 現在のリージョン情報
# =============================================================================

data "aws_region" "current" {
  # 現在使用しているリージョンの情報を取得
}

# 使用例：
# region_name = data.aws_region.current.name
```

### 6. main.tf（リソース定義）

#### 役割

```
main.tf の役割：
1. 実際のAWSリソースを作成
2. プロジェクトのメインとなる設定
3. リソース間の依存関係を定義

なぜmain.tfという名前か：
- プロジェクトの中心（main）
- 最初に見るべきファイル
- Terraform の慣習
```

#### 現在のコードから残す部分

**現在のmain.tfから、resource ブロックのみを残します：**

```hcl
# main.tf（リソース定義のみ）

# =============================================================================
# VPCの作成
# =============================================================================

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# =============================================================================
# インターネットゲートウェイ
# =============================================================================

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-igw"
  }
}

# =============================================================================
# パブリックサブネット
# =============================================================================

resource "aws_subnet" "public" {
  count = length(local.public_subnet_cidrs)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = local.public_subnet_cidrs[count.index]
  availability_zone       = local.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project_name}-public-${count.index + 1}"
    Type = "Public"
  }
}

# =============================================================================
# プライベートサブネット
# =============================================================================

resource "aws_subnet" "private" {
  count = length(local.private_subnet_cidrs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_subnet_cidrs[count.index]
  availability_zone = local.availability_zones[count.index]

  tags = {
    Name = "${var.project_name}-private-${count.index + 1}"
    Type = "Private"
  }
}

# =============================================================================
# パブリックルートテーブル
# =============================================================================

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

# =============================================================================
# ルートテーブルの関連付け
# =============================================================================

resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

#### main.tfの整理のコツ

```hcl
# 良い整理：セクションごとにコメントで区切る

# =============================================================================
# ネットワーク基盤
# =============================================================================

resource "aws_vpc" "main" { }

resource "aws_internet_gateway" "main" { }

# =============================================================================
# サブネット
# =============================================================================

resource "aws_subnet" "public" { }

resource "aws_subnet" "private" { }

# =============================================================================
# ルーティング
# =============================================================================

resource "aws_route_table" "public" { }

resource "aws_route_table_association" "public" { }
```

### 7. outputs.tf（出力定義）

#### 役割

```
outputs.tf の役割：
1. terraform apply 後に表示する情報
2. 他のモジュールから参照する値
3. 重要な情報の可視化

なぜ分離するのか：
- 出力する情報を一箇所で管理
- ドキュメントとしての役割
- API的な役割（他から参照される）
```

#### 現在のコードから抜き出す

**現在のmain.tfから、この部分を抜き出します：**

```hcl
# main.tf（現在）

output "vpc_id" {                    # これらの
  description = "VPC ID"             # output ブロックを
  value       = aws_vpc.main.id      # すべて
}                                    # outputs.tf に
                                     # 移動します
output "vpc_cidr" {
  description = "VPC CIDR block"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "availability_zones" {
  description = "Availability zones used"
  value       = local.availability_zones
}
```

#### 新しいoutputs.tfファイルを作成

```hcl
# outputs.tf（新規作成）

# =============================================================================
# VPC情報
# =============================================================================

output "vpc_id" {
  description = "作成したVPCのID。他のリソースでVPCを参照する時に使用します。"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "VPCのCIDRブロック。ネットワーク設定の確認に使用します。"
  value       = aws_vpc.main.cidr_block
}

# =============================================================================
# サブネット情報
# =============================================================================

output "public_subnet_ids" {
  description = "パブリックサブネットのIDリスト。ロードバランサーやEC2インスタンスの配置に使用します。"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "プライベートサブネットのIDリスト。データベースやアプリサーバーの配置に使用します。"
  value       = aws_subnet.private[*].id
}

# =============================================================================
# その他の情報
# =============================================================================

output "availability_zones" {
  description = "使用しているアベイラビリティゾーンのリスト"
  value       = local.availability_zones
}

# =============================================================================
# まとめ情報（マップ形式）
# =============================================================================

output "network_summary" {
  description = "ネットワーク構成の概要"
  value = {
    vpc_id             = aws_vpc.main.id
    vpc_cidr           = aws_vpc.main.cidr_block
    availability_zones = local.availability_zones
    public_subnets     = length(aws_subnet.public)
    private_subnets    = length(aws_subnet.private)
  }
}
```

#### 出力の活用例

```hcl
# outputs.tf（実用的な例）

# =============================================================================
# 接続情報（後で使用する情報）
# =============================================================================

output "ssh_connection_info" {
  description = "EC2インスタンスへのSSH接続情報"
  value = {
    command = "ssh -i keypair.pem ec2-user@${aws_instance.web.public_ip}"
    ip      = aws_instance.web.public_ip
    port    = 22
  }
}

# =============================================================================
# URL情報
# =============================================================================

output "application_url" {
  description = "アプリケーションのURL"
  value       = "https://${aws_lb.main.dns_name}"
}

# =============================================================================
# 機密情報の出力
# =============================================================================

output "database_password" {
  description = "データベースの管理者パスワード"
  value       = random_password.db_password.result
  sensitive   = true  # 重要：機密情報は sensitive = true
}

# 使用方法：
# terraform output database_password
```

### 8. terraform.tfvars（変数の値）

#### 役割

```
terraform.tfvars の役割：
1. 変数の実際の値を設定
2. 環境ごとに異なる値を管理
3. デフォルト値の上書き

なぜ分離するのか：
- コードと設定を分離
- 環境ごとに異なるファイル
- 機密情報の管理
```

#### terraform.tfvarsファイルを作成

```hcl
# terraform.tfvars（新規作成）

# =============================================================================
# 基本設定
# =============================================================================

aws_region   = "ap-northeast-1"
environment  = "development"
project_name = "terraform-practice"

# =============================================================================
# ネットワーク設定
# =============================================================================

vpc_cidr = "10.88.0.0/16"
```

#### 環境別のファイル

```hcl
# dev.tfvars（開発環境）

aws_region   = "ap-northeast-1"
environment  = "development"
project_name = "terraform-practice"
vpc_cidr     = "10.88.0.0/16"

# 開発環境用の設定
instance_type  = "t4g.micro"
instance_count = 1
enable_backup  = false
```

```hcl
# staging.tfvars（ステージング環境）

aws_region   = "ap-northeast-1"
environment  = "staging"
project_name = "terraform-practice"
vpc_cidr     = "10.89.0.0/16"  # 開発と異なるCIDR

# ステージング環境用の設定
instance_type  = "t4g.small"
instance_count = 2
enable_backup  = true
```

```hcl
# prod.tfvars（本番環境）

aws_region   = "ap-northeast-1"
environment  = "production"
project_name = "terraform-practice"
vpc_cidr     = "10.90.0.0/16"  # 本番用のCIDR

# 本番環境用の設定
instance_type  = "t4g.medium"
instance_count = 4
enable_backup  = true
```

#### 使用方法

```bash
# デフォルト（terraform.tfvars）
terraform apply

# 開発環境
terraform apply -var-file="dev.tfvars"

# ステージング環境
terraform apply -var-file="staging.tfvars"

# 本番環境
terraform apply -var-file="prod.tfvars"
```

### 9. .gitignore

#### 役割

```
.gitignore の役割：
1. Gitで管理しないファイルを指定
2. 機密情報の保護
3. 自動生成ファイルの除外

なぜ必要か：
- 機密情報の漏洩防止
- リポジトリサイズの削減
- チーム開発の円滑化
```

#### .gitignoreファイルを作成

```gitignore
# .gitignore（新規作成）

# =============================================================================
# Terraform関連
# =============================================================================

# .terraform ディレクトリ（プロバイダープラグインなど）
.terraform/
.terraform.lock.hcl

# 状態ファイル（機密情報を含む可能性）
*.tfstate
*.tfstate.*
*.tfstate.backup

# クラッシュログ
crash.log
crash.*.log

# 変数ファイル（機密情報を含む場合）
*.tfvars
*.tfvars.json

# ただし、サンプルファイルは除外しない
!example.tfvars
!terraform.tfvars.example

# =============================================================================
# OS関連
# =============================================================================

# macOS
.DS_Store
.AppleDouble
.LSOverride

# Windows
Thumbs.db
Desktop.ini

# Linux
*~

# =============================================================================
# IDE関連
# =============================================================================

# Visual Studio Code
.vscode/

# IntelliJ IDEA
.idea/
*.iml

# =============================================================================
# その他
# =============================================================================

# SSH秘密鍵
*.pem
*.key

# バックアップファイル
*.bak
*.backup
```

**重要な注意：**

```
機密情報を含むファイルは必ず.gitignoreに追加

含めるべき：
- terraform.tfvars.example（サンプル）
- README.md（ドキュメント）

含めないべき：
- *.tfstate（状態ファイル）
- *.tfvars（実際の値、特に本番環境）
- *.pem（秘密鍵）
- .terraform/（プロバイダープラグイン）
```

## 分割作業の実践手順

### ステップ1：バックアップを作成

```bash
# 現在のmain.tfをバックアップ
cp main.tf main.tf.backup

# 念のためディレクトリ全体もバックアップ
cd ..
cp -r terraform-practice terraform-practice-backup
cd terraform-practice
```

### ステップ2：新しいファイルを作成

```bash
# 新しいファイルを作成
touch versions.tf
touch backend.tf
touch providers.tf
touch variables.tf
touch data.tf
touch outputs.tf
touch terraform.tfvars
touch .gitignore
```

### ステップ3：コードを移動

#### 3-1. versions.tfにterraformブロックを移動

```bash
# main.tfを開いて、terraformブロックをコピー
# versions.tfに貼り付け
# main.tfから該当部分を削除
```

**移動するコード：**

```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "3.7.2"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.1"
    }
  }
}
```

#### 3-2. backend.tfを作成（新規）

```bash
# backend.tfに新規作成
```

**記載する内容：**

```hcl
# backend.tf

# =============================================================================
# バックエンド設定（状態ファイルの保存場所）
# =============================================================================

# 現在はローカルバックエンドを使用（デフォルト）
# チーム開発時はS3バックエンドに移行

# 移行手順：
# 1. S3バケットとDynamoDBテーブルを作成
# 2. 以下の設定のコメントを外す
# 3. terraform init -migrate-state を実行

# terraform {
#   backend "s3" {
#     bucket         = "my-terraform-state-bucket"
#     key            = "terraform-practice/terraform.tfstate"
#     region         = "ap-northeast-1"
#     dynamodb_table = "terraform-state-lock"
#     encrypt        = true
#   }
# }
```

#### 3-3. providers.tfにproviderブロックを移動

```bash
# main.tfを開いて、providerブロックをコピー
# providers.tfに貼り付け
# main.tfから該当部分を削除
```

**移動するコード：**

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project_name
      ManagedBy   = "Terraform"
    }
  }
}

provider "random" {}

provider "tls" {}
```

#### 3-4. variables.tfにvariableブロックを移動

```bash
# main.tfを開いて、すべてのvariableブロックをコピー
# variables.tfに貼り付け
# main.tfから該当部分を削除
```

**移動するコード：**

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-northeast-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "development"
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "terraform-practice"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.88.0.0/16"
}
```

#### 3-5. data.tfにdataブロックとlocalsを移動

```bash
# main.tfを開いて、dataブロックとlocalsをコピー
# data.tfに貼り付け
# main.tfから該当部分を削除
```

**移動するコード：**

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  availability_zones = slice(
    data.aws_availability_zones.available.names,
    0,
    2
  )

  public_subnet_cidrs = [
    "10.88.1.0/24",
    "10.88.2.0/24"
  ]

  private_subnet_cidrs = [
    "10.88.11.0/24",
    "10.88.12.0/24"
  ]
}
```

#### 3-6. outputs.tfにoutputブロックを移動

```bash
# main.tfを開いて、すべてのoutputブロックをコピー
# outputs.tfに貼り付け
# main.tfから該当部分を削除
```

**移動するコード：**

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "VPC CIDR block"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "availability_zones" {
  description = "Availability zones used"
  value       = local.availability_zones
}
```

#### 3-7. main.tfにはresourceブロックのみ残す

```bash
# main.tfにはresourceブロックだけが残る
```

**残すコード：**

```hcl
# main.tf

# =============================================================================
# VPCの作成
# =============================================================================

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# ... 他のresourceブロック ...
```

#### 3-8. terraform.tfvarsを作成

```bash
# terraform.tfvarsに変数の値を記載
```

**記載する内容：**

```hcl
# terraform.tfvars

aws_region   = "ap-northeast-1"
environment  = "development"
project_name = "terraform-practice"
vpc_cidr     = "10.88.0.0/16"
```

#### 3-9. .gitignoreを作成

```bash
# .gitignoreを作成
```

**記載する内容：**

```gitignore
# Terraform
.terraform/
.terraform.lock.hcl
*.tfstate
*.tfstate.*
*.tfstate.backup
crash.log
crash.*.log

# Variables
*.tfvars
*.tfvars.json
!terraform.tfvars.example

# OS
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
*.iml

# Keys
*.pem
*.key
```

### ステップ4：動作確認

```bash
# フォーマット確認
terraform fmt -recursive

# 初期化（念のため）
terraform init

# 検証
terraform validate

# プラン（変更がないことを確認）
terraform plan

# 期待する結果：
# No changes. Your infrastructure matches the configuration.
```

### ステップ5：最終確認

```bash
# ファイル構成を確認
ls -la

# 期待する結果：
# backend.tf
# data.tf
# main.tf
# outputs.tf
# providers.tf
# terraform.tfvars
# variables.tf
# versions.tf
# .gitignore
```

## プロジェクト規模別のファイル構成

### パターン1：小規模プロジェクト（学習・個人開発）

```
my-project/
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars
```

**特徴：**

- ファイル数が少ない
- シンプルで理解しやすい
- 学習に最適

**いつ使うか：**

- 学習中
- 個人の小さなプロジェクト
- リソースが10個以下

### パターン2：中規模プロジェクト（チーム開発）

```
my-project/
├── versions.tf
├── backend.tf
├── providers.tf
├── variables.tf
├── data.tf
├── main.tf
├── outputs.tf
├── terraform.tfvars
├── dev.tfvars
├── prod.tfvars
├── .gitignore
└── README.md
```

**特徴：**

- 役割ごとにファイル分離
- 環境別設定が可能
- チーム開発に適している

**いつ使うか：**

- チーム開発
- リソースが10-50個
- 複数環境を管理

### パターン3：大規模プロジェクト（エンタープライズ）

```
my-project/
├── versions.tf
├── backend.tf
├── providers.tf
├── variables.tf
├── data.tf
├── main.tf              # モジュールの呼び出しのみ
├── outputs.tf
├── terraform.tfvars
├── environments/
│   ├── dev.tfvars
│   ├── staging.tfvars
│   └── prod.tfvars
├── modules/             # 機能ごとに分離
│   ├── network/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── database/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── .gitignore
└── README.md
```

**特徴：**

- モジュールを使用して機能を分離
- ルートのmain.tfはモジュール呼び出しのみ
- 再利用可能な構造
- 大規模チーム開発に最適

**いつ使うか：**

- リソースが50個以上
- 複数のプロジェクトで同じインフラを使用
- 大規模なチーム開発

**重要な注意：**

```
間違った方法：
my-project/
├── main.tf
├── network.tf      # ルートに直接配置はNG
├── compute.tf      # ルートに直接配置はNG
└── database.tf     # ルートに直接配置はNG

理由：
1. ファイルが増えすぎる
2. 依存関係が複雑になる
3. 再利用できない

正しい方法：
my-project/
├── main.tf
└── modules/        # モジュールフォルダに整理
    ├── network/
    ├── compute/
    └── database/

理由：
1. 整理されている
2. 再利用可能
3. テストしやすい
```

## ファイル分割の正しい進化

### ステージ1：学習段階（すべて1つのファイル）

```
my-project/
└── main.tf (すべてのコード)
```

**状況：**

- Terraformを学び始めた
- リソースが5-10個程度
- 個人で実験中

**問題：**

- コードが長くなってきた（300行以上）
- スクロールが大変
- 探しにくい

**次のステップ：** ステージ2へ

### ステージ2：基本的な分割（役割別にファイル分離）

```
my-project/
├── versions.tf      # バージョン設定
├── backend.tf       # バックエンド設定
├── providers.tf     # プロバイダー設定
├── variables.tf     # 変数定義
├── data.tf          # データソース + locals
├── main.tf          # リソース定義（まだ1つのファイル）
├── outputs.tf       # 出力定義
└── terraform.tfvars # 変数の値
```

**状況：**

- チームで開発開始
- リソースが10-50個
- main.tfだけが大きい（500行以上）

**問題：**

- main.tfが大きすぎる
- VPC、EC2、RDSなど複数の種類が混在
- 特定のリソースを探すのが大変

**正しい次のステップ：** ステージ3（モジュール化）へ

### ステージ3：モジュール化（機能別に分離）

```
my-project/
├── versions.tf
├── backend.tf
├── providers.tf
├── variables.tf
├── data.tf
├── main.tf          # モジュールの呼び出しのみ（シンプル）
├── outputs.tf
├── terraform.tfvars
└── modules/         # 機能ごとにモジュール化
    ├── network/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── compute/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── database/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

**状況：**

- リソースが50個以上
- 複数の環境（dev、staging、prod）
- 複数のプロジェクトで同じインフラを使用

**ルートのmain.tfの内容（シンプルになる）：**

```hcl
# main.tf（ルート）

# ネットワークモジュールの呼び出し
module "network" {
  source = "./modules/network"

  vpc_cidr     = var.vpc_cidr
  project_name = var.project_name
  environment  = var.environment
}

# コンピュートモジュールの呼び出し
module "compute" {
  source = "./modules/compute"

  vpc_id            = module.network.vpc_id
  public_subnet_ids = module.network.public_subnet_ids
  instance_type     = var.instance_type
  instance_count    = var.instance_count
}

# データベースモジュールの呼び出し
module "database" {
  source = "./modules/database"

  vpc_id             = module.network.vpc_id
  private_subnet_ids = module.network.private_subnet_ids
  db_instance_class  = var.db_instance_class
}
```

**メリット：**

- main.tfが短くシンプル
- 各機能が独立している
- 再利用可能
- テストしやすい

**注意：** モジュールの詳細は別の章で学習します。今は「機能を分離する時はmodulesフォルダを使う」ということだけ覚えてください。

## ベストプラクティスまとめ

### ファイル命名規則

```
推奨される命名：
  versions.tf        # バージョン設定
  backend.tf         # バックエンド設定
  providers.tf       # プロバイダー設定
  variables.tf       # 変数定義
  data.tf            # データソース
  main.tf            # メインリソース
  outputs.tf         # 出力

避けるべき命名：
  myfile.tf          # 役割が不明
  test.tf            # 曖昧
  backup.tf          # 目的が不明
  old_main.tf        # バージョン管理で対応すべき
```

### ファイル分割の基準

**1. 役割別に分割（推奨）**

```
versions.tf    → バージョン管理
backend.tf     → バックエンド設定
providers.tf   → プロバイダー設定
variables.tf   → 変数定義
data.tf        → データソース
outputs.tf     → 出力定義
```

**2. main.tfが大きくなったらモジュール化（次の章で学習）**

```
modules/
├── network/
├── compute/
└── database/
```

**3. ルートにリソース種類別ファイル（これはNG）**

```
network.tf     # ルートに直接配置しない
compute.tf     # モジュールを使うべき
database.tf    # モジュールを使うべき
```

### localsブロックの配置

```
方法1：data.tf にまとめる（小規模向け）
  - データソースと関連する locals を同じファイルに
  - 今回のチュートリアルではこの方法を採用

方法2：使用箇所の近くに配置（大規模向け）
  - locals を使うリソースの直前に配置
  - コードの流れが分かりやすい

選択基準：
  - プロジェクトの規模
  - チームの慣習
  - locals の数（5個以下なら方法1、5個以上なら方法2）
```

### ファイルサイズの目安

```
理想的なファイルサイズ：
- 200-300行以内
- スクロールなしで全体が見渡せる
- 1つの責任範囲に集中

ファイルが大きくなったら：
1. リソース種類別に分割（NG）
2. モジュール化を検討（正しい）
3. 複雑なロジックをlocalsに抽出
```

### コメントの書き方

```hcl
# 良いコメント：目的と理由を説明

# =============================================================================
# パブリックサブネット
# インターネットからアクセス可能なサブネット
# Webサーバーやロードバランサーを配置
# =============================================================================

resource "aws_subnet" "public" {
  # 2つのAZに分散配置（高可用性のため）
  count = 2

  # パブリックIPの自動割り当て（インターネットアクセスに必要）
  map_public_ip_on_launch = true

  # ...
}

# 悪いコメント：コードを繰り返しているだけ

# サブネットを作成
resource "aws_subnet" "public" {
  # countは2
  count = 2

  # map_public_ip_on_launchをtrueに設定
  map_public_ip_on_launch = true
}
```

### .gitignoreの必須項目

```gitignore
# 必ず含めるべき項目

# Terraform状態ファイル（機密情報を含む）
*.tfstate
*.tfstate.*

# Terraformプラグイン
.terraform/

# 変数ファイル（機密情報を含む可能性）
*.tfvars
!terraform.tfvars.example

# クラッシュログ
crash.log

# 秘密鍵
*.pem
*.key
```

## まとめ

### ファイル構成の進化

```
ステップ1：すべてmain.tfに（学習段階）
  └── main.tf

ステップ2：基本的な分割（個人開発）
  ├── main.tf
  ├── variables.tf
  └── outputs.tf

ステップ3：推奨構成（チーム開発）
  ├── versions.tf
  ├── backend.tf
  ├── providers.tf
  ├── variables.tf
  ├── data.tf
  ├── main.tf
  └── outputs.tf

ステップ4：大規模構成（エンタープライズ）
  ├── versions.tf
  ├── backend.tf
  ├── providers.tf
  ├── variables.tf
  ├── data.tf
  ├── main.tf
  ├── outputs.tf
  └── modules/
      ├── network/
      ├── compute/
      └── database/
```

### 重要なポイント

1. **役割ごとにファイルを分割**

- versions.tf: バージョン管理
- backend.tf: 状態管理
- providers.tf: プロバイダー設定
- variables.tf: 変数定義
- data.tf: データソース
- outputs.tf: 出力定義

2. **localsの配置**

- 小規模: data.tf にまとめる
- 大規模: 使用箇所の近くに配置

3. **機密情報の保護**

- .gitignoreを必ず作成
- \*.tfstateは除外
- \*.tfvarsは除外

4. **環境別の設定**

- ファイル分割ではなくtfvarsで対応
- dev.tfvars、prod.tfvars

5. **モジュール化のタイミング**

- main.tfが500行以上
- リソースが50個以上
- 再利用の必要性

### 現在の推奨構成（このチュートリアルでの目標）

```
terraform-practice/
├── versions.tf       # Terraformバージョン設定
├── backend.tf        # 状態管理設定
├── providers.tf      # プロバイダー設定
├── variables.tf      # 変数定義
├── data.tf           # データソース + locals
├── main.tf           # リソース定義
├── outputs.tf        # 出力定義
├── terraform.tfvars  # 変数の値
├── .gitignore        # Git除外設定
└── README.md         # ドキュメント
```

**この構成で問題が出るまで（main.tfが500行以上）は、この構成のままで大丈夫です。**

**それ以上大きくなったら、モジュール化を学習して移行します。**
