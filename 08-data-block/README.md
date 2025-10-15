# Terraformのdataブロック：既存リソースの情報取得

## この章で学ぶこと

Terraformの`data`ブロックについて学習します。`data`ブロックを使用すると、既存のリソースやプロバイダーから情報を取得して、それを設定に活用できます。

**学習内容：**

- dataブロックとresourceブロックの違い
- dataブロックの基本的な使い方
- プロバイダードキュメントの読み方
- 実践的な活用例
- ベストプラクティス
- よくある間違いと対策

## dataブロックとは何か

### 日常生活での例え

**例えて言うなら：** 図書館での本の検索

```
resourceブロック = 新しい本を書いて図書館に寄贈する
dataブロック     = 図書館で既にある本を検索して情報を得る
```

### 基本的な違い

| 項目        | resourceブロック         | dataブロック             |
| ----------- | ------------------------ | ------------------------ |
| 目的        | 新しいリソースを**作成** | 既存のリソースを**検索** |
| 操作        | 作成・更新・削除         | 読み取りのみ             |
| AWSへの影響 | リソースが作成される     | 何も作成されない         |
| 使用例      | VPC、EC2、S3を作る       | 既存のAMI IDを調べる     |

**重要なポイント：** `data`ブロックはAWS上に何も作成しません。既にあるものを調べるだけです。

## dataブロックの基本構文

### 基本形式

```hcl
data "プロバイダータイプ" "ローカル名" {
  # 検索条件
}
```

**構成要素：**

- `data`：dataブロックであることを示すキーワード
- `"プロバイダータイプ"`：何を検索するか（例：`aws_ami`、`aws_vpc`）
- `"ローカル名"`：この検索結果に付ける名前
- ブロック内：検索条件や絞り込み条件

### 簡単な例

公式ドキュメント：

- [Data Source: aws_ami](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ami)
- [Data Source: aws_vpc](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/vpc)

```hcl
# 例1：最新のAmazon Linux 2023 AMIを検索
data "aws_ami" "latest_amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*"]
  }
}

# 例2：既存のVPCを検索
data "aws_vpc" "selected" {
  filter {
    name   = "tag:Name"
    values = ["main-vpc"]
  }
}
```

## dataブロックとresourceブロックの比較

### 実例で理解する

```hcl
# ========================================
# resourceブロック：新しいVPCを作成
# ========================================
resource "aws_vpc" "new_vpc" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "new-vpc"
  }
}
# → AWSに新しいVPCが作成される

# ========================================
# dataブロック：既存のVPCを検索
# ========================================
data "aws_vpc" "existing_vpc" {
  filter {
    name   = "tag:Name"
    values = ["existing-vpc"]
  }
}
# → AWSには何も作成されない
# → 既存のVPCの情報を取得するだけ
```

### 参照方法の違い

```hcl
# resourceブロックの参照
aws_vpc.new_vpc.id          # 作成したVPCのID
aws_vpc.new_vpc.cidr_block  # 作成したVPCのCIDR

# dataブロックの参照
data.aws_vpc.existing_vpc.id          # 検索したVPCのID
data.aws_vpc.existing_vpc.cidr_block  # 検索したVPCのCIDR
```

**重要な違い：** `data`ブロックの参照には必ず `data.` プレフィックスが付きます。

## プロバイダー公式ドキュメントの読み方

### ドキュメントへのアクセス方法

**手順：**

1. Terraform Registry（https://registry.terraform.io/）にアクセス
2. 上部の検索バーで「aws」を検索
3. 「hashicorp/aws」プロバイダーを選択
4. 左側メニューから対象のリソースをクリックし、「Data Sources」を選択
5. 使用したいデータソース（例：`aws_availability_zones`）を選択

### ドキュメントの構成

すべてのdata sourceドキュメントは、以下の構成になっています：

```
1. 概要説明（データソースの目的）
2. 注意事項（Note/Warning）
3. Example Usage（使用例）
4. Argument Reference（引数一覧）
5. Attribute Reference（参照可能な属性）
```

### 実例：aws_availability_zones の読み方

公式ドキュメント：[Data Source: aws_availability_zones
ja](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/availability_zones)

#### 1. 概要説明を読む

ドキュメントの最初の部分：

```
Data Source: aws_availability_zones

The Availability Zones data source allows access to the list of AWS
Availability Zones which can be accessed by an AWS account within
the region configured in the provider.
```

**日本語で理解：**

- このデータソースは、AWSアカウントがアクセスできるアベイラビリティゾーンの一覧を取得する
- プロバイダーで設定されたリージョン内のAZが対象

#### 2. 注意事項を確認

```
Note: When Local Zones are enabled in a region, by default the API
and this data source include both Local Zones and Availability Zones.
To return only Availability Zones, see the example section below.
```

**日本語で理解：**

- デフォルトでは、ローカルゾーンとアベイラビリティゾーンの両方が含まれる
- AZのみを取得したい場合は、フィルターを使用する必要がある

#### 3. Example Usage を確認

ドキュメントには複数の使用例が記載されています：

**例1：状態で絞り込み**

```hcl
# Declare the data source
data "aws_availability_zones" "available" {
  state = "available"
}

# e.g., Create subnets in the first two available availability zones

resource "aws_subnet" "primary" {
  availability_zone = data.aws_availability_zones.available.names[0]
  # ...
}

resource "aws_subnet" "secondary" {
  availability_zone = data.aws_availability_zones.available.names[1]
  # ...
}
```

**例2：フィルターを使用（AZのみを取得）**

```hcl
# Only Availability Zones (no Local Zones):
data "aws_availability_zones" "example" {
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}
```

#### 4. Argument Reference を読む

設定可能な引数の一覧：

| 引数                     | 必須/任意 | 説明                                             |
| ------------------------ | --------- | ------------------------------------------------ |
| `region`                 | 任意      | リージョンを指定（デフォルトはプロバイダー設定） |
| `all_availability_zones` | 任意      | すべてのAZとローカルゾーンを含める               |
| `filter`                 | 任意      | フィルター条件を指定                             |
| `exclude_names`          | 任意      | 除外するAZ名のリスト                             |
| `exclude_zone_ids`       | 任意      | 除外するAZ IDのリスト                            |
| `state`                  | 任意      | AZの状態で絞り込み（available等）                |

**重要：**

- 「Required」と書かれている引数は必須
- 「Optional」と書かれている引数は任意
- 任意の引数を省略した場合、デフォルト値が使用される

#### 5. Attribute Reference を読む

取得できる属性の一覧：

| 属性          | 説明                 |
| ------------- | -------------------- |
| `group_names` | AZグループ名のセット |
| `id`          | AZのリージョン       |
| `names`       | AZ名のリスト         |
| `zone_ids`    | AZ IDのリスト        |

**使用例：**

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

# 参照方法
data.aws_availability_zones.available.names     # ["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]
data.aws_availability_zones.available.zone_ids  # ["apne1-az4", "apne1-az1", "apne1-az2"]
data.aws_availability_zones.available.id        # "ap-northeast-1"
```

### 現在のコードとドキュメントの対応

あなたの現在のコードを、ドキュメントと照らし合わせて見てみましょう：

```hcl
# あなたのコード
data "aws_availability_zones" "available" {
  state = "available"
}
```

**ドキュメントから分かること：**

1. **使用している引数：**

- `state = "available"` → Argument Referenceに記載
- 「available」状態のAZのみを取得

2. **取得できる属性：**

- `.names` → AZ名のリスト
- `.zone_ids` → AZ IDのリスト
- `.id` → リージョン名

3. **実際の使用：**

```hcl
# あなたの現在のコードから
locals {
  # slice()関数：リストから一部を取り出す
  # slice(リスト, 開始位置, 終了位置)
  availability_zones = slice(
    # 全AZのリスト（["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]）
    data.aws_availability_zones.available.names,
    0,  # 0番目から（最初の要素）
    2   # 2番目の手前まで（0番目と1番目を取得）
  )
}
# 結果: 最初の2つのAZのみが取得される
# 例: ["ap-northeast-1a", "ap-northeast-1c"]
```

### ドキュメントを読む際のポイント

#### ポイント1：Example Usageから始める

```
最初にExample Usageを見る理由：
1. 実際の使用方法が分かる
2. 必須の設定が何かが分かる
3. コピー&ペーストで試せる
```

#### ポイント2：必須引数を確認

```hcl
# ドキュメントで「Required」と書かれている引数は必ず設定

# 例：aws_amiの場合
data "aws_ami" "example" {
  most_recent = true  # Optional
  owners      = ["amazon"]  # Required（一部のケースで）

  filter {  # Optional
    name   = "name"  # Requiredと書かれている
    values = ["al2023-ami-*"]  # Requiredと書かれている
  }
}
```

#### ポイント3：Attribute Referenceで参照方法を確認

```hcl
# Attribute Referenceに記載されている属性のみ参照可能

data "aws_availability_zones" "available" {
  state = "available"
}

# 使用可能
data.aws_availability_zones.available.names      # ✓
data.aws_availability_zones.available.zone_ids   # ✓
data.aws_availability_zones.available.id         # ✓

# 使用不可（ドキュメントに記載なし）
data.aws_availability_zones.available.region     # ✗ エラー
```

#### ポイント4：filterの使い方

多くのデータソースでは`filter`ブロックが使用できます：

```hcl
data "aws_availability_zones" "example" {
  filter {
    name   = "opt-in-status"  # フィルター名
    values = ["opt-in-not-required"]  # フィルター値（リスト）
  }
}
```

**filterで使用できる`name`の値：**

- ドキュメントに記載：「Valid values can be found in the EC2 DescribeAvailabilityZones API Reference」
- AWS APIドキュメントを参照する必要がある
- 一般的な例はExample Usageに記載されている

### 他のデータソースでも同じパターン

すべてのデータソースは同じドキュメント構成です：

```hcl
# aws_ami の場合
data "aws_ami" "example" {
  # Argument Reference に記載されている引数
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*"]
  }
}

# Attribute Reference に記載されている属性を参照
ami_id   = data.aws_ami.example.id
ami_name = data.aws_ami.example.name
```

```hcl
# aws_vpc の場合
data "aws_vpc" "example" {
  # Argument Reference に記載されている引数
  filter {
    name   = "tag:Name"
    values = ["my-vpc"]
  }
}

# Attribute Reference に記載されている属性を参照
vpc_id   = data.aws_vpc.example.id
vpc_cidr = data.aws_vpc.example.cidr_block
```

## なぜdataブロックが必要なのか

### 理由1：ハードコーディングを避ける

```hcl
# 悪い例：AMI IDをハードコーディング
resource "aws_instance" "web" {
  ami = "ami-0c55b159cbfafe1f0"  # この値は古くなる可能性がある
  instance_type = "t2.micro"
}

# 良い例：dataブロックで最新のAMIを自動取得
data "aws_ami" "latest" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*"]
  }
}

resource "aws_instance" "web" {
  ami = data.aws_ami.latest.id  # 常に最新のAMI IDが使用される
  instance_type = "t2.micro"
}
```

### 理由2：既存インフラとの連携

```hcl
# シナリオ：既存のVPC内にサブネットを作成したい

# 既存のVPCを検索
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

# 既存のVPC内に新しいサブネットを作成
resource "aws_subnet" "new_subnet" {
  vpc_id     = data.aws_vpc.existing.id  # 既存VPCのIDを使用
  cidr_block = "10.0.10.0/24"
}
```

### 理由3：リージョン間でのリソース情報共有

```hcl
# デフォルトは東京リージョン
provider "aws" {
  region = "ap-northeast-1"
}

# 東京リージョンの既存VPCを検索
data "aws_vpc" "tokyo_vpc" {
  filter {
    name   = "tag:Environment"
    values = ["production"]
  }
}

# 大阪リージョンに新しいVPCを作成
resource "aws_vpc" "osaka_vpc" {
  region     = "ap-northeast-3" # region属性で大阪リージョンわ指定
  cidr_block = "10.1.0.0/16"

  tags = {
    Name = "osaka-vpc"
  }
}

# VPCピアリング接続を作成
resource "aws_vpc_peering_connection" "tokyo_osaka" {
  vpc_id      = data.aws_vpc.tokyo_vpc.id  # 東京のVPC
  peer_vpc_id = aws_vpc.osaka_vpc.id       # 大阪のVPC
  peer_region = "ap-northeast-3"

  tags = {
    Name = "tokyo-osaka-peering"
  }
}
```

## よく使うdataブロックの例

### 1. AMI（Amazon Machine Image）の取得

```hcl
# Amazon Linux 2023（ARM64）の最新AMI
data "aws_ami" "amazon_linux_arm64" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-kernel-6.1-arm64"]
  }

  filter {
    name   = "architecture"
    values = ["arm64"]
  }
}

# 使用例
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_arm64.id
  instance_type = "t4g.micro"
}
```

### 2. VPCの検索

```hcl
# 既存のVPCをタグで検索
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

# デフォルトVPCを取得
data "aws_vpc" "default" {
  default = true
}

# 使用例
resource "aws_subnet" "app" {
  vpc_id     = data.aws_vpc.main.id
  cidr_block = "10.0.20.0/24"
}
```

### 3. サブネットの検索

```hcl
# 特定のVPC内のサブネット一覧を取得
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }

  filter {
    name   = "tag:Type"
    values = ["private"]
  }
}

# 使用例：複数のサブネットIDを参照
resource "aws_db_subnet_group" "main" {
  name       = "main"
  subnet_ids = data.aws_subnets.private.ids
}
```

### 4. セキュリティグループの検索

```hcl
# 既存のセキュリティグループを検索
data "aws_security_group" "default" {
  filter {
    name   = "group-name"
    values = ["default"]
  }

  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
}

# 使用例
resource "aws_instance" "web" {
  ami                    = data.aws_ami.latest.id
  instance_type          = "t2.micro"
  vpc_security_group_ids = [data.aws_security_group.default.id]
}
```

### 5. アベイラビリティゾーンの取得

```hcl
# 利用可能なAZ一覧を取得（あなたのコードで既に使用中）
data "aws_availability_zones" "available" {
  state = "available"
}

# 特定のAZ情報を取得
data "aws_availability_zone" "az_a" {
  name = "ap-northeast-1a"
}

# 使用例（output）
output "az_names" {
  value = data.aws_availability_zones.available.names
}
```

### 6. IAMポリシードキュメントの作成

```hcl
# IAMポリシーを動的に生成
data "aws_iam_policy_document" "s3_policy" {
  statement {
    effect = "Allow"

    actions = [
      "s3:GetObject",
      "s3:ListBucket"
    ]

    resources = [
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/*"
    ]
  }
}

# 使用例
resource "aws_iam_policy" "s3_read" {
  name   = "s3-read-policy"
  policy = data.aws_iam_policy_document.s3_policy.json
}
```

## dataブロックの詳細設定

### フィルター条件の指定

多くのdataブロックでは、`filter`を使って検索条件を指定できます：

```hcl
# 例1：タグで検索
data "aws_vpc" "selected" {
  filter {
    name   = "tag:Environment"
    values = ["production"]
  }

  filter {
    name   = "tag:Team"
    values = ["platform"]
  }
}

# 例2：複数の条件で検索
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical（Ubuntuの提供元）

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

### 最新のリソースを取得

```hcl
data "aws_ami" "latest" {
  most_recent = true  # ← 最新のものを取得
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*"]
  }
}
```

**重要：** `most_recent = true` がないと、複数のAMIが見つかった場合にエラーになります。

### 依存関係の指定

```hcl
# 先にリソースを作成してから検索
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "main-vpc"
  }
}

# VPC作成後にサブネット情報を取得
data "aws_subnets" "main" {
  filter {
    name   = "vpc-id"
    values = [aws_vpc.main.id]
  }

  # VPCが作成されるまで待つ
  depends_on = [aws_vpc.main]
}
```

## dataブロックの実行タイミング

### いつデータが取得されるか

dataブロックは、いつAWSから情報を取得するのでしょうか？答えは「状況による」です。

**2つのパターンがあります：**

1. **plan時に取得される**（固定値のみを使用している場合）
2. **apply時に取得される**（resourceの属性を参照している場合）

### パターン1：plan時に取得（固定値のみ）

```hcl
data "aws_ami" "latest" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*"]  # ← 固定の文字列
  }
}
```

**なぜplan時に取得できるのか：**

- すべての条件が固定値（文字列）
- resourceを参照していない
- Terraformはすぐに検索を実行できる

**terraform planを実行すると：**

```bash
$ terraform plan

data.aws_ami.latest: Reading...
data.aws_ami.latest: Read complete after 1s [id=ami-0c55b159cbfafe1f0]

# planの出力に実際のAMI IDが表示される
+ resource "aws_instance" "web" {
    ami = "ami-0c55b159cbfafe1f0"  # ← 実際のIDが表示
    # ...
  }
```

**重要：** AMI IDが具体的に表示されます。これにより、applyする前にどのAMIが使用されるか確認できます。

### パターン2：apply時に取得（resourceを参照）

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

data "aws_subnets" "main" {
  filter {
    name   = "vpc-id"
    values = [aws_vpc.main.id]  # ← resourceの属性を参照
  }
}
```

**なぜapply時まで待つのか：**

- VPCはまだ作成されていない
- VPC IDはapply後にしか分からない
- TerraformはVPCを作成してからサブネットを検索する

**terraform planを実行すると：**

```bash
$ terraform plan

# VPCの作成計画
+ resource "aws_vpc" "main" {
    id         = (known after apply)  # ← まだ不明
    cidr_block = "10.0.0.0/16"
    # ...
  }

# dataブロックの状態
data.aws_subnets.main: Reading...
data.aws_subnets.main: Read complete after 0s (known after apply)
```

**重要：** "(known after apply)" と表示されます。これは「applyを実行するまで値が分からない」という意味です。

### 実際の動作の違い

**パターン1（plan時）：**

```
1. terraform plan 実行
2. dataブロックがすぐに実行される
3. 実際の値がplanに表示される
4. terraform apply 実行
5. 表示された値でリソースが作成される
```

**パターン2（apply時）：**

```
1. terraform plan 実行
2. dataブロックは実行されない
3. "(known after apply)" と表示される
4. terraform apply 実行
5. resourceが作成される
6. dataブロックが実行される
7. 取得した値でリソースが作成される
```

### あなたの現在のコードでの例

```hcl
# これはplan時に実行される
data "aws_availability_zones" "available" {
  state = "available"  # ← 固定値のみ
}

# terraform plan で確認してみましょう
```

**実行してみると：**

```bash
$ terraform plan

data.aws_availability_zones.available: Reading...
data.aws_availability_zones.available: Read complete after 1s

# 実際のAZ名がplanで確認できる
```

### なぜこれを理解する必要があるのか

1. **planで確認できる情報が分かる**

- 固定値のdataブロック → 実際の値が見える
- resourceを参照するdataブロック → "(known after apply)"

2. **エラーの原因が分かる**

- resourceがまだ存在しない → dataブロックが失敗
- 依存関係の問題 → depends_onで解決

3. **実行順序が理解できる**

- Terraformがどの順番で処理するか
- なぜ特定の値が後で決まるのか

````

**What I added:**
1. **WHY** it happens (fixed values vs resource references)
2. **WHAT IT MEANS** for beginners ("known after apply" explanation)
3. **HOW** the execution flow differs (step-by-step)
4. **WHY THEY SHOULD CARE** (3 practical reasons)
5. Real terminal output **with explanation** of what each line means

Should I update the tutorial with this clearer explanation?

## ベストプラクティス

### 1. 明確な検索条件を指定する

```hcl
# 悪い例：曖昧な検索条件
data "aws_ami" "latest" {
  most_recent = true
  owners      = ["amazon"]
  # フィルターが足りない - 意図しないAMIが見つかる可能性
}

# 良い例：明確な検索条件
data "aws_ami" "latest" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-kernel-6.1-arm64"]
  }

  filter {
    name   = "architecture"
    values = ["arm64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
````

### 2. ローカル名は分かりやすく

```hcl
# 悪い例：意味が分からない名前
data "aws_ami" "a" { }
data "aws_vpc" "v" { }

# 良い例：目的が明確な名前
data "aws_ami" "amazon_linux_2023_arm64" { }
data "aws_vpc" "production_vpc" { }
```

### 3. 説明コメントを追加

```hcl
# 本番環境のVPCを検索
# このVPCは手動で作成された既存のリソース
data "aws_vpc" "production" {
  filter {
    name   = "tag:Environment"
    values = ["production"]
  }

  filter {
    name   = "tag:ManagedBy"
    values = ["manual"]
  }
}
```

### 4. most_recentを忘れずに指定

```hcl
# 悪い例：複数の結果が返る可能性
data "aws_ami" "amazon_linux" {
  owners = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*"]
  }
}
# 複数のAMIが見つかった場合エラーになる

# 良い例：most_recentで最新を指定
data "aws_ami" "amazon_linux" {
  most_recent = true  # ← これを追加
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*"]
  }
}
```

### 5. 依存関係を明示する

```hcl
# 明示的な依存関係
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

data "aws_vpc" "main" {
  id = aws_vpc.main.id

  # VPC作成を待つ
  depends_on = [aws_vpc.main]
}
```

### 6. ドキュメントを必ず確認する

```
新しいデータソースを使用する前に：
1. Terraform Registryでドキュメントを開く
2. Example Usageを読む
3. Argument Referenceで必須引数を確認
4. Attribute Referenceで使用可能な属性を確認
5. コピー&ペーストで試す
```

## よくある間違いと対策

### 間違い1：dataとresourceの混同

```hcl
# 間違い：既存リソースをresourceで定義
resource "aws_vpc" "existing" {
  # これは既存のVPCを上書きしようとする！
  cidr_block = "10.0.0.0/16"
}

# 正しい：既存リソースはdataで取得
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["existing-vpc"]
  }
}
```

### 間違い2：参照時のdata.プレフィックス忘れ

```hcl
data "aws_ami" "latest" {
  most_recent = true
  owners      = ["amazon"]
}

resource "aws_instance" "web" {
  # 間違い
  ami = aws_ami.latest.id

  # 正しい
  ami = data.aws_ami.latest.id
}
```

### 間違い3：存在しないリソースの検索

```hcl
# 存在しないVPCを検索するとエラー
data "aws_vpc" "nonexistent" {
  filter {
    name   = "tag:Name"
    values = ["does-not-exist"]
  }
}
# エラー: no matching VPC found
```

**対策：** 検索前にリソースが存在することを確認するか、エラーメッセージを確認して対処する。

### 間違い4：most_recentの指定忘れ

```hcl
# 間違い：複数のAMIが見つかる場合エラー
data "aws_ami" "latest" {
  owners = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*"]
  }
}
# エラー: multiple AMIs matched

# 正しい：most_recentを指定
data "aws_ami" "latest" {
  most_recent = true  # ← これを追加
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*"]
  }
}
```

### 間違い5：循環参照

```hcl
# 間違い：dataとresourceが互いに参照
resource "aws_instance" "web" {
  ami           = data.aws_ami.for_web.id
  instance_type = "t2.micro"

  tags = {
    Name = "web-server"
  }
}

data "aws_ami" "for_web" {
  filter {
    name   = "tag:UsedBy"
    values = [aws_instance.web.id]  # ← 循環参照！
  }
}

# 正しい：依存関係を整理
data "aws_ami" "web" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.web.id
  instance_type = "t2.micro"

  tags = {
    Name = "web-server"
  }
}
```

### 間違い6：フィルター値のタイプミス

```hcl
# 間違い：タグ名のタイプミス
data "aws_vpc" "main" {
  filter {
    name   = "tag:Environemnt"  # ← "Environment"のスペルミス
    values = ["production"]
  }
}
# 結果: VPCが見つからない

# 正しい：タグ名を正確に指定
data "aws_vpc" "main" {
  filter {
    name   = "tag:Environment"  # ← 正しいスペル
    values = ["production"]
  }
}
```

### 間違い7：ドキュメントにない属性を参照

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

# 間違い：ドキュメントにない属性
output "region" {
  value = data.aws_availability_zones.available.region  # ← エラー！
}

# 正しい：ドキュメントに記載されている属性
output "region" {
  value = data.aws_availability_zones.available.id  # ← これが正しい
}
```

## dataブロックとoutputの組み合わせ

dataブロックで取得した情報は、outputで出力して確認できます：

```hcl
# AMI情報を取得
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-kernel-6.1-arm64"]
  }
}

# 取得した情報を出力
output "ami_id" {
  description = "最新のAmazon Linux AMI ID"
  value       = data.aws_ami.amazon_linux.id
}

output "ami_name" {
  description = "AMI名"
  value       = data.aws_ami.amazon_linux.name
}

output "ami_creation_date" {
  description = "AMI作成日"
  value       = data.aws_ami.amazon_linux.creation_date
}
```

確認方法：

```bash
# planで確認
terraform plan

# applyで出力
terraform apply

# 出力のみ表示
terraform output
terraform output ami_id
```

## まとめ

### dataブロックの重要ポイント

1. **読み取り専用**

- AWSに何も作成しない
- 既存のリソース情報を取得するだけ
- `data.` プレフィックスで参照

2. **動的な設定を実現**

- ハードコーディングを避ける
- 最新情報を自動取得
- 環境に応じた柔軟な設定

3. **既存インフラとの連携**

- 手動で作成したリソースを参照
- 他のTerraformプロジェクトのリソースを参照
- リージョン間でのリソース共有

4. **ドキュメントを活用**

- Terraform Registryで必ずドキュメントを確認
- Example Usageから始める
- Argument ReferenceとAttribute Referenceを理解

5. **検索条件は明確に**

- フィルターを適切に設定
- `most_recent` を使用
- タグ名のスペルミスに注意

6. **実行タイミングに注意**

- 固定値のみ → plan時に実行
- resource参照 → apply時に実行
- 依存関係を理解する

### dataブロックを使うべき場面

- 最新のAMI IDを取得したい
- 既存のVPCやサブネットを参照したい
- 利用可能なAZ一覧を取得したい
- リージョン固有の情報が必要
- 他のチームが管理するリソースを参照したい

### dataブロックを使わない場合

- 新しいリソースを作成する → `resource`を使用
- 固定値で問題ない → 変数に直接記述
- Terraform管理外のリソースを変更する → 手動操作g
