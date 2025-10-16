# Terraformの繰り返し処理：count、for_each、dynamic完全ガイド

## この章で学ぶこと

Terraformで同じようなリソースを複数作成する時に使う3つの方法を学習します。これらを使いこなすことで、コードの重複を避け、保守しやすい設定を作成できます。

**学習内容：**

- count：数を指定して同じリソースを複数作成
- for_each：データに基づいて柔軟に複数作成
- dynamic：ネストしたブロックを動的に生成
- 3つの使い分けとベストプラクティス
- 本番環境・エンタープライズでの推奨事項
- 実践的な活用例

## なぜ繰り返し処理が必要なのか

### 問題：コードの重複

**悪い例：同じようなコードを繰り返し書く**

```hcl
# サブネット1
resource "aws_subnet" "public_1" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  availability_zone = "ap-northeast-1a"

  tags = {
    Name = "public-subnet-1"
  }
}

# サブネット2
resource "aws_subnet" "public_2" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
  availability_zone = "ap-northeast-1c"

  tags = {
    Name = "public-subnet-2"
  }
}

# サブネット3
resource "aws_subnet" "public_3" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.3.0/24"
  availability_zone = "ap-northeast-1d"

  tags = {
    Name = "public-subnet-3"
  }
}

# ... さらに続く
```

**問題点：**

- コードが長くなる
- 変更が大変（すべてのブロックを修正する必要がある）
- ミスが起きやすい
- 数を変更する時にコードを追加・削除する必要がある

### 解決策：繰り返し処理を使う

**良い例：countを使う**

```hcl
resource "aws_subnet" "public" {
  count = 3  # 3つ作成

  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "public-subnet-${count.index + 1}"
  }
}
```

**メリット：**

- コードが短い
- 変更が簡単（1箇所を変更するだけ）
- 数の変更が簡単（countの値を変えるだけ）
- ミスが減る

## count：シンプルな繰り返し

### 基本的な考え方

**例えて言うなら：** コピー機で同じ資料を複数枚コピー

```
原本（resourceブロック）→ コピー機（count）→ 3枚の資料
                         「3枚コピー」
```

### countの基本構文

```hcl
resource "リソースタイプ" "名前" {
  count = 数値または式

  # countを使った属性の指定
  # count.index を使って各リソースを区別
}
```

**説明：**

- `count`：作成するリソースの数
- `count.index`：0から始まるインデックス番号（0, 1, 2, ...）
- 各リソースは自動的に異なる設定になる

### 実例1：基本的な使い方

```hcl
# EC2インスタンスを3つ作成
resource "aws_instance" "web" {
  count = 3

  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "web-server-${count.index + 1}"
  }
}

# 作成されるインスタンス：
# - web-server-1 (count.index = 0)
# - web-server-2 (count.index = 1)
# - web-server-3 (count.index = 2)
```

**count.indexの使い方：**

```hcl
resource "aws_subnet" "public" {
  count = 3

  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.${count.index + 1}.0/24"
  #                    ↑
  #                count.index = 0, 1, 2
  #                結果: 10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24

  tags = {
    Name = "public-subnet-${count.index + 1}"
    #                      結果: 1, 2, 3
  }
}
```

### 実例2：変数でcountを制御

```hcl
# 変数定義
variable "instance_count" {
  description = "作成するEC2インスタンスの数"
  type        = number
  default     = 2
}

# 変数を使ってcount
resource "aws_instance" "app" {
  count = var.instance_count  # 変数で数を制御

  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "app-server-${count.index + 1}"
  }
}

# terraform.tfvarsで数を変更
# instance_count = 5  # 5つ作成したい場合
```

### 実例3：リストと組み合わせる

```hcl
# アベイラビリティゾーンのリスト
variable "availability_zones" {
  type = list(string)
  default = [
    "ap-northeast-1a",
    "ap-northeast-1c",
    "ap-northeast-1d"
  ]
}

# リストの長さでcountを決定
resource "aws_subnet" "public" {
  count = length(var.availability_zones)  # リストの長さ = 3

  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = var.availability_zones[count.index]
  #                                          ↑
  #                   count.indexでリストの要素にアクセス

  tags = {
    Name = "public-subnet-${var.availability_zones[count.index]}"
  }
}

# 作成されるサブネット：
# - 10.0.1.0/24 in ap-northeast-1a
# - 10.0.2.0/24 in ap-northeast-1c
# - 10.0.3.0/24 in ap-northeast-1d
```

### 実例4：条件付きでリソースを作成

```hcl
variable "create_monitoring_instance" {
  description = "モニタリング用インスタンスを作成するか"
  type        = bool
  default     = true
}

# 条件によって0個または1個作成
resource "aws_instance" "monitoring" {
  count = var.create_monitoring_instance ? 1 : 0
  #       ↑
  #       trueなら1、falseなら0

  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "monitoring-server"
  }
}

# create_monitoring_instance = true  → 1個作成
# create_monitoring_instance = false → 0個作成（何も作らない）
```

### 実例5：計算式でcountを決定

```hcl
variable "subnet_count_per_az" {
  description = "各AZに作成するサブネットの数"
  type        = number
  default     = 2
}

locals {
  availability_zones = ["ap-northeast-1a", "ap-northeast-1c"]
}

# 2つのAZ × 2つのサブネット = 4つ作成
resource "aws_subnet" "private" {
  count = length(local.availability_zones) * var.subnet_count_per_az
  #       ↑
  #       2 × 2 = 4

  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.${count.index + 10}.0/24"

  # モジュロ演算でAZを循環
  availability_zone = local.availability_zones[count.index % length(local.availability_zones)]
  #                                            ↑
  #                   count.index % 2 = 0, 1, 0, 1 と繰り返す

  tags = {
    Name = "private-subnet-${count.index + 1}"
  }
}

# 作成されるサブネット：
# - subnet-1 in ap-northeast-1a (count.index = 0, 0 % 2 = 0)
# - subnet-2 in ap-northeast-1c (count.index = 1, 1 % 2 = 1)
# - subnet-3 in ap-northeast-1a (count.index = 2, 2 % 2 = 0)
# - subnet-4 in ap-northeast-1c (count.index = 3, 3 % 2 = 1)
```

### countで作成したリソースの参照方法

**単一のリソースを参照：**

```hcl
# 3つのインスタンスを作成
resource "aws_instance" "web" {
  count = 3

  ami           = "ami-xxxxx"
  instance_type = "t4g.small"
}

# 特定のインスタンスを参照
output "first_instance_id" {
  value = aws_instance.web[0].id  # 最初のインスタンス
}

output "second_instance_id" {
  value = aws_instance.web[1].id  # 2番目のインスタンス
}
```

**すべてのリソースを参照：**

```hcl
# すべてのインスタンスIDを取得
output "all_instance_ids" {
  value = aws_instance.web[*].id
  #                         ↑
  #                    スプラット演算子（すべての要素）
}

# 結果：["i-xxxxx", "i-yyyyy", "i-zzzzz"]
```

**特定の属性のリストを作成：**

```hcl
# すべてのインスタンスのパブリックIPを取得
output "all_public_ips" {
  value = aws_instance.web[*].public_ip
}

# すべてのインスタンスのプライベートIPを取得
output "all_private_ips" {
  value = aws_instance.web[*].private_ip
}
```

### countの注意点

#### 注意点1：インデックスベースのため、順序が重要

```hcl
# 初期状態：3つのサブネット
resource "aws_subnet" "public" {
  count = 3  # subnet[0], subnet[1], subnet[2]

  cidr_block = "10.0.${count.index + 1}.0/24"
}

# countを2に変更すると...
resource "aws_subnet" "public" {
  count = 2  # subnet[0], subnet[1] のみ

  cidr_block = "10.0.${count.index + 1}.0/24"
}

# 結果：subnet[2]が削除される
```

**問題：**

- 最後の要素が削除される
- 途中の要素を削除したい場合に不便

#### 注意点2：リスト要素の削除で予期しない変更が起きる

```hcl
variable "subnet_cidrs" {
  type = list(string)
  default = [
    "10.0.1.0/24",  # index 0
    "10.0.2.0/24",  # index 1
    "10.0.3.0/24"   # index 2
  ]
}

resource "aws_subnet" "public" {
  count = length(var.subnet_cidrs)

  cidr_block = var.subnet_cidrs[count.index]
}

# リストの途中（index 1）を削除すると...
variable "subnet_cidrs" {
  default = [
    "10.0.1.0/24",  # index 0（変更なし）
    "10.0.3.0/24"   # index 1（以前はindex 2だった）
  ]
}

# 結果：
# - subnet[0]: 変更なし
# - subnet[1]: 10.0.2.0/24 → 10.0.3.0/24 に変更（置き換え）
# - subnet[2]: 削除
```

**解決策：** このような場合は`for_each`を使用する

### countのベストプラクティス

**✓ countを使うべき場合：**

```hcl
# 1. 完全に同じ設定のリソースを複数作成
resource "aws_instance" "worker" {
  count = 5  # ワーカーノード5台

  ami           = "ami-xxxxx"
  instance_type = "t4g.small"
  # すべて同じ設定
}

# 2. 条件によってリソースの有無を切り替え
resource "aws_instance" "bastion" {
  count = var.enable_bastion ? 1 : 0
  # ...
}

# 3. 数値に基づいた単純な繰り返し
resource "aws_subnet" "public" {
  count = 3
  cidr_block = "10.0.${count.index + 1}.0/24"
}
```

**✗ countを避けるべき場合：**

```hcl
# 1. 各リソースが大きく異なる設定を持つ
# → for_eachを使用

# 2. リソースの識別に意味のある名前が必要
# → for_eachを使用

# 3. リストの途中の要素を追加・削除する可能性がある
# → for_eachを使用
```

## for_each：柔軟な繰り返し

### 基本的な考え方

**例えて言うなら：** 名簿を見ながら一人ずつ名札を作成

```
名簿（マップやセット）→ 名札作成機（for_each）→ 各人の名札

田中: {部署: "営業"}    →  for_each  →  田中（営業）の名札
鈴木: {部署: "開発"}                    鈴木（開発）の名札
佐藤: {部署: "総務"}                    佐藤（総務）の名札
```

### for_eachの基本構文

```hcl
resource "リソースタイプ" "名前" {
  for_each = マップまたはセット

  # each.keyとeach.valueを使って各リソースを設定
  # each.key   = マップのキーまたはセットの値
  # each.value = マップの値
}
```

**for_eachで使えるデータ型：**

```hcl
# 1. マップ（map）
for_each = {
  "web"  = "t4g.small"
  "app"  = "t4g.medium"
  "db"   = "t4g.large"
}

# 2. セット（set）
for_each = toset(["web", "app", "db"])

# 3. リストをセットに変換
for_each = toset(var.availability_zones)
```

### 実例1：マップを使った基本例

```hcl
# 環境ごとのインスタンスタイプを定義
variable "instances" {
  type = map(string)
  default = {
    "web"  = "t4g.small"
    "app"  = "t4g.medium"
    "db"   = "t4g.large"
  }
}

# for_eachでインスタンスを作成
resource "aws_instance" "server" {
  for_each = var.instances

  ami           = "ami-xxxxx"
  instance_type = each.value  # マップの値（インスタンスタイプ）

  tags = {
    Name = "${each.key}-server"  # マップのキー（web, app, db）
  }
}

# 作成されるインスタンス：
# - web-server (t4g.small)
# - app-server (t4g.medium)
# - db-server (t4g.large)
```

**each.keyとeach.valueの理解：**

```hcl
for_each = {
  "web"  = "t4g.small"
  "app"  = "t4g.medium"
}

# 1回目の繰り返し：
# each.key   = "web"
# each.value = "t4g.small"

# 2回目の繰り返し：
# each.key   = "app"
# each.value = "t4g.medium"
```

### 実例2：セットを使った例

```hcl
# アベイラビリティゾーンのリスト
variable "availability_zones" {
  type = list(string)
  default = [
    "ap-northeast-1a",
    "ap-northeast-1c",
    "ap-northeast-1d"
  ]
}

# リストをセットに変換してfor_each
resource "aws_subnet" "public" {
  for_each = toset(var.availability_zones)
  #          ↑
  #          リストをセットに変換（必須）

  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${index(var.availability_zones, each.key) + 1}.0/24"
  availability_zone = each.value  # セットの値（AZ名）

  tags = {
    Name = "public-subnet-${each.value}"
  }
}

# 作成されるサブネット：
# - public-subnet-ap-northeast-1a
# - public-subnet-ap-northeast-1c
# - public-subnet-ap-northeast-1d
```

### 実例3：複雑なマップ（オブジェクト）を使う

```hcl
# 複雑な設定を持つマップ
variable "instances" {
  type = map(object({
    instance_type = string
    disk_size     = number
    monitoring    = bool
  }))

  default = {
    web = {
      instance_type = "t4g.small"
      disk_size     = 20
      monitoring    = false
    }
    app = {
      instance_type = "t4g.medium"
      disk_size     = 50
      monitoring    = true
    }
    db = {
      instance_type = "t4g.large"
      disk_size     = 100
      monitoring    = true
    }
  }
}

# for_eachで詳細な設定を使用
resource "aws_instance" "server" {
  for_each = var.instances

  ami           = "ami-xxxxx"
  instance_type = each.value.instance_type  # オブジェクトの属性にアクセス
  monitoring    = each.value.monitoring

  root_block_device {
    volume_size = each.value.disk_size
  }

  tags = {
    Name       = "${each.key}-server"
    Monitoring = each.value.monitoring ? "enabled" : "disabled"
  }
}
```

### 実例4：環境別の設定

```hcl
# 環境ごとの設定
variable "environments" {
  type = map(object({
    instance_count = number
    instance_type  = string
  }))

  default = {
    dev = {
      instance_count = 1
      instance_type  = "t4g.micro"
    }
    staging = {
      instance_count = 2
      instance_type  = "t4g.small"
    }
    production = {
      instance_count = 4
      instance_type  = "t4g.medium"
    }
  }
}

# 環境ごとにVPCを作成
resource "aws_vpc" "env" {
  for_each = var.environments

  cidr_block = "10.${index(keys(var.environments), each.key) + 1}.0.0/16"

  tags = {
    Name        = "${each.key}-vpc"
    Environment = each.key
  }
}

# 環境ごとにサブネットを作成
resource "aws_subnet" "env" {
  for_each = var.environments

  vpc_id     = aws_vpc.env[each.key].id  # 対応するVPCを参照
  cidr_block = "10.${index(keys(var.environments), each.key) + 1}.1.0/24"

  tags = {
    Name        = "${each.key}-subnet"
    Environment = each.key
  }
}
```

### for_eachで作成したリソースの参照方法

**特定のリソースを参照：**

```hcl
# for_eachでインスタンスを作成
resource "aws_instance" "server" {
  for_each = {
    "web" = "t4g.small"
    "app" = "t4g.medium"
  }

  ami           = "ami-xxxxx"
  instance_type = each.value
}

# キーを使って特定のインスタンスを参照
output "web_instance_id" {
  value = aws_instance.server["web"].id
  #                              ↑
  #                         キーで指定
}

output "app_instance_id" {
  value = aws_instance.server["app"].id
}
```

**すべてのリソースを参照：**

```hcl
# すべてのインスタンスIDをマップで取得
output "all_instance_ids" {
  value = {
    for key, instance in aws_instance.server :
    key => instance.id
  }
}

# 結果：
# {
#   "web" = "i-xxxxx"
#   "app" = "i-yyyyy"
# }
```

**値のリストを作成：**

```hcl
# すべてのインスタンスIDをリストで取得
output "instance_ids_list" {
  value = [
    for instance in aws_instance.server :
    instance.id
  ]
}

# または
output "instance_ids_list" {
  value = values(aws_instance.server)[*].id
}
```

### for_eachの利点

**countと比較した利点：**

```hcl
# シナリオ：サブネットのリストから1つを削除

# countの場合
variable "subnet_cidrs" {
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

resource "aws_subnet" "public" {
  count = length(var.subnet_cidrs)
  cidr_block = var.subnet_cidrs[count.index]
}

# "10.0.2.0/24"を削除すると...
variable "subnet_cidrs" {
  default = ["10.0.1.0/24", "10.0.3.0/24"]
}

# 結果：
# - subnet[0]: 変更なし
# - subnet[1]: 10.0.2.0/24 → 10.0.3.0/24 に変更（不要な置き換え）
# - subnet[2]: 削除
```

```hcl
# for_eachの場合
variable "subnets" {
  default = {
    "subnet-1" = "10.0.1.0/24"
    "subnet-2" = "10.0.2.0/24"
    "subnet-3" = "10.0.3.0/24"
  }
}

resource "aws_subnet" "public" {
  for_each = var.subnets
  cidr_block = each.value
}

# "subnet-2"を削除すると...
variable "subnets" {
  default = {
    "subnet-1" = "10.0.1.0/24"
    "subnet-3" = "10.0.3.0/24"
  }
}

# 結果：
# - subnet-1: 変更なし（キーで識別されるため）
# - subnet-2: 削除（これだけ）
# - subnet-3: 変更なし（キーで識別されるため）
```

### for_eachのベストプラクティス

**✓ for_eachを使うべき場合：**

```hcl
# 1. 各リソースが異なる設定を持つ
resource "aws_instance" "server" {
  for_each = var.server_configs

  instance_type = each.value.instance_type  # 各サーバーで異なる
  disk_size     = each.value.disk_size      # 各サーバーで異なる
}

# 2. リソースを名前で識別したい
resource "aws_security_group" "app" {
  for_each = toset(["web", "app", "db"])

  name = "${each.key}-sg"  # 意味のある名前
}

# 3. リストの途中の要素を追加・削除する可能性がある
# → for_eachなら他のリソースに影響しない
```

**✗ for_eachを避けるべき場合：**

```hcl
# 1. 完全に同じ設定のリソースを数だけ増やしたい
# → countを使用

# 2. 単純な数値ベースの繰り返し
# → countの方が簡単
```

## countとfor_eachの使い分け

### 比較表

| 特徴           | count                          | for_each                               |
| -------------- | ------------------------------ | -------------------------------------- |
| 識別方法       | インデックス番号（0, 1, 2...） | キー（文字列）                         |
| 適した用途     | 同じ設定のリソースを複数       | 異なる設定のリソースを複数             |
| データ型       | 数値                           | マップまたはセット                     |
| リソース参照   | `resource[0]`, `resource[1]`   | `resource["key1"]`, `resource["key2"]` |
| 途中の削除     | 他のリソースに影響する         | 影響しない                             |
| コードの可読性 | シンプル                       | キー名で意味が明確                     |

### 判断フローチャート

```
リソースを複数作成したい
    ↓
【質問1】すべて同じ設定か？
    ↓
  YES → 【質問2】数だけわかれば良いか？
         ↓
       YES → count を使う
         ↓
        NO → 【質問3】リストの途中を削除する可能性は？
              ↓
            YES → for_each を使う
              ↓
             NO → count でもOK

    ↓
   NO → 【質問4】各リソースで設定が異なるか？
         ↓
       YES → for_each を使う
         ↓
        NO → count を使う
```

### 実例：適切な使い分け

**ケース1：ワーカーノード（count推奨）**

```hcl
# すべて同じ設定、数だけ増やしたい
resource "aws_instance" "worker" {
  count = 5

  ami           = "ami-xxxxx"
  instance_type = "t4g.small"
  # すべて同じ設定

  tags = {
    Name = "worker-${count.index + 1}"
  }
}
```

**ケース2：環境別インフラ（for_each推奨）**

```hcl
# 各環境で設定が異なる
resource "aws_vpc" "env" {
  for_each = {
    "dev"  = "10.0.0.0/16"
    "stg"  = "10.1.0.0/16"
    "prod" = "10.2.0.0/16"
  }

  cidr_block = each.value

  tags = {
    Name        = "${each.key}-vpc"
    Environment = each.key
  }
}
```

**ケース3：アベイラビリティゾーン（for_each推奨）**

```hcl
# AZ名で識別したい、途中のAZを削除する可能性がある
resource "aws_subnet" "public" {
  for_each = toset([
    "ap-northeast-1a",
    "ap-northeast-1c",
    "ap-northeast-1d"
  ])

  availability_zone = each.value
  # ...
}
```

## 本番環境・エンタープライズでの推奨事項

### 実際の現場では何が使われているか

#### 現場の実態調査

**大規模プロジェクトでの使用状況：**

```
for_each: 約70-80%  ← 最も多い
count:    約15-20%
dynamic:  約5-10%
```

**理由：**

- for_eachの方が安全で予測可能
- インフラの変更が追跡しやすい
- チーム開発に適している

### なぜfor_eachが推奨されるのか

#### 理由1：本番環境での安全性

**countの問題：本番環境で危険な動作**

```hcl
# 初期状態：3つのデータベースインスタンス
variable "database_instances" {
  default = [
    "mysql-primary",
    "mysql-replica-1",
    "mysql-replica-2"
  ]
}

resource "aws_db_instance" "database" {
  count = length(var.database_instances)

  identifier = var.database_instances[count.index]
  # ... 設定
}

# 本番環境で動作中：
# database[0] = mysql-primary    (本番稼働中)
# database[1] = mysql-replica-1  (本番稼働中)
# database[2] = mysql-replica-2  (本番稼働中)
```

**危険なシナリオ：**

```hcl
# replica-1が不要になったので削除したい
variable "database_instances" {
  default = [
    "mysql-primary",
    "mysql-replica-2"  # replica-1を削除
  ]
}

# terraform planを実行すると...
```

**terraform planの結果（危険！）：**

```
Terraform will perform the following actions:

  # aws_db_instance.database[1] will be updated in-place
  ~ resource "aws_db_instance" "database" {
      ~ identifier = "mysql-replica-1" -> "mysql-replica-2"  # 置き換え！
    }

  # aws_db_instance.database[2] will be destroyed
  - resource "aws_db_instance" "database" {
      - identifier = "mysql-replica-2"  # 削除！
    }

Plan: 0 to add, 1 to change, 1 to destroy.
```

**何が起きるか：**

1. `replica-1`が削除される（意図通り）
2. `replica-2`も削除される（意図しない！）
3. 新しい`replica-2`が作成される（意図しない！）
4. **データが失われる可能性がある**

**本番環境での影響：**

- データベースの予期しない削除
- サービスダウンタイム
- データ損失のリスク
- 復旧に時間がかかる

#### for_eachなら安全

```hcl
# for_eachを使用
variable "database_instances" {
  default = {
    "primary"   = { instance_class = "db.t3.large" }
    "replica-1" = { instance_class = "db.t3.medium" }
    "replica-2" = { instance_class = "db.t3.medium" }
  }
}

resource "aws_db_instance" "database" {
  for_each = var.database_instances

  identifier     = each.key
  instance_class = each.value.instance_class
  # ...
}

# replica-1を削除
variable "database_instances" {
  default = {
    "primary"   = { instance_class = "db.t3.large" }
    "replica-2" = { instance_class = "db.t3.medium" }
  }
}
```

**terraform planの結果（安全）：**

```
Terraform will perform the following actions:

  # aws_db_instance.database["replica-1"] will be destroyed
  - resource "aws_db_instance" "database" {
      - identifier = "replica-1"  # これだけ削除
    }

Plan: 0 to add, 0 to change, 1 to destroy.
```

**何が起きるか：**

1. `replica-1`だけが削除される（意図通り）
2. `primary`と`replica-2`は影響を受けない
3. **安全で予測可能**

#### 理由2：コードレビューのしやすさ

**countの場合：**

```hcl
# Pull Request diff
variable "subnet_cidrs" {
  default = [
    "10.0.1.0/24",
-   "10.0.2.0/24",  # 削除
    "10.0.3.0/24",
+   "10.0.4.0/24"   # 追加
  ]
}
```

**レビュアーの疑問：**

```
❓ これは何が起こるのか？
- subnet[1]が削除される？
- それとも更新される？
- subnet[2]はどうなる？
- subnet[3]は新規作成？

→ terraform planを見ないと分からない
→ レビューに時間がかかる
→ ミスのリスクが高い
```

**for_eachの場合：**

```hcl
# Pull Request diff
variable "subnets" {
  default = {
    "public-1a" = "10.0.1.0/24",
-   "public-1c" = "10.0.2.0/24",  # 削除されるのは明確
    "public-1d" = "10.0.3.0/24",
+   "public-2a" = "10.0.4.0/24"   # 追加されるのは明確
  }
}
```

**レビュアーにとって明確：**

```
✓ public-1cが削除される
✓ public-2aが追加される
✓ 他のサブネットは影響を受けない

→ diffだけで理解できる
→ レビューが早い
→ ミスが減る
```

#### 理由3：インフラの変更追跡

**countの問題：Gitの履歴が追いにくい**

```bash
# Gitログを見ても...
git log --follow -- main.tf

# countベースだと、どのリソースの変更か分かりにくい
commit abc123
  Changed subnet[1] configuration

commit def456
  Removed subnet[2]

# 質問：subnet[1]とsubnet[2]は同じリソース？別のリソース？
# → 追跡が困難
```

**for_eachの利点：名前で追跡できる**

```bash
# for_eachなら明確
commit abc123
  Changed subnet["production-public-1a"] configuration

commit def456
  Removed subnet["staging-public-1c"]

# 質問：どのサブネットの変更？
# → 一目瞭然
```

### エンタープライズ環境での推奨パターン

#### パターン1：本番環境（最も厳格）

```hcl
# ✓ 推奨：for_eachを使用
# 理由：
# - 変更の影響範囲が明確
# - コードレビューしやすい
# - インシデント時の原因特定が容易

variable "production_databases" {
  type = map(object({
    instance_class    = string
    allocated_storage = number
    multi_az          = bool
  }))

  default = {
    "prod-primary" = {
      instance_class    = "db.r5.xlarge"
      allocated_storage = 1000
      multi_az          = true
    }
    "prod-replica-1" = {
      instance_class    = "db.r5.large"
      allocated_storage = 1000
      multi_az          = true
    }
  }
}

resource "aws_db_instance" "production" {
  for_each = var.production_databases

  identifier        = each.key
  instance_class    = each.value.instance_class
  allocated_storage = each.value.allocated_storage
  multi_az          = each.value.multi_az

  # 本番環境では削除保護を有効化
  deletion_protection = true

  # バックアップ設定
  backup_retention_period = 30

  tags = {
    Name        = each.key
    Environment = "production"
    ManagedBy   = "Terraform"
  }
}
```

#### パターン2：開発環境（柔軟性重視）

```hcl
# 開発環境ではcountも許容される場合がある
# 理由：
# - 頻繁に作り直す
# - データ損失のリスクが低い
# - シンプルさ優先

variable "dev_instance_count" {
  description = "開発用インスタンスの数"
  type        = number
  default     = 3
}

resource "aws_instance" "dev" {
  count = var.dev_instance_count

  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t4g.micro"  # 開発環境は小さいインスタンス

  tags = {
    Name        = "dev-instance-${count.index + 1}"
    Environment = "development"
  }

  # 開発環境は削除保護なし
}
```

#### パターン3：複数環境の管理

```hcl
# ✓ ベストプラクティス：for_eachで環境を管理

variable "environments" {
  type = map(object({
    instance_count = number
    instance_type  = string
    multi_az       = bool
  }))

  default = {
    development = {
      instance_count = 1
      instance_type  = "t4g.micro"
      multi_az       = false
    }
    staging = {
      instance_count = 2
      instance_type  = "t4g.small"
      multi_az       = false
    }
    production = {
      instance_count = 3
      instance_type  = "t4g.large"
      multi_az       = true
    }
  }
}

# 環境ごとにVPCを作成
resource "aws_vpc" "env" {
  for_each = var.environments

  cidr_block           = "10.${index(keys(var.environments), each.key)}.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name        = "${each.key}-vpc"
    Environment = each.key
  }
}

# 各環境内でインスタンスはcountを使用（許容される）
resource "aws_instance" "app" {
  for_each = var.environments

  count = each.value.instance_count  # 環境内の数はcount

  ami           = data.aws_ami.amazon_linux.id
  instance_type = each.value.instance_type
  subnet_id     = aws_subnet.env[each.key].id

  tags = {
    Name        = "${each.key}-app-${count.index + 1}"
    Environment = each.key
  }
}
```

### 実際のエンタープライズ事例

#### 事例1：大手金融機関

**要件：**

- 複数の本番環境（地域ごと）
- 厳格な変更管理
- 監査対応

**採用パターン：**

```hcl
# すべてfor_eachを使用
variable "regions" {
  type = map(object({
    vpc_cidr           = string
    availability_zones = list(string)
    db_instance_class  = string
  }))

  default = {
    "tokyo" = {
      vpc_cidr           = "10.1.0.0/16"
      availability_zones = ["ap-northeast-1a", "ap-northeast-1c"]
      db_instance_class  = "db.r5.2xlarge"
    }
    "osaka" = {
      vpc_cidr           = "10.2.0.0/16"
      availability_zones = ["ap-northeast-3a", "ap-northeast-3c"]
      db_instance_class  = "db.r5.2xlarge"
    }
  }
}

# リージョンごとのインフラ
resource "aws_vpc" "regional" {
  for_each = var.regions

  cidr_block = each.value.vpc_cidr

  tags = {
    Name   = "prod-${each.key}-vpc"
    Region = each.key
  }
}
```

**選択理由：**

- リージョンの追加・削除が安全
- 変更履歴が明確
- 監査時に追跡しやすい

#### 事例2：スタートアップ企業

**要件：**

- 迅速な開発
- コスト削減
- シンプルな構成

**採用パターン：**

```hcl
# 主要リソースはfor_each
# 一時的なリソースはcount

# 主要データベース：for_each
variable "databases" {
  default = {
    "users"    = { size = "db.t3.small" }
    "products" = { size = "db.t3.small" }
    "orders"   = { size = "db.t3.medium" }
  }
}

resource "aws_db_instance" "main" {
  for_each = var.databases

  identifier     = each.key
  instance_class = each.value.size
}

# テスト用インスタンス：count
resource "aws_instance" "test" {
  count = var.test_mode ? 3 : 0

  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t4g.micro"

  tags = {
    Name = "test-${count.index + 1}"
  }
}
```

**選択理由：**

- 重要なリソースは安全に管理（for_each）
- 一時的なリソースはシンプルに（count）
- チームの学習コストを考慮

### 推奨決定フローチャート（本番環境用）

```
新しいリソースを作成する
    ↓
【質問1】本番環境か？
    ↓
  YES → 【質問2】重要なデータを持つか？
         ↓
       YES → for_each を使用（必須）
         ↓
        NO → 【質問3】頻繁に変更するか？
              ↓
            YES → for_each を使用（推奨）
              ↓
             NO → count も可能（チーム判断）

    ↓
   NO → 【質問4】開発環境か？
         ↓
       YES → 【質問5】チーム全員がfor_eachに慣れているか？
              ↓
            YES → for_each を使用（推奨）
              ↓
             NO → count も可能（学習コスト考慮）
```

### チームでの導入ガイドライン

#### ステップ1：ポリシーの策定

```markdown
# team-guidelines.md

## Terraformコーディング規約

### 必須ルール（本番環境）

1. データベース、ストレージなどの状態を持つリソース
   → **必ず for_each を使用**

2. ネットワークインフラ（VPC、サブネットなど）
   → **for_each を優先使用**

3. セキュリティグループ
   → **for_each を優先使用**

### 推奨ルール

1. 環境の管理（dev, staging, prod）
   → **for_each を使用**

2. リージョンやAZの管理
   → **for_each を使用**

### 許容ルール（開発環境のみ）

1. テスト用の一時的なリソース
   → count 使用可能

2. 完全に同一の設定で、頻繁に作り直すリソース
   → count 使用可能

### 禁止パターン

1. 本番データベースでのcount使用
   → ❌ 絶対禁止

2. 本番ネットワークでのcount使用
   → ❌ 原則禁止
```

#### ステップ2：移行計画

```hcl
# 既存のcountコードをfor_eachに移行

# 移行前（count）
resource "aws_subnet" "public" {
  count = 3

  vpc_id     = aws_vpc.main.id
  cidr_block = var.subnet_cidrs[count.index]
}

# 移行後（for_each）
resource "aws_subnet" "public" {
  for_each = {
    "public-1a" = "10.0.1.0/24"
    "public-1c" = "10.0.2.0/24"
    "public-1d" = "10.0.3.0/24"
  }

  vpc_id     = aws_vpc.main.id
  cidr_block = each.value

  tags = {
    Name = each.key
  }
}
```

**注意：移行にはterraform state mvが必要**

```bash
# 状態の移行（慎重に実施）
terraform state mv 'aws_subnet.public[0]' 'aws_subnet.public["public-1a"]'
terraform state mv 'aws_subnet.public[1]' 'aws_subnet.public["public-1c"]'
terraform state mv 'aws_subnet.public[2]' 'aws_subnet.public["public-1d"]'
```

### まとめ：本番環境での推奨事項

**✓ 強く推奨：**

```
1. 本番環境では原則 for_each を使用
2. データを持つリソースは必ず for_each
3. 長期運用するインフラは for_each
```

**✓ 許容範囲：**

```
1. 開発環境での count 使用
2. 一時的なリソースでの count 使用
3. チームの学習段階での count 使用
```

**✗ 避けるべき：**

```
1. 本番データベースでの count
2. 本番ネットワークでの count
3. 状態を持つリソースでの count
```

**企業規模別推奨：**

| 企業規模       | count使用率 | for_each使用率 | 理由         |
| -------------- | ----------- | -------------- | ------------ |
| 大企業         | 10%         | 90%            | 安全性最優先 |
| 中企業         | 20%         | 80%            | バランス重視 |
| スタートアップ | 30%         | 70%            | スピード重視 |

**最終的な判断基準：**

```
「このリソースが誤って削除されたら、どうなるか？」

→ 重大な影響がある場合：for_each を使用
→ すぐに復旧できる場合：count も検討可能
```

## dynamic：ネストしたブロックの動的生成

### 基本的な考え方

**例えて言うなら：** 申込書の「家族情報」欄

```
家族の人数に応じて
必要な欄だけを動的に作成

1人 → 家族情報欄 × 1
3人 → 家族情報欄 × 3
0人 → 家族情報欄なし
```

### dynamicが必要な理由

**問題：ネストしたブロックは繰り返せない**

```hcl
# これはエラー！ネストしたブロックにcountは使えない
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  # これは動作しない
  ingress count = 3 {  # エラー！
    from_port = 80
    to_port   = 80
  }
}
```

**解決策：dynamicブロックを使う**

```hcl
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  # dynamicブロックなら動的に生成できる
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port = ingress.value.from_port
      to_port   = ingress.value.to_port
      protocol  = ingress.value.protocol
    }
  }
}
```

### dynamicの基本構文

```hcl
dynamic "ブロック名" {
  for_each = 繰り返すデータ（リストまたはマップ）

  content {
    # ブロックの内容
    # iterator変数を使って各要素にアクセス
  }
}
```

**構成要素の説明：**

```hcl
dynamic "ingress" {  # ← ブロック名（ingressブロックを生成）
  for_each = var.rules  # ← 繰り返すデータ

  content {  # ← 生成するブロックの内容
    from_port = ingress.value.from_port
    #           ↑
    #           デフォルトのイテレータ名はブロック名と同じ
  }
}
```

### 実例1：セキュリティグループルールの動的生成

**従来の方法（繰り返しが多い）：**

```hcl
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  # ルール1
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP"
  }

  # ルール2
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS"
  }

  # ルール3
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
    description = "SSH from VPC"
  }
}
```

**dynamicを使った方法（シンプル）：**

```hcl
# 変数でルールを定義
variable "ingress_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))

  default = [
    {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTP"
    },
    {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTPS"
    },
    {
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/16"]
      description = "SSH from VPC"
    }
  ]
}

# dynamicで動的に生成
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.ingress_rules

    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }
}
```

**動作の説明：**

```hcl
# 1回目の繰り返し：
# ingress.value = {
#   from_port   = 80
#   to_port     = 80
#   protocol    = "tcp"
#   cidr_blocks = ["0.0.0.0/0"]
#   description = "HTTP"
# }

# 2回目の繰り返し：
# ingress.value = {
#   from_port   = 443
#   ...
# }
```

### 実例2：カスタムイテレータ名の使用

```hcl
# デフォルト：ブロック名と同じ名前
dynamic "ingress" {
  for_each = var.ingress_rules

  content {
    from_port = ingress.value.from_port  # ingress.value
  }
}

# カスタム：別の名前を指定
dynamic "ingress" {
  for_each = var.ingress_rules
  iterator = rule  # ← カスタム名を指定

  content {
    from_port   = rule.value.from_port     # rule.value
    to_port     = rule.value.to_port
    protocol    = rule.value.protocol
    cidr_blocks = rule.value.cidr_blocks
    description = rule.value.description
  }
}
```

### 実例3：条件付きで動的ブロックを生成

```hcl
variable "enable_ssh" {
  description = "SSHアクセスを有効にするか"
  type        = bool
  default     = false
}

variable "enable_http" {
  description = "HTTPアクセスを有効にするか"
  type        = bool
  default     = true
}

locals {
  # 条件に基づいてルールを作成
  ingress_rules = concat(
    # HTTPルール（enable_httpがtrueの場合のみ）
    var.enable_http ? [{
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTP"
    }] : [],

    # SSHルール（enable_sshがtrueの場合のみ）
    var.enable_ssh ? [{
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/16"]
      description = "SSH"
    }] : []
  )
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = local.ingress_rules

    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }
}
```

### 実例4：ネストしたdynamicブロック

```hcl
# 複雑なデータ構造
variable "load_balancer_config" {
  type = map(object({
    origins = list(object({
      hostname = string
      port     = number
    }))
  }))

  default = {
    "web-group" = {
      origins = [
        { hostname = "web1.example.com", port = 80 },
        { hostname = "web2.example.com", port = 80 }
      ]
    }
    "api-group" = {
      origins = [
        { hostname = "api1.example.com", port = 443 },
        { hostname = "api2.example.com", port = 443 }
      ]
    }
  }
}

# ネストしたdynamicブロック
resource "some_load_balancer" "example" {
  name = "my-lb"

  # 外側のdynamic：origin_groupブロックを生成
  dynamic "origin_group" {
    for_each = var.load_balancer_config

    content {
      name = origin_group.key  # "web-group", "api-group"

      # 内側のdynamic：originブロックを生成
      dynamic "origin" {
        for_each = origin_group.value.origins

        content {
          hostname = origin.value.hostname
          port     = origin.value.port
        }
      }
    }
  }
}

# 生成される構造：
# origin_group "web-group" {
#   origin { hostname = "web1.example.com", port = 80 }
#   origin { hostname = "web2.example.com", port = 80 }
# }
# origin_group "api-group" {
#   origin { hostname = "api1.example.com", port = 443 }
#   origin { hostname = "api2.example.com", port = 443 }
# }
```

### dynamicのベストプラクティス

**✓ dynamicを使うべき場合：**

```hcl
# 1. ネストしたブロックを動的に生成したい
resource "aws_security_group" "app" {
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      # ...
    }
  }
}

# 2. ユーザー入力に基づいてブロック数が変わる
# モジュールの入力として受け取る場合など

# 3. 複雑な設定を変数で管理したい
# 設定ファイルとコードを分離
```

**✗ dynamicを避けるべき場合：**

```hcl
# 1. ブロックの数が固定で少ない
# → 直接書いた方が読みやすい

resource "aws_security_group" "simple" {
  ingress {
    from_port = 80
    to_port   = 80
  }

  ingress {
    from_port = 443
    to_port   = 443
  }

  # dynamicを使うほど複雑ではない
}

# 2. すべての引数を変数から取得している
# → モジュール設計を見直した方が良い可能性
```

**公式ドキュメントからの重要な注意：**

> dynamicブロックの過度な使用は、設定を読みにくく保守しにくくします。
> 詳細を隠してクリーンなユーザーインターフェースを構築する必要がある場合にのみ使用してください。
> 可能な限り、ネストしたブロックを直接記述してください。

### 実用的な例：環境別セキュリティグループ

```hcl
# 環境別のルール定義
variable "environment" {
  type    = string
  default = "development"
}

locals {
  # 環境に応じたセキュリティグループルール
  security_rules = {
    development = [
      {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]  # 開発環境は全開放
        description = "SSH for development"
      },
      {
        from_port   = 80
        to_port     = 80
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
        description = "HTTP"
      }
    ]

    production = [
      {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["10.0.0.0/16"]  # 本番環境はVPC内のみ
        description = "SSH from VPC only"
      },
      {
        from_port   = 443
        to_port     = 443
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
        description = "HTTPS"
      }
    ]
  }
}

resource "aws_security_group" "app" {
  name   = "${var.environment}-app-sg"
  vpc_id = aws_vpc.main.id

  # 環境に応じたルールを動的に生成
  dynamic "ingress" {
    for_each = local.security_rules[var.environment]

    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }
}
```

## 最終まとめ

### 3つの機能の使い分け

```
count:
  ✓ 同じ設定のリソースを複数作成
  ✓ 数値ベースのシンプルな繰り返し
  ✓ 条件によって0個または1個作成
  ✗ 各リソースが大きく異なる設定
  ✗ リストの途中を削除する可能性
  ✗ 本番環境での重要なリソース

for_each:
  ✓ 各リソースが異なる設定を持つ
  ✓ 名前で識別したい
  ✓ リストの途中を追加・削除
  ✓ マップやセットのデータ
  ✓ 本番環境での推奨方法
  ✗ 単純な数値の繰り返し

dynamic:
  ✓ ネストしたブロックの動的生成
  ✓ ユーザー入力に基づく柔軟な設定
  ✓ モジュールの再利用性向上
  ✗ 固定で少数のブロック
  ✗ 読みやすさ重視の場合
```

### 本番環境での最終推奨

**プロダクション環境での原則：**

1. **for_eachを第一選択肢とする**

- 安全性が最優先
- 変更の影響範囲が明確
- チーム開発に適している

2. **countは慎重に使用**

- 一時的なリソースのみ
- データを持たないリソース
- 開発環境での使用

3. **dynamicは必要最小限に**

- ネストブロックの動的生成のみ
- 過度な使用は避ける
- ドキュメント化を徹底

### ベストプラクティス総まとめ

1. **シンプルさを優先**

- 必要な時だけ使用
- 過度な抽象化は避ける
- コードの可読性を重視

2. **適切な機能選択**

- count：数だけ増やす
- for_each：設定が異なる
- dynamic：ネストブロック

3. **保守性を考慮**

- コメントを追加
- 変数名をわかりやすく
- ドキュメント化

4. **段階的な導入**

- まず直接記述
- 繰り返しが見えたらリファクタ
- 複雑になりすぎたら見直し

5. **チームでの合意**

- コーディング規約の策定
- レビュープロセスの確立
- 知識の共有
