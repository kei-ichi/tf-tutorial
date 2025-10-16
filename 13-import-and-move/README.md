# TerraformのインポートとリファクタリングBlocked完全ガイド

## この章で学ぶこと

既存のインフラをTerraformで管理する方法と、Terraformコードをリファクタリング（再構成）する方法を学習します。

**学習内容：**

- importブロック：既存のインフラをTerraformに取り込む
- dataブロックとの違い：管理 vs 参照
- movedブロック：リソースのアドレスを安全に変更
- movedを使わない場合の問題
- 実践的な使用例
- ベストプラクティス

## なぜインポートとリファクタリングが必要なのか

### 問題1：手動で作成したリソースをTerraformで管理したい

**シナリオ：**

```
状況：
- AWSコンソールでVPCとサブネットを手動作成
- EC2インスタンスもマニュアルで起動
- 運用が進むにつれて管理が複雑に
- Terraformで管理したいが、既存のリソースは削除できない

問題：
- 既存のリソースを削除せずにTerraformで管理したい
- しかしTerraformは通常、新しくリソースを作成しようとする
```

**解決策：importブロック**

```hcl
# 既存のVPCをTerraformにインポート
import {
  to = aws_vpc.main
  id = "vpc-0abc123def456789"  # 既存のVPC ID
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  # Terraformは既存のVPCを削除せず、管理下に置く
}
```

### 問題2：Terraformコードをリファクタリングしたい

**シナリオ：**

```
状況：
- プロジェクトが成長してコードが複雑に
- リソース名を変更したい
- モジュールに分割したい
- しかし、名前を変えるとTerraformは削除と再作成を試みる

問題：
resource "aws_instance" "web" { }
 ↓ リソース名を変更
resource "aws_instance" "web_server" { }

通常の動作（movedブロックを使わない場合）：
1. aws_instance.web を削除
2. aws_instance.web_server を新規作成
→ サービス停止、データ損失のリスク
```

**解決策：movedブロック**

```hcl
# リソースの名前を安全に変更
moved {
  from = aws_instance.web
  to   = aws_instance.web_server
}

resource "aws_instance" "web_server" {
  # 既存のインスタンスを削除せず、名前だけ変更
}
```

## importブロック：既存リソースの取り込み

### 基本的な考え方

**例えて言うなら：** 既存の家をリフォーム会社の管理下に置く

```
状況：
- 自分で建てた家がある（既存のAWSリソース）
- リフォーム会社に管理を任せたい（Terraformで管理）
- しかし家を壊して建て直すわけにはいかない

解決：
- 家の図面を作成（Terraformコード）
- 既存の家を会社の管理台帳に登録（import）
- これで管理下に置ける
```

### importブロックの基本構文

```hcl
import {
  to = リソースアドレス
  id = "クラウドプロバイダーのリソースID"
}

resource "リソースタイプ" "名前" {
  # 既存リソースの設定を記述
}
```

**構成要素：**

```hcl
import {
  to = aws_instance.web          # インポート先のTerraformアドレス
  id = "i-0abc123def456789"      # AWSのリソースID
}

resource "aws_instance" "web" {
  # 既存インスタンスの現在の設定を記述
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"
}
```

### 実例1：EC2インスタンスのインポート

**ステップ1：既存リソースの情報を確認**

```bash
# AWSコンソールまたはCLIで既存リソースを確認
aws ec2 describe-instances --instance-ids i-0abc123def456789
```

**ステップ2：importブロックとresourceブロックを作成**

```hcl
# import.tf（importブロック専用ファイル推奨）

import {
  to = aws_instance.web
  id = "i-0abc123def456789"
}
```

```hcl
# main.tf

resource "aws_instance" "web" {
  # 既存インスタンスの現在の設定を記述
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t4g.small"

  tags = {
    Name = "web-server"
  }
}
```

**ステップ3：インポート実行**

```bash
# プラン確認（インポートが正しく計画されているか）
terraform plan

# 出力例：
# aws_instance.web: Importing from ID "i-0abc123def456789"...
# aws_instance.web: Import prepared!
#
# Terraform will perform the following actions:
#
#   # aws_instance.web will be imported
#   resource "aws_instance.web" "web" {
#       id            = "i-0abc123def456789"
#       # ... その他の属性
#   }

# インポート実行
terraform apply
```

**ステップ4：確認**

```bash
# 状態を確認
terraform state list

# 出力：
# aws_instance.web

# 詳細確認
terraform state show aws_instance.web
```

### 実例2：VPCとサブネットのインポート

**シナリオ：既存のVPC、サブネットをまとめてインポート**

```hcl
# import.tf

# VPCをインポート
import {
  to = aws_vpc.main
  id = "vpc-0abc123def456789"
}

# パブリックサブネットをインポート
import {
  to = aws_subnet.public_1
  id = "subnet-0abc123def456789"
}

import {
  to = aws_subnet.public_2
  id = "subnet-1abc123def456789"
}
```

```hcl
# main.tf

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "main-vpc"
  }
}

resource "aws_subnet" "public_1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "ap-northeast-1a"

  tags = {
    Name = "public-subnet-1"
  }
}

resource "aws_subnet" "public_2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "ap-northeast-1c"

  tags = {
    Name = "public-subnet-2"
  }
}
```

### 実例3：for_eachでの複数リソースインポート

**シナリオ：複数のS3バケットを一度にインポート**

```hcl
# import.tf

locals {
  buckets = {
    "staging" = "my-app-staging-bucket"
    "uat"     = "my-app-uat-bucket"
    "prod"    = "my-app-prod-bucket"
  }
}

import {
  for_each = local.buckets
  to       = aws_s3_bucket.this[each.key]
  id       = each.value
}
```

```hcl
# main.tf

locals {
  buckets = {
    "staging" = "my-app-staging-bucket"
    "uat"     = "my-app-uat-bucket"
    "prod"    = "my-app-prod-bucket"
  }
}

resource "aws_s3_bucket" "this" {
  for_each = local.buckets

  bucket = each.value

  tags = {
    Environment = each.key
  }
}
```

**実行：**

```bash
terraform plan
terraform apply

# 結果：3つのバケットが一度にインポートされる
# aws_s3_bucket.this["staging"]
# aws_s3_bucket.this["uat"]
# aws_s3_bucket.this["prod"]
```

## importとdataの違い：管理 vs 参照

### 基本的な違い

**重要な違い：**

```
import：既存リソースをTerraformで管理する
- リソースをTerraformの管理下に置く
- 変更、削除ができる
- terraform applyで設定を変更できる

data：既存リソースを参照するだけ
- リソースはTerraform管理外
- 読み取り専用
- 変更も削除もできない
```

### 具体例で理解する

#### シナリオ：既存のVPCがある

**例えて言うなら：**

```
import：
  賃貸物件を購入して所有者になる
  - リフォームできる（変更可能）
  - 売却できる（削除可能）
  - 自分の管理台帳に記載（terraform.tfstate）

data：
  近所の建物を観察するだけ
  - 見るだけ（読み取り専用）
  - 変更できない
  - 所有していない
```

#### パターン1：importを使う場合（管理したい）

```hcl
# import.tf
import {
  to = aws_vpc.main
  id = "vpc-0abc123def456789"
}

# main.tf
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  # タグを変更できる
  tags = {
    Name = "main-vpc"
    Team = "platform"  # 新しいタグを追加
  }
}
```

**実行結果：**

```bash
terraform apply

# Terraformはタグを追加する
# aws_vpc.main will be updated in-place
#   ~ tags = {
#       + Team = "platform"
#     }
```

**特徴：**

```
管理下にある：
- タグの追加・変更が可能
- terraform destroyで削除可能
- terraform.tfstateに記録される
```

#### パターン2：dataを使う場合（参照のみ）

```hcl
# main.tf
data "aws_vpc" "existing" {
  id = "vpc-0abc123def456789"
}

# VPCの情報を参照するだけ
resource "aws_subnet" "public" {
  vpc_id     = data.aws_vpc.existing.id  # 参照
  cidr_block = "10.0.1.0/24"
}

# このVPCはTerraform管理外なので変更できない
```

**実行結果：**

```bash
terraform apply

# VPCは変更されない（読み取り専用）
# サブネットだけが作成される
```

**特徴：**

```
管理外：
- VPCの変更は不可能
- terraform destroyでもVPCは削除されない
- サブネット作成時にVPC IDを参照するだけ
```

### 使い分けの基準

#### importを使うべき場合

```
状況：
1. 既存リソースをTerraformで管理したい
   例：手動作成したVPCをTerraformで管理

2. リソースを変更・削除する権限がある
   例：自分のプロジェクトのリソース

3. リソースのライフサイクルを制御したい
   例：terraform destroyで削除したい

実例：
- 開発初期に手動で作成したインフラ
- 他のツールで作成したリソースを引き継ぐ
- 既存プロジェクトのTerraform化
```

**コード例：**

```hcl
# 既存のEC2インスタンスをTerraformで管理
import {
  to = aws_instance.app
  id = "i-0abc123def456789"
}

resource "aws_instance" "app" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  # これ以降、インスタンスタイプの変更などが可能
}
```

#### dataを使うべき場合

```
状況：
1. 他のチームが管理しているリソースを参照
   例：ネットワークチームが管理するVPC

2. 変更権限がない、または変更すべきでない
   例：共有インフラストラクチャ

3. 情報を読み取るだけでよい
   例：VPC IDだけ必要

実例：
- 共有VPCの情報取得
- 組織全体で使用するAMIの取得
- 他部門が管理するリソースの参照
```

**コード例：**

```hcl
# 他チームが管理するVPCを参照
data "aws_vpc" "shared" {
  tags = {
    Name = "shared-vpc"
    Team = "network-team"
  }
}

# このVPC内にサブネットを作成（VPC自体は変更しない）
resource "aws_subnet" "app" {
  vpc_id     = data.aws_vpc.shared.id
  cidr_block = "10.0.10.0/24"
}
```

### 比較表

| 項目              | import                    | data                     |
| ----------------- | ------------------------- | ------------------------ |
| 目的              | リソースを管理下に置く    | リソースを参照する       |
| 管理              | Terraform管理下           | Terraform管理外          |
| 変更              | 可能                      | 不可能                   |
| 削除              | 可能（terraform destroy） | 不可能                   |
| terraform.tfstate | 記録される                | 記録されない（毎回取得） |
| 使用場面          | 既存リソースのTerraform化 | 他チームのリソース参照   |
| resourceブロック  | 必要                      | 不要（dataブロックのみ） |

### 実践例：両方を組み合わせる

**シナリオ：**

- ネットワークチームが管理する共有VPC（data）
- 自チームのアプリ用サブネットとEC2（import + resource）

```hcl
# =============================================================================
# 共有VPCを参照（data：管理外）
# =============================================================================
data "aws_vpc" "shared" {
  tags = {
    Name = "shared-vpc"
    Team = "network-team"
  }
}

# =============================================================================
# 既存のサブネットをインポート（import：管理下に置く）
# =============================================================================
import {
  to = aws_subnet.app
  id = "subnet-0abc123def456789"
}

resource "aws_subnet" "app" {
  vpc_id     = data.aws_vpc.shared.id  # 共有VPCを参照
  cidr_block = "10.0.10.0/24"

  tags = {
    Name = "app-subnet"
    Team = "app-team"
  }
}

# =============================================================================
# 新しいEC2インスタンスを作成（resource：新規作成して管理）
# =============================================================================
resource "aws_instance" "app" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"
  subnet_id     = aws_subnet.app.id  # インポートしたサブネットを使用

  tags = {
    Name = "app-server"
  }
}
```

**この構成の説明：**

```
data.aws_vpc.shared：
- ネットワークチームが管理
- 参照のみ
- 変更不可

aws_subnet.app：
- 自チームが管理
- importで取り込み
- 変更・削除可能

aws_instance.app：
- 自チームが管理
- 新規作成
- 変更・削除可能
```

## movedブロック：リソースの安全な移動

### 基本的な考え方

**例えて言うなら：** 引っ越しの住所変更届

```
状況：
- 引っ越しをする（リソース名を変更）
- 郵便局に住所変更届を出す（movedブロック）
- 新しい住所で郵便物を受け取れる（リソースは削除されない）

movedブロックがない場合：
- 旧住所の契約解除（リソース削除）
- 新住所の契約（新規リソース作成）
→ データ損失、サービス停止
```

### movedブロックを使わない場合の問題

#### 問題のシナリオ1：リソース名の変更

**初期状態：**

```hcl
# main.tf
resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "web-server"
  }
}
```

```bash
# この状態でterraform apply済み
# EC2インスタンスが作成されている：i-0abc123def456789
```

**リソース名を変更（movedブロックなし）：**

```hcl
# main.tf（変更後）
resource "aws_instance" "web_server" {  # 名前を変更
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "web-server"
  }
}
```

**terraform planの結果：**

```bash
terraform plan

# 出力：
Terraform will perform the following actions:

  # aws_instance.web will be destroyed
  # (because aws_instance.web is not in configuration)
  - resource "aws_instance" "web" {
      - id            = "i-0abc123def456789"
      - instance_type = "t4g.small"
      # ... その他の属性
    }

  # aws_instance.web_server will be created
  + resource "aws_instance" "web_server" {
      + id            = (known after apply)
      + instance_type = "t4g.small"
      # ... その他の属性
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```

**問題点：**

```
危険な動作：
1. 既存のEC2インスタンス（i-0abc123def456789）が削除される
2. 新しいEC2インスタンスが作成される

影響：
- サービス停止（ダウンタイム発生）
- IPアドレスが変わる
- ディスク内のデータが失われる
- DNSの再設定が必要
- ユーザーがアクセスできなくなる
```

**terraform apply実行すると：**

```bash
terraform apply

# 実際に起こること：
Step 1: Destroying aws_instance.web... (サービス停止)
Step 2: aws_instance.web: Destruction complete
Step 3: Creating aws_instance.web_server...
Step 4: aws_instance.web_server: Creation complete (新しいIP)

# 結果：
# - 旧インスタンス削除
# - 新インスタンス作成
# - サービス停止時間：数分
```

#### 解決策：movedブロックを使用

```hcl
# main.tf（正しい方法）

# リソース名を変更
resource "aws_instance" "web_server" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "web-server"
  }
}

# 旧名から新名への移動を記録
moved {
  from = aws_instance.web
  to   = aws_instance.web_server
}
```

**terraform planの結果：**

```bash
terraform plan

# 出力：
Terraform will perform the following actions:

  # aws_instance.web has moved to aws_instance.web_server
    resource "aws_instance" "web_server" {
        id            = "i-0abc123def456789"
        instance_type = "t4g.small"
        # ... その他の属性
    }

Plan: 0 to add, 0 to change, 0 to destroy.
```

**メリット：**

```
安全な動作：
1. 既存のインスタンスは削除されない
2. terraform.tfstate内のアドレスだけが変更される
3. 実際のリソースには一切影響なし

影響：
- サービス停止なし
- IPアドレス変更なし
- データ保持
- ダウンタイムゼロ
```

#### 問題のシナリオ2：countからfor_eachへの移行

**初期状態：countを使用**

```hcl
# main.tf
resource "aws_subnet" "public" {
  count = 2

  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "public-subnet-${count.index + 1}"
  }
}

# 作成されているリソース：
# aws_subnet.public[0] - 10.0.1.0/24 in ap-northeast-1a
# aws_subnet.public[1] - 10.0.2.0/24 in ap-northeast-1c
```

**for_eachに変更（movedブロックなし）：**

```hcl
# main.tf（変更後）
locals {
  subnets = {
    "public-1a" = {
      cidr_block = "10.0.1.0/24"
      az         = "ap-northeast-1a"
    }
    "public-1c" = {
      cidr_block = "10.0.2.0/24"
      az         = "ap-northeast-1c"
    }
  }
}

resource "aws_subnet" "public" {
  for_each = local.subnets  # countからfor_eachに変更

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.az

  tags = {
    Name = each.key
  }
}
```

**terraform planの結果：**

```bash
terraform plan

# 出力：
Terraform will perform the following actions:

  # aws_subnet.public[0] will be destroyed
  - resource "aws_subnet" "public" {
      - id         = "subnet-0abc123def456789"
      - cidr_block = "10.0.1.0/24"
      # ...
    }

  # aws_subnet.public[1] will be destroyed
  - resource "aws_subnet" "public" {
      - id         = "subnet-1abc123def456789"
      - cidr_block = "10.0.2.0/24"
      # ...
    }

  # aws_subnet.public["public-1a"] will be created
  + resource "aws_subnet" "public" {
      + id         = (known after apply)
      + cidr_block = "10.0.1.0/24"
      # ...
    }

  # aws_subnet.public["public-1c"] will be created
  + resource "aws_subnet" "public" {
      + id         = (known after apply)
      + cidr_block = "10.0.2.0/24"
      # ...
    }

Plan: 2 to add, 0 to change, 2 to destroy.
```

**問題点：**

```
危険な動作：
1. 既存の2つのサブネット削除
2. 新しい2つのサブネット作成

影響：
- これらのサブネット内のEC2インスタンスが影響を受ける
- ネットワーク接続が一時的に切断される
- 関連するルートテーブル、NACLの設定が失われる
- サブネットIDが変わるため、他のリソースで参照エラー
```

#### 解決策：movedブロックを使用

```hcl
# main.tf（正しい方法）

locals {
  subnets = {
    "public-1a" = {
      cidr_block = "10.0.1.0/24"
      az         = "ap-northeast-1a"
    }
    "public-1c" = {
      cidr_block = "10.0.2.0/24"
      az         = "ap-northeast-1c"
    }
  }
}

resource "aws_subnet" "public" {
  for_each = local.subnets

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.az

  tags = {
    Name = each.key
  }
}

# 既存のリソースを新しいキーに移動
moved {
  from = aws_subnet.public[0]
  to   = aws_subnet.public["public-1a"]
}

moved {
  from = aws_subnet.public[1]
  to   = aws_subnet.public["public-1c"]
}
```

**terraform planの結果：**

```bash
terraform plan

# 出力：
Terraform will perform the following actions:

  # aws_subnet.public[0] has moved to aws_subnet.public["public-1a"]
    resource "aws_subnet" "public" {
        id         = "subnet-0abc123def456789"
        cidr_block = "10.0.1.0/24"
        # ...
    }

  # aws_subnet.public[1] has moved to aws_subnet.public["public-1c"]
    resource "aws_subnet" "public" {
        id         = "subnet-1abc123def456789"
        cidr_block = "10.0.2.0/24"
        # ...
    }

Plan: 0 to add, 0 to change, 0 to destroy.
```

**メリット：**

```
安全な動作：
1. 既存のサブネットは削除されない
2. terraform.tfstate内のアドレスだけが変更される
3. サブネットID、CIDR、AZはすべてそのまま

影響：
- サービス停止なし
- ネットワーク接続維持
- 関連リソースへの影響なし
```

### movedブロックの基本構文

```hcl
moved {
  from = 旧アドレス
  to   = 新アドレス
}
```

**構成要素：**

```hcl
moved {
  from = aws_instance.web          # 変更前のアドレス
  to   = aws_instance.web_server   # 変更後のアドレス
}
```

### 実例1：リソース名の変更

```hcl
# 変更前のコード
resource "aws_instance" "a" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "web-server"
  }
}
```

**ステップ1：リソース名を変更してmovedブロックを追加**

```hcl
# main.tf

# リソース名を変更
resource "aws_instance" "web_server" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "web-server"
  }
}

# 旧名から新名への移動を記録
moved {
  from = aws_instance.a
  to   = aws_instance.web_server
}
```

**ステップ2：プラン確認**

```bash
terraform plan

# 出力例：
# aws_instance.web_server has moved to aws_instance.web_server
#
# Terraform will perform the following actions:
#
#   # aws_instance.a has moved to aws_instance.web_server
#     resource "aws_instance" "web_server" {
#         id            = "i-0abc123def456789"
#         # ...
#     }
#
# Plan: 0 to add, 0 to change, 0 to destroy.
```

**ステップ3：適用**

```bash
terraform apply

# リソースは削除されず、アドレスだけが変更される
```

### 実例2：モジュールへの移動

**シナリオ：リソースをモジュールに整理**

**変更前：すべてルートモジュールに記述**

```hcl
# main.tf（変更前）

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}
```

**movedブロックなしで変更した場合：**

```hcl
# main.tf（変更後：movedなし）

module "network" {
  source = "./modules/network"

  vpc_cidr    = "10.0.0.0/16"
  subnet_cidr = "10.0.1.0/24"
}

# terraform planの結果：
# すべてのネットワークリソースが削除・再作成される
# Plan: 3 to add, 0 to change, 3 to destroy.
```

**正しい方法：movedブロックを使用**

```hcl
# modules/network/main.tf（モジュール作成）

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = var.subnet_cidr
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}
```

```hcl
# main.tf（ルートモジュール：変更後）

# モジュールを呼び出す
module "network" {
  source = "./modules/network"

  vpc_cidr    = "10.0.0.0/16"
  subnet_cidr = "10.0.1.0/24"
}

# 既存リソースをモジュール内に移動
moved {
  from = aws_vpc.main
  to   = module.network.aws_vpc.main
}

moved {
  from = aws_subnet.public
  to   = module.network.aws_subnet.public
}

moved {
  from = aws_internet_gateway.main
  to   = module.network.aws_internet_gateway.main
}
```

**terraform planの結果：**

```bash
terraform plan

# 出力例：
# aws_vpc.main has moved to module.network.aws_vpc.main
# aws_subnet.public has moved to module.network.aws_subnet.public
# aws_internet_gateway.main has moved to module.network.aws_internet_gateway.main
#
# Plan: 0 to add, 0 to change, 0 to destroy.
```

### 実例3：モジュール名の変更

**変更前：**

```hcl
module "net" {
  source = "./modules/network"
  # ...
}
```

**movedブロックなしで変更：**

```hcl
module "network" {
  source = "./modules/network"
  # ...
}

# terraform planの結果：
# module.netのすべてのリソースが削除される
# module.networkのすべてのリソースが新規作成される
```

**正しい方法：**

```hcl
module "network" {
  source = "./modules/network"
  # ...
}

moved {
  from = module.net
  to   = module.network
}

# terraform planの結果：
# Plan: 0 to add, 0 to change, 0 to destroy.
```

### 実例4：モジュールの分割

**変更前：1つの大きなモジュール**

```hcl
# modules/infrastructure/main.tf

resource "aws_vpc" "main" {
  # ...
}

resource "aws_instance" "web" {
  # ...
}

resource "aws_s3_bucket" "data" {
  # ...
}
```

**変更後：機能ごとに分割**

```hcl
# modules/network/main.tf
resource "aws_vpc" "main" {
  # ...
}

# modules/compute/main.tf
resource "aws_instance" "web" {
  # ...
}

# modules/storage/main.tf
resource "aws_s3_bucket" "data" {
  # ...
}
```

```hcl
# main.tf（ルートモジュール）

module "network" {
  source = "./modules/network"
  # ...
}

module "compute" {
  source = "./modules/compute"
  # ...
}

module "storage" {
  source = "./modules/storage"
  # ...
}

# 旧モジュールから新モジュールへリソースを移動
moved {
  from = module.infrastructure.aws_vpc.main
  to   = module.network.aws_vpc.main
}

moved {
  from = module.infrastructure.aws_instance.web
  to   = module.compute.aws_instance.web
}

moved {
  from = module.infrastructure.aws_s3_bucket.data
  to   = module.storage.aws_s3_bucket.data
}
```

## ベストプラクティス

### 1. importブロックのベストプラクティス

#### 1-1. import.tf ファイルを作成

```
推奨構成：

my-project/
├── main.tf
├── variables.tf
├── outputs.tf
├── import.tf        ← importブロック専用ファイル
└── terraform.tfvars

理由：
- importブロックを一箇所にまとめる
- メインコードと分離
- 管理しやすい
```

**例：**

```hcl
# import.tf

# VPCのインポート
import {
  to = aws_vpc.main
  id = "vpc-0abc123def456789"
}

# サブネットのインポート
import {
  to = aws_subnet.public_1
  id = "subnet-0abc123def456789"
}

import {
  to = aws_subnet.public_2
  id = "subnet-1abc123def456789"
}
```

#### 1-2. リソース設定の確認

```bash
# ステップ1：既存リソースの設定を確認
aws ec2 describe-instances --instance-ids i-0abc123def456789

# ステップ2：resourceブロックに正確に記述
# 特に以下の属性に注意：
# - タグ
# - セキュリティグループ
# - サブネット
# - IAMロール

# ステップ3：planで差分を確認
terraform plan

# ステップ4：差分があれば修正
# No changes になるまで調整
```

#### 1-3. インポート後のクリーンアップ

```hcl
# インポート成功後、importブロックの扱い

方法1：コメントアウト（推奨）
# import {
#   to = aws_instance.web
#   id = "i-0abc123def456789"
# }

方法2：削除
# 完全に削除してもOK
# ただし、履歴として残す方が良い場合もある

方法3：別ファイルに移動
# imported-resources.tf.bak などにバックアップ
```

### 2. importとdataの使い分け

```
自チームのリソース：
- 管理権限がある → import
- 変更・削除したい → import

他チームのリソース：
- 管理権限がない → data
- 参照のみ → data

共有インフラ：
- 変更すべきでない → data
- 読み取り専用でOK → data
```

### 3. movedブロックのベストプラクティス

#### 3-1. movedブロックは残す

```hcl
# 推奨：movedブロックを削除しない

moved {
  from = aws_instance.a
  to   = aws_instance.web_server
}

resource "aws_instance" "web_server" {
  # ...
}

理由：
1. アップグレードパスの保持
   - 古いバージョンから新しいバージョンへの移行をサポート

2. 履歴の記録
   - どのように変更されたか分かる

3. チーム開発での混乱防止
   - 古いブランチからマージする時に安全

削除してよい場合：
- すべてのユーザーが新バージョンに移行済み
- プライベートモジュールで管理が容易
- 十分な時間が経過（例：1年以上）
```

#### 3-2. 段階的な移動を記録

```hcl
# 複数回の名前変更を記録

# 第1回の変更
moved {
  from = aws_instance.a
  to   = aws_instance.web
}

# 第2回の変更
moved {
  from = aws_instance.web
  to   = aws_instance.web_server
}

# これにより、すべての履歴をサポート：
# - aws_instance.a からの移行
# - aws_instance.web からの移行
# どちらも aws_instance.web_server に正しく移動
```

#### 3-3. コメントで理由を記録

```hcl
# リソース名を変更
# 理由：より分かりやすい命名規則に統一
# 日付：2025-01-15
moved {
  from = aws_instance.a
  to   = aws_instance.web_server
}

# モジュールに移動
# 理由：ネットワークリソースを独立したモジュールに整理
# 日付：2025-02-01
moved {
  from = aws_vpc.main
  to   = module.network.aws_vpc.main
}
```

#### 3-4. planで変更を確認

```bash
# movedブロック追加後、必ず確認

terraform plan

# 期待する結果：
# Plan: 0 to add, 0 to change, 0 to destroy.
#
# ただし以下のメッセージが表示される：
# aws_instance.a has moved to aws_instance.web_server

# NGパターン：削除や作成が計画される
# Plan: 1 to add, 0 to change, 1 to destroy.
# これはmovedが正しく動作していない
```

### 4. 共通のベストプラクティス

#### 4-1. バックアップを取る

```bash
# 重要な変更の前にバックアップ

# ステップ1：状態ファイルをバックアップ
cp terraform.tfstate terraform.tfstate.backup

# ステップ2：コードをコミット
git add .
git commit -m "Before refactoring"

# ステップ3：変更を実施
# ...

# ステップ4：問題があれば戻す
# git revert または terraform state mv で修正
```

#### 4-2. 段階的に実施

```
一度に大量の変更をしない

悪い例：
- 50個のリソースを一度にインポート
- 複数のモジュールを同時にリファクタリング

良い例：
ステップ1：VPCとサブネットをインポート
  ↓ 確認
ステップ2：セキュリティグループをインポート
  ↓ 確認
ステップ3：EC2インスタンスをインポート
  ↓ 確認
...
```

#### 4-3. ドキュメント化

```markdown
# REFACTORING.md

## 実施した変更

### 2025-01-15：EC2インスタンス名の変更

- aws_instance.a → aws_instance.web_server
- 理由：命名規則の統一
- 影響：なし

### 2025-02-01：ネットワークモジュールへの移行

- aws_vpc.main → module.network.aws_vpc.main
- aws*subnet.* → module.network.aws*subnet.*
- 理由：コードの整理
- 影響：なし

## 既知の問題

なし
```

#### 4-4. チーム内での共有

```
重要な変更の前に：

1. チームに通知
   - Slackなどで事前に周知
   - 影響範囲を説明

2. レビューを受ける
   - Pull Requestを作成
   - importやmovedの妥当性を確認

3. 段階的なロールアウト
   - まず開発環境で実施
   - 問題なければステージング
   - 最後に本番環境
```

## トラブルシューティング

### エラー1：インポート時の設定不一致

```
Error: Conflicting configuration

The following settings in the resource configuration conflict with the
imported resource:
  - instance_type: "t4g.medium" (configuration) vs "t4g.small" (actual)
```

**解決方法：**

```hcl
# ステップ1：実際の設定を確認
aws ec2 describe-instances --instance-ids i-xxx

# ステップ2：resourceブロックを修正
resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"  # 実際の値に合わせる
}

# ステップ3：再度plan
terraform plan
```

### エラー2：movedブロックのアドレスミス

```
Error: Invalid moved block

The "from" and "to" addresses must refer to valid resource or module
instances within this module.
```

**解決方法：**

```hcl
# 悪い例：存在しないアドレス
moved {
  from = aws_instance.old_name  # このリソースは存在しない
  to   = aws_instance.new_name
}

# 良い例：正しいアドレスを指定
moved {
  from = aws_instance.web  # terraform state listで確認
  to   = aws_instance.web_server
}
```

### エラー3：循環参照

```
Error: Cycle in moved blocks

The moved blocks form a cycle:
  aws_instance.a -> aws_instance.b
  aws_instance.b -> aws_instance.a
```

**解決方法：**

```hcl
# 悪い例：循環参照
moved {
  from = aws_instance.a
  to   = aws_instance.b
}

moved {
  from = aws_instance.b
  to   = aws_instance.a
}

# 良い例：一方向の移動
moved {
  from = aws_instance.a
  to   = aws_instance.b
}
```

## まとめ

### importブロックの使用

```
いつ使う：
- 既存のAWSリソースをTerraformで管理したい
- 他のツールで作成したリソースを引き継ぐ
- マニュアル作成したリソースをコード化

重要なポイント：
1. リソース設定を正確に記述
2. No changesになるまで調整
3. 段階的にインポート
4. importブロックは履歴として保持
```

### importとdataの違い

```
import：管理下に置く
- リソースを変更・削除できる
- terraform.tfstateに記録
- 自チームのリソース

data：参照するだけ
- リソースは変更できない
- 読み取り専用
- 他チームのリソース
```

### movedブロックの使用

```
いつ使う：
- リソース名を変更したい
- モジュールに整理したい
- countからfor_eachに移行したい

重要なポイント：
1. movedブロックは削除しない（履歴として保持）
2. 段階的な移動を記録
3. planで必ず確認（削除・作成がないこと）
4. コメントで理由を記録
```

### movedを使わない場合の危険性

```
危険な動作：
- リソースが削除される
- 新しいリソースが作成される
- サービス停止
- データ損失

必ずmovedブロックを使う場面：
- リソース名の変更
- countとfor_eachの切り替え
- モジュール構造の変更
```

### ベストプラクティス

```
1. バックアップを取る
   - 状態ファイル
   - コードのコミット

2. 段階的に実施
   - 一度に大量の変更をしない
   - 確認しながら進める

3. ドキュメント化
   - 変更履歴を記録
   - 理由を明記

4. チームで共有
   - 事前に通知
   - レビューを受ける
   - 段階的なロールアウト
```
