# Terraform関数とCLIコマンド完全ガイド

## この章で学ぶこと

Terraformの実践的な関数とCLIコマンドを学習します。日常的に使用する機能に焦点を当てます。

**学習内容：**

- よく使うTerraform関数
- 実践的なCLIコマンド
- デバッグとトラブルシューティング
- 実際の使用例

## Terraform関数とは

### 基本的な考え方

**例えて言うなら：** Excelの関数

```
Excel:
  SUM(A1:A10)     - 合計を計算
  UPPER(A1)       - 大文字に変換
  IF(A1>10, "大", "小") - 条件分岐

Terraform:
  length(list)    - リストの長さを取得
  upper(string)   - 大文字に変換
  condition ? true_val : false_val - 条件分岐
```

**Terraformでの使用：**

```hcl
# 関数を使わない場合
resource "aws_subnet" "public_1" {
  cidr_block = "10.0.1.0/24"
}

resource "aws_subnet" "public_2" {
  cidr_block = "10.0.2.0/24"
}

resource "aws_subnet" "public_3" {
  cidr_block = "10.0.3.0/24"
}

# 関数を使う場合（自動計算）
resource "aws_subnet" "public" {
  count = 3

  cidr_block = cidrsubnet(var.vpc_cidr, 8, count.index)
}
```

## よく使う文字列関数

### 1. format() - 文字列のフォーマット

**基本的な使い方：**

```hcl
# 基本構文
format(format_string, arguments...)

# 実例
locals {
  # %s：文字列を挿入
  server_name = format("%s-server-%s", var.project_name, var.environment)
  # 結果: "myapp-server-production"

  # %d：数値を挿入
  instance_name = format("web-%02d", 1)
  # 結果: "web-01"（2桁でゼロ埋め）
}
```

**実践例：リソース名の生成**

```hcl
variable "project_name" {
  type    = string
  default = "myapp"
}

variable "environment" {
  type    = string
  default = "production"
}

resource "aws_instance" "web" {
  count = 3

  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    # web-01, web-02, web-03
    Name = format("web-%02d", count.index + 1)

    # myapp-production-web-01
    FullName = format("%s-%s-web-%02d",
      var.project_name,
      var.environment,
      count.index + 1
    )
  }
}
```

**フォーマット指定子：**

```hcl
locals {
  # %s：文字列
  str_example = format("Hello %s", "World")
  # 結果: "Hello World"

  # %d：10進数
  num_example = format("Number: %d", 42)
  # 結果: "Number: 42"

  # %02d：2桁でゼロ埋め
  zero_pad = format("ID: %02d", 5)
  # 結果: "ID: 05"

  # %03d：3桁でゼロ埋め
  zero_pad_3 = format("Code: %03d", 7)
  # 結果: "Code: 007"

  # 複数の値
  multi = format("%s has %d items", "Basket", 10)
  # 結果: "Basket has 10 items"
}
```

### 2. join() と split() - 文字列の結合と分割

**join() - リストを文字列に結合：**

```hcl
# 基本構文
join(separator, list)

# 実例
locals {
  subnet_ids = [
    "subnet-abc123",
    "subnet-def456",
    "subnet-ghi789"
  ]

  # カンマで結合
  subnet_string = join(",", local.subnet_ids)
  # 結果: "subnet-abc123,subnet-def456,subnet-ghi789"

  # スペースで結合
  subnet_spaces = join(" ", local.subnet_ids)
  # 結果: "subnet-abc123 subnet-def456 subnet-ghi789"
}
```

**実践例：タグの結合**

```hcl
variable "tags_list" {
  type    = list(string)
  default = ["web", "api", "production"]
}

resource "aws_instance" "app" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "app-server"
    # タグをハイフンで結合
    Tags = join("-", var.tags_list)
    # 結果: "web-api-production"
  }
}
```

**split() - 文字列をリストに分割：**

```hcl
# 基本構文
split(separator, string)

# 実例
locals {
  # カンマ区切りの文字列
  csv_string = "apple,banana,orange"

  # リストに分割
  fruits = split(",", local.csv_string)
  # 結果: ["apple", "banana", "orange"]

  # IPアドレスの分割
  ip_address = "192.168.1.100"
  ip_parts   = split(".", local.ip_address)
  # 結果: ["192", "168", "1", "100"]
}
```

**実践例：CIDR範囲の処理**

```hcl
variable "allowed_cidrs" {
  description = "許可するCIDR範囲（カンマ区切り）"
  type        = string
  default     = "10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
}

locals {
  # 文字列をリストに変換
  cidr_list = split(",", var.allowed_cidrs)
  # 結果: ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"]
}

resource "aws_security_group" "app" {
  name        = "app-sg"
  description = "Application security group"
  vpc_id      = aws_vpc.main.id

  # 各CIDRに対してルールを作成
  dynamic "ingress" {
    for_each = local.cidr_list

    content {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = [ingress.value]
    }
  }
}
```

### 3. upper()、lower()、title() - 大文字小文字の変換

```hcl
locals {
  original = "Hello World"

  # すべて大文字
  uppercase = upper(local.original)
  # 結果: "HELLO WORLD"

  # すべて小文字
  lowercase = lower(local.original)
  # 結果: "hello world"

  # 各単語の先頭を大文字
  titlecase = title(local.original)
  # 結果: "Hello World"
}
```

**実践例：環境名の正規化**

```hcl
variable "environment" {
  type = string
}

locals {
  # 入力値を小文字に統一
  env_normalized = lower(var.environment)

  # タグ用には大文字に
  env_tag = upper(var.environment)
}

resource "aws_instance" "app" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name        = "${local.env_normalized}-app-server"
    Environment = local.env_tag
  }
}
```

### 4. trimspace()、trim() - 空白の削除

```hcl
locals {
  # 前後の空白を削除
  text_with_spaces = "  Hello World  "
  trimmed          = trimspace(local.text_with_spaces)
  # 結果: "Hello World"

  # 特定の文字を削除
  text_with_chars = "###Hello###"
  trimmed_chars   = trim(local.text_with_chars, "#")
  # 結果: "Hello"
}
```

**実践例：ユーザー入力の正規化**

```hcl
variable "project_name" {
  type = string
}

locals {
  # 前後の空白を削除して小文字化
  normalized_name = lower(trimspace(var.project_name))
}

resource "aws_s3_bucket" "data" {
  # クリーンな名前を使用
  bucket = "${local.normalized_name}-data-bucket"
}
```

## よく使うコレクション関数

### 1. length() - 長さを取得

```hcl
locals {
  # リストの長さ
  subnet_ids = ["subnet-1", "subnet-2", "subnet-3"]
  subnet_count = length(local.subnet_ids)
  # 結果: 3

  # 文字列の長さ
  text = "Hello"
  text_length = length(local.text)
  # 結果: 5

  # マップの要素数
  tags = {
    Name = "server"
    Env  = "prod"
  }
  tag_count = length(local.tags)
  # 結果: 2
}
```

**実践例：動的なリソース作成**

```hcl
variable "availability_zones" {
  type    = list(string)
  default = ["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]
}

# AZの数だけサブネットを作成
resource "aws_subnet" "public" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = format("public-subnet-%d-of-%d",
      count.index + 1,
      length(var.availability_zones)
    )
  }
}
```

### 2. concat() - リストの結合

```hcl
locals {
  public_subnets  = ["subnet-pub-1", "subnet-pub-2"]
  private_subnets = ["subnet-priv-1", "subnet-priv-2"]

  # リストを結合
  all_subnets = concat(local.public_subnets, local.private_subnets)
  # 結果: ["subnet-pub-1", "subnet-pub-2", "subnet-priv-1", "subnet-priv-2"]
}
```

**実践例：複数のセキュリティグループの結合**

```hcl
variable "base_security_groups" {
  type = list(string)
  default = ["sg-base-1", "sg-base-2"]
}

variable "additional_security_groups" {
  type    = list(string)
  default = []
}

resource "aws_instance" "app" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  # 基本SGと追加SGを結合
  vpc_security_group_ids = concat(
    var.base_security_groups,
    var.additional_security_groups
  )
}
```

### 3. merge() - マップの結合

```hcl
locals {
  base_tags = {
    ManagedBy = "Terraform"
    Project   = "MyApp"
  }

  env_tags = {
    Environment = "production"
  }

  # マップを結合
  all_tags = merge(local.base_tags, local.env_tags)
  # 結果: {
  #   ManagedBy   = "Terraform"
  #   Project     = "MyApp"
  #   Environment = "production"
  # }
}
```

**実践例：共通タグとリソース固有タグの結合**

```hcl
variable "project_name" {
  type = string
}

variable "environment" {
  type = string
}

locals {
  # すべてのリソースに付ける共通タグ
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  # 共通タグとリソース固有タグを結合
  tags = merge(
    local.common_tags,
    {
      Name = "web-server"
      Role = "frontend"
    }
  )
}

resource "aws_instance" "db" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.medium"

  # 共通タグとリソース固有タグを結合
  tags = merge(
    local.common_tags,
    {
      Name = "database-server"
      Role = "backend"
    }
  )
}
```

### 4. contains() - リストに値が含まれるかチェック

```hcl
locals {
  allowed_regions = ["ap-northeast-1", "ap-northeast-3", "us-east-1"]

  # 値が含まれるかチェック
  is_tokyo = contains(local.allowed_regions, "ap-northeast-1")
  # 結果: true

  is_london = contains(local.allowed_regions, "eu-west-2")
  # 結果: false
}
```

**実践例：変数のバリデーション**

```hcl
variable "instance_type" {
  type = string
}

locals {
  allowed_instance_types = [
    "t4g.micro",
    "t4g.small",
    "t4g.medium",
    "t4g.large"
  ]
}

resource "aws_instance" "app" {
  ami           = "ami-xxxxx"
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition = contains(
        local.allowed_instance_types,
        var.instance_type
      )
      error_message = "許可されていないインスタンスタイプです。許可: ${join(", ", local.allowed_instance_types)}"
    }
  }
}
```

### 5. distinct() - 重複を削除

```hcl
locals {
  subnet_ids = [
    "subnet-1",
    "subnet-2",
    "subnet-1",  # 重複
    "subnet-3",
    "subnet-2"   # 重複
  ]

  # 重複を削除
  unique_subnets = distinct(local.subnet_ids)
  # 結果: ["subnet-1", "subnet-2", "subnet-3"]
}
```

**実践例：重複するAZの削除**

```hcl
variable "subnet_azs" {
  type = list(string)
  default = [
    "ap-northeast-1a",
    "ap-northeast-1c",
    "ap-northeast-1a",  # 重複
    "ap-northeast-1c"   # 重複
  ]
}

locals {
  # 重複を削除してユニークなAZリストを作成
  unique_azs = distinct(var.subnet_azs)
}

output "availability_zones_used" {
  description = "使用するユニークなAZ"
  value       = local.unique_azs
  # 結果: ["ap-northeast-1a", "ap-northeast-1c"]
}
```

### 6. flatten() - ネストされたリストを平坦化

```hcl
locals {
  # ネストされたリスト
  nested_list = [
    ["subnet-1", "subnet-2"],
    ["subnet-3", "subnet-4"],
    ["subnet-5"]
  ]

  # 平坦化
  flat_list = flatten(local.nested_list)
  # 結果: ["subnet-1", "subnet-2", "subnet-3", "subnet-4", "subnet-5"]
}
```

**実践例：複数のサブネットグループの統合**

```hcl
variable "public_subnet_ids" {
  type    = list(string)
  default = ["subnet-pub-1", "subnet-pub-2"]
}

variable "private_subnet_ids" {
  type    = list(string)
  default = ["subnet-priv-1", "subnet-priv-2"]
}

variable "database_subnet_ids" {
  type    = list(string)
  default = ["subnet-db-1", "subnet-db-2"]
}

locals {
  # すべてのサブネットを1つのリストに
  all_subnet_ids = flatten([
    var.public_subnet_ids,
    var.private_subnet_ids,
    var.database_subnet_ids
  ])
  # 結果: ["subnet-pub-1", "subnet-pub-2", "subnet-priv-1", "subnet-priv-2", "subnet-db-1", "subnet-db-2"]
}
```

## よく使うネットワーク関数

### 1. cidrsubnet() - サブネットCIDRの自動計算

**基本的な使い方：**

```hcl
# 基本構文
cidrsubnet(prefix, newbits, netnum)

# prefix: 元のCIDR（例："10.0.0.0/16"）
# newbits: 追加するビット数
# netnum: サブネット番号（0から開始）
```

**実例：**

```hcl
locals {
  vpc_cidr = "10.0.0.0/16"

  # /16 から /24 を作成（8ビット追加）
  subnet_1 = cidrsubnet(local.vpc_cidr, 8, 0)
  # 結果: "10.0.0.0/24"

  subnet_2 = cidrsubnet(local.vpc_cidr, 8, 1)
  # 結果: "10.0.1.0/24"

  subnet_3 = cidrsubnet(local.vpc_cidr, 8, 2)
  # 結果: "10.0.2.0/24"
}
```

**実践例：複数のサブネットを自動作成**

```hcl
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "availability_zones" {
  type    = list(string)
  default = ["ap-northeast-1a", "ap-northeast-1c"]
}

# パブリックサブネット
resource "aws_subnet" "public" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "public-${count.index + 1}"
  }
}

# プライベートサブネット（番号を10から開始）
resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "private-${count.index + 1}"
  }
}

# 結果：
# public-1:  10.0.0.0/24 (ap-northeast-1a)
# public-2:  10.0.1.0/24 (ap-northeast-1c)
# private-1: 10.0.10.0/24 (ap-northeast-1a)
# private-2: 10.0.11.0/24 (ap-northeast-1c)
```

### 2. cidrhost() - IPアドレスの計算

```hcl
locals {
  subnet_cidr = "10.0.1.0/24"

  # サブネット内の特定のIPアドレスを計算
  first_ip = cidrhost(local.subnet_cidr, 0)
  # 結果: "10.0.1.0"

  second_ip = cidrhost(local.subnet_cidr, 1)
  # 結果: "10.0.1.1"

  tenth_ip = cidrhost(local.subnet_cidr, 10)
  # 結果: "10.0.1.10"
}
```

**実践例：固定IPアドレスの割り当て**

```hcl
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

locals {
  private_subnet_cidr = cidrsubnet(var.vpc_cidr, 8, 10)
}

resource "aws_network_interface" "database" {
  subnet_id = aws_subnet.private.id

  # サブネット内の10番目のIPアドレスを使用
  private_ips = [cidrhost(local.private_subnet_cidr, 10)]

  tags = {
    Name = "database-eni"
  }
}
```

## よく使う型変換関数

### 1. tolist()、toset()、tomap() - 型変換

```hcl
locals {
  # 文字列からリストへ
  string_list = tolist(["a", "b", "c"])

  # リストからセットへ（重複を削除）
  list_input = ["a", "b", "a", "c"]
  unique_set = toset(local.list_input)
  # 結果: ["a", "b", "c"]（順序は保証されない）

  # オブジェクトからマップへ
  obj = {
    key1 = "value1"
    key2 = "value2"
  }
  map_output = tomap(local.obj)
}
```

**実践例：for_eachでリストを使用**

```hcl
variable "subnet_names" {
  type    = list(string)
  default = ["web", "app", "db"]
}

# for_eachはセットまたはマップが必要
resource "aws_subnet" "app" {
  for_each = toset(var.subnet_names)

  vpc_id     = aws_vpc.main.id
  cidr_block = cidrsubnet(var.vpc_cidr, 8, index(var.subnet_names, each.value))

  tags = {
    Name = "${each.value}-subnet"
  }
}
```

### 2. tonumber()、tostring() - 数値と文字列の変換

```hcl
locals {
  # 文字列から数値へ
  string_number = "42"
  number_value  = tonumber(local.string_number)
  # 結果: 42

  # 数値から文字列へ
  count_value = 10
  string_value = tostring(local.count_value)
  # 結果: "10"
}
```

**実践例：環境変数からの数値取得**

```hcl
variable "instance_count_string" {
  description = "インスタンス数（文字列として受け取る）"
  type        = string
  default     = "3"
}

locals {
  # 文字列を数値に変換
  instance_count = tonumber(var.instance_count_string)
}

resource "aws_instance" "app" {
  count = local.instance_count

  ami           = "ami-xxxxx"
  instance_type = "t4g.small"
}
```

## よく使う条件式と制御構文

### 1. 条件式（三項演算子）

```hcl
# 基本構文
condition ? true_value : false_value

# 実例
locals {
  environment = "production"

  # 環境に応じてインスタンスタイプを選択
  instance_type = var.environment == "production" ? "t4g.large" : "t4g.small"

  # 本番環境のみバックアップを有効化
  enable_backup = var.environment == "production" ? true : false
}
```

**実践例：環境別の設定**

```hcl
variable "environment" {
  type = string
}

locals {
  # 環境に応じた設定
  instance_config = {
    type          = var.environment == "production" ? "t4g.large" : "t4g.small"
    monitoring    = var.environment == "production" ? true : false
    multi_az      = var.environment == "production" ? true : false
    backup_retention = var.environment == "production" ? 7 : 1
  }
}

resource "aws_instance" "app" {
  ami           = "ami-xxxxx"
  instance_type = local.instance_config.type
  monitoring    = local.instance_config.monitoring

  tags = {
    Name = "app-server"
  }
}

resource "aws_db_instance" "main" {
  identifier        = "app-db"
  engine            = "mysql"
  instance_class    = "db.t3.small"
  multi_az          = local.instance_config.multi_az
  backup_retention_period = local.instance_config.backup_retention
}
```

### 2. for式 - リストとマップの変換

**リストの変換：**

```hcl
variable "instance_names" {
  type    = list(string)
  default = ["web", "api", "worker"]
}

locals {
  # リストの各要素を大文字に変換
  uppercase_names = [
    for name in var.instance_names : upper(name)
  ]
  # 結果: ["WEB", "API", "WORKER"]

  # フィルタリング付き変換
  filtered_names = [
    for name in var.instance_names : name
    if length(name) > 3
  ]
  # 結果: ["worker"]
}
```

**マップの変換：**

```hcl
variable "instances" {
  type = map(object({
    type = string
    ami  = string
  }))
  default = {
    web = {
      type = "t4g.small"
      ami  = "ami-web"
    }
    api = {
      type = "t4g.medium"
      ami  = "ami-api"
    }
  }
}

locals {
  # マップの値を変換
  instance_types = {
    for key, value in var.instances :
    key => value.type
  }
  # 結果: { web = "t4g.small", api = "t4g.medium" }
}
```

**実践例：リソースの動的作成**

```hcl
variable "subnets" {
  type = map(object({
    cidr = string
    az   = string
    type = string
  }))
  default = {
    public_1 = {
      cidr = "10.0.1.0/24"
      az   = "ap-northeast-1a"
      type = "public"
    }
    public_2 = {
      cidr = "10.0.2.0/24"
      az   = "ap-northeast-1c"
      type = "public"
    }
    private_1 = {
      cidr = "10.0.11.0/24"
      az   = "ap-northeast-1a"
      type = "private"
    }
  }
}

locals {
  # パブリックサブネットのみフィルタリング
  public_subnets = {
    for key, value in var.subnets :
    key => value
    if value.type == "public"
  }
}

resource "aws_subnet" "main" {
  for_each = var.subnets

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az

  # パブリックサブネットのみパブリックIP割り当て
  map_public_ip_on_launch = each.value.type == "public"

  tags = {
    Name = each.key
    Type = each.value.type
  }
}
```

## Terraform CLIコマンド

### 基本コマンド

#### 1. terraform init - 初期化

**基本的な使い方：**

```bash
# 基本的な初期化
terraform init

# プロバイダープラグインをアップグレード
terraform init -upgrade

# バックエンド設定を変更後に再初期化
terraform init -reconfigure

# バックエンドを変更して状態を移行
terraform init -migrate-state
```

**いつ使うか：**

```
必須の場面：
- 新しいプロジェクトを開始した時
- プロバイダーのバージョンを変更した時
- モジュールを追加・変更した時
- バックエンド設定を変更した時
- 他の人のコードをクローンした後
```

**実行例：**

```bash
# 新しいプロジェクトで初期化
cd terraform-project
terraform init

# 出力例：
Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.34.0"...
- Installing hashicorp/aws v5.34.0...
- Installed hashicorp/aws v5.34.0 (signed by HashiCorp)

Terraform has been successfully initialized!
```

#### 2. terraform plan - 実行計画の確認

**基本的な使い方：**

```bash
# 基本的なプラン確認
terraform plan

# 特定の変数ファイルを使用
terraform plan -var-file="production.tfvars"

# 特定の変数を指定
terraform plan -var="instance_type=t4g.large"

# プランをファイルに保存
terraform plan -out=tfplan

# 特定のリソースのみプラン
terraform plan -target=aws_instance.web
```

**実行例：**

```bash
terraform plan

# 出力例：
Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami                         = "ami-xxxxx"
      + instance_type               = "t4g.small"
      + id                          = (known after apply)
      + public_ip                   = (known after apply)
      # ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

**プランの読み方：**

```
記号の意味：
  + 作成される
  - 削除される
  ~ 変更される
  -/+ 削除後に再作成される
  <= データソースを読み取る
```

#### 3. terraform apply - リソースの作成・変更

**基本的な使い方：**

```bash
# 基本的な適用（確認プロンプトあり）
terraform apply

# 自動承認（CI/CDで使用）
terraform apply -auto-approve

# 保存したプランを適用
terraform apply tfplan

# 特定のリソースのみ適用
terraform apply -target=aws_instance.web

# 変数を指定して適用
terraform apply -var="environment=production"
```

**実行例：**

```bash
terraform apply

# 出力例：
Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      # ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_instance.web: Creating...
aws_instance.web: Creation complete after 45s [id=i-xxxxx]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

#### 4. terraform destroy - リソースの削除

**基本的な使い方：**

```bash
# すべてのリソースを削除
terraform destroy

# 自動承認
terraform destroy -auto-approve

# 特定のリソースのみ削除
terraform destroy -target=aws_instance.web

# 変数を指定して削除
terraform destroy -var-file="production.tfvars"
```

**実行例：**

```bash
terraform destroy

# 出力例：
Terraform will perform the following actions:

  # aws_instance.web will be destroyed
  - resource "aws_instance" "web" {
      - id            = "i-xxxxx" -> null
      - instance_type = "t4g.small" -> null
      # ...
    }

Plan: 0 to add, 0 to change, 1 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

aws_instance.web: Destroying... [id=i-xxxxx]
aws_instance.web: Destruction complete after 30s

Destroy complete! Resources: 1 destroyed.
```

### 状態管理コマンド

#### 5. terraform state list - 状態ファイルのリソース一覧

```bash
# すべてのリソースを表示
terraform state list

# 出力例：
aws_vpc.main
aws_subnet.public[0]
aws_subnet.public[1]
aws_subnet.private[0]
aws_subnet.private[1]
aws_instance.web[0]
aws_instance.web[1]

# 特定のパターンでフィルタ
terraform state list | grep subnet

# 出力例：
aws_subnet.public[0]
aws_subnet.public[1]
aws_subnet.private[0]
aws_subnet.private[1]
```

#### 6. terraform state show - リソースの詳細表示

```bash
# 特定のリソースの詳細を表示
terraform state show aws_instance.web

# 出力例：
# aws_instance.web:
resource "aws_instance" "web" {
    ami                         = "ami-xxxxx"
    instance_type               = "t4g.small"
    id                          = "i-xxxxx"
    public_ip                   = "18.182.xxx.xxx"
    private_ip                  = "10.0.1.10"
    vpc_security_group_ids      = ["sg-xxxxx"]
    subnet_id                   = "subnet-xxxxx"
    # ...
}
```

#### 7. terraform state rm - 状態からリソースを削除

```bash
# 状態ファイルからリソースを削除（実際のリソースは削除されない）
terraform state rm aws_instance.web

# 複数のリソースを削除
terraform state rm aws_instance.web[0] aws_instance.web[1]

# モジュール全体を削除
terraform state rm module.network

# 注意：このコマンドは実際のAWSリソースを削除しません
# Terraformの管理下から外すだけです
```

**使用例：**

```
いつ使うか：
- リソースを手動管理に移行する時
- Terraformの管理から外したい時
- 誤ってインポートしたリソースを削除する時

注意：
- 実際のリソースは削除されない
- 次回のapplyで再作成される（コードが残っている場合）
```

#### 8. terraform state mv - リソースのアドレス変更

```bash
# リソース名を変更
terraform state mv aws_instance.old_name aws_instance.new_name

# インデックスを変更
terraform state mv 'aws_instance.web[0]' 'aws_instance.web[2]'

# モジュールに移動
terraform state mv aws_instance.web module.compute.aws_instance.web

# モジュールから取り出す
terraform state mv module.compute.aws_instance.web aws_instance.web
```

**使用例：moved ブロックの代わり**

```bash
# リソース名を変更した後
# moved ブロックを使わない場合は state mv を使用

terraform state mv aws_instance.web aws_instance.web_server

# これでリソースが削除・再作成されない
```

### 出力コマンド

#### 9. terraform output - 出力値の表示

```bash
# すべての出力を表示
terraform output

# 出力例：
instance_id = "i-xxxxx"
instance_public_ip = "18.182.xxx.xxx"
vpc_id = "vpc-xxxxx"

# 特定の出力のみ表示
terraform output instance_public_ip

# 出力例：
"18.182.xxx.xxx"

# JSON形式で出力
terraform output -json

# 出力例：
{
  "instance_id": {
    "sensitive": false,
    "type": "string",
    "value": "i-xxxxx"
  },
  "instance_public_ip": {
    "sensitive": false,
    "type": "string",
    "value": "18.182.xxx.xxx"
  }
}

# 特定の出力をJSON形式で
terraform output -json instance_public_ip

# 機密情報を含む出力を表示
terraform output database_password
```

**実践例：スクリプトでの使用**

```bash
#!/bin/bash

# 出力値を変数に格納
INSTANCE_IP=$(terraform output -raw instance_public_ip)

# SSH接続
ssh -i keypair.pem ec2-user@$INSTANCE_IP

# または複数の値を使用
VPC_ID=$(terraform output -raw vpc_id)
SUBNET_ID=$(terraform output -raw subnet_id)

echo "VPC: $VPC_ID"
echo "Subnet: $SUBNET_ID"
```

### デバッグコマンド

#### 10. terraform console - 対話的なコンソール

```bash
# コンソールを起動
terraform console

# コンソール内で式を評価
> var.vpc_cidr
"10.0.0.0/16"

> cidrsubnet(var.vpc_cidr, 8, 0)
"10.0.0.0/24"

> length(var.availability_zones)
2

> upper("hello")
"HELLO"

> local.common_tags
{
  "Environment" = "production"
  "ManagedBy" = "Terraform"
}

# 終了
> exit
```

**実践例：複雑な式のテスト**

```bash
terraform console

# CIDRの計算をテスト
> cidrsubnet("10.0.0.0/16", 8, 10)
"10.0.10.0/24"

# for式をテスト
> [for i in range(3) : format("subnet-%02d", i + 1)]
[
  "subnet-01",
  "subnet-02",
  "subnet-03",
]

# 条件式をテスト
> var.environment == "production" ? "t4g.large" : "t4g.small"
"t4g.small"
```

#### 11. terraform validate - 設定の検証

```bash
# 設定ファイルを検証
terraform validate

# 成功時の出力：
Success! The configuration is valid.

# エラー時の出力：
Error: Invalid resource type

  on main.tf line 10, in resource "aws_instanse" "web":
  10: resource "aws_instanse" "web" {

The resource type "aws_instanse" is not supported by provider
"registry.terraform.io/hashicorp/aws".

Did you mean "aws_instance"?
```

#### 12. terraform fmt - コードフォーマット

```bash
# 現在のディレクトリをフォーマット
terraform fmt

# すべてのサブディレクトリもフォーマット
terraform fmt -recursive

# 変更があるファイルを表示
terraform fmt -diff

# チェックのみ（変更しない）
terraform fmt -check -recursive
```

### 高度なコマンド

#### 13. terraform import - 既存リソースのインポート

```bash
# 既存のEC2インスタンスをインポート
terraform import aws_instance.web i-xxxxx

# 既存のS3バケットをインポート
terraform import aws_s3_bucket.data my-bucket-name

# モジュール内のリソースをインポート
terraform import module.network.aws_vpc.main vpc-xxxxx

# for_eachで作成したリソースをインポート
terraform import 'aws_instance.web["prod"]' i-xxxxx
```

#### 14. terraform taint / untaint - リソースの再作成マーク

```bash
# リソースを次回のapplyで再作成するようマーク（非推奨）
terraform taint aws_instance.web

# マークを解除（非推奨）
terraform untaint aws_instance.web

# 推奨：代わりに -replace を使用
terraform apply -replace=aws_instance.web
```

#### 15. terraform refresh - 状態の更新

```bash
# 状態ファイルを実際のリソースの状態に更新
terraform refresh

# 注意：このコマンドは通常使用しません
# plan や apply が自動的に refresh を実行するため
```

### 便利なオプション

#### よく使うグローバルオプション

```bash
# JSON形式で出力
terraform plan -json

# 詳細なログを出力
TF_LOG=DEBUG terraform apply

# ログをファイルに保存
TF_LOG=DEBUG TF_LOG_PATH=./terraform.log terraform apply

# 並列実行数を指定
terraform apply -parallelism=20

# カラー出力を無効化
terraform plan -no-color
```

### 実践的なワークフロー例

#### 日常的な開発フロー

```bash
# 1. コードを編集
# ... ファイルを編集 ...

# 2. フォーマット
terraform fmt -recursive

# 3. バリデーション
terraform validate

# 4. プラン確認
terraform plan

# 5. 問題なければ適用
terraform apply

# 6. 出力確認
terraform output
```

#### 環境別のデプロイ

```bash
# 開発環境
terraform plan -var-file="environments/dev.tfvars"
terraform apply -var-file="environments/dev.tfvars"

# ステージング環境
terraform plan -var-file="environments/staging.tfvars"
terraform apply -var-file="environments/staging.tfvars"

# 本番環境
terraform plan -var-file="environments/prod.tfvars" -out=prod.tfplan
terraform apply prod.tfplan
```

#### トラブルシューティング

```bash
# 詳細ログで問題を調査
TF_LOG=DEBUG terraform apply

# 特定のリソースのみ再作成
terraform apply -replace=aws_instance.web

# 状態ファイルとの不整合を確認
terraform plan -refresh-only

# 状態ファイルの内容を確認
terraform state list
terraform state show aws_instance.web

# 設定のテスト
terraform console
> cidrsubnet("10.0.0.0/16", 8, 0)
```

## まとめ

### よく使う関数

```
文字列関数：
- format()：文字列のフォーマット
- join()、split()：結合と分割
- upper()、lower()：大文字小文字変換

コレクション関数：
- length()：長さを取得
- concat()：リストの結合
- merge()：マップの結合
- contains()：値の存在チェック
- distinct()：重複削除
- flatten()：平坦化

ネットワーク関数：
- cidrsubnet()：サブネットCIDRの計算
- cidrhost()：IPアドレスの計算

型変換関数：
- tolist()、toset()、tomap()：型変換
- tonumber()、tostring()：数値と文字列の変換
```

### 必須CLIコマンド

```
基本コマンド：
- terraform init：初期化
- terraform plan：実行計画
- terraform apply：適用
- terraform destroy：削除

状態管理：
- terraform state list：リソース一覧
- terraform state show：詳細表示
- terraform state rm：削除
- terraform state mv：移動

その他：
- terraform output：出力表示
- terraform console：対話的コンソール
- terraform validate：検証
- terraform fmt：フォーマット
```

### 実践的な使い方

```
日常的な作業：
1. コード編集
2. terraform fmt -recursive
3. terraform validate
4. terraform plan
5. terraform apply

デバッグ：
1. terraform console で式をテスト
2. TF_LOG=DEBUG で詳細ログ
3. terraform state show で状態確認

環境管理：
1. 環境別の .tfvars ファイル
2. terraform workspace（複数環境）
3. -var-file オプションで切り替え
```
