# Terraformスタイルガイド完全版

## この章で学ぶこと

Terraformコードを読みやすく、保守しやすく、チームで共有しやすくするためのスタイルガイドを学習します。

**学習内容：**

- コードの書き方とフォーマット
- ファイルとディレクトリの整理方法
- 命名規則
- コメントの書き方
- バージョン管理とGitの使い方
- セキュリティのベストプラクティス
- チーム開発での運用方法

## なぜスタイルガイドが必要なのか

### 問題：統一されていないコード

**シナリオ：3人のエンジニアが同じプロジェクトで作業**

```hcl
# エンジニアA のコード
resource "aws_instance" "web"{
ami="ami-xxxxx"
instance_type="t4g.small"
tags={Name="web-server"}}

# エンジニアB のコード
resource "aws_instance" "web_server" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "web-server"
  }
}

# エンジニアC のコード
resource "aws_instance" "WebServer" {
    ami = "ami-xxxxx"
    instance_type = "t4g.small"
    tags = { Name = "web-server" }
}
```

**問題点：**

```
1. フォーマットがバラバラ
   - インデントの違い
   - 改行の違い
   - スペースの有無

2. 命名規則が異なる
   - web vs web_server vs WebServer
   - スネークケース vs キャメルケース

3. 読みにくい
   - 統一感がない
   - レビューが困難
   - 保守が大変
```

### 解決策：スタイルガイドの採用

```hcl
# 統一されたスタイル

resource "aws_instance" "web_server" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "web-server"
  }
}
```

**メリット：**

```
1. 読みやすさ
   - 誰が書いても同じスタイル
   - 予測可能な構造

2. 保守性
   - 変更が容易
   - バグを見つけやすい

3. チーム開発
   - レビューがスムーズ
   - 知識の共有が容易
```

## コードフォーマット

### 基本ルール

#### 1. インデントは2スペース

```hcl
# 正しい例：2スペース
resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "web-server"
  }
}

# 間違った例：4スペースやタブ
resource "aws_instance" "web" {
    ami           = "ami-xxxxx"  # 4スペース（NG）
	instance_type = "t4g.small"  # タブ（NG）
}
```

#### 2. イコール記号を揃える

```hcl
# 正しい例：イコールが揃っている
resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"
  subnet_id     = aws_subnet.public.id
}

# 間違った例：バラバラ
resource "aws_instance" "web" {
  ami = "ami-xxxxx"
  instance_type = "t4g.small"
  subnet_id = aws_subnet.public.id
}
```

#### 3. ブロックの区切り

```hcl
# 正しい例：適切な空行

# VPCの作成
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# サブネットの作成（1行空ける）
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}
```

#### 4. 引数とブロックの順序

```hcl
# 正しい例：メタ引数 → 通常の引数 → ネストされたブロック → lifecycle

resource "aws_instance" "web" {
  # メタ引数を最初
  count = 2

  # 通常の引数
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"
  subnet_id     = aws_subnet.public.id

  # ネストされたブロック
  tags = {
    Name = "web-server"
  }

  # lifecycleブロックを最後
  lifecycle {
    create_before_destroy = true
  }
}
```

### terraform fmtの使用

**基本的な使い方：**

```bash
# 現在のディレクトリをフォーマット
terraform fmt

# すべてのサブディレクトリもフォーマット
terraform fmt -recursive

# 変更されたファイルを表示
terraform fmt -diff

# 実際には変更せず、チェックのみ
terraform fmt -check
```

**推奨ワークフロー：**

```bash
# コミット前に必ず実行
terraform fmt -recursive

# Gitにコミット
git add .
git commit -m "Add web server configuration"
```

**Git pre-commit hookの設定：**

```bash
# .git/hooks/pre-commit ファイルを作成

#!/bin/bash

# Terraformファイルをフォーマット
echo "Running terraform fmt..."
terraform fmt -recursive -check

# フォーマットが必要な場合はコミットを中止
if [ $? -ne 0 ]; then
  echo "Error: Terraform files need formatting. Run 'terraform fmt -recursive'"
  exit 1
fi

echo "All Terraform files are properly formatted"
```

### terraform validateの使用

```bash
# 構文チェック
terraform validate

# 出力例（エラーがある場合）：
Error: Invalid resource type

  on main.tf line 10:
  10: resource "aws_instanse" "web" {

# 出力例（エラーがない場合）：
Success! The configuration is valid.
```

**推奨ワークフロー：**

```bash
# コード編集後
terraform fmt
terraform validate

# 問題なければコミット
git add .
git commit -m "Add infrastructure configuration"
```

## ファイル構成

### 基本的なファイル構成

```
terraform-project/
├── backend.tf         # バックエンド設定
├── terraform.tf       # Terraformとプロバイダーのバージョン設定
├── providers.tf       # プロバイダー設定
├── variables.tf       # 変数定義
├── locals.tf          # ローカル値定義
├── main.tf            # メインのリソース定義
├── outputs.tf         # 出力定義
├── terraform.tfvars   # 変数の値（開発環境）
├── .gitignore         # Git除外設定
└── README.md          # ドキュメント
```

### 各ファイルの詳細

#### backend.tf

```hcl
# backend.tf

# =============================================================================
# バックエンド設定
# 状態ファイルの保存場所を定義
# =============================================================================

terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "project/terraform.tfstate"
    region         = "ap-northeast-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

#### terraform.tf

```hcl
# terraform.tf

# =============================================================================
# Terraformとプロバイダーのバージョン管理
# =============================================================================

terraform {
  required_version = ">= 1.7"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.34"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}
```

#### providers.tf

```hcl
# providers.tf

# =============================================================================
# プロバイダー設定
# =============================================================================

# デフォルトプロバイダー（東京リージョン）
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}

# 追加プロバイダー（大阪リージョン）
provider "aws" {
  alias  = "osaka"
  region = "ap-northeast-3"

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      Region      = "osaka"
      ManagedBy   = "Terraform"
    }
  }
}
```

#### variables.tf

```hcl
# variables.tf

# =============================================================================
# 基本設定
# =============================================================================

variable "aws_region" {
  description = "AWSリージョン。リソースを作成する地理的な場所。"
  type        = string
  default     = "ap-northeast-1"
}

variable "environment" {
  description = "環境名。development、staging、productionのいずれか。"
  type        = string

  validation {
    condition = contains(
      ["development", "staging", "production"],
      var.environment
    )
    error_message = "environmentはdevelopment、staging、productionのいずれかである必要があります。"
  }
}

variable "project_name" {
  description = "プロジェクト名。リソース名の接頭辞として使用。"
  type        = string
}

# =============================================================================
# ネットワーク設定
# =============================================================================

variable "vpc_cidr" {
  description = "VPCのCIDRブロック。"
  type        = string
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "有効なCIDR形式を指定してください。"
  }
}
```

**変数定義の順序：**

```
1. type
2. description
3. default（オプション）
4. sensitive（オプション）
5. validation（オプション）
```

#### locals.tf

```hcl
# locals.tf

# =============================================================================
# 共通のローカル値
# =============================================================================

locals {
  # 共通の名前接頭辞
  name_prefix = "${var.project_name}-${var.environment}"

  # 共通タグ
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }

  # 環境別の設定
  is_production = var.environment == "production"
  instance_type = local.is_production ? "t4g.medium" : "t4g.small"
}
```

#### main.tf

```hcl
# main.tf

# =============================================================================
# VPC
# =============================================================================

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-vpc"
    }
  )
}

# =============================================================================
# サブネット
# =============================================================================

resource "aws_subnet" "public" {
  count = 2

  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-public-${count.index + 1}"
      Type = "Public"
    }
  )
}
```

#### outputs.tf

```hcl
# outputs.tf

# =============================================================================
# ネットワーク出力
# =============================================================================

output "vpc_id" {
  description = "VPCのID"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "VPCのCIDRブロック"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "パブリックサブネットのIDリスト"
  value       = aws_subnet.public[*].id
}
```

**出力定義の順序：**

```
1. description
2. value
3. sensitive（オプション）
```

### プロジェクトが大きくなった場合

#### オプション1：機能別にファイルを分割（同じディレクトリ内）

**いつ使うか：**

```
リソース数が増えてきたが、まだモジュール化するほどではない場合
目安：リソース数が20-50個程度
```

**構造：**

```
terraform-project/
├── backend.tf
├── terraform.tf
├── providers.tf
├── variables.tf
├── locals.tf
├── outputs.tf
├── terraform.tfvars
├── network.tf         # ネットワーク関連リソース
├── compute.tf         # コンピュート関連リソース
├── database.tf        # データベース関連リソース
├── storage.tf         # ストレージ関連リソース
├── .gitignore
└── README.md
```

**network.tf の例：**

```hcl
# network.tf

# =============================================================================
# VPC
# =============================================================================

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-vpc"
    }
  )
}

# =============================================================================
# サブネット
# =============================================================================

resource "aws_subnet" "public" {
  count = 2

  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-public-${count.index + 1}"
      Type = "Public"
    }
  )
}

resource "aws_subnet" "private" {
  count = 2

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-private-${count.index + 1}"
      Type = "Private"
    }
  )
}

# =============================================================================
# インターネットゲートウェイ
# =============================================================================

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-igw"
    }
  )
}

# =============================================================================
# ルートテーブル
# =============================================================================

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-public-rt"
    }
  )
}

resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

**compute.tf の例：**

```hcl
# compute.tf

# =============================================================================
# セキュリティグループ
# =============================================================================

resource "aws_security_group" "web" {
  name        = "${local.name_prefix}-web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-web-sg"
    }
  )
}

# =============================================================================
# EC2インスタンス
# =============================================================================

resource "aws_instance" "web" {
  count = var.web_instance_count

  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public[count.index % length(aws_subnet.public)].id
  vpc_security_group_ids = [aws_security_group.web.id]

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-web-${count.index + 1}"
    }
  )
}
```

**このアプローチの特徴：**

```
メリット：
- main.tfが巨大になることを防ぐ
- 機能ごとに見やすく整理
- ファイルの切り替えが容易
- すべて同じディレクトリなので管理が簡単

デメリット：
- リソース間の依存関係がファイル間にまたがる
- 再利用性がない（他のプロジェクトで使えない）
- 状態ファイルは1つ（すべてのリソースが同じ状態）

適用場面：
- 中規模プロジェクト（リソース20-50個）
- まだモジュール化するほど複雑ではない
- 単一プロジェクト専用の構成
```

#### オプション2：モジュール化（推奨：大規模プロジェクト）

**いつ使うか：**

```
さらにリソースが増えた場合、またはコードを再利用したい場合
目安：リソース数が50個以上、または複数環境で同じ構成を使う場合
```

**構造：**

```
terraform-project/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
├── backend.tf
├── terraform.tf
├── providers.tf
└── modules/
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

**ルートモジュール（main.tf）：**

```hcl
# main.tf（ルートモジュール）

module "network" {
  source = "./modules/network"

  project_name = var.project_name
  environment  = var.environment
  vpc_cidr     = var.vpc_cidr
}

module "compute" {
  source = "./modules/compute"

  project_name       = var.project_name
  environment        = var.environment
  vpc_id             = module.network.vpc_id
  public_subnet_ids  = module.network.public_subnet_ids
  instance_type      = var.instance_type
  instance_count     = var.instance_count
}

module "database" {
  source = "./modules/database"

  project_name       = var.project_name
  environment        = var.environment
  vpc_id             = module.network.vpc_id
  private_subnet_ids = module.network.private_subnet_ids
}
```

**このアプローチの特徴：**

```
メリット：
- 高い再利用性（他のプロジェクトで使える）
- 明確な依存関係（モジュール間の接続が明確）
- 独立したテストが可能
- 状態を分割可能（モジュールごとに別の状態ファイル）

デメリット：
- 初期設定が複雑
- 学習コストが高い
- 小規模プロジェクトにはオーバースペック

適用場面：
- 大規模プロジェクト（リソース50個以上）
- 複数環境で同じ構成を使う
- チーム間でコードを共有
- 再利用性が重要
```

#### どちらを選ぶべきか

```
プロジェクトの進化：

ステージ1：すべてmain.tfに（リソース < 20個）
  main.tf だけで管理

ステージ2：機能別ファイル分割（リソース 20-50個）
  network.tf, compute.tf, database.tf に分割
  まだ同じディレクトリ内

ステージ3：モジュール化（リソース > 50個）
  modules/ ディレクトリに分離
  再利用可能な構造に


判断基準：

機能別ファイル分割を選ぶ場合：
- リソース数が増えてきた（20-50個）
- まだ1つのプロジェクト専用
- 再利用の予定がない
- シンプルさを保ちたい

モジュール化を選ぶ場合：
- リソース数が多い（50個以上）
- 複数環境で使う（dev, staging, prod）
- 他のプロジェクトでも使いたい
- チームでコードを共有したい
```

**推奨アプローチ：**

```
段階的な移行：

1. 最初はmain.tfだけ
   ↓ リソースが増えてきた

2. 機能別ファイルに分割
   network.tf, compute.tf など
   ↓ さらに複雑になった、または再利用が必要に

3. モジュール化
   modules/network/, modules/compute/ など

無理に最初からモジュール化する必要はありません
プロジェクトの成長に合わせて段階的に移行することが重要です
```

## 命名規則

### リソース名

**基本ルール：**

```
1. 名詞を使用
2. スネークケース（アンダースコア区切り）
3. リソースタイプを名前に含めない
4. ダブルクォートで囲む
```

**良い例：**

```hcl
resource "aws_instance" "web_server" { }
resource "aws_instance" "database_server" { }
resource "aws_s3_bucket" "application_logs" { }
resource "aws_vpc" "main" { }
```

**悪い例：**

```hcl
resource "aws_instance" "webServerInstance" { }  # キャメルケース（NG）
resource "aws_instance" "web-server" { }         # ハイフン（NG）
resource "aws_instance" "aws_instance_web" { }   # リソースタイプ重複（NG）
resource aws_instance web { }                    # クォートなし（NG）
```

### 変数名と出力名

**良い例：**

```hcl
# 変数名
variable "vpc_cidr_block" { }
variable "instance_type" { }
variable "enable_monitoring" { }
variable "database_password" { }

# 出力名
output "vpc_id" { }
output "public_subnet_ids" { }
output "instance_public_ip" { }
output "database_endpoint" { }
```

**悪い例：**

```hcl
# 変数名
variable "VPCCidr" { }           # キャメルケース（NG）
variable "vpc-cidr" { }          # ハイフン（NG）
variable "the_vpc_cidr" { }      # 不要な冠詞（NG）

# 出力名
output "VpcId" { }               # キャメルケース（NG）
output "output1" { }             # 意味不明（NG）
```

## コメントの書き方

### 基本ルール

```hcl
# 単一行コメントには # を使用

# 複数行コメントも
# # を使用
# // や /* */ は使わない
```

### コメントの使い方

**セクションの区切り：**

```hcl
# =============================================================================
# VPCとネットワーク設定
# =============================================================================

resource "aws_vpc" "main" {
  # ...
}

# =============================================================================
# セキュリティグループ
# =============================================================================

resource "aws_security_group" "web" {
  # ...
}
```

**複雑なロジックの説明：**

```hcl
# 本番環境では高可用性のためマルチAZ構成を使用
# 開発環境ではコスト削減のためシングルAZで運用
resource "aws_db_instance" "main" {
  multi_az = var.environment == "production"
  # ...
}
```

**一時的な変更の記録：**

```hcl
# TODO: インスタンスタイプを本番環境に合わせて変更する
# 現在は開発用の設定
resource "aws_instance" "app" {
  instance_type = "t4g.small"
  # ...
}
```

**避けるべきコメント：**

```hcl
# 悪い例：コードの繰り返し
# VPCを作成
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"  # CIDRブロックを10.0.0.0/16に設定
}

# 良い例：理由や意図を説明
# 開発環境と本番環境で同じCIDRレンジを使用
# VPCピアリングは不要と判断
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
```

## リソースの順序

### データソースとリソースの配置

**推奨：データソースをリソースの直前に配置**

```hcl
# データソース：利用可能なAZ
data "aws_availability_zones" "available" {
  state = "available"
}

# データソース：最新のAMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-kernel-6.1-arm64"]
  }
}

# リソース：データソースを使用
resource "aws_subnet" "public" {
  count = 2

  availability_zone = data.aws_availability_zones.available.names[count.index]
  # ...
}

resource "aws_instance" "web" {
  ami = data.aws_ami.amazon_linux.id
  # ...
}
```

### リソース内のパラメータ順序

```hcl
resource "aws_instance" "web" {
  # 1. メタ引数（count、for_each）
  count = 2

  # 2. 必須パラメータ
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type

  # 3. オプションパラメータ
  subnet_id                   = aws_subnet.public[count.index].id
  vpc_security_group_ids      = [aws_security_group.web.id]
  associate_public_ip_address = true

  # 4. ネストされたブロック
  tags = {
    Name = "${var.project_name}-web-${count.index + 1}"
  }

  # 5. lifecycle ブロック
  lifecycle {
    create_before_destroy = true
  }

  # 6. depends_on（必要な場合のみ）
  depends_on = [
    aws_security_group.web
  ]
}
```

## 変数と出力のベストプラクティス

### 変数の使用

**使いすぎない：**

```hcl
# 悪い例：すべてを変数にする
variable "vpc_enable_dns_hostnames" {
  type    = bool
  default = true
}

variable "vpc_enable_dns_support" {
  type    = bool
  default = true
}

# これらは通常変更しないので変数にする必要がない

# 良い例：変更の可能性があるもののみ変数化
variable "vpc_cidr" {
  description = "VPCのCIDRブロック。環境によって変更する。"
  type        = string
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true  # 固定値
  enable_dns_support   = true  # 固定値
}
```

### 変数定義の完全な例

```hcl
# 良い変数定義

variable "instance_type" {
  description = <<-EOT
    EC2インスタンスのタイプ。
    開発環境: t4g.micro または t4g.small
    本番環境: t4g.medium 以上を推奨
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

variable "database_password" {
  description = "データベースの管理者パスワード。最小8文字。"
  type        = string
  sensitive   = true

  validation {
    condition     = length(var.database_password) >= 8
    error_message = "パスワードは8文字以上である必要があります。"
  }
}
```

### 出力定義の完全な例

```hcl
# 良い出力定義

output "vpc_id" {
  description = "作成したVPCのID。他のリソースでVPCを参照する時に使用。"
  value       = aws_vpc.main.id
}

output "instance_public_ip" {
  description = "Webサーバーのパブリック IP アドレス。SSH接続に使用。"
  value       = aws_instance.web.public_ip
}

output "database_password" {
  description = "データベースの管理者パスワード。"
  value       = random_password.db.result
  sensitive   = true
}
```

## .gitignoreの設定

```gitignore
# .gitignore

# =============================================================================
# Terraform関連
# =============================================================================

# .terraformディレクトリ（プロバイダープラグイン）
.terraform/
.terraform.lock.hcl

# 状態ファイル（機密情報を含む）
*.tfstate
*.tfstate.*
*.tfstate.backup

# 状態ロックファイル
.terraform.tfstate.lock.info

# クラッシュログ
crash.log
crash.*.log

# 変数ファイル（機密情報を含む可能性）
*.tfvars
*.tfvars.json

# ただし、サンプルファイルは除外しない
!terraform.tfvars.example
!example.tfvars

# プランファイル
*.tfplan

# =============================================================================
# OS関連
# =============================================================================

# macOS
.DS_Store

# Windows
Thumbs.db

# Linux
*~

# =============================================================================
# IDE関連
# =============================================================================

# VS Code
.vscode/

# IntelliJ
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
```

**必ずコミットするファイル：**

```
必須：
- すべての .tf ファイル
- .terraform.lock.hcl（依存関係ロックファイル）
- .gitignore
- README.md

任意（プロジェクトによる）：
- terraform.tfvars.example（サンプル）
```

## バージョン管理

### Terraformバージョンの固定

```hcl
# terraform.tf

terraform {
  # Terraformバージョンを固定
  required_version = ">= 1.7"

  required_providers {
    # プロバイダーバージョンも固定
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.34.0"
    }
  }
}
```

**バージョン指定の考え方：**

```
Terraformバージョン：
  >= 1.7        - 最小バージョンを指定（推奨）
  = 1.7.0       - 厳密に固定（CI/CDで使用）

プロバイダーバージョン：
  ~> 5.34.0     - 5.34.x の最新（推奨）
  ~> 5.34       - 5.x.x の最新
  = 5.34.0      - 厳密に固定
```

### モジュールバージョンの固定

```hcl
# モジュールの使用

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"  # メジャーバージョンを固定

  # ...
}
```

## リンターの使用

### TFLintのインストール

```bash
# macOS
brew install tflint

# Linux
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# Windows
choco install tflint
```

### TFLintの基本設定

```hcl
# .tflint.hcl

plugin "aws" {
  enabled = true
  version = "0.29.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_naming_convention" {
  enabled = true
  format  = "snake_case"
}

rule "terraform_documented_variables" {
  enabled = true
}

rule "terraform_documented_outputs" {
  enabled = true
}
```

### TFLintの実行

```bash
# 初期化
tflint --init

# チェック実行
tflint

# 再帰的にチェック
tflint --recursive

# 出力例：
3 issue(s) found:

Warning: variable "instance_type" has no description (terraform_documented_variables)

  on variables.tf line 10:
  10: variable "instance_type" {

Error: "t2.micro" is not a valid value for instance_type (aws_instance_invalid_type)

  on main.tf line 15:
  15:   instance_type = "t2.micro"
```

## 開発ワークフロー

### 推奨ワークフロー

```bash
# ステップ1：ブランチ作成
git checkout -b feature/add-web-server

# ステップ2：コード作成
# ... ファイルを編集 ...

# ステップ3：フォーマット
terraform fmt -recursive

# ステップ4：バリデーション
terraform validate

# ステップ5：リンターチェック
tflint --recursive

# ステップ6：プラン確認
terraform plan

# ステップ7：コミット
git add .
git commit -m "Add web server configuration"

# ステップ8：プッシュ
git push origin feature/add-web-server

# ステップ9：プルリクエスト作成
# GitHubでプルリクエストを作成

# ステップ10：レビュー後マージ
```

### Git pre-commit hookの完全な例

```bash
#!/bin/bash
# .git/hooks/pre-commit

echo "Running pre-commit checks..."

# Terraformファイルをフォーマット
echo "1. Formatting Terraform files..."
terraform fmt -recursive -check
if [ $? -ne 0 ]; then
  echo "Error: Run 'terraform fmt -recursive' to format files"
  exit 1
fi
echo "Formatting check passed"

# バリデーション
echo "2. Validating Terraform configuration..."
terraform validate
if [ $? -ne 0 ]; then
  echo "Error: Terraform validation failed"
  exit 1
fi
echo "Validation passed"

# TFLintチェック
if command -v tflint &> /dev/null; then
  echo "3. Running TFLint..."
  tflint --recursive
  if [ $? -ne 0 ]; then
    echo "Error: TFLint found issues"
    exit 1
  fi
  echo "TFLint passed"
fi

echo "All pre-commit checks passed!"
```

## セキュリティのベストプラクティス

### 機密情報の管理

**絶対にやってはいけないこと：**

```hcl
# 悪い例：パスワードをハードコーディング

resource "aws_db_instance" "main" {
  username = "admin"
  password = "MyPassword123!"  # NG
}
```

**正しい方法：**

```hcl
# 良い例：環境変数を使用

variable "database_password" {
  description = "データベースパスワード"
  type        = string
  sensitive   = true
}

resource "aws_db_instance" "main" {
  username = "admin"
  password = var.database_password
}
```

```bash
# 環境変数で渡す
export TF_VAR_database_password="MySecurePassword123!"
terraform apply
```

**さらに良い方法：AWS Secrets Managerを使用**

```hcl
# Secrets Managerからパスワードを取得
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "production/database/password"
}

resource "aws_db_instance" "main" {
  username = "admin"
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

### 状態ファイルの保護

```hcl
# backend.tf

terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "project/terraform.tfstate"
    region         = "ap-northeast-1"
    encrypt        = true                      # 暗号化必須
    dynamodb_table = "terraform-state-lock"    # ロック機能

    # バージョニング有効化（S3バケット側で設定）
  }
}
```

## モジュール開発

### モジュールの構造

```
terraform-aws-vpc/
├── README.md              # モジュールの説明
├── main.tf                # メインのリソース定義
├── variables.tf           # 入力変数
├── outputs.tf             # 出力
├── versions.tf            # バージョン制約
├── examples/              # 使用例
│   └── complete/
│       ├── main.tf
│       └── README.md
└── tests/                 # テスト
    └── vpc_test.go
```

### モジュールの命名規則

```
命名パターン：
terraform-<PROVIDER>-<NAME>

例：
terraform-aws-vpc
terraform-aws-ec2-instance
terraform-google-kubernetes-cluster
```

### モジュールの使用

```hcl
# ローカルモジュール
module "vpc" {
  source = "./modules/vpc"

  vpc_cidr = "10.0.0.0/16"
}

# レジストリのモジュール
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}
```

## テストの実施

### Terraformテストの基本

```hcl
# tests/vpc_test.tftest.hcl

run "valid_vpc_cidr" {
  command = plan

  variables {
    vpc_cidr = "10.0.0.0/16"
  }

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR が期待値と一致しません"
  }
}
```

```bash
# テスト実行
terraform test
```

## 完全なプロジェクト例

```
production-infrastructure/
├── .git/
├── .github/
│   └── workflows/
│       └── terraform.yml
├── environments/
│   ├── development/
│   │   ├── backend.tf
│   │   ├── terraform.tf
│   │   ├── providers.tf
│   │   ├── variables.tf
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   └── ...
│   └── production/
│       └── ...
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── compute/
│   │   └── ...
│   └── database/
│       └── ...
├── .gitignore
├── .tflint.hcl
├── README.md
└── CONTRIBUTING.md
```

## チェックリスト

### コミット前のチェックリスト

```
terraform fmt -recursive を実行
terraform validate を実行
tflint --recursive を実行
terraform plan を実行して確認
コメントを適切に追加
機密情報が含まれていないか確認
README.md を更新（必要な場合）
```

### プルリクエストのチェックリスト

```
変更内容の説明が明確
terraform plan の出力を添付
レビュー依頼
CI/CD が成功している
ドキュメントが更新されている
```

## まとめ

### コードスタイル

```
フォーマット：
- terraform fmt を必ず実行
- インデントは2スペース
- イコール記号を揃える

命名規則：
- リソース名：スネークケース、名詞
- 変数名：説明的、スネークケース
- 出力名：明確、スネークケース
```

### ファイル構成

```
基本ファイル：
- backend.tf
- terraform.tf
- providers.tf
- variables.tf
- locals.tf
- main.tf
- outputs.tf

中規模プロジェクト（機能別ファイル分割）：
- network.tf
- compute.tf
- database.tf
- storage.tf

大規模プロジェクト（モジュール化）：
- modules/network/
- modules/compute/
- modules/database/
```

### ベストプラクティス

```
コード品質：
- terraform validate で検証
- TFLint でチェック
- テストを書く

セキュリティ：
- 機密情報はハードコーディングしない
- 状態ファイルを暗号化
- .gitignore を適切に設定

バージョン管理：
- Terraform バージョンを固定
- プロバイダーバージョンを固定
- モジュールバージョンを固定
```
