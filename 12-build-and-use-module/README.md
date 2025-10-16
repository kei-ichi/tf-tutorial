# Terraformモジュールの作成と使用：完全ガイド

## この章で学ぶこと

モジュールは、複数のリソースをまとめて管理する仕組みです。この章では、再利用可能なモジュールの作成方法と使用方法を学習します。

**学習内容：**

- モジュールとは何か（ルートモジュールと子モジュール）
- モジュールを作成する理由
- モジュールの構造
- ネットワークモジュールの作成（実践例）
- ストレージモジュールの作成（実践例）
- モジュール間での値の受け渡し
- ベストプラクティス

## モジュールとは

### ルートモジュールと子モジュール

**基本的な考え方：**

```
会社組織：
本社（ルート）
├── 営業部（子）
├── 開発部（子）
└── 総務部（子）

Terraformのモジュール：
ルートモジュール（プロジェクトのメイン）
├── networkモジュール（子）
├── storageモジュール（子）
└── computeモジュール（子）
```

**ルートモジュールとは：**

```
ルートモジュール：
- terraform applyを実行するディレクトリ
- プロジェクトのメインとなる設定
- 子モジュールを呼び出す
- main.tf、variables.tf、outputs.tfなどを含む

例：
my-project/
├── main.tf          ← ルートモジュールのファイル
├── variables.tf     ← ルートモジュールのファイル
├── outputs.tf       ← ルートモジュールのファイル
└── modules/         ← 子モジュールのディレクトリ
    ├── network/
    └── storage/
```

**子モジュール（チャイルドモジュール）とは：**

```
子モジュール：
- ルートモジュールから呼び出される
- 特定の機能を持つ独立したモジュール
- 再利用可能な部品
- modules/ディレクトリ配下に配置

例：
modules/network/     ← 子モジュール
├── main.tf          ← モジュール内のファイル
├── variables.tf     ← モジュール内のファイル
└── outputs.tf       ← モジュール内のファイル
```

**ルートモジュールと子モジュールの関係：**

```hcl
# ルートモジュール（main.tf）

# 子モジュールを呼び出す
module "network" {
  source = "./modules/network"

  # 子モジュールに値を渡す
  project_name = var.project_name
  vpc_cidr     = "10.0.0.0/16"
}

# 子モジュールの出力を使用
resource "aws_instance" "web" {
  subnet_id = module.network.public_subnet_ids[0]
  # ...
}
```

```
データの流れ：

ルートモジュール
    ↓（入力変数を渡す）
子モジュール（network）
    ↓（リソースを作成）
VPC、サブネットなど
    ↓（出力値を返す）
ルートモジュール
    ↓（出力値を使用）
他のリソース作成
```

### 具体例で理解する

**例えて言うなら：** レゴブロックのセット

```
レゴブロック：
- 基本ブロック → 個別のリソース（VPC、サブネットなど）
- セット → 子モジュール（ネットワーク一式、データベース一式など）
- 作品全体 → ルートモジュール

小さな家を作る：
  基本ブロックを1つずつ組み立てる（大変）

大きな街を作る：
  ルートモジュール：街全体の設計
    ├── 家セット（子モジュール）
    ├── 公園セット（子モジュール）
    └── 学校セット（子モジュール）
```

**Terraformでも同じ：**

```
モジュールなし（ルートモジュールのみ）：
my-project/
└── main.tf
    ├── すべてのリソースを1つずつ定義
    ├── VPCを定義
    ├── サブネットを定義
    ├── ルートテーブルを定義
    └── ...（50個のリソース）

モジュールあり（ルート + 子）：
my-project/
├── main.tf（ルートモジュール）
│   └── 子モジュールを呼び出すだけ
└── modules/
    ├── network/（子モジュール）
    │   └── ネットワーク関連を一式管理
    ├── storage/（子モジュール）
    │   └── ストレージ関連を一式管理
    └── compute/（子モジュール）
        └── コンピュート関連を一式管理
```

## なぜモジュールを作成するのか

### 問題：コードの重複

**例：3つの環境でネットワークを作成**

```hcl
# ルートモジュール（main.tf）にすべて書く場合

# 開発環境のネットワーク
resource "aws_vpc" "dev" {
  cidr_block = "10.0.0.0/16"
  # ...（50行の設定）
}

resource "aws_subnet" "dev_public_1" {
  # ...（20行の設定）
}

resource "aws_subnet" "dev_public_2" {
  # ...（20行の設定）
}

# ... 他のリソース（合計200行）

# ステージング環境のネットワーク
resource "aws_vpc" "staging" {
  cidr_block = "10.1.0.0/16"
  # ...（50行の設定、ほぼ同じ）
}

resource "aws_subnet" "staging_public_1" {
  # ...（20行の設定、ほぼ同じ）
}

# ... （さらに200行の重複コード）

# 本番環境のネットワーク
resource "aws_vpc" "prod" {
  cidr_block = "10.2.0.0/16"
  # ...（50行の設定、ほぼ同じ）
}

# ... （さらに200行の重複コード）

# 合計：600行以上の重複コード
```

**問題点：**

- コードが長すぎる（600行以上）
- 同じコードを3回書いている
- バグ修正を3箇所で行う必要がある
- 保守が大変

### 解決策：子モジュールを使用

```hcl
# modules/network/main.tf （子モジュール：1回だけ定義）
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  # ...
}

resource "aws_subnet" "public" {
  # ...
}

# ルートモジュール（main.tf）
module "network_dev" {
  source   = "./modules/network"
  vpc_cidr = "10.0.0.0/16"
}

module "network_staging" {
  source   = "./modules/network"
  vpc_cidr = "10.1.0.0/16"
}

module "network_prod" {
  source   = "./modules/network"
  vpc_cidr = "10.2.0.0/16"
}

# 合計：約100行（大幅に削減）
```

**メリット：**

- コードの重複を削減
- 保守が簡単
- バグ修正は1箇所だけ
- 再利用可能

## モジュールの基本構造

### 標準的なモジュール構造

```
my-project/
├── main.tf              # ルートモジュール（モジュール呼び出し）
├── variables.tf         # ルートモジュールの変数
├── outputs.tf           # ルートモジュールの出力
├── terraform.tfvars     # 変数の値
└── modules/             # 子モジュールのディレクトリ
    ├── network/         # ネットワーク子モジュール
    │   ├── main.tf      # リソース定義
    │   ├── variables.tf # モジュールの入力変数
    │   └── outputs.tf   # モジュールの出力
    └── storage/         # ストレージ子モジュール
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

### モジュールの入力と出力

**注意：** 入力（変数）と出力の詳細は、既に学習した「Terraformの設定ブロック完全ガイド：variable、locals、output」で説明されています。ここでは、モジュール間でのやり取りに焦点を当てます。

```
子モジュールの構造：

入力（Input Variables）：
  ↓
【変数定義（variables.tf）】
  - ルートモジュールから値を受け取る
  - 必要なデフォルト値を設定
  ↓
【処理（main.tf）】
  - 実際にインフラを作成
  - 受け取った変数を使用
  ↓
【出力（outputs.tf）】
  - モジュールが作成したリソースの情報
  - ルートモジュールに返す
  ↓
ルートモジュールで使用可能
```

## 実践：ネットワークモジュールの作成

### ステップ1：プロジェクト構造を作成

```bash
# プロジェクトディレクトリを作成
mkdir terraform-modules-practice
cd terraform-modules-practice

# 子モジュールディレクトリを作成
mkdir -p modules/network
mkdir -p modules/storage

# ルートモジュールのファイルを作成
touch main.tf
touch variables.tf
touch outputs.tf
touch terraform.tfvars
```

### ステップ2：ネットワーク子モジュールを作成

#### modules/network/variables.tf

```hcl
# modules/network/variables.tf

# =============================================================================
# 入力変数（このモジュールが受け取る値）
# =============================================================================

variable "project_name" {
  description = "プロジェクト名。リソース名の接頭辞として使用されます。"
  type        = string
}

variable "environment" {
  description = "環境名（development、staging、production）"
  type        = string

  validation {
    condition = contains(
      ["development", "staging", "production"],
      var.environment
    )
    error_message = "environmentはdevelopment、staging、productionのいずれかである必要があります。"
  }
}

variable "vpc_cidr" {
  description = "VPCのCIDRブロック"
  type        = string

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "有効なCIDR形式を指定してください（例：10.0.0.0/16）。"
  }
}

variable "availability_zones" {
  description = "使用するアベイラビリティゾーンのリスト"
  type        = list(string)
  default     = ["ap-northeast-1a", "ap-northeast-1c"]
}
```

#### modules/network/main.tf

```hcl
# modules/network/main.tf

# =============================================================================
# データソース（このモジュール内でのみ使用）
# =============================================================================

# 注意：data.tfファイルに分離してもよいが、
# 小さなモジュールではmain.tfに含めることが多い

data "aws_availability_zones" "available" {
  state = "available"
}

# =============================================================================
# ローカル値（このモジュール内でのみ使用）
# =============================================================================

locals {
  # 使用するアベイラビリティゾーン
  availability_zones = var.availability_zones

  # サブネットCIDRを自動計算
  public_subnet_cidrs = [
    for i in range(length(local.availability_zones)) :
    cidrsubnet(var.vpc_cidr, 8, i)
  ]

  private_subnet_cidrs = [
    for i in range(length(local.availability_zones)) :
    cidrsubnet(var.vpc_cidr, 8, i + 10)
  ]
}

# =============================================================================
# VPCの作成
# =============================================================================

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.project_name}-${var.environment}-vpc"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# =============================================================================
# インターネットゲートウェイ
# =============================================================================

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name        = "${var.project_name}-${var.environment}-igw"
    Environment = var.environment
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
    Name        = "${var.project_name}-${var.environment}-public-${count.index + 1}"
    Environment = var.environment
    Type        = "Public"
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
    Name        = "${var.project_name}-${var.environment}-private-${count.index + 1}"
    Environment = var.environment
    Type        = "Private"
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
    Name        = "${var.project_name}-${var.environment}-public-rt"
    Environment = var.environment
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

#### modules/network/outputs.tf

```hcl
# modules/network/outputs.tf

# =============================================================================
# 出力値（このモジュールが返す値）
# =============================================================================

output "vpc_id" {
  description = "作成したVPCのID"
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

output "private_subnet_ids" {
  description = "プライベートサブネットのIDリスト"
  value       = aws_subnet.private[*].id
}

output "internet_gateway_id" {
  description = "インターネットゲートウェイのID"
  value       = aws_internet_gateway.main.id
}

output "public_route_table_id" {
  description = "パブリックルートテーブルのID"
  value       = aws_route_table.public.id
}
```

### モジュールの入力と出力の流れ

```
ルートモジュール（呼び出し側）
    ↓
module "network" {
  source       = "./modules/network"
  project_name = "myapp"      ← 入力変数を渡す
  environment  = "dev"
  vpc_cidr     = "10.0.0.0/16"
}
    ↓
【network子モジュール】
    ↓
variables.tf で受け取る
    ↓
main.tf で処理（VPC、サブネットなどを作成）
    ↓
outputs.tf で返す
    ↓
ルートモジュールで使用
    ↓
module.network.vpc_id
module.network.public_subnet_ids
```

## 実践：ストレージモジュールの作成

### ステップ3：ストレージ子モジュールを作成

#### modules/storage/variables.tf

```hcl
# modules/storage/variables.tf

# =============================================================================
# 入力変数
# =============================================================================

variable "project_name" {
  description = "プロジェクト名"
  type        = string
}

variable "environment" {
  description = "環境名"
  type        = string
}

variable "bucket_name" {
  description = "S3バケット名（グローバルに一意である必要があります）"
  type        = string
}

variable "enable_versioning" {
  description = "バージョニングを有効にするか"
  type        = bool
  default     = false
}

variable "enable_encryption" {
  description = "暗号化を有効にするか"
  type        = bool
  default     = true
}
```

#### modules/storage/main.tf

```hcl
# modules/storage/main.tf

# =============================================================================
# S3バケットの作成
# =============================================================================

resource "aws_s3_bucket" "main" {
  bucket = var.bucket_name

  tags = {
    Name        = var.bucket_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# =============================================================================
# バージョニング設定
# =============================================================================

resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id

  versioning_configuration {
    status = var.enable_versioning ? "Enabled" : "Disabled"
  }
}

# =============================================================================
# 暗号化設定
# =============================================================================

resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  count = var.enable_encryption ? 1 : 0

  bucket = aws_s3_bucket.main.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# =============================================================================
# パブリックアクセスブロック（セキュリティのため）
# =============================================================================

resource "aws_s3_bucket_public_access_block" "main" {
  bucket = aws_s3_bucket.main.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

#### modules/storage/outputs.tf

```hcl
# modules/storage/outputs.tf

# =============================================================================
# 出力値
# =============================================================================

output "bucket_id" {
  description = "S3バケットのID"
  value       = aws_s3_bucket.main.id
}

output "bucket_arn" {
  description = "S3バケットのARN"
  value       = aws_s3_bucket.main.arn
}

output "bucket_domain_name" {
  description = "S3バケットのドメイン名"
  value       = aws_s3_bucket.main.bucket_domain_name
}

output "bucket_regional_domain_name" {
  description = "S3バケットのリージョナルドメイン名"
  value       = aws_s3_bucket.main.bucket_regional_domain_name
}
```

## ルートモジュールで子モジュールを使用

### ステップ4：ルートモジュールを作成

#### versions.tf

```hcl
# versions.tf

terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

#### providers.tf

```hcl
# providers.tf

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project   = var.project_name
      ManagedBy = "Terraform"
    }
  }
}
```

#### variables.tf（ルートモジュール）

```hcl
# variables.tf（ルートモジュール）

# =============================================================================
# 基本設定
# =============================================================================

variable "aws_region" {
  description = "AWSリージョン"
  type        = string
  default     = "ap-northeast-1"
}

variable "project_name" {
  description = "プロジェクト名"
  type        = string
  default     = "terraform-modules-practice"
}

variable "environment" {
  description = "環境名"
  type        = string
  default     = "development"
}

# =============================================================================
# ネットワーク設定
# =============================================================================

variable "vpc_cidr" {
  description = "VPCのCIDRブロック"
  type        = string
  default     = "10.0.0.0/16"
}

# =============================================================================
# ストレージ設定
# =============================================================================

variable "bucket_name" {
  description = "S3バケット名（グローバルに一意である必要があります）"
  type        = string
}
```

#### main.tf（ルートモジュール）

```hcl
# main.tf（ルートモジュール）

# =============================================================================
# ネットワーク子モジュールの呼び出し
# =============================================================================

module "network" {
  source = "./modules/network"

  # モジュールへの入力値
  project_name       = var.project_name
  environment        = var.environment
  vpc_cidr           = var.vpc_cidr
  availability_zones = ["ap-northeast-1a", "ap-northeast-1c"]
}

# =============================================================================
# ストレージ子モジュールの呼び出し
# =============================================================================

module "storage" {
  source = "./modules/storage"

  # モジュールへの入力値
  project_name      = var.project_name
  environment       = var.environment
  bucket_name       = var.bucket_name
  enable_versioning = true
  enable_encryption = true
}
```

### 子モジュール呼び出しの説明

```hcl
module "network" {           # モジュールの識別名
  source = "./modules/network"  # 子モジュールの場所

  # 以下は入力変数（子モジュールのvariables.tfで定義された変数）
  project_name = var.project_name
  environment  = var.environment
  vpc_cidr     = var.vpc_cidr
}
```

**重要なポイント：**

```
1. source（必須）：
   - 子モジュールの場所を指定
   - ローカルパス：./modules/network
   - Gitリポジトリ：git::https://...
   - Terraform Registry：hashicorp/consul/aws

2. 入力変数：
   - 子モジュールのvariables.tfで定義された変数
   - ルートモジュールから値を渡す
   - 必須の変数は必ず指定する
   - デフォルト値がある変数は省略可能

3. 識別名：
   - module "network" の "network" 部分
   - 出力値を参照する時に使用
   - 例：module.network.vpc_id
```

#### outputs.tf（ルートモジュール）

```hcl
# outputs.tf（ルートモジュール）

# =============================================================================
# ネットワーク情報
# =============================================================================

output "vpc_id" {
  description = "VPC ID"
  value       = module.network.vpc_id
  #             ↑モジュール名.出力名
}

output "public_subnet_ids" {
  description = "パブリックサブネットのIDリスト"
  value       = module.network.public_subnet_ids
}

output "private_subnet_ids" {
  description = "プライベートサブネットのIDリスト"
  value       = module.network.private_subnet_ids
}

# =============================================================================
# ストレージ情報
# =============================================================================

output "bucket_id" {
  description = "S3バケットID"
  value       = module.storage.bucket_id
}

output "bucket_arn" {
  description = "S3バケットARN"
  value       = module.storage.bucket_arn
}
```

#### terraform.tfvars

```hcl
# terraform.tfvars

aws_region   = "ap-northeast-1"
project_name = "terraform-modules-practice"
environment  = "development"
vpc_cidr     = "10.0.0.0/16"

# 注意：バケット名はグローバルに一意である必要があります
# 自分の名前や日付を含めると良いでしょう
bucket_name = "your-name-terraform-practice-20250101"
```

### ステップ5：モジュールを初期化して実行

```bash
# モジュールをダウンロード（初期化）
terraform init

# 実行計画を確認
terraform plan

# リソースを作成
terraform apply
```

**terraform initの実行結果：**

```
Initializing modules...
- network in modules/network
- storage in modules/storage

Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.x.x...

Terraform has been successfully initialized!
```

## モジュール間での値の受け渡し

### パターン1：子モジュールの出力を別の子モジュールの入力に渡す

```hcl
# main.tf（ルートモジュール）

# ネットワーク子モジュール
module "network" {
  source = "./modules/network"

  project_name = var.project_name
  environment  = var.environment
  vpc_cidr     = var.vpc_cidr
}

# コンピュート子モジュール（network子モジュールの出力を使用）
module "compute" {
  source = "./modules/compute"

  project_name = var.project_name
  environment  = var.environment

  # network子モジュールの出力を入力として渡す
  vpc_id            = module.network.vpc_id
  public_subnet_ids = module.network.public_subnet_ids
}
```

**データの流れ：**

```
network子モジュール
    ↓
  出力：vpc_id
    ↓
compute子モジュールの入力
    ↓
  使用：VPC内にEC2を作成
```

### パターン2：複数の子モジュール出力を組み合わせる

```hcl
# main.tf（ルートモジュール）

module "network" {
  source = "./modules/network"
  # ...
}

module "storage" {
  source = "./modules/storage"
  # ...
}

module "application" {
  source = "./modules/application"

  # 複数の子モジュールの出力を組み合わせる
  vpc_id             = module.network.vpc_id
  private_subnet_ids = module.network.private_subnet_ids
  bucket_arn         = module.storage.bucket_arn
  bucket_id          = module.storage.bucket_id
}
```

### パターン3：ローカル値で加工してから渡す

```hcl
# main.tf（ルートモジュール）

module "network" {
  source = "./modules/network"
  # ...
}

# 子モジュールの出力を加工
locals {
  # サブネットIDを文字列に変換
  subnet_ids_string = join(",", module.network.public_subnet_ids)

  # 環境に応じた設定
  instance_count = var.environment == "production" ? 4 : 2
}

module "compute" {
  source = "./modules/compute"

  subnet_ids     = local.subnet_ids_string
  instance_count = local.instance_count
}
```

## 依存関係の制御

### 暗黙的な依存関係

```hcl
# Terraformが自動的に依存関係を検出

module "network" {
  source = "./modules/network"
  # ...
}

module "compute" {
  source = "./modules/compute"

  # network子モジュールの出力を参照
  # Terraformは自動的にnetwork → computeの順序を理解
  vpc_id = module.network.vpc_id
}

# 実行順序：
# 1. network子モジュール
# 2. compute子モジュール（networkの後）
```

### 明示的な依存関係

```hcl
# depends_onで明示的に依存関係を指定

module "network" {
  source = "./modules/network"
  # ...
}

module "monitoring" {
  source = "./modules/monitoring"
  # ...
}

module "application" {
  source = "./modules/application"
  # ...

  # 明示的な依存関係
  # networkとmonitoringの後に実行
  depends_on = [
    module.network,
    module.monitoring
  ]
}

# 実行順序：
# 1. network子モジュールとmonitoring子モジュール（並列）
# 2. application子モジュール（network + monitoring の後）
```

## モジュールのベストプラクティス

### 1. モジュールは単一の責任を持つ

```
良い例：
  - network子モジュール：ネットワーク関連のみ
  - storage子モジュール：ストレージ関連のみ
  - compute子モジュール：コンピュート関連のみ

悪い例：
  - infrastructure子モジュール：すべてを含む
  理由：大きすぎて再利用できない
```

### 2. 明確な入力と出力を定義

```hcl
# 良い例：descriptionを詳しく書く

variable "vpc_cidr" {
  description = <<-EOT
    VPCのCIDRブロック。
    例：10.0.0.0/16（65536個のIPアドレス）
    推奨：/16から/24の範囲
  EOT
  type        = string
}

output "vpc_id" {
  description = "作成したVPCのID。他のリソースでVPCを参照する時に使用します。"
  value       = aws_vpc.main.id
}

# 悪い例：説明が不十分

variable "vpc_cidr" {
  description = "CIDR"  # 不十分
  type        = string
}
```

### 3. デフォルト値を適切に設定

```hcl
# 良い例：一般的な値をデフォルトに

variable "enable_versioning" {
  description = "S3バケットのバージョニングを有効にするか"
  type        = bool
  default     = false  # デフォルトは無効（コストを考慮）
}

variable "availability_zones" {
  description = "使用するアベイラビリティゾーン"
  type        = list(string)
  default     = ["ap-northeast-1a", "ap-northeast-1c"]  # 一般的な設定
}

# 必須の値はデフォルトを設定しない

variable "project_name" {
  description = "プロジェクト名（必須）"
  type        = string
  # defaultなし = 必須入力
}
```

### 4. バリデーションを追加

```hcl
variable "environment" {
  description = "環境名"
  type        = string

  validation {
    condition = contains(
      ["development", "staging", "production"],
      var.environment
    )
    error_message = "environmentはdevelopment、staging、productionのいずれかである必要があります。"
  }
}

variable "vpc_cidr" {
  description = "VPCのCIDRブロック"
  type        = string

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "有効なCIDR形式を指定してください（例：10.0.0.0/16）。"
  }
}
```

### 5. READMEを作成

````markdown
# modules/network/README.md

# ネットワークモジュール

VPC、サブネット、インターネットゲートウェイなどのネットワークインフラを作成するモジュール。

## 使用例

```hcl
module "network" {
  source = "./modules/network"

  project_name       = "my-project"
  environment        = "development"
  vpc_cidr           = "10.0.0.0/16"
  availability_zones = ["ap-northeast-1a", "ap-northeast-1c"]
}
```
````

## 入力変数

| 名前               | 説明              | 型           | デフォルト                             | 必須 |
| ------------------ | ----------------- | ------------ | -------------------------------------- | ---- |
| project_name       | プロジェクト名    | string       | -                                      | yes  |
| environment        | 環境名            | string       | -                                      | yes  |
| vpc_cidr           | VPCのCIDRブロック | string       | -                                      | yes  |
| availability_zones | 使用するAZ        | list(string) | ["ap-northeast-1a", "ap-northeast-1c"] | no   |

## 出力値

| 名前               | 説明                             |
| ------------------ | -------------------------------- |
| vpc_id             | VPCのID                          |
| public_subnet_ids  | パブリックサブネットのIDリスト   |
| private_subnet_ids | プライベートサブネットのIDリスト |

```

### 6. モジュールの入れ子は浅く保つ

```

推奨：フラットな構造

root/
├── main.tf
└── modules/
├── network/
├── compute/
└── storage/

main.tfから各子モジュールを直接呼び出す

避けるべき：深い入れ子

root/
├── main.tf
└── modules/
└── infrastructure/
└── modules/
└── network/
└── modules/
└── vpc/

理由：

- 複雑すぎる
- 再利用が困難
- デバッグが大変

````

## モジュールの再利用パターン

### パターン1：複数環境での使用

```hcl
# main.tf（ルートモジュール）

# 開発環境
module "network_dev" {
  source = "./modules/network"

  project_name = "myapp"
  environment  = "development"
  vpc_cidr     = "10.0.0.0/16"
}

# ステージング環境
module "network_staging" {
  source = "./modules/network"

  project_name = "myapp"
  environment  = "staging"
  vpc_cidr     = "10.1.0.0/16"
}

# 本番環境
module "network_prod" {
  source = "./modules/network"

  project_name = "myapp"
  environment  = "production"
  vpc_cidr     = "10.2.0.0/16"
}
````

### パターン2：for_eachで複数インスタンス

```hcl
# main.tf（ルートモジュール）

variable "environments" {
  type = map(object({
    vpc_cidr           = string
    availability_zones = list(string)
  }))

  default = {
    development = {
      vpc_cidr           = "10.0.0.0/16"
      availability_zones = ["ap-northeast-1a"]
    }
    staging = {
      vpc_cidr           = "10.1.0.0/16"
      availability_zones = ["ap-northeast-1a", "ap-northeast-1c"]
    }
    production = {
      vpc_cidr           = "10.2.0.0/16"
      availability_zones = ["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]
    }
  }
}

# for_eachで複数環境を一度に作成
module "network" {
  for_each = var.environments

  source = "./modules/network"

  project_name       = "myapp"
  environment        = each.key
  vpc_cidr           = each.value.vpc_cidr
  availability_zones = each.value.availability_zones
}

# 参照方法
output "dev_vpc_id" {
  value = module.network["development"].vpc_id
}

output "prod_vpc_id" {
  value = module.network["production"].vpc_id
}
```

## トラブルシューティング

### エラー1：モジュールが見つからない

```
Error: Module not installed

  on main.tf line 10:
  10: module "network" {

This module is not yet installed. Run "terraform init" to install all modules
required by this configuration.
```

**解決方法：**

```bash
# モジュールを初期化
terraform init
```

### エラー2：必須変数が指定されていない

```
Error: Missing required argument

  on main.tf line 10, in module "network":
  10: module "network" {

The argument "project_name" is required, but no definition was found.
```

**解決方法：**

```hcl
# 必須変数を追加
module "network" {
  source = "./modules/network"

  project_name = "myapp"  # 追加
  environment  = "development"
  vpc_cidr     = "10.0.0.0/16"
}
```

### エラー3：子モジュールの出力が見つからない

```
Error: Unsupported attribute

  on main.tf line 20:
  20:   vpc_id = module.network.vpc_id

This object does not have an attribute named "vpc_id".
```

**解決方法：**

```hcl
# modules/network/outputs.tfを確認
# 出力が定義されているか確認

output "vpc_id" {
  value = aws_vpc.main.id
}
```

## まとめ

### モジュールの基本概念

```
ルートモジュール：
- terraform applyを実行するディレクトリ
- 子モジュールを呼び出す
- プロジェクトのメイン

子モジュール：
- modules/ディレクトリ配下
- 再利用可能な部品
- 特定の機能を持つ
```

### モジュールの作成手順

```
1. ディレクトリ構造を作成
   modules/
   └── network/
       ├── main.tf
       ├── variables.tf
       └── outputs.tf

2. variables.tfで入力変数を定義
   - 必要な入力を明確に
   - descriptionを詳しく
   - デフォルト値を適切に

3. main.tfでリソースを定義
   - 変数を使って柔軟に
   - タグを適切に付ける

4. outputs.tfで出力を定義
   - 他のモジュールで使う値
   - descriptionを詳しく

5. ルートモジュールから呼び出し
   - sourceを指定
   - 必要な変数を渡す
```

### 重要なポイント

1. **単一責任の原則**

- 1つの子モジュール = 1つの機能
- 小さく、再利用可能に

2. **明確なインターフェース**

- 入力変数：何を受け取るか
- 出力値：何を返すか
- ドキュメント化

3. **依存関係の管理**

- 出力→入力で接続
- depends_onで明示的に

4. **フラットな構造**

- モジュールの入れ子は浅く
- ルートから直接呼び出す

5. **再利用性**

- 環境ごとに使い回す
- for_eachで複数作成
