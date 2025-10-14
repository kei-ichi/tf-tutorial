# resourceブロック実践演習：EC2インスタンスとセキュリティグループの作成

## 演習の目的

この演習では、学習したresourceブロックの知識を実際に活用して、以下のリソースを作成します：

- セキュリティグループ（SSH接続用）
- SSHキーペア（TLSプロバイダー使用）
- 秘密鍵の保存（Parameter Store）
- EC2インスタンス（ランダムな名前付き）

**重要：** この演習は前回のプロバイダー設定演習の続きです。AWS、Random、TLSプロバイダーが既に設定されている状態から開始します。

## 事前準備

現在のプロジェクトフォルダ `terraform-aws-practice` で作業を続けます。

### 前提条件の確認

以下のコマンドで、3つのプロバイダーが設定されていることを確認してください：

```bash
terraform providers
```

**期待される出力：**

```
Providers required by configuration:
.
├── provider[registry.terraform.io/hashicorp/aws] ~> 6.0
├── provider[registry.terraform.io/hashicorp/random] 3.7.2
└── provider[registry.terraform.io/hashicorp/tls] ~> 4.1
```

## 事前設定：データソースの追加

**データソースとは何か：**

- `resource`：新しいものを「作成」する
- `data`：既存の情報を「取得」する

**例えて言うなら：**

- `resource`：新しい家を建てる（作成）
- `data`：既存の地図から住所を調べる（取得）

**今回の使用目的：**
EC2インスタンスを作成するには「AMI ID」が必要です。AMI IDはリージョンやOSバージョンによって異なるため、毎回最新のAMI IDを自動的に取得します。

### データソースの追加手順

`main.tf`ファイルの**データソースセクション**（既存の`data "aws_availability_zones"`の後）に、以下を追加してください：

```hcl
# =============================================================================
# データソース：Amazon Linux 2023 AMI（ARM64）を取得
# 最新のAMI IDを自動的に取得するため、手動でIDを調べる必要がない
# =============================================================================
data "aws_ami" "amazon_linux_2023" {
  most_recent = true  # 最新のAMIを取得
  owners      = ["amazon"]  # Amazonが提供する公式AMIのみ

  # AMI名でフィルター：Amazon Linux 2023、カーネル6.1、ARM64
  filter {
    name   = "name"
    values = ["al2023-ami-*-kernel-6.1-arm64"]
  }

  # 仮想化タイプでフィルター
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  # アーキテクチャでフィルター
  filter {
    name   = "architecture"
    values = ["arm64"]
  }
}
```

### 動作確認

```bash
# データソースが正しく動作するか確認
terraform plan
```

**期待される出力：**

```
data.aws_ami.amazon_linux_2023: Reading...
data.aws_ami.amazon_linux_2023: Read complete after 1s [id=ami-xxxxx]

No changes. Your infrastructure matches the configuration.
```

**データソースの参照方法：**

- AMI ID：`data.aws_ami.amazon_linux_2023.id`
- AMI名：`data.aws_ami.amazon_linux_2023.name`

これで準備完了です。次のタスクからresourceブロックの作成に進みます。

## 演習の流れ

この演習では、以下の順序でリソースを追加していきます：

```
1. ランダム文字列生成（インスタンス名用）
   ↓
2. TLS秘密鍵生成（SSH接続用、ED25519アルゴリズム）
   ↓
3. 秘密鍵をParameter Storeに保存
   ↓
4. AWS Key Pair作成（SSH鍵登録）
   ↓
5. セキュリティグループ作成（SSH許可）
   ↓
6. EC2インスタンス作成
```

## タスク1：ランダム文字列リソースの作成

### 背景知識：プロバイダードキュメントの読み方

**ドキュメントの確認手順：**

1. https://registry.terraform.io/ にアクセス
2. 「random」で検索
3. 「random_string」リソースを選択
4. 以下のセクションを確認：

- **Example Usage**：基本的な使用例
- **Argument Reference**：設定可能なパラメータ
- **Attribute Reference**：参照可能な属性

### 実践タスク

`random_string`リソースを作成して、6文字のランダム文字列を生成してください。

**要件：**

- リソースタイプ：`random_string`
- ローカル名：`instance_suffix`（インスタンス名の末尾に使用）
- 長さ：6文字
- 使用文字：小文字（lower）と数字（number）のみ
- 大文字は使用しない（upper = false）
- 特殊文字は使用しない（special = false）

**ヒント（ドキュメントから）：**

```hcl
# ドキュメントの Example Usage から抜粋
resource "random_string" "example" {
  length  = 16        # ← length は必須パラメータ
  special = true      # ← special はオプション（デフォルトtrue）
  upper   = true      # ← upper はオプション（デフォルトtrue）
}

# Attribute Reference から
# result - 生成されたランダム文字列
```

### 質問1：random_stringリソースの理解

**質問1-A：** 作成した`random_string`リソースの結果（ランダム文字列）は、どの属性で参照できますか？

**質問1-B：** このリソースを`terraform apply`で2回実行すると、ランダム文字列は変わりますか？それとも同じままですか？

**質問1-C：** もしランダム文字列を変更したい場合、どうすればよいですか？

### 検証

```bash
# 設定を検証
terraform validate

# 実行計画を確認
terraform plan
```

**期待される出力（plan）：**

```
Terraform will perform the following actions:

  # random_string.instance_suffix will be created
  + resource "random_string" "instance_suffix" {
      + id          = (known after apply)
      + length      = 6
      + lower       = true
      + number      = true
      + result      = (known after apply)
      + special     = false
      + upper       = false
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

### 質問2：terraform plan出力の理解

**質問2-A：** `+ resource "random_string" "instance_suffix"` の `+` 記号は何を意味しますか？

**質問2-B：** `result = (known after apply)` とはどういう意味ですか？

**質問2-C：** この段階で、実際のランダム文字列は生成されていますか？

## タスク2：TLS秘密鍵リソースの作成

### 背景知識：TLSプロバイダーとED25519

**TLSプロバイダーでできること：**

1. 秘密鍵の生成（SSH接続用など）
2. 公開鍵の生成
3. 証明書の作成

**ED25519とは：**

- RSAより新しく、安全で高速な暗号化アルゴリズム
- SSH鍵として推奨される現代的な方式
- より短い鍵長で同等以上のセキュリティを実現

**SSH接続の仕組み：**

```
秘密鍵（あなたが保持）+ 公開鍵（サーバーに登録）= SSH接続可能
```

### プロバイダードキュメントを確認

1. https://registry.terraform.io/providers/hashicorp/tls にアクセス
2. 「tls_private_key」リソースを選択
3. Example UsageでED25519の使用例を確認

### 実践タスク

`tls_private_key`リソースを作成してください。

**要件：**

- リソースタイプ：`tls_private_key`
- ローカル名：`ec2_key`
- アルゴリズム：`"ED25519"`（RSAではなくED25519を使用）

**ヒント（ドキュメントから）：**

```hcl
# Example Usage - ED25519の場合
resource "tls_private_key" "example" {
  algorithm = "ED25519"
}

# Attribute Reference（参照可能な属性）
# private_key_openssh - OpenSSH形式の秘密鍵
# private_key_pem - PEM形式の秘密鍵
# public_key_openssh - OpenSSH形式の公開鍵
```

### 質問3：TLSリソースの理解

**質問3-A：** OpenSSH形式の秘密鍵は、どの属性で参照できますか？

**質問3-B：** OpenSSH形式の公開鍵は、どの属性で参照できますか？

**質問3-C：** ED25519アルゴリズムを使用する場合、RSAのような`rsa_bits`パラメータは必要ですか？

**質問3-D：** このリソースで生成される秘密鍵は、AWSに保存されますか？それともTerraformの状態ファイルに保存されますか？

### 検証

```bash
terraform plan
```

**期待される出力：**

```
  # tls_private_key.ec2_key will be created
  + resource "tls_private_key" "ec2_key" {
      + algorithm                     = "ED25519"
      + private_key_openssh           = (sensitive value)
      + private_key_pem               = (sensitive value)
      + public_key_openssh            = (known after apply)
    }
```

### 質問4：sensitive値の理解

**質問4-A：** plan出力で `(sensitive value)` と表示される理由は何ですか？

**質問4-B：** `(sensitive value)` と `(known after apply)` の違いは何ですか？

## タスク3：秘密鍵をParameter Storeに保存

### 背景知識：AWS Systems Manager Parameter Store

**Parameter Storeとは：**

- AWSが提供する安全な設定値・秘密情報の保存サービス
- 暗号化された状態で保存可能（SecureString）
- IAMで厳密なアクセス制御が可能

**なぜParameter Storeに保存するのか：**

- 秘密鍵をローカルファイルに保存するよりも安全
- チームメンバーが必要な時にAWSから取得可能
- バックアップや監査ログが自動的に管理される

### プロバイダードキュメントを確認

1. AWS Provider ドキュメントで「aws_ssm_parameter」を検索
2. Argument Referenceを確認

### 実践タスク

`aws_ssm_parameter`リソースを作成して、TLS秘密鍵を保存してください。

**要件：**

- リソースタイプ：`aws_ssm_parameter`
- ローカル名：`ec2_private_key`
- パラメータ名：`"/ec2/keypair/${var.project_name}-private-key"`
- タイプ：`"SecureString"`（暗号化保存）
- 値：TLSリソースで生成したOpenSSH形式の秘密鍵
  - ヒント：`tls_private_key.ec2_key.private_key_openssh`
- 説明：`"Private key for EC2 SSH access"`

**ドキュメントのヒント：**

```hcl
# Example Usage
resource "aws_ssm_parameter" "example" {
  name  = "/myapp/database/password"
  type  = "SecureString"
  value = "secret_password"
}

# Argument Reference
# name - (Required) パラメータの名前
# type - (Required) パラメータのタイプ（String, StringList, SecureString）
# value - (Required) パラメータの値
```

### 質問5：Parameter Storeリソースの理解

**質問5-A：** `type = "SecureString"` と `type = "String"` の違いは何ですか？

**質問5-B：** パラメータ名を `/ec2/keypair/` で始める理由は何ですか？

**質問5-C：** このリソースは`tls_private_key`リソースの「後」に作成されますか？Terraformはどうやってそれを判断しますか？

### 検証

```bash
terraform plan
```

**期待される出力：**

```
  # aws_ssm_parameter.ec2_private_key will be created
  + resource "aws_ssm_parameter" "ec2_private_key" {
      + name  = "/ec2/keypair/terraform-practice-private-key"
      + type  = "SecureString"
      + value = (sensitive value)
    }
```

## タスク4：AWS Key Pairリソースの作成

### 背景知識：AWS Key Pairとは

**Key Pairの役割：**

- EC2インスタンスにSSH接続するための鍵をAWSに登録
- 公開鍵のみをAWSに保存（秘密鍵は保存されない）
- EC2起動時にこのKey Pairを指定する

**重要なポイント：**

- TLSリソースで生成した公開鍵を使用
- ハードコーディングではなく、リソース参照を使用

### プロバイダードキュメントを確認

1. AWS Provider ドキュメントで「aws_key_pair」を検索
2. Argument Referenceを確認

### 実践タスク

`aws_key_pair`リソースを作成してください。

**要件：**

- リソースタイプ：`aws_key_pair`
- ローカル名：`ec2_key`
- キーペア名：`"${var.project_name}-key-${random_string.instance_suffix.result}"`
- 公開鍵：TLSリソースで生成した公開鍵（OpenSSH形式）
  - **重要**：ハードコーディングではなく、`tls_private_key.ec2_key.public_key_openssh`を参照

**ドキュメントのヒント：**

```hcl
# Example Usage
resource "aws_key_pair" "deployer" {
  key_name   = "deployer-key"
  public_key = "ssh-rsa AAAAB3NzaC1yc2E..."  # ← これはハードコーディング
}

# Argument Reference
# key_name - (Required) キーペアの名前
# public_key - (Required) 公開鍵の内容
```

### 質問6：リソース参照の理解

**質問6-A：** `random_string.instance_suffix.result` は何を参照していますか？

**質問6-B：** `tls_private_key.ec2_key.public_key_openssh` は何を参照していますか？

**質問6-C：** なぜ公開鍵をハードコーディングせず、リソース参照を使用するのですか？

**質問6-D：** このaws_key_pairリソースは、random_stringとtls_private_keyの「後」に作成されますか？それとも順序は関係ありませんか？

### 検証

```bash
terraform plan
```

**期待される出力：**

```
  # aws_key_pair.ec2_key will be created
  + resource "aws_key_pair" "ec2_key" {
      + key_name   = "terraform-practice-key-abc123"  # ランダム文字列付き
      + public_key = (known after apply)
    }
```

## タスク5：セキュリティグループリソースの作成

### 背景知識：セキュリティグループとは

**例えて言うなら：** マンションの入口にいる警備員

- **インバウンドルール**：外から中への訪問を許可するルール
- **アウトバウンドルール**：中から外への外出を許可するルール

**SSH接続の場合：**

- ポート22（SSHのポート番号）への接続を許可する必要がある

### プロバイダードキュメントを確認

1. AWS Provider ドキュメントで「aws_security_group」を検索
2. ingressとegressの設定方法を確認

### 実践タスク

SSH接続を許可するセキュリティグループを作成してください。

**要件：**

- リソースタイプ：`aws_security_group`
- ローカル名：`ssh_access`
- 名前：`"${var.project_name}-ssh-sg"`
- 説明：`"Allow SSH access"`
- VPC：既存のVPCを参照（`aws_vpc.main.id`）
- インバウンドルール（ingress）：
  - 説明：`"Allow SSH from anywhere"`
  - プロトコル：`"tcp"`
  - ポート範囲：22から22（`from_port = 22`, `to_port = 22`）
  - 許可するIP範囲：全て（`cidr_blocks = ["0.0.0.0/0"]`）
- アウトバウンドルール（egress）：
  - 説明：`"Allow all outbound traffic"`
  - プロトコル：全て（`"-1"`）
  - ポート範囲：0から0
  - 許可するIP範囲：全て（`["0.0.0.0/0"]`）

**ドキュメントのヒント：**

```hcl
# Example Usage
resource "aws_security_group" "example" {
  name        = "example-sg"
  description = "Example security group"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### 質問7：セキュリティグループの理解

**質問7-A：** `from_port = 22`、`to_port = 22` の意味は何ですか？

**質問7-B：** `cidr_blocks = ["0.0.0.0/0"]` は何を意味しますか？これは安全な設定ですか？

**質問7-C：** egressで `protocol = "-1"` の意味は何ですか？

**質問7-D：** このセキュリティグループを削除したい場合、それを使用しているEC2インスタンスがあっても削除できますか？

### 検証

```bash
terraform plan
```

**期待される出力：**

```
  # aws_security_group.ssh_access will be created
  + resource "aws_security_group" "ssh_access" {
      + name        = "terraform-practice-ssh-sg"
      + description = "Allow SSH access"
      + vpc_id      = "vpc-xxxxx"

      + ingress {
          + description = "Allow SSH from anywhere"
          + from_port   = 22
          + to_port     = 22
          + protocol    = "tcp"
          + cidr_blocks = ["0.0.0.0/0"]
        }

      + egress {
          + description = "Allow all outbound traffic"
          + from_port   = 0
          + to_port     = 0
          + protocol    = "-1"
          + cidr_blocks = ["0.0.0.0/0"]
        }
    }
```

## タスク6：EC2インスタンスリソースの作成

### 背景知識：EC2インスタンスとは

**例えて言うなら：** クラウド上の仮想コンピューター

**必要な設定：**

1. どのOSを使うか（AMI）
2. どのサイズのコンピューターか（インスタンスタイプ）
3. どのネットワークに置くか（サブネット）
4. どのセキュリティルールを適用するか（セキュリティグループ）
5. SSH接続用の鍵は何か（Key Pair）

### プロバイダードキュメントを確認

1. AWS Provider ドキュメントで「aws_instance」を検索
2. 必須パラメータとオプションパラメータを確認

### 実践タスク

EC2インスタンスを作成してください。

**要件：**

- リソースタイプ：`aws_instance`
- ローカル名：`web`
- AMI：データソースから取得（`data.aws_ami.amazon_linux_2023.id`）
- インスタンスタイプ：`"t4g.small"`（ARM64アーキテクチャ対応）
- サブネット：最初のパブリックサブネット（`aws_subnet.public[0].id`）
- セキュリティグループ：作成したセキュリティグループ（`[aws_security_group.ssh_access.id]`）
  - 注意：リスト形式で指定
- キーペア名：作成したKey Pair（`aws_key_pair.ec2_key.key_name`）
- タグ：
  - Name：`"${var.project_name}-instance-${random_string.instance_suffix.result}"`

**ドキュメントのヒント：**

```hcl
# Example Usage
resource "aws_instance" "example" {
  ami           = "ami-xxxxx"
  instance_type = "t2.micro"

  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.example.id]
  key_name               = aws_key_pair.deployer.key_name

  tags = {
    Name = "example-instance"
  }
}
```

### 質問8：EC2インスタンスの理解

**質問8-A：** `vpc_security_group_ids`はなぜリスト形式（`[]`）で指定する必要がありますか？

**質問8-B：** `aws_subnet.public[0].id`の`[0]`は何を意味しますか？

**質問8-C：** このEC2インスタンスは、以下のどのリソースに依存していますか？

- A: random_string
- B: tls_private_key
- C: aws_ssm_parameter
- D: aws_key_pair
- E: aws_security_group
- F: data.aws_ami
- G: aws_subnet

### 検証

```bash
terraform plan
```

**期待される出力：**

```
  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami                         = "ami-xxxxx"
      + instance_type               = "t4g.small"
      + key_name                    = "terraform-practice-key-abc123"
      + subnet_id                   = "subnet-xxxxx"
      + vpc_security_group_ids      = [
          + "sg-xxxxx",
        ]
      + tags                        = {
          + "Name" = "terraform-practice-instance-abc123"
        }
    }
```

## タスク7：すべてのリソースを作成

### 実行前の最終確認

```bash
# すべての設定が正しいか確認
terraform validate

# 実行計画を詳細に確認
terraform plan
```

**確認すべきポイント：**

1. **作成されるリソースの数**

```
Plan: X to add, 0 to change, 0 to destroy.
```

- 期待値：6個のリソース
  - random_string（1個）
  - tls_private_key（1個）
  - aws_ssm_parameter（1個）
  - aws_key_pair（1個）
  - aws_security_group（1個）
  - aws_instance（1個）

2. **依存関係の順序**

```
作成順序（Terraformが自動的に決定）:
1. random_string (ランダム文字列)
2. tls_private_key (秘密鍵)
3. aws_ssm_parameter (Parameter Store - tls_private_keyに依存)
4. aws_key_pair (Key Pair - tls_private_key & random_stringに依存)
5. aws_security_group (セキュリティグループ)
6. aws_instance (EC2 - 上記すべてに依存)
```

3. **予期しない変更や削除がないか**

```
0 to change, 0 to destroy  ← これらが0であることを確認
```

### 質問9：terraform plan全体の理解

**質問9-A：** planで表示される作成順序は、コードを書いた順序と同じですか？

**質問9-B：** もし`random_string`を先に作成せずに、`aws_instance`を作成しようとした場合、何が起こりますか？

**質問9-C：** `terraform plan`を実行した時点で、AWS上に何かリソースは作成されていますか？

### リソースの作成

すべての確認が完了したら、実際にリソースを作成します：

```bash
terraform apply
```

**実行中の出力例：**

```
random_string.instance_suffix: Creating...
tls_private_key.ec2_key: Creating...
random_string.instance_suffix: Creation complete after 0s
tls_private_key.ec2_key: Creation complete after 1s
aws_ssm_parameter.ec2_private_key: Creating...
aws_key_pair.ec2_key: Creating...
aws_security_group.ssh_access: Creating...
aws_ssm_parameter.ec2_private_key: Creation complete after 1s
aws_key_pair.ec2_key: Creation complete after 2s
aws_security_group.ssh_access: Creation complete after 3s
aws_instance.web: Creating...
aws_instance.web: Still creating... [10s elapsed]
aws_instance.web: Still creating... [20s elapsed]
aws_instance.web: Creation complete after 25s

Apply complete! Resources: 6 added, 0 changed, 0 destroyed.
```

### 質問10：terraform applyの理解

**質問10-A：** EC2インスタンスの作成が他のリソースより時間がかかる理由は何ですか？

**質問10-B：** 今、`terraform apply`をもう一度実行すると何が起こりますか？

**質問10-C：** Parameter Storeに保存された秘密鍵は、どのように取得できますか？

## タスク8：作成したリソースの確認

### terraform showコマンドで確認

```bash
# 現在の状態をすべて表示
terraform show

# 特定のリソースのみ表示
terraform state show aws_instance.web
```

### 質問11：作成されたリソースの確認

**質問11-A：** `terraform show`で表示される情報と、`terraform plan`で表示された情報の違いは何ですか？

**質問11-B：** EC2インスタンスの`id`属性は、plan時と異なる実際の値になっていますか？

**質問11-C：** tls_private_keyリソースの秘密鍵は、`terraform show`で表示されますか？

### AWSマネジメントコンソールで確認

1. AWSマネジメントコンソールにログイン
2. EC2サービスに移動
3. インスタンスを確認
4. セキュリティグループを確認
5. Key Pairを確認
6. Systems Manager → Parameter Store を確認

### 質問12：実際のAWSリソースとTerraformの関係

**質問12-A：** AWSコンソールで見えるEC2インスタンスと、Terraformで管理しているEC2インスタンスは同じものですか？

**質問12-B：** もしAWSコンソールから手動でこのEC2インスタンスを削除した場合、次回`terraform plan`を実行すると何が起こりますか？

## タスク9：plan出力の詳細理解（in-place update）

### AMI IDの取得

**手順：**

1. AWSマネジメントコンソールにログイン
2. EC2 → AMI カタログ に移動
3. 「クイックスタート」タブを選択
4. 「Ubuntu」を検索
5. **Ubuntu Server 24.04 LTS (HVM), SSD Volume Type, arm64** を見つける
6. AMI IDをコピー（例：`ami-0a1b2c3d4e5f6g7h8`）

### 変更を加えてplanを確認

コピーしたUbuntu AMI IDを使用して、EC2インスタンスのAMIを変更してみましょう：

```hcl
resource "aws_instance" "web" {
  # AMIを変更（コピーしたUbuntu AMI IDに置き換える）
  ami = "ami-あなたがコピーしたUbuntu_ARM64のAMI_ID"

  # 他の設定は変更なし
  instance_type          = "t4g.small"
  subnet_id              = aws_subnet.public[0].id
  vpc_security_group_ids = [aws_security_group.ssh_access.id]
  key_name               = aws_key_pair.ec2_key.key_name

  tags = {
    Name = "${var.project_name}-instance-${random_string.instance_suffix.result}"
  }
}
```

```bash
terraform plan
```

**出力例：**

```
  # aws_instance.web must be replaced
-/+ resource "aws_instance" "web" {
      ~ ami                         = "ami-0xxxxx" -> "ami-0yyyyy" # forces replacement
      ~ id                          = "i-xxxxx" -> (known after apply)
        instance_type               = "t4g.small"
        # ... 他の属性 ...
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```

### 質問13：forces replacementの理解

**質問13-A：** `-/+` 記号は何を意味しますか？

**質問13-B：** `forces replacement` とはどういう意味ですか？

**質問13-C：** この変更を適用すると、何が起こりますか？（順序も含めて）

**質問13-D：** なぜAMIの変更は`forces replacement`になるのですか？

**質問13-E：** どのような変更が`forces replacement`になりやすいですか？

**重要：** この変更は適用せず、元に戻してください：

```hcl
resource "aws_instance" "web" {
  ami = data.aws_ami.amazon_linux_2023.id  # ← 元に戻す
  # ... 他の設定 ...
}
```

## タスク10：リソースの削除

### 1つのリソースのみ削除

まず、EC2インスタンスのみを削除してみましょう：

```bash
# 削除計画を確認
terraform plan -destroy -target=aws_instance.web

# 実際に削除
terraform destroy -target=aws_instance.web
```

### 質問14：ターゲット削除の理解

**質問14-A：** EC2インスタンスを削除すると、セキュリティグループやKey Pairも一緒に削除されますか？

**質問14-B：** もし先にaws_key_pairを削除しようとした場合、どうなりますか？

**質問14-C：** aws_ssm_parameterは、EC2インスタンスが削除されても残りますか？

### すべてのリソースを削除

```bash
# すべてのリソースの削除計画を確認
terraform plan -destroy

# すべてのリソースを削除
terraform destroy
```

**出力例：**

```
Plan: 0 to add, 0 to change, 6 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value:
```

### 質問15：リソース削除の理解

**質問15-A：** `terraform destroy`を実行すると、リソースはどの順序で削除されますか？

**質問15-B：** 削除の順序は、作成の順序と同じですか？逆ですか？

**質問15-C：** `terraform destroy`を実行した後、もう一度`terraform plan`を実行すると何が表示されますか？

**質問15-D：** Parameter Storeに保存した秘密鍵も削除されますか？

## 解答例と解説

### タスク1の解答

**コード例：**

```hcl
resource "random_string" "instance_suffix" {
  length  = 6
  special = false
  upper   = false
  lower   = true
  number  = true
}
```

**質問1-A：** `result`属性（例：`random_string.instance_suffix.result`）

**質問1-B：** 同じまま（Terraformの状態に保存されるため）

**質問1-C：** リソースを削除して再作成（`terraform destroy -target=random_string.instance_suffix`の後、`terraform apply`）

**質問2-A：** 新規作成されるリソース

**質問2-B：** リソース作成後に決まる値（plan時点では未確定）

**質問2-C：** いいえ、`terraform apply`を実行するまで生成されません

### タスク2の解答

**コード例：**

```hcl
resource "tls_private_key" "ec2_key" {
  algorithm = "ED25519"
}
```

**質問3-A：** `tls_private_key.ec2_key.private_key_openssh`

**質問3-B：** `tls_private_key.ec2_key.public_key_openssh`

**質問3-C：** いいえ、ED25519は固定長のため追加パラメータは不要

**質問3-D：** Terraformの状態ファイル（terraform.tfstate）に保存される（AWSには保存されない）

**質問4-A：** 機密情報（秘密鍵など）を画面に表示しないため

**質問4-B：**

- `(sensitive value)`：既に値はあるが、セキュリティのため非表示
- `(known after apply)`：まだ値が決まっていない（作成後に決まる）

### タスク3の解答

**コード例：**

```hcl
resource "aws_ssm_parameter" "ec2_private_key" {
  name        = "/ec2/keypair/${var.project_name}-private-key"
  description = "Private key for EC2 SSH access"
  type        = "SecureString"
  value       = tls_private_key.ec2_key.private_key_openssh
}
```

**質問5-A：**

- `SecureString`：AWS KMSで暗号化して保存（推奨）
- `String`：平文で保存（機密情報には使用しない）

**質問5-B：** パラメータを階層的に整理するため（`/ec2/keypair/`というグループ）

**質問5-C：** 後に作成される。`tls_private_key.ec2_key.private_key_openssh`を参照しているため、Terraformが自動的に依存関係を検出

### タスク4の解答

**コード例：**

```hcl
resource "aws_key_pair" "ec2_key" {
  key_name   = "${var.project_name}-key-${random_string.instance_suffix.result}"
  public_key = tls_private_key.ec2_key.public_key_openssh
}
```

**質問6-A：** random_stringリソースで生成されたランダム文字列（6文字）

**質問6-B：** tls_private_keyリソースで生成されたOpenSSH形式の公開鍵

**質問6-C：**

- リソース間の依存関係を明確にするため
- 鍵が変更された場合、自動的に更新されるため
- コードの再利用性と保守性が向上するため

**質問6-D：** 後に作成される（Terraformが自動的に依存関係を解決）

### タスク5の解答

**コード例：**

```hcl
resource "aws_security_group" "ssh_access" {
  name        = "${var.project_name}-ssh-sg"
  description = "Allow SSH access"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "Allow SSH from anywhere"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project_name}-ssh-sg"
  }
}
```

**質問7-A：** ポート22のみを許可（SSHのデフォルトポート）

**質問7-B：** すべてのIPアドレスからのアクセスを許可。学習環境では問題ないが、本番環境では特定のIPのみに制限すべき

**質問7-C：** すべてのプロトコルを許可

**質問7-D：** いいえ、使用中のセキュリティグループは削除できません。先にEC2インスタンスを削除する必要があります

### タスク6の解答

**コード例：**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2023.id
  instance_type = "t4g.small"

  subnet_id              = aws_subnet.public[0].id
  vpc_security_group_ids = [aws_security_group.ssh_access.id]
  key_name               = aws_key_pair.ec2_key.key_name

  tags = {
    Name = "${var.project_name}-instance-${random_string.instance_suffix.result}"
  }
}
```

**質問8-A：** 1つのEC2インスタンスに複数のセキュリティグループを適用できるため

**質問8-B：** public配列の最初の要素（0番目）を参照

**質問8-C：** A, D, E, F, Gに直接依存（B, Cには間接的に依存）

- A: random_string（Name タグで使用）
- D: aws_key_pair（key_name で使用）
- E: aws_security_group（vpc_security_group_ids で使用）
- F: data.aws_ami（ami で使用）
- G: aws_subnet（subnet_id で使用）

### タスク7の解答

**質問9-A：** いいえ、依存関係に基づいてTerraformが自動的に決定します

**質問9-B：** エラーになります。random_stringの結果を参照しているため、先にrandom_stringを作成する必要があります

**質問9-C：** いいえ、`terraform plan`は計画を表示するだけで、実際には何も作成しません

**質問10-A：** EC2インスタンスは物理的なリソースを起動するため、OSの起動などで時間がかかります

**質問10-B：** `No changes`と表示されます。すでにリソースが作成済みで、設定に変更がないため

**質問10-C：** AWS CLIまたはマネジメントコンソールから：

```bash
aws ssm get-parameter --name "/ec2/keypair/terraform-practice-private-key" --with-decryption --query "Parameter.Value" --output text
```

### タスク8の解答

**質問11-A：**

- `terraform plan`：これから行う変更の計画
- `terraform show`：現在存在するリソースの実際の状態

**質問11-B：** はい、実際のAWSリソースIDに変わっています

**質問11-C：** はい、表示されますが`(sensitive)`とマークされています

**質問12-A：** はい、同じものです

**質問12-B：** Terraformは「リソースが存在しない」ことを検出し、再作成する計画を表示します

### タスク9の解答

**質問13-A：** リソースを削除してから再作成する

**質問13-B：** このパラメータを変更するには、リソースを作り直す必要がある

**質問13-C：**

1. 古いEC2インスタンスを削除
2. 新しいEC2インスタンスを作成（Ubuntu AMI使用）
3. 新しいIDが付与される

**質問13-D：** AMIはインスタンスの基盤となるOSイメージであり、実行中のインスタンスでは変更できないため

**質問13-E：** AMI、インスタンスタイプ、サブネット、アベイラビリティゾーンなど、EC2の基本的な設定の変更

### タスク10の解答

**質問14-A：** いいえ、EC2インスタンスのみが削除されます

**質問14-B：** エラーになります。EC2インスタンスがそのKey Pairを使用しているため

**質問14-C：** はい、Parameter Storeのパラメータは独立したリソースなので残ります

**質問15-A：** 依存関係の逆順

1. aws_instance（最も依存されている）
2. aws_security_group / aws_key_pair（並行削除可能）
3. aws_ssm_parameter / tls_private_key（並行削除可能）
4. random_string（最も依存されていない）

**質問15-B：** 逆順です（依存されているリソースを先に削除できないため）

**質問15-C：** すべてのリソースを作成する計画が表示されます

**質問15-D：** はい、aws_ssm_parameterリソースとして定義されているため削除されます

## まとめ

### この演習で学んだこと

1. **プロバイダードキュメントの読み方**

- Example Usageの活用
- Argument Referenceの確認
- Attribute Referenceの理解

2. **resourceブロックの基本**

- リソースタイプとローカル名
- 必須パラメータとオプションパラメータ
- タグの設定

3. **リソース間の参照**

- `リソースタイプ.ローカル名.属性名`の形式
- ハードコーディングではなくリソース参照を使用
- 依存関係の自動解決

4. **terraform plan出力の理解**

- 変更記号（+, -/+）の意味
- forces replacementの影響
- sensitive値の表示

5. **リソースのライフサイクル**

- 作成（terraform apply）
- 置換（forces replacement）
- 削除（terraform destroy）

6. **複数プロバイダーの連携**

- Random：ランダム値の生成
- TLS：暗号化キー（ED25519）の生成
- AWS：実際のインフラ作成とParameter Store

7. **セキュリティのベストプラクティス**

- ED25519アルゴリズムの使用
- 秘密鍵のParameter Storeへの安全な保存
- リソース参照によるハードコーディング回避

### 重要なポイント

- **ドキュメントファースト**：公式ドキュメントを読むことが最も重要
- **plan before apply**：必ず計画を確認してから実行
- **リソース参照の活用**：ハードコーディングを避け、リソース間の参照を使用
- **依存関係の理解**：Terraformが自動的に順序を決定
- **状態管理**：terraform.tfstateが現在の状態を記録

### 次のステップ

現在、あなたは以下のスキルを習得しました：

- resourceブロックの作成
- 複数プロバイダーの活用
- リソース間の参照
- terraform planの読み方
- セキュアなキー管理
