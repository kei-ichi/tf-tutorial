# Terraform言語入門：実際のコードで学ぼう

## この章で学ぶこと

この章では、Terraform言語の基本的な構造を実際のコードを使って学びます。AWSでネットワーク（VPC）を作りながら、Terraformの各要素を理解していきましょう。

**重要な注意：** このレッスンで作成するコードは、この後の全てのレッスンで使用します。必ず保存しておいてください。

## AWSアカウントの準備

始める前に、以下の準備が必要です：

1. AWSアカウントを作成している
2. AWS CLIがインストールされ、認証情報が設定されている
3. AWS CLIのプロフィールは「default」に設定されている

## 完全なサンプルコード

まず、完全なコードを見てみましょう。新しいフォルダ `terraform-aws-practice` を作成し、`main.tf` ファイルに以下の内容を保存してください：

```hcl
# =============================================================================
# Terraformとプロバイダーの設定
# ベストプラクティス：バージョンを明確に指定してチーム全体で同じ環境を保つ
# =============================================================================
terraform {
  # 最低限必要なTerraformバージョンを指定
  # これによりチームメンバー全員が同じバージョンを使用
  required_version = ">= 1.0"

  # 使用するプロバイダー（AWSとの接続ツール）を明確に定義
  # バージョン固定により予期しない変更を防ぐ
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"  # 6.x.x の最新を使用（メジャー変更は避ける）
    }
  }
}

# =============================================================================
# AWSプロバイダーの設定
# ベストプラクティス：default_tagsで全リソースに統一されたタグを自動適用
# =============================================================================
provider "aws" {
  # 変数を使用してリージョンを指定（環境ごとに変更可能）
  region = var.aws_region

  # 全てのAWSリソースに自動的に付与されるタグ
  # 管理、コスト分析、セキュリティに重要
  # これによりタグの付け忘れを防ぐ
  default_tags {
    tags = {
      Environment = var.environment    # 開発/本番環境の識別
      Project     = var.project_name   # プロジェクト名
      ManagedBy   = "Terraform"        # Terraformで管理されていることを明示
    }
  }
}

# =============================================================================
# 変数の定義
# ベストプラクティス：ハードコードを避け、再利用可能なコードにする
# =============================================================================

# AWSリージョンの設定（東京リージョンをデフォルトに）
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-northeast-1"  # 東京リージョン

  # バリデーション追加も可能（今回は省略）
}

# 環境名の設定（development, staging, production など）
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "development"
}

# プロジェクト名（リソース名の一部として使用）
variable "project_name" {
  description = "Project name"
  type        = string
  default     = "terraform-practice"
}

# VPCのCIDRブロック（ネットワークアドレス範囲）
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.88.0.0/16"  # 65,536個のIPアドレスを含む
}

# =============================================================================
# データソース：既存のAWS情報を取得
# ベストプラクティス：動的な情報は実行時に取得する
# =============================================================================
data "aws_availability_zones" "available" {
  # 利用可能（正常に動作している）AZのみを取得
  state = "available"

  # このデータは実行時にAWSから取得される
  # リージョンによってAZの数や名前が異なるため動的取得が重要
}

# =============================================================================
# ローカル値の定義
# ベストプラクティス：複雑な計算や繰り返し使用する値を一箇所で管理
# =============================================================================
locals {
  # 最初の2つのAZのみを使用（高可用性のため2AZ構成）
  availability_zones = slice(data.aws_availability_zones.available.names, 0, 2)

  # パブリックサブネットのCIDR（インターネットアクセス可能）
  public_subnet_cidrs  = ["10.88.1.0/24", "10.88.2.0/24"]

  # プライベートサブネットのCIDR（内部通信のみ）
  private_subnet_cidrs = ["10.88.11.0/24", "10.88.12.0/24"]
}

# =============================================================================
# VPCの作成（仮想ネットワークの基盤）
# ベストプラクティス：DNS機能を有効にして名前解決を可能にする
# =============================================================================
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  # DNS機能の有効化（重要：これがないとサービス間通信が困難）
  enable_dns_hostnames = true  # DNSホスト名の割り当てを有効化
  enable_dns_support   = true  # DNS解決を有効化

  # Nameタグのみ指定（Environment、Project、ManagedByは自動適用）
  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# =============================================================================
# インターネットゲートウェイの作成
# パブリックサブネットがインターネットにアクセスするために必要
# =============================================================================
resource "aws_internet_gateway" "main" {
  # 作成したVPCにアタッチ
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-igw"
  }
}

# =============================================================================
# パブリックサブネットの作成
# ベストプラクティス：countを使用して複数サブネットを効率的に作成
# =============================================================================
resource "aws_subnet" "public" {
  # 配列の長さ分だけ繰り返し作成
  count = length(local.public_subnet_cidrs)

  vpc_id = aws_vpc.main.id

  # count.indexを使用して配列から値を取得
  cidr_block = local.public_subnet_cidrs[count.index]

  # 異なるAZに配置（高可用性のため）
  availability_zone = local.availability_zones[count.index]

  # パブリックIPの自動割り当て（インターネットアクセスに必要）
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project_name}-public-${count.index + 1}"
    Type = "Public"  # サブネットタイプを明示
  }
}

# =============================================================================
# プライベートサブネットの作成
# インターネットに直接接続しない安全なサブネット
# =============================================================================
resource "aws_subnet" "private" {
  count = length(local.private_subnet_cidrs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_subnet_cidrs[count.index]
  availability_zone = local.availability_zones[count.index]

  # パブリックIPは割り当てない（セキュリティのため）

  tags = {
    Name = "${var.project_name}-private-${count.index + 1}"
    Type = "Private"
  }
}

# =============================================================================
# パブリックサブネット用ルートテーブル
# ネットワークトラフィックの経路を定義
# =============================================================================
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  # インターネット向けトラフィック（0.0.0.0/0）をIGWに転送
  route {
    cidr_block = "0.0.0.0/0"            # 全てのインターネットトラフィック
    gateway_id = aws_internet_gateway.main.id  # IGWを経由
  }

  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

# =============================================================================
# パブリックサブネットとルートテーブルの関連付け
# サブネットにルーティング規則を適用
# =============================================================================
resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id

  # これにより各パブリックサブネットがインターネットアクセス可能になる
}

# =============================================================================
# 出力値の定義
# ベストプラクティス：他のモジュールや人が使用する重要な情報を出力
# =============================================================================

# VPC IDを出力（他のリソースで参照される重要な値）
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

# VPCのCIDRブロックを出力
output "vpc_cidr" {
  description = "VPC CIDR block"
  value       = aws_vpc.main.cidr_block
}

# パブリックサブネットのIDリストを出力（EC2やロードバランサーで使用）
output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id  # [*]で全要素のidを取得
}

# プライベートサブネットのIDリストを出力（データベースやアプリサーバーで使用）
output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = aws_subnet.private[*].id
}

# 使用したAZの情報を出力（確認用）
output "availability_zones" {
  description = "Availability zones used"
  value       = local.availability_zones
}
```

## コメントについて

コードを見ると、`#` で始まる行がたくさんあることに気づいたでしょう。

### コメント（#）の役割

```hcl
# これはコメントです - Terraformは実行しません
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr  # この部分もコメントです
}
```

**役割：** コードの説明や注釈を書く
**例えて言うなら：** 設計図に書かれた説明文やメモ

### コメントの特徴

1. **実行されない**：`#` 以降はTerraformに無視される
2. **説明のため**：コードの動作や理由を説明
3. **どこでも使用可能**：行の最初でも途中でも使用できる

### コメントの使い方

```hcl
# 1. 行全体をコメントにする場合
# これは VPC を作成するためのリソースです

# 2. コードの後ろに説明を追加する場合
cidr_block = "10.88.0.0/16"  # 65,536個のIPアドレスを含む

# 3. 複数行のコメント
# =============================================================================
# VPCの作成（仮想ネットワークの基盤）
# ベストプラクティス：DNS機能を有効にして名前解決を可能にする
# =============================================================================
```

### なぜコメントが重要なのか？

1. **理解しやすくなる**：後から見返した時に内容が分かる
2. **チームでの共有**：他の人がコードを理解できる
3. **ベストプラクティスの説明**：なぜそのような書き方をしたかを記録
4. **学習に役立つ**：各部分の役割を覚えやすい

**注意：** 今回のコードにはたくさんのコメントが書かれています。これらのコメントを読むことで、各部分が何をしているかを理解できます。

## コードを実行してみましょう

1. プロジェクトフォルダに移動：

```bash
cd terraform-aws-practice
```

2. 初期化：

```bash
terraform init
```

3. プランを確認：

```bash
terraform plan
```

4. リソースを作成：

```bash
terraform apply
```

`yes`と入力して実行してください。

## 各ブロックの説明

コードが動作することを確認したら、各ブロックがどのような役割を持っているかを学びましょう。

### 1. terraformブロック

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

**役割：** Terraform自体の設定
**例えて言うなら：** 工事に使う道具や材料の指定

- `required_version`：使用するTerraformのバージョン
- `required_providers`：使用するプロバイダー（今回はAWS）とそのバージョン

### 2. providerブロック

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
```

**役割：** クラウドサービス（AWS）との接続設定
**例えて言うなら：** 工事現場へのアクセス許可と作業場所の指定

- `region`：作業するAWSの地域
- `default_tags`：全てのリソースに自動で付けるラベル（重要なベストプラクティス）

### 3. variableブロック

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-northeast-1"
}
```

**役割：** 設定値を外部から変更可能にする
**例えて言うなら：** 建物の色やサイズなど、後から変更できる設計要素

- `description`：この変数の説明
- `type`：データの種類（文字列、数値など）
- `default`：デフォルト値

### 4. dataブロック

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}
```

**役割：** 既存の情報を取得する
**例えて言うなら：** 建設予定地の地図や土地の情報を調べる

- この例では、利用可能なアベイラビリティゾーン（データセンター）の一覧を取得

### 5. localsブロック

```hcl
locals {
  availability_zones = slice(data.aws_availability_zones.available.names, 0, 2)

  public_subnet_cidrs  = ["10.88.1.0/24", "10.88.2.0/24"]
  private_subnet_cidrs = ["10.88.11.0/24", "10.88.12.0/24"]
}
```

**役割：** 計算結果や複雑な値を保存する
**例えて言うなら：** 設計図の中で使う、計算済みの寸法や仕様

- 他の場所で何度も使う値を、一箇所で定義して管理
- **注意：** `common_tags`は削除しました（`default_tags`が自動的に適用されるため）

### 6. resourceブロック

```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}
```

**役割：** 実際に作成するAWSリソース
**例えて言うなら：** 設計図に基づいて実際に建設する建物や設備

- `aws_vpc`：VPC（仮想ネットワーク）を作成
- `cidr_block`：ネットワークのアドレス範囲
- `tags`：リソースに付けるラベル（`Name`のみ指定、他は自動適用）

### 7. outputブロック

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}
```

**役割：** 作成したリソースの情報を表示・提供する
**例えて言うなら：** 完成した建物の住所や電話番号を記録する

- 他の人やシステムが使える形で、重要な情報を出力

## default_tagsのベストプラクティス

今回のコードでは、`default_tags`を使用した重要なベストプラクティスを実装しています：

### なぜdefault_tagsを使うのか？

1. **タグの付け忘れ防止**：全てのリソースに自動的に適用される
2. **一貫性の保持**：チーム全体で同じタグが使用される
3. **コードの簡潔性**：各リソースで繰り返しタグを書く必要がない
4. **管理の容易さ**：タグの変更は一箇所で済む

### タグの役割

- `Environment`：開発、テスト、本番環境の識別
- `Project`：プロジェクトやチームの識別
- `ManagedBy`：リソースの管理方法を明示

## ネットワーク構成の説明

このコードで作成されるネットワーク構成：

```
VPC (10.88.0.0/16)
├── パブリックサブネット1 (10.88.1.0/24)  - AZ-1
├── パブリックサブネット2 (10.88.2.0/24)  - AZ-2
├── プライベートサブネット1 (10.88.11.0/24) - AZ-1
└── プライベートサブネット2 (10.88.12.0/24) - AZ-2
```

- **VPC**：あなた専用の仮想ネットワーク
- **パブリックサブネット**：インターネットに接続できる区域
- **プライベートサブネット**：内部通信専用の区域
- **アベイラビリティゾーン（AZ）**：異なるデータセンター

## コードの流れ

1. **terraform/providerブロック**：基本設定とタグ自動適用
2. **variableブロック**：設定可能な値を定義
3. **dataブロック**：AWS情報を取得
4. **localsブロック**：計算・整理
5. **resourceブロック**：実際にリソースを作成
6. **outputブロック**：結果を表示

## 重要な注意事項

**警告：このコードは今後のレッスンで継続使用します**

- このフォルダとファイルは削除しないでください
- `terraform.tfstate`ファイルは特に重要です
- 今後のレッスンでは、このコードに機能を追加していきます

## 費用について

このコードで作成されるリソースは、基本的に無料利用枠内で動作しますが：

- VPC、サブネット：無料
- インターネットゲートウェイ：無料
- データ転送：一定量まで無料

## まとめ

Terraformの基本構造：

- **terraform**：Terraform自体の設定
- **provider**：クラウドサービスとの接続（`default_tags`で統一管理）
- **variable**：変更可能な設定値
- **data**：既存情報の取得
- **locals**：計算結果の保存
- **resource**：実際に作成するもの
- **output**：結果の出力
