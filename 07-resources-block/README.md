# resourceブロック詳細解説

## この章で学ぶこと

resourceブロックは、Terraformで実際のインフラストラクチャを作成・管理するための最も重要なブロックです。VPCやEC2インスタンス、データベースなど、あらゆるAWSリソースはresourceブロックで定義します。

## resourceブロックとは

### 役割と目的

**例えて言うなら：** 建築設計図における個別の部屋や設備の設計

- 各resourceブロック：「リビングルーム」「キッチン」「バスルーム」の詳細設計
- Terraform：設計図通りに実際の部屋を建設する建築業者

### resourceブロックの基本構文

```hcl
resource "<RESOURCE_TYPE>" "<LOCAL_NAME>" {
  # 設定パラメータ
  <ARGUMENT_NAME> = <VALUE>
<ARGUMENT_NAME> = <VALUE>

# タグ設定（オプション）
tags = {
Name = "リソース名"
}
}
```

### 構文要素の説明

| 要素            | 説明           | 例                      |
| --------------- | -------------- | ----------------------- |
| `RESOURCE_TYPE` | リソースの種類 | `aws_vpc`, `aws_subnet` |
| `LOCAL_NAME`    | ローカル参照名 | `main`, `public`        |
| `ARGUMENT_NAME` | 設定パラメータ | `cidr_block`, `vpc_id`  |

## プロバイダードキュメントの読み方

リソースを作成する際は、必ずプロバイダーの公式ドキュメントを参照します。ここでは、VPCリソースを例に、ドキュメントの読み方を学びます。

### ドキュメントの基本構造

AWS プロバイダーのドキュメントは以下の構造になっています：

```
1. Resource: aws_vpc         ← リソース名
2. 概要説明                  ← リソースの役割
3. Example Usage            ← 使用例
4. Argument Reference       ← 設定可能なパラメータ
5. Attribute Reference      ← 参照可能な属性
6. Import                   ← 既存リソースの取り込み方法
```

### Example Usage（使用例）の読み方

ドキュメントの使用例は、そのリソースの基本的な使い方を示しています：

#### 基本的な例

```hcl
# ドキュメントの例
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
```

**説明：**

- `aws_vpc`：VPC（仮想ネットワーク）を作成
- `"main"`：ローカル名（設定ファイル内での参照名）
- `cidr_block`：必須パラメータ（ネットワークアドレス範囲）

#### タグ付きの例

```hcl
# ドキュメントの例
resource "aws_vpc" "main" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"

  tags = {
    Name = "main"
  }
}
```

**追加要素の説明：**

- `instance_tenancy`：オプションパラメータ（インスタンスの配置方法）
- `tags`：リソースの識別用ラベル

### Argument Reference（引数リファレンス）の読み方

この部分には、リソースで使用できる全てのパラメータが記載されています。

#### 必須パラメータとオプションパラメータの見分け方

```hcl
# ドキュメントから抜粋
cidr_block - (Optional) The IPv4 CIDR block for the VPC
enable_dns_support - (Optional) A boolean flag to enable/disable DNS support
```

**表記の読み方：**

- `(Optional)`：オプションパラメータ（省略可能）
- `(Required)`：必須パラメータ（省略不可）※明記されていない場合もあり

#### 重要なパラメータの理解

VPCの場合、以下が重要なパラメータです：

```hcl
resource "aws_vpc" "main" {
  # 必須ではないが、実質的に必要
  cidr_block = "10.0.0.0/16"

  # DNS機能（重要：有効にしないとサービス間通信が困難）
  enable_dns_support   = true  # DNS解決を有効化
  enable_dns_hostnames = true  # DNSホスト名の割り当てを有効化

  # インスタンスの配置方法（通常はデフォルトでOK）
  instance_tenancy = "default"  # 共有ハードウェア
}
```

### パラメータの詳細説明

#### 1. cidr_block（CIDRブロック）

```hcl
cidr_block = "10.0.0.0/16"
```

**意味：**

- VPCで使用するIPアドレスの範囲を指定
- `10.0.0.0/16`：10.0.0.0から10.0.255.255まで（65,536個のアドレス）

**実際のプロジェクトでは：**

```hcl
cidr_block = var.vpc_cidr  # 変数を使用して柔軟に設定
```

#### 2. enable_dns_support（DNS解決の有効化）

```hcl
enable_dns_support = true
```

**意味：**

- VPC内でのDNS解決機能を有効化
- `true`：DNSによる名前解決が可能
- デフォルトは `true`

#### 3. enable_dns_hostnames（DNSホスト名の有効化）

```hcl
enable_dns_hostnames = true
```

**意味：**

- EC2インスタンスにDNSホスト名を割り当て
- `true`：パブリックIPに対応するDNS名が付与される
- デフォルトは `false`

**重要：** この設定がないと、サービス間の通信で問題が発生する可能性があります。

#### 4. instance_tenancy（インスタンス配置）

```hcl
instance_tenancy = "default"  # または "dedicated"
```

**選択肢：**

- `"default"`：共有ハードウェア（通常はこちら）
- `"dedicated"`：専用ハードウェア（高コスト：時間あたり$2＋インスタンス使用料）

### Attribute Reference（属性リファレンス）の読み方

この部分には、リソース作成後に参照できる情報が記載されています。

#### 主要な属性

```hcl
# 作成されたVPCの情報を他のリソースで参照可能
resource "aws_subnet" "example" {
  vpc_id = aws_vpc.main.id          # VPCのID
  # その他の設定...
}

# 出力でVPC情報を表示
output "vpc_info" {
  value = {
    id         = aws_vpc.main.id                    # VPCのID
    arn        = aws_vpc.main.arn                   # AWSリソース番号
    cidr_block = aws_vpc.main.cidr_block            # CIDRブロック
    owner_id   = aws_vpc.main.owner_id              # AWSアカウントID
  }
}
```

#### 重要な属性の説明

| 属性名                | 説明                   | 使用例                 |
| --------------------- | ---------------------- | ---------------------- |
| `id`                  | VPCのID                | 他のリソースでの参照   |
| `arn`                 | AWS リソース番号       | IAMポリシーでの指定    |
| `cidr_block`          | CIDRブロック           | ネットワーク設計の確認 |
| `main_route_table_id` | メインルートテーブルID | ルート設定の参照       |

### 現在のコードとの対照

あなたの現在のVPCリソース：

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  # DNS機能の有効化（ドキュメント推奨設定）
  enable_dns_hostnames = true
  enable_dns_support   = true

  # タグ設定（ドキュメントの例を参考）
  tags = {
    Name = "${var.project_name}-vpc"
  }
}
```

**ドキュメントとの比較：**

| 設定項目               | ドキュメント例            | あなたのコード                     | 説明                   |
| ---------------------- | ------------------------- | ---------------------------------- | ---------------------- |
| `cidr_block`           | `"10.0.0.0/16"`           | `var.vpc_cidr`                     | 変数使用でより柔軟     |
| `enable_dns_support`   | 省略（デフォルト`true`）  | `true`                             | 明示的に設定（推奨）   |
| `enable_dns_hostnames` | 省略（デフォルト`false`） | `true`                             | 明示的に有効化（重要） |
| `tags`                 | `Name = "main"`           | `Name = "${var.project_name}-vpc"` | 動的な名前生成         |

### ドキュメント活用のベストプラクティス

#### 1. 必要なパラメータの特定

```hcl
# ステップ1：基本的な設定から開始
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"  # 最低限必要
}

# ステップ2：重要なオプションを追加
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  # DNS機能（実質必須）
  enable_dns_support   = true
  enable_dns_hostnames = true
}

# ステップ3：管理しやすくするためのタグ追加
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "my-vpc"
  }
}
```

#### 2. デフォルト値の確認

```hcl
# ドキュメントでデフォルト値を確認
enable_dns_support = true    # デフォルトがtrueなので省略可能
enable_dns_hostnames = true  # デフォルトがfalseなので明示的に設定
```

#### 3. 複雑な例の理解

ドキュメントには高度な使用例も含まれています：

```hcl
# IPAM使用例（上級者向け）
resource "aws_vpc" "test" {
  ipv4_ipam_pool_id   = aws_vpc_ipam_pool.test.id
  ipv4_netmask_length = 28
  depends_on = [
    aws_vpc_ipam_pool_cidr.test
  ]
}
```

**初心者の場合：** このような複雑な例はスキップし、基本的な使用例に集中してください。

### 他のリソースのドキュメント読み方

同じパターンで他のAWSリソースも理解できます：

#### aws_subnet の場合

```hcl
# ドキュメントの基本例
resource "aws_subnet" "example" {
  vpc_id     = aws_vpc.main.id     # 必須：所属するVPC
  cidr_block = "10.0.1.0/24"       # 必須：サブネットのアドレス範囲
}

# よく使用されるオプション
resource "aws_subnet" "example" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "ap-northeast-1a"       # 配置するAZ

  # パブリックIPの自動割り当て
  map_public_ip_on_launch = true

  tags = {
    Name = "example-subnet"
  }
}
```

## terraform plan出力の読み方

`terraform plan`コマンドは、実際にインフラを変更する前に、Terraformが何を行うかを表示します。この出力を正しく理解することは、安全な運用のために非常に重要です。

### planコマンドの基本

```bash
# 実行計画を確認
terraform plan

# 実行計画を保存（推奨）
terraform plan -out=tfplan
```

### 出力の基本構造

```
Terraform will perform the following actions:

  # リソースの変更内容

Plan: X to add, Y to change, Z to destroy.
```

### 変更記号の意味

| 記号  | 意味     | 説明                           |
| ----- | -------- | ------------------------------ |
| `+`   | 作成     | 新しいリソースを作成           |
| `-`   | 削除     | 既存のリソースを削除           |
| `~`   | 変更     | 既存のリソースを変更           |
| `-/+` | 置換     | リソースを削除して再作成       |
| `<=`  | 読み取り | データソースから情報を読み取り |

### 1. 新規リソースの作成

#### 出力例：VPCを新規作成

```
Terraform will perform the following actions:

  # aws_vpc.main will be created
  + resource "aws_vpc" "main" {
      + arn                              = (known after apply)
      + cidr_block                       = "10.88.0.0/16"
      + default_network_acl_id           = (known after apply)
      + default_route_table_id           = (known after apply)
      + default_security_group_id        = (known after apply)
      + dhcp_options_id                  = (known after apply)
      + enable_dns_hostnames             = true
      + enable_dns_support               = true
      + id                               = (known after apply)
      + instance_tenancy                 = "default"
      + ipv6_association_id              = (known after apply)
      + ipv6_cidr_block                  = (known after apply)
      + main_route_table_id              = (known after apply)
      + owner_id                         = (known after apply)
      + tags                             = {
          + "Name" = "terraform-practice-vpc"
        }
      + tags_all                         = {
          + "Environment" = "development"
          + "ManagedBy"   = "Terraform"
          + "Name"        = "terraform-practice-vpc"
          + "Project"     = "terraform-practice"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

#### 読み方のポイント

```
# aws_vpc.main will be created
  + resource "aws_vpc" "main" {
  ↑                     ↑
  作成される            リソース名
```

**重要な項目：**

1. **設定値（左側に値がある）**

```
+ cidr_block = "10.88.0.0/16"
↑ 設定ファイルで指定した値
```

2. **作成後に決まる値（known after apply）**

```
+ id = (known after apply)
↑ AWSが実際に作成した後に決まる値
```

3. **自動適用されるタグ（tags_all）**

```
+ tags_all = {
    + "Environment" = "development"  # default_tagsから
    + "ManagedBy"   = "Terraform"    # default_tagsから
    + "Name"        = "terraform-practice-vpc"
    + "Project"     = "terraform-practice"  # default_tagsから
  }
```

### 2. リソースの変更

#### 出力例：VPCのタグを変更

```
Terraform will perform the following actions:

  # aws_vpc.main will be updated in-place
  ~ resource "aws_vpc" "main" {
        id                               = "vpc-0123456789abcdef0"
      ~ tags                             = {
          ~ "Name" = "old-vpc-name" -> "new-vpc-name"
        }
      ~ tags_all                         = {
          ~ "Name" = "old-vpc-name" -> "new-vpc-name"
            # (3 unchanged elements hidden)
        }
        # (15 unchanged attributes hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

#### 読み方のポイント

```
# aws_vpc.main will be updated in-place
  ~ resource "aws_vpc" "main" {
  ↑                     ↑
  変更される            リソース名
                        (既存のリソースを修正)
```

**変更の種類：**

1. **値の変更**

```
~ tags = {
    ~ "Name" = "old-vpc-name" -> "new-vpc-name"
  }
  ↑          ↑                   ↑
  変更       現在の値             新しい値
```

2. **変更されない属性**

```
# (15 unchanged attributes hidden)
↑ 変更されない15個の属性は表示を省略
```

### 3. リソースの削除

#### 出力例：サブネットを削除

```
Terraform will perform the following actions:

  # aws_subnet.unnecessary will be destroyed
  - resource "aws_subnet" "unnecessary" {
      - arn                             = "arn:aws:ec2:ap-northeast-1:123456789012:subnet/subnet-0123456789abcdef0" -> null
      - availability_zone               = "ap-northeast-1a" -> null
      - cidr_block                      = "10.88.99.0/24" -> null
      - id                              = "subnet-0123456789abcdef0" -> null
      - map_public_ip_on_launch         = false -> null
      - owner_id                        = "123456789012" -> null
      - tags                            = {
          - "Name" = "unnecessary-subnet"
        } -> null
      - tags_all                        = {
          - "Environment" = "development"
          - "ManagedBy"   = "Terraform"
          - "Name"        = "unnecessary-subnet"
          - "Project"     = "terraform-practice"
        } -> null
      - vpc_id                          = "vpc-0123456789abcdef0" -> null
    }

Plan: 0 to add, 0 to change, 1 to destroy.
```

#### 読み方のポイント

```
# aws_subnet.unnecessary will be destroyed
  - resource "aws_subnet" "unnecessary" {
  ↑                      ↑
  削除される              リソース名
```

**重要な確認事項：**

1. **削除されるリソースの確認**

```
- id = "subnet-0123456789abcdef0" -> null
  ↑    ↑                            ↑
  削除  現在のID                     削除後はnull（存在しない）
```

2. **依存関係の警告**

```
# 他のリソースがこのサブネットを参照している場合、
# Terraformが自動的に依存関係を解決して削除順序を決定
```

### 4. リソースの置換（削除して再作成）

#### 出力例：VPCのCIDRブロックを変更

```
Terraform will perform the following actions:

  # aws_vpc.main must be replaced
-/+ resource "aws_vpc" "main" {
      ~ arn                              = "arn:aws:ec2:ap-northeast-1:123456789012:vpc/vpc-0123456789abcdef0" -> (known after apply)
      ~ cidr_block                       = "10.88.0.0/16" -> "10.99.0.0/16" # forces replacement
      ~ default_network_acl_id           = "acl-0123456789abcdef0" -> (known after apply)
      ~ default_route_table_id           = "rtb-0123456789abcdef0" -> (known after apply)
      ~ default_security_group_id        = "sg-0123456789abcdef0" -> (known after apply)
      ~ dhcp_options_id                  = "dopt-0123456789abcdef0" -> (known after apply)
        enable_dns_hostnames             = true
        enable_dns_support               = true
      ~ id                               = "vpc-0123456789abcdef0" -> (known after apply)
      ~ main_route_table_id              = "rtb-0123456789abcdef0" -> (known after apply)
      ~ owner_id                         = "123456789012" -> (known after apply)
        tags                             = {
            "Name" = "terraform-practice-vpc"
        }
      ~ tags_all                         = {
            "Environment" = "development"
            "ManagedBy"   = "Terraform"
            "Name"        = "terraform-practice-vpc"
            "Project"     = "terraform-practice"
        } -> (known after apply)
        # (3 unchanged attributes hidden)
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```

#### 読み方のポイント

```
# aws_vpc.main must be replaced
-/+ resource "aws_vpc" "main" {
 ↑                      ↑
 削除して再作成          リソース名
```

**置換が必要な理由：**

```
~ cidr_block = "10.88.0.0/16" -> "10.99.0.0/16" # forces replacement
                                                  ↑
                                           置換を強制する変更
```

**重要な警告：**

- 置換は「削除→作成」の順序で実行されます
- 既存のリソースIDは失われます
- 依存している他のリソースも影響を受ける可能性があります

### 5. 複数の変更が含まれる例

#### 出力例：VPC、サブネット、IGWの変更

```
Terraform will perform the following actions:

  # aws_internet_gateway.main will be created
  + resource "aws_internet_gateway" "main" {
      + arn      = (known after apply)
      + id       = (known after apply)
      + owner_id = (known after apply)
      + tags     = {
          + "Name" = "terraform-practice-igw"
        }
      + tags_all = {
          + "Environment" = "development"
          + "ManagedBy"   = "Terraform"
          + "Name"        = "terraform-practice-igw"
          + "Project"     = "terraform-practice"
        }
      + vpc_id   = (known after apply)
    }

  # aws_subnet.public will be created
  + resource "aws_subnet" "public" {
      + arn                             = (known after apply)
      + cidr_block                      = "10.88.1.0/24"
      + id                              = (known after apply)
      + map_public_ip_on_launch         = true
      + vpc_id                          = (known after apply)
      # ... 省略 ...
    }

  # aws_vpc.main will be updated in-place
  ~ resource "aws_vpc" "main" {
        id                     = "vpc-0123456789abcdef0"
      ~ tags                   = {
          ~ "Name" = "old-name" -> "new-name"
        }
        # (15 unchanged attributes hidden)
    }

Plan: 2 to add, 1 to change, 0 to destroy.
```

#### 読み方のポイント

```
Plan: 2 to add, 1 to change, 0 to destroy.
      ↑        ↑             ↑
      新規作成  変更           削除
```

### planの確認時にチェックすべきポイント

#### 1. 予期しない削除がないか

```
# 削除予定のリソースを必ず確認
Plan: X to add, Y to change, Z to destroy.
                            ↑
                      この数字が0でない場合は要注意
```

#### 2. 置換（-/+）が発生していないか

```
# 置換は影響が大きいため、必ず理由を確認
-/+ resource "aws_vpc" "main" {
    ↑
    この記号がある場合は要注意
```

#### 3. forces replacementの確認

```
~ cidr_block = "10.88.0.0/16" -> "10.99.0.0/16" # forces replacement
                                                  ↑
                                          置換を強制する変更
```

#### 4. 依存関係の影響範囲

```
# VPCを置換すると、依存している全てのリソースも影響を受ける
aws_vpc.main (置換)
  ↓
aws_subnet.public (置換)
  ↓
aws_instance.web (置換)
```

### plan出力の保存と確認

#### 実行計画の保存

```bash
# 計画を保存
terraform plan -out=tfplan

# 保存した計画の詳細を確認
terraform show tfplan

# 保存した計画を実行
terraform apply tfplan
```

#### チーム作業での活用

```bash
# 計画をファイルに保存して共有
terraform plan -out=tfplan > plan_output.txt

# チームメンバーが確認後、承認を得てから実行
terraform apply tfplan
```

### よくあるplan出力パターン

#### パターン1：初回実行（全て新規作成）

```
Plan: 10 to add, 0 to change, 0 to destroy.
      ↑
      全てのリソースが新規作成
```

#### パターン2：変更なし

```
No changes. Your infrastructure matches the configuration.
```

#### パターン3：タグのみ変更

```
Plan: 0 to add, 5 to change, 0 to destroy.
                ↑
                タグなどの軽微な変更のみ
```

#### パターン4：危険な変更

```
Plan: 5 to add, 0 to change, 5 to destroy.
                            ↑
                      削除が含まれる場合は要注意
```

## 現在のコードでのresourceブロック例

### VPCリソースの詳細解説

```hcl
resource "aws_vpc" "main" {
  # 必須パラメータ：ネットワークアドレス範囲
  cidr_block = var.vpc_cidr

  # DNS機能の設定（重要）
  enable_dns_hostnames = true
  enable_dns_support   = true

  # リソース識別用のタグ
  tags = {
    Name = "${var.project_name}-vpc"
  }
}
```

### 構成要素の詳細解説

#### 1. リソースタイプ：`aws_vpc`

```hcl
resource "aws_vpc" "main" {
#         ↑
#    リソースタイプ
#    - プロバイダー名（aws）+ リソース種類（vpc）
#    - AWSのVirtual Private Cloudを作成
```

#### 2. ローカル名：`main`

```hcl
resource "aws_vpc" "main" {
#                   ↑
#              ローカル名
#    - この設定ファイル内での参照名
#    - 他のリソースから aws_vpc.main.id で参照可能
```

#### 3. 設定パラメータ

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr           # 必須パラメータ
  enable_dns_hostnames = true         # オプションパラメータ
  enable_dns_support   = true         # オプションパラメータ
}
```

## resourceブロックの種類

### AWSリソースの例

あなたのコードに含まれているAWSリソース：

#### 1. VPC（仮想ネットワーク）

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.88.0.0/16"  # ネットワークアドレス範囲

  # DNS機能を有効化
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "terraform-practice-vpc"
  }
}
```

#### 2. インターネットゲートウェイ（外部接続）

```hcl
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id  # どのVPCに接続するか

  tags = {
    Name = "terraform-practice-igw"
  }
}
```

#### 3. サブネット（ネットワークの区画）

```hcl
resource "aws_subnet" "example" {
  vpc_id            = aws_vpc.main.id          # 所属するVPC
  cidr_block        = "10.88.1.0/24"          # サブネットのアドレス範囲
  availability_zone = "ap-northeast-1a"       # 配置するAZ

  # パブリックIPを自動割り当て
  map_public_ip_on_launch = true

  tags = {
    Name = "terraform-practice-subnet"
    Type = "Public"
  }
}
```

#### 4. ルートテーブル（通信経路の設定）

```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  # インターネット向けの通信をIGWに送信
  route {
    cidr_block = "0.0.0.0/0"                  # 全てのインターネット通信
    gateway_id = aws_internet_gateway.main.id # IGW経由で送信
  }

  tags = {
    Name = "terraform-practice-public-rt"
  }
}
```

## リソース間の参照方法

### 基本的な参照構文

```hcl
<RESOURCE_TYPE>.<LOCAL_NAME>.<ATTRIBUTE>
```

### 実際の参照例

#### 他のリソースのIDを参照

```hcl
# VPCのIDを使用してサブネットを作成
resource "aws_subnet" "example" {
  vpc_id = aws_vpc.main.id  # VPCリソースのid属性を参照
  # その他の設定...
}
```

#### 複数の属性を参照

```hcl
# IGWのIDを使用してルートを設定
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id  # VPCのID参照

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id  # IGWのID参照
  }
}
```

### 参照可能な属性の確認方法

各リソースが持つ属性は、公式ドキュメントで確認できます：

```hcl
# aws_vpc リソースの主な属性例
aws_vpc.main.id           # VPCのID
aws_vpc.main.cidr_block   # CIDRブロック
aws_vpc.main.arn          # AWS リソース番号

# aws_subnet リソースの主な属性例
aws_subnet.example.id     # サブネットのID
aws_subnet.example.arn    # AWS リソース番号
```

## タグの設定方法

### 個別タグの設定

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  # リソース固有のタグ
  tags = {
    Name = "${var.project_name}-vpc"
    Type = "Network"
    Owner = "DevTeam"
  }
}
```

### default_tagsとの組み合わせ

あなたのコードでは、providerレベルでdefault_tagsが設定されているため：

```hcl
# providerで自動適用されるタグ
# Environment = "development"
# Project     = "terraform-practice"
# ManagedBy   = "Terraform"

# リソースで追加するタグ
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  # リソース固有のタグのみ設定
  tags = {
    Name = "${var.project_name}-vpc"  # 個別の識別名のみ
  }
}
```

**結果として付与されるタグ：**

- Name = "terraform-practice-vpc"
- Environment = "development" （自動適用）
- Project = "terraform-practice" （自動適用）
- ManagedBy = "Terraform" （自動適用）

## リソースの作成手順

### ステップ1：resourceブロックの記述

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.88.0.0/16"

  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "my-vpc"
  }
}
```

### ステップ2：設定の検証

```bash
terraform validate
```

### ステップ3：実行計画の確認

```bash
terraform plan
```

**出力例：**

```
Terraform will perform the following actions:

  # aws_vpc.main will be created
  + resource "aws_vpc" "main" {
      + cidr_block           = "10.88.0.0/16"
      + enable_dns_hostnames = true
      + enable_dns_support   = true
      + id                   = (known after apply)
      + tags                 = {
          + "Name" = "my-vpc"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

### ステップ4：リソースの作成

```bash
terraform apply
```

## リソースの削除方法

### 方法1：設定ファイルからリソースブロックを削除

**最も一般的で安全な方法**

**手順：**

1. 削除したいresourceブロックを設定ファイルから除去
2. そのリソースを参照している他の箇所も修正
3. `terraform validate` で構文エラーがないか確認
4. `terraform plan` で削除予定の内容を確認
5. `terraform apply` で実際に削除

**例：**

```hcl
# この resourceブロック を削除
# resource "aws_subnet" "unnecessary" {
#   vpc_id     = aws_vpc.main.id
#   cidr_block = "10.88.99.0/24"
# }
```

### 方法2：terraform destroyコマンド

#### 特定のリソースのみ削除

```bash
# 特定のリソースのみ削除
terraform destroy -target=aws_subnet.unnecessary

# 削除前に必ず確認
terraform plan -destroy -target=aws_subnet.unnecessary
```

#### 全てのリソースを削除

```bash
# 全リソース削除（注意！元に戻せません）
terraform destroy

# 削除前に必ず確認
terraform plan -destroy
```

### 削除時の重要な注意点

#### 1. 依存関係の自動処理

Terraformは依存関係を自動的に解決して、正しい順序で削除します：

```hcl
# 依存関係の例
VPC → Subnet → EC2インスタンス

# 削除順序（自動的に決定される）
EC2インスタンス → Subnet → VPC
```

#### 2. 削除できないリソース

一部のリソースは、他のリソースに依存されている間は削除できません：

```bash
Error: Error deleting VPC: DependencyViolation:
The vpc 'vpc-xxxxx' has dependencies and cannot be deleted.
```

**解決方法：** 依存しているリソースを先に削除する

#### 3. 削除の取り消しは不可能

**重要：** `terraform destroy` や `terraform apply`（削除を含む）は取り消しできません。実行前に必ず計画を確認してください。

## 現在のコードの構造理解

### リソース間の依存関係

あなたのコードでは、以下の依存関係が存在します：

```hcl
aws_vpc.main
├── aws_internet_gateway.main  # VPCのIDを参照
├── aws_subnet.public          # VPCのIDを参照
├── aws_subnet.private         # VPCのIDを参照
└── aws_route_table.public     # VPCのIDとIGWのIDを参照
```

### リソースの目的

1. **aws_vpc.main**：基盤となる仮想ネットワーク
2. **aws_internet_gateway.main**：インターネット接続用ゲートウェイ
3. **aws_subnet.public/private**：ネットワーク内の区画
4. **aws_route_table.public**：通信経路の設定

## resourceブロックのベストプラクティス

### 1. 意味のあるローカル名を使用

```hcl
# 良い例：目的が明確
resource "aws_vpc" "main" { }
resource "aws_subnet" "public" { }
resource "aws_subnet" "private" { }

# 悪い例：意味が不明
resource "aws_vpc" "vpc1" { }
resource "aws_subnet" "s1" { }
```

### 2. 適切なコメントを追加

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  # DNS機能の有効化（重要：これがないとサービス間通信が困難）
  enable_dns_hostnames = true
  enable_dns_support   = true
}
```

### 3. 変数を活用した柔軟な設定

```hcl
# 良い例：変数使用
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr  # 環境ごとに変更可能

  tags = {
    Name = "${var.project_name}-vpc"  # プロジェクト名を動的に設定
  }
}
```

### 4. 一貫したタグ付け

```hcl
# default_tagsと組み合わせて使用
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  tags = {
    Name = "${var.project_name}-vpc"  # リソース固有のタグのみ
  }
}
```

## トラブルシューティング

### よくあるエラー1：必須パラメータの不足

```bash
Error: Missing required argument
│ The argument "cidr_block" is required, but no definition was found.
```

**解決方法：**

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"  # 必須パラメータを追加
}
```

### よくあるエラー2：無効な参照

```bash
Error: Reference to undeclared resource
│ A resource "aws_vpc" "nonexistent" has not been declared.
```

**解決方法：**

```hcl
# 正しいリソース名を使用
vpc_id = aws_vpc.main.id  # "main" が正しいローカル名
```

### よくあるエラー3：重複するCIDRブロック

```bash
Error: InvalidVpc.Range
│ The CIDR '10.0.0.0/16' conflicts with another subnet
```

**解決方法：**
重複しないCIDRブロックを設定する

## まとめ

### resourceブロックの重要ポイント

1. **基本構文の理解**

- `resource "リソースタイプ" "ローカル名" { }`
- 必須パラメータとオプションパラメータの区別

2. **プロバイダードキュメントの活用**

- Example Usage で基本的な使い方を理解
- Argument Reference で設定可能なパラメータを確認
- Attribute Reference で参照可能な属性を確認

3. **terraform plan出力の理解**

- 変更記号（+, -, ~, -/+）の意味
- 予期しない変更の検出
- 置換（forces replacement）の影響範囲確認

4. **リソース参照の方法**

- `リソースタイプ.ローカル名.属性名`
- 依存関係の自動解決

5. **削除方法の理解**

- 設定ファイルからの削除（推奨）
- terraform destroyコマンドの慎重な使用

6. **安全な運用**

- 実行前の計画確認
- 段階的な変更
- 適切なバックアップ
