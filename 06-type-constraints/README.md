# Terraformのデータ型：基礎から実践まで

## この章で学ぶこと

Terraformで使用するデータ型（data type）について学習します。データ型を理解することで、変数やリソースの設定をより正確に記述できるようになります。

**学習内容：**

- プリミティブ型（string、number、bool）
- コレクション型（list、map、set）
- 構造型（object、tuple）
- 型変換の仕組み
- 実践的な使用例

## データ型とは何か

### 日常生活での例え

**例えて言うなら：** データの入れ物の種類

```
文字列（string）  = 文字を書くノート
数値（number）    = 計算機
真偽値（bool）    = スイッチ（ON/OFF）
リスト（list）    = 順番に並んだ本棚
マップ（map）     = 住所録（名前で検索）
```

### なぜデータ型が重要なのか

1. **エラー防止**：正しい形式でデータを扱える
2. **コード品質**：意図を明確に表現できる
3. **自動検証**：Terraformが自動的にチェック
4. **ドキュメント**：どんな値が必要か一目で分かる

## プリミティブ型（基本的な型）

### 1. string（文字列）

**説明：** 文字の並びを表す型

**使用例：**

```hcl
# 変数での使用
variable "instance_name" {
  description = "EC2インスタンスの名前"
  type        = string
  default     = "web-server"
}

# リソースでの使用
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = var.instance_name  # string型の値
  }
}
```

**よくある使用場面：**

- リソース名
- AMI ID
- 説明文
- タグの値

**現在のコードでの使用例：**

```hcl
# あなたの現在のコードから
variable "aws_region" {
  description = "AWS region"
  type        = string              # ← string型を明示
  default     = "ap-northeast-1"
}

variable "project_name" {
  description = "Project name"
  type        = string              # ← string型を明示
  default     = "terraform-practice"
}
```

### 2. number（数値）

**説明：** 整数や小数を表す型

**使用例：**

```hcl
# 整数の例
variable "instance_count" {
  description = "作成するインスタンスの数"
  type        = number
  default     = 3
}

# 小数の例
variable "cpu_credits" {
  description = "CPUクレジット"
  type        = number
  default     = 0.5
}

# リソースでの使用
resource "aws_instance" "web" {
  count = var.instance_count  # number型の値

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

**よくある使用場面：**

- カウント数
- ポート番号
- サイズ指定
- タイムアウト時間

**現在のコードでの使用例：**

```hcl
# count でnumber型を使用
resource "aws_subnet" "public" {
  count = length(local.public_subnet_cidrs)  # ← number型を返す

  vpc_id = aws_vpc.main.id
  cidr_block = local.public_subnet_cidrs[count.index]  # ← count.index もnumber型
  # ...
}
```

### 3. bool（真偽値）

**説明：** trueまたはfalseの2つの値のみを持つ型

**使用例：**

```hcl
# 変数での使用
variable "enable_monitoring" {
  description = "モニタリングを有効化するか"
  type        = bool
  default     = true
}

variable "create_elastic_ip" {
  description = "Elastic IPを作成するか"
  type        = bool
  default     = false
}

# リソースでの使用
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  monitoring = var.enable_monitoring  # bool型の値
}

# 条件分岐での使用
resource "aws_eip" "example" {
  count = var.create_elastic_ip ? 1 : 0  # boolで条件判定

  instance = aws_instance.example.id
}
```

**よくある使用場面：**

- 機能の有効/無効
- リソースの作成/スキップ
- 設定のON/OFF

**現在のコードでの使用例：**

```hcl
# あなたの現在のVPCリソースから
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  # bool型で機能を制御
  enable_dns_hostnames = true  # ← bool型
  enable_dns_support   = true  # ← bool型
}

# サブネットでの使用
resource "aws_subnet" "public" {
  # ...
  map_public_ip_on_launch = true  # ← bool型
}
```

### プリミティブ型の自動変換

Terraformは必要に応じて型を自動的に変換します：

```hcl
# 数値から文字列への変換
variable "port" {
  type    = number
  default = 443
}

resource "aws_security_group_rule" "example" {
  type        = "ingress"
  from_port   = var.port
  to_port     = var.port
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]

  # descriptionは文字列を期待しているが、
  # 数値を含む文字列を作成できる
  description = "Allow port ${var.port}"  # "Allow port 443" になる
}

# 真偽値から文字列への変換
variable "enabled" {
  type    = bool
  default = true
}

locals {
  status = var.enabled ? "enabled" : "disabled"
  # または
  status_string = tostring(var.enabled)  # "true" になる
}
```

**変換の例：**

- `true` → `"true"`
- `false` → `"false"`
- `15` → `"15"`
- `"true"` → `true`
- `"123"` → `123`

## コレクション型（複数の値を扱う型）

### 1. list（リスト）

**説明：** 順序付けられた値の並び（同じ型の要素のみ）

**例えて言うなら：** 番号が振られた本棚

```hcl
# リストの定義
variable "availability_zones" {
  description = "使用するアベイラビリティゾーン"
  type        = list(string)
  default     = ["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]
}

variable "allowed_ports" {
  description = "許可するポート番号"
  type        = list(number)
  default     = [80, 443, 8080]
}

# リストの使用
resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  availability_zone = var.availability_zones[count.index]  # インデックスで取得
  cidr_block        = "10.0.${count.index}.0/24"
}

# 動的ブロックでの使用
resource "aws_security_group" "example" {
  name = "example-sg"

  dynamic "ingress" {
    for_each = var.allowed_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

**リストの操作：**

```hcl
# 要素の取得
locals {
  first_az  = var.availability_zones[0]   # "ap-northeast-1a"
  second_az = var.availability_zones[1]   # "ap-northeast-1c"

  # 長さの取得
  az_count = length(var.availability_zones)  # 3

  # 結合
  all_zones = concat(
    var.availability_zones,
    ["ap-northeast-1b"]
  )
}
```

**現在のコードでの使用例：**

```hcl
# あなたの現在のローカル値から
locals {
  # list(string)型
  availability_zones = slice(
    data.aws_availability_zones.available.names,
    0,
    2
  )

  public_subnet_cidrs = ["10.88.1.0/24", "10.88.2.0/24"]  # list(string)
  private_subnet_cidrs = ["10.88.11.0/24", "10.88.12.0/24"]  # list(string)
}

# リストを使用したリソース作成
resource "aws_subnet" "public" {
  count = length(local.public_subnet_cidrs)  # リストの長さ

  cidr_block = local.public_subnet_cidrs[count.index]  # インデックスで取得
  # ...
}
```

### 2. map（マップ）

**説明：** キーと値のペアで構成される（同じ型の値のみ）

**例えて言うなら：** 住所録（名前で情報を検索）

```hcl
# マップの定義
variable "instance_types" {
  description = "環境ごとのインスタンスタイプ"
  type        = map(string)
  default = {
    dev  = "t2.micro"
    stg  = "t2.small"
    prod = "t2.medium"
  }
}

variable "common_tags" {
  description = "全リソースに適用するタグ"
  type        = map(string)
  default = {
    Project     = "MyApp"
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

# マップの使用
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_types["prod"]  # キーで値を取得

  tags = var.common_tags
}

# 環境変数で切り替え
variable "environment" {
  type    = string
  default = "dev"
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_types[var.environment]  # 動的に選択
}
```

**マップの操作：**

```hcl
locals {
  # 値の取得
  prod_instance_type = var.instance_types["prod"]

  # マージ
  all_tags = merge(
    var.common_tags,
    {
      Name = "web-server"
      Role = "frontend"
    }
  )

  # キーの取得
  environments = keys(var.instance_types)  # ["dev", "stg", "prod"]

  # 値の取得
  types = values(var.instance_types)  # ["t2.micro", "t2.small", "t2.medium"]
}
```

**現在のコードでの使用例：**

```hcl
# あなたの現在のprovider設定から
provider "aws" {
  region = var.aws_region

  # map(string)型のタグ
  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project_name
      ManagedBy   = "Terraform"
    }
  }
}

# リソースのタグ（map(string)型）
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  tags = {
    Name = "${var.project_name}-vpc"
  }
}
```

### 3. set（セット）

**説明：** 順序がなく、重複しない値の集合

**例えて言うなら：** 袋に入った一意のカード

```hcl
# セットの定義
variable "required_tags" {
  description = "必須タグのキー"
  type        = set(string)
  default     = ["Environment", "Project", "Owner"]
}

variable "allowed_cidr_blocks" {
  description = "許可するCIDRブロック"
  type        = set(string)
  default = [
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16"
  ]
}

# セットの使用
resource "aws_security_group_rule" "allow_private" {
  for_each = var.allowed_cidr_blocks  # セットの各要素に対して実行

  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = [each.value]
  security_group_id = aws_security_group.example.id
}
```

**重要な特徴：**

```hcl
# セットは重複を自動的に除去
variable "example_set" {
  type    = set(string)
  default = ["a", "b", "c", "b", "a"]  # 実際は ["a", "b", "c"] になる
}

# セットは順序を保証しない
# インデックスで直接アクセスできない
# 必要な場合はリストに変換
locals {
  # セットをリストに変換
  zones_list = tolist(var.required_tags)
  first_tag  = local.zones_list[0]  # これでアクセス可能
}
```

### リスト、マップ、セットの使い分け

```hcl
# リスト：順序が重要な場合
variable "subnet_cidrs" {
  type = list(string)
  default = [
    "10.0.1.0/24",  # 1番目のサブネット
    "10.0.2.0/24",  # 2番目のサブネット
    "10.0.3.0/24"   # 3番目のサブネット
  ]
}

# マップ：名前で値を取得したい場合
variable "ami_ids" {
  type = map(string)
  default = {
    us-east-1      = "ami-12345"
    ap-northeast-1 = "ami-67890"
  }
}

# セット：重複を避けたい、順序が不要な場合
variable "security_groups" {
  type = set(string)
  default = [
    "sg-web",
    "sg-app",
    "sg-db"
  ]
}
```

## 構造型（複雑な型）

### 1. object（オブジェクト）

**説明：** 異なる型の値をまとめた型

**例えて言うなら：** フォーム（各項目が異なる種類の情報）

```hcl
# オブジェクトの定義
variable "vpc_config" {
  description = "VPC設定"
  type = object({
    cidr_block           = string
    enable_dns_hostnames = bool
    enable_dns_support   = bool
    instance_tenancy     = string
  })

  default = {
    cidr_block           = "10.0.0.0/16"
    enable_dns_hostnames = true
    enable_dns_support   = true
    instance_tenancy     = "default"
  }
}

# より複雑なオブジェクト
variable "server_config" {
  description = "サーバー設定"
  type = object({
    name          = string
    instance_type = string
    disk_size     = number
    monitoring    = bool
    tags          = map(string)
  })

  default = {
    name          = "web-server"
    instance_type = "t2.micro"
    disk_size     = 20
    monitoring    = true
    tags = {
      Environment = "production"
      Team        = "platform"
    }
  }
}

# オブジェクトの使用
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_config.cidr_block
  enable_dns_hostnames = var.vpc_config.enable_dns_hostnames
  enable_dns_support   = var.vpc_config.enable_dns_support
  instance_tenancy     = var.vpc_config.instance_tenancy
}

resource "aws_instance" "server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.server_config.instance_type

  tags = merge(
    var.server_config.tags,
    { Name = var.server_config.name }
  )
}
```

### オプショナル属性（optional）

Terraform 1.3以降では、オブジェクトの属性をオプションにできます：

```hcl
# オプショナル属性の定義
variable "app_config" {
  description = "アプリケーション設定"
  type = object({
    name        = string                      # 必須
    port        = optional(number, 8080)      # オプション（デフォルト値あり）
    ssl_enabled = optional(bool)              # オプション（デフォルトはnull）
    domain      = optional(string)            # オプション
    tags        = optional(map(string), {})   # オプション（空マップがデフォルト）
  })
}

# 使用例1：最小限の設定
app_config = {
  name = "myapp"
}
# 結果：
# {
#   name        = "myapp"
#   port        = 8080        # デフォルト値が使用される
#   ssl_enabled = null        # 指定なしでnull
#   domain      = null        # 指定なしでnull
#   tags        = {}          # 空マップ
# }

# 使用例2：一部をオーバーライド
app_config = {
  name        = "myapp"
  port        = 3000
  ssl_enabled = true
  tags = {
    Environment = "production"
  }
}
```

### ネストしたオブジェクト

```hcl
# 複雑なネストしたオブジェクト
variable "website_config" {
  description = "ウェブサイト設定"
  type = object({
    name    = string
    enabled = optional(bool, true)
    domain  = optional(object({
      name            = string
      use_https       = optional(bool, true)
      certificate_arn = optional(string)
    }))
    backend = object({
      port            = number
      health_check    = optional(object({
        path     = optional(string, "/health")
        interval = optional(number, 30)
        timeout  = optional(number, 5)
      }), {})
    })
  })
}

# 使用例
website_config = {
  name = "mysite"
  domain = {
    name      = "example.com"
    use_https = true
  }
  backend = {
    port = 8080
    health_check = {
      path     = "/api/health"
      interval = 60
    }
  }
}
```

### 2. tuple（タプル）

**説明：** 異なる型の値を順序付けて並べた型

**例えて言うなら：** 決まった形式の書類（各欄が異なる種類の情報）

```hcl
# タプルの定義
variable "server_spec" {
  description = "サーバースペック [名前, CPU数, メモリGB, SSD有無]"
  type        = tuple([string, number, number, bool])
  default     = ["web-server", 2, 4, true]
}

# 複数のサーバー定義
variable "servers" {
  description = "サーバーリスト"
  type = list(tuple([string, string, number]))
  default = [
    ["web-1", "t2.micro", 8],
    ["web-2", "t2.small", 16],
    ["app-1", "t2.medium", 32]
  ]
}

# タプルの使用
locals {
  server_name   = var.server_spec[0]  # "web-server"
  server_cpu    = var.server_spec[1]  # 2
  server_memory = var.server_spec[2]  # 4
  server_ssd    = var.server_spec[3]  # true
}

resource "aws_instance" "servers" {
  count = length(var.servers)

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.servers[count.index][1]  # インスタンスタイプ

  root_block_device {
    volume_size = var.servers[count.index][2]  # ディスクサイズ
  }

  tags = {
    Name = var.servers[count.index][0]  # サーバー名
  }
}
```

**タプルとリストの違い：**

```hcl
# リスト：すべて同じ型
variable "ports" {
  type    = list(number)
  default = [80, 443, 8080]  # すべてnumber
}

# タプル：それぞれ異なる型が可能
variable "endpoint" {
  type    = tuple([string, number, bool])
  default = ["example.com", 443, true]  # string, number, bool
}
```

## 型変換

### 自動型変換

Terraformは多くの場合、自動的に型を変換します：

```hcl
# 文字列から数値への変換
variable "port_string" {
  type    = string
  default = "443"
}

resource "aws_security_group_rule" "example" {
  type      = "ingress"
  from_port = var.port_string  # "443" → 443 に自動変換
  to_port   = var.port_string
  protocol  = "tcp"
}

# 数値から文字列への変換
variable "count_value" {
  type    = number
  default = 3
}

locals {
  message = "Created ${var.count_value} instances"  # 3 → "3" に自動変換
}

# 真偽値から文字列への変換
variable "is_production" {
  type    = bool
  default = true
}

resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Environment = var.is_production ? "production" : "development"
    IsProduction = tostring(var.is_production)  # true → "true"
  }
}
```

### コレクション型の変換

```hcl
# リストとタプルの変換
variable "zones_tuple" {
  type    = tuple([string, string, string])
  default = ["a", "b", "c"]
}

locals {
  # タプルをリストとして使用（自動変換）
  zones_list = var.zones_tuple

  # リストをタプルとして使用（要素数が一致する場合）
  fixed_zones = ["x", "y", "z"]  # これをtuple型として使える
}

# マップとオブジェクトの変換
variable "tags_map" {
  type = map(string)
  default = {
    Name        = "example"
    Environment = "production"
  }
}

locals {
  # マップをオブジェクトとして使用（自動変換）
  # ただし、オブジェクトに定義されていないキーは無視される
  required_tags = var.tags_map
}
```

### 明示的な型変換関数

```hcl
locals {
  # 文字列変換
  num_as_string  = tostring(42)         # "42"
  bool_as_string = tostring(true)       # "true"

  # 数値変換
  string_as_num = tonumber("123")       # 123

  # リスト変換
  set_to_list = tolist(["a", "b", "c"])

  # セット変換
  list_to_set = toset(["a", "b", "a"])  # ["a", "b"] 重複除去

  # マップ変換
  obj_to_map = tomap({
    key1 = "value1"
    key2 = "value2"
  })
}
```

## 実践例：現在のコードの改善

### 現在の変数定義を見直す

あなたの現在のコードは既に型が適切に指定されていますが、さらに構造化することもできます：

```hcl
# 現在のコード（良い例）
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.88.0.0/16"
}

# より構造化した例（オプション）
variable "network_config" {
  description = "ネットワーク設定"
  type = object({
    vpc_cidr             = string
    public_subnet_cidrs  = list(string)
    private_subnet_cidrs = list(string)
    availability_zones   = number  # 使用するAZ数
  })

  default = {
    vpc_cidr             = "10.88.0.0/16"
    public_subnet_cidrs  = ["10.88.1.0/24", "10.88.2.0/24"]
    private_subnet_cidrs = ["10.88.11.0/24", "10.88.12.0/24"]
    availability_zones   = 2
  }
}

# 使用例
resource "aws_vpc" "main" {
  cidr_block = var.network_config.vpc_cidr
  # ...
}

locals {
  availability_zones = slice(
    data.aws_availability_zones.available.names,
    0,
    var.network_config.availability_zones
  )

  public_subnet_cidrs  = var.network_config.public_subnet_cidrs
  private_subnet_cidrs = var.network_config.private_subnet_cidrs
}
```

## 実践演習：型の理解を深める

### 演習1：terraform consoleで型を確認

```bash
# Terraformコンソールを起動
terraform console

# 型を確認
> type("hello")
string

> type(123)
number

> type(true)
bool

> type(["a", "b", "c"])
tuple([string, string, string])

> type({name = "John", age = 30})
object({age: number, name: string})

# リストの操作
> length(["a", "b", "c"])
3

> ["a", "b", "c"][0]
"a"

# マップの操作
> {dev = "t2.micro", prod = "t2.small"}["dev"]
"t2.micro"

# 型変換
> tostring(42)
"42"

> tonumber("123")
123

> tolist(toset(["a", "b", "a"]))
["a", "b"]

# 現在のコードの変数を確認
> var.aws_region
"ap-northeast-1"

> var.vpc_cidr
"10.88.0.0/16"

> local.availability_zones
tolist([
  "ap-northeast-1a",
  "ap-northeast-1c",
])

> local.public_subnet_cidrs
[
  "10.88.1.0/24",
  "10.88.2.0/24",
]
```

### 演習2：型を使った新しい変数の追加

`main.tf`に以下を追加して、型の使い方を練習してください：

```hcl
# 型の練習用変数（main.tfの変数セクションに追加）

# プリミティブ型の例
variable "enable_vpc_flow_logs" {
  description = "VPCフローログを有効化するか"
  type        = bool
  default     = false
}

variable "log_retention_days" {
  description = "ログの保持日数"
  type        = number
  default     = 7
}

# コレクション型の例
variable "allowed_ssh_cidrs" {
  description = "SSH接続を許可するCIDRブロック"
  type        = list(string)
  default     = ["0.0.0.0/0"]
}

variable "resource_tags" {
  description = "追加のリソースタグ"
  type        = map(string)
  default = {
    Owner = "Platform Team"
    CostCenter = "Engineering"
  }
}

# オブジェクト型の例（オプション）
variable "monitoring_config" {
  description = "モニタリング設定"
  type = object({
    enabled           = bool
    alarm_email       = optional(string)
    retention_days    = optional(number, 30)
    detailed_metrics  = optional(bool, false)
  })

  default = {
    enabled = false
  }
}

# 出力して確認
output "type_practice" {
  description = "型の練習用出力"
  value = {
    flow_logs_enabled = var.enable_vpc_flow_logs
    retention_days    = var.log_retention_days
    ssh_cidrs         = var.allowed_ssh_cidrs
    tags              = var.resource_tags
    monitoring        = var.monitoring_config
  }
}
```

実行して確認：

```bash
# 検証
terraform validate

# 出力を確認
terraform refresh
terraform output type_practice
```

## よくある間違いと解決方法

### 間違い1：型の不一致

```hcl
# エラー例
variable "port" {
  type    = string
  default = 443  # エラー：numberをstringに割り当て
}

# 解決方法1：型を修正
variable "port" {
  type    = number  # 型をnumberに変更
  default = 443
}

# 解決方法2：値を修正
variable "port" {
  type    = string
  default = "443"  # 文字列に変更
}
```

### 間違い2：リスト内の型の不統一

```hcl
# エラー例
variable "mixed_list" {
  type = list(string)
  default = ["a", 1, true]  # エラー：異なる型が混在
}

# 解決方法1：型を統一
variable "string_list" {
  type    = list(string)
  default = ["a", "1", "true"]  # すべて文字列に
}

# 解決方法2：タプルを使用
variable "mixed_tuple" {
  type    = tuple([string, number, bool])
  default = ["a", 1, true]  # タプルなら異なる型OK
}
```

### 間違い3：オブジェクトの必須属性の欠落

```hcl
# エラー例
variable "server" {
  type = object({
    name = string
    port = number
  })

  default = {
    name = "web-server"
    # port が欠落している
  }
}

# 解決方法1：必須属性を追加
variable "server" {
  type = object({
    name = string
    port = number
  })

  default = {
    name = "web-server"
    port = 8080  # 必須属性を追加
  }
}

# 解決方法2：optionalを使用
variable "server" {
  type = object({
    name = string
    port = optional(number, 8080)  # デフォルト値付きでオプション化
  })

  default = {
    name = "web-server"
    # port は省略可能
  }
}
```

### 間違い4：nullの扱い

```hcl
# エラー例
variable "instance_type" {
  type    = string
  default = null  # エラー：nullはstringではない
}

# 解決方法：nullableを使用
variable "instance_type" {
  type     = string
  default  = null
  nullable = true  # nullを許可
}

# 使用時の注意
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type != null ? var.instance_type : "t2.micro"
}
```

## まとめ

### 型の選び方

```
単一の値を扱う
├─ 文字列 → string
├─ 数値 → number
└─ ON/OFF → bool

複数の値を扱う（同じ型）
├─ 順序が重要 → list
├─ 名前で検索 → map
└─ 重複を避ける → set

複数の値を扱う（異なる型）
├─ 名前で検索 → object
└─ 順序で検索 → tuple
```

### 重要なポイント

1. **型は必ず指定する**

- コードの意図が明確になる
- エラーを早期に発見できる
- ドキュメントとしても機能

2. **適切な型を選ぶ**

- データの性質に合った型を選択
- 過度に複雑な型は避ける
- 必要に応じてoptionalを活用

3. **型変換を理解する**

- 自動変換される場合とされない場合
- 明示的な変換関数の使用
- 型変換でのデータ損失に注意

4. **ベストプラクティスに従う**

- 型を明示的に指定
- 変数に説明を追加
- デフォルト値を適切に設定

型はTerraformの基礎となる重要な概念です。しっかりと理解して、正確で保守しやすいコードを書けるようになりましょう。
