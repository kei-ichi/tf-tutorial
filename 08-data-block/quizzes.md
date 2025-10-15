# dataブロック実践演習：既存リソースの検索と参照

## 演習の目的

この演習では、学習した`data`ブロックの知識を実際に活用して、以下を実践します：

- AWSコンソールで手動作成したリソースをdataブロックで検索
- 最新のAMI情報を動的に取得
- 既存リソースの情報を使用してEC2インスタンスを作成
- プロバイダードキュメントの読み方を実践

**重要：** この演習は前回のresourceブロック演習の続きです。既存のVPC、サブネット、ルートテーブルなどのネットワークインフラが既に設定されている状態から開始します。

**演習後のクリーンアップ：** この演習で作成するリソースは、演習終了時に削除します。既存のVPCやサブネットなどのネットワークインフラは残したまま、演習用のリソースのみを削除します。

## 事前準備

現在のプロジェクトフォルダ `terraform-aws-practice` で作業を続けます。

### 前提条件の確認

以下のコマンドで、既存のネットワークインフラが存在することを確認してください：

```bash
terraform state list
```

**期待される出力：**

```
data.aws_availability_zones.available
aws_internet_gateway.main
aws_route_table.public
aws_route_table_association.public[0]
aws_route_table_association.public[1]
aws_subnet.private[0]
aws_subnet.private[1]
aws_subnet.public[0]
aws_subnet.public[1]
aws_vpc.main
```

## 演習の流れ

この演習では、以下の順序で作業を進めます：

```
1. AWSコンソールでセキュリティグループを手動作成
   ↓
2. AWSコンソールでキーペアを手動作成
   ↓
3. dataブロックでセキュリティグループを検索
   ↓
4. dataブロックでキーペアを検索
   ↓
5. dataブロックで最新のAMIを検索
   ↓
6. 取得した情報を使ってEC2インスタンスを作成
   ↓
7. outputで取得した情報を確認
   ↓
8. リソースの動作確認
   ↓
9. 演習用リソースのクリーンアップ
```

## パート1：AWSコンソールでリソースを手動作成

### タスク1：セキュリティグループの作成

**目的：** SSH接続用のセキュリティグループを作成し、後でTerraformのdataブロックから検索できるようにします。

**手順：**

1. AWSマネジメントコンソールにログイン
2. EC2サービスを開く
3. 左メニューから「セキュリティグループ」を選択
4. 「セキュリティグループを作成」をクリック

**設定内容：**

```
セキュリティグループ名: terraform-quiz-ssh-sg
説明: Security group for SSH access - Terraform data block practice
VPC: あなたが作成したVPCを選択（terraform-practice-vpc）

インバウンドルール:
- タイプ: SSH
- プロトコル: TCP
- ポート範囲: 22
- ソース: マイIP（または 0.0.0.0/0）

アウトバウンドルール:
- デフォルトのまま（全て許可）

タグ:
- Key: Name, Value: terraform-quiz-ssh-sg
- Key: Purpose, Value: terraform-data-quiz
- Key: ManagedBy, Value: Console
```

5. 「セキュリティグループを作成」をクリック

**重要：** `Purpose = terraform-data-quiz` というタグを必ず追加してください。これをフィルター条件として使用します。

### 質問1：セキュリティグループ作成の理解

**質問1-A：** AWSコンソールで作成したセキュリティグループは、Terraformの状態ファイル（terraform.tfstate）に記録されますか？

**質問1-B：** なぜ特定のタグ（Purpose）を付ける必要があるのですか？

**質問1-C：** 作成したセキュリティグループのIDは、どこで確認できますか？

### タスク2：EC2キーペアの作成

**目的：** EC2インスタンスにSSH接続するためのキーペアを作成します。

**手順：**

1. EC2サービスの左メニューから「キーペア」を選択
2. 「キーペアを作成」をクリック

**設定内容：**

```
名前: terraform-quiz-key
キーペアのタイプ: RSA
プライベートキーファイル形式: .pem

タグを追加:
- Key: Purpose, Value: terraform-data-quiz
- Key: ManagedBy, Value: Console
```

3. 「キーペアを作成」をクリック
4. ダウンロードされた `.pem` ファイルを安全な場所に保存

**重要：** このキーファイルは一度しかダウンロードできません。紛失しないよう注意してください。

### 質問2：キーペア作成の理解

**質問2-A：** ダウンロードした `.pem` ファイルには何が含まれていますか？

**質問2-B：** AWSに保存されているのは秘密鍵ですか、公開鍵ですか？

**質問2-C：** 前回の演習でTLSプロバイダーを使って作成したキーペアと、今回AWSコンソールで作成したキーペアの違いは何ですか？

### 作成確認チェックリスト

以下を確認してください：

```
✓ セキュリティグループが作成され、Purpose タグが設定されている
✓ セキュリティグループがあなたのVPCに関連付けられている
✓ キーペアが作成され、Purpose タグが設定されている
✓ キーファイル（.pem）がダウンロードされている
```

## パート2：dataブロックで既存リソースを検索

### タスク3：プロバイダードキュメントの確認

**作業前の重要なステップ：**

dataブロックを書く前に、必ず公式ドキュメントを確認します。

**手順：**

1. https://registry.terraform.io/ にアクセス
2. 「aws」で検索
3. 「hashicorp/aws」プロバイダーを選択
4. 左側メニューから「Data Sources」を選択
5. 以下のデータソースを探してください：

- `aws_security_group`
- `aws_key_pair`
- `aws_ami`

### 質問3：ドキュメントの構成理解

**質問3-A：** データソースのドキュメントには、どのようなセクションがありますか？（5つ挙げてください）

**質問3-B：** 「Argument Reference」と「Attribute Reference」の違いは何ですか？

**質問3-C：** 新しいデータソースを使用する際、どのセクションから読み始めるべきですか？

### タスク4：セキュリティグループをdataブロックで検索

`main.tf` の**データソースセクション**（既存の`data "aws_availability_zones"`の後）に、以下を追加してください：

```hcl
# =============================================================================
# 演習用：既存リソースの検索
# =============================================================================

# タスク4：AWSコンソールで作成したセキュリティグループを検索
# ヒント：
# - データソース名は "aws_security_group"
# - Purposeタグでフィルター（tag:Purpose）
# - VPC IDでも絞り込む（vpc-id）
# - filter ブロックを2つ使用
data "aws_security_group" "ssh" {
  # ここにコードを記述してください




}
```

**ドキュメントから確認すべき情報：**

```hcl
# Example Usage（ドキュメントから）
data "aws_security_group" "example" {
  filter {
    name   = "group-name"
    values = ["example-sg"]
  }

  filter {
    name   = "vpc-id"
    values = [aws_vpc.main.id]
  }
}

# Argument Reference
# filter - (Optional) One or more name/value pairs to filter
#   name - (Required) Name of the filter field
#   values - (Required) Set of values for filtering
```

### 質問4：セキュリティグループ検索の理解

**質問4-A：** タグでフィルターする場合、filter の name には何を指定しますか？

**質問4-B：** なぜVPC IDでも絞り込む必要があるのですか？

**質問4-C：** このdataブロックは、`terraform plan` 時に実行されますか、それとも `terraform apply` 時ですか？なぜですか？

**質問4-D：** もしセキュリティグループが見つからない場合、どのようなエラーが表示されますか？

### 検証

```bash
# 構文チェック
terraform validate

# dataブロックの動作確認
terraform plan
```

**期待される出力：**

```
data.aws_security_group.ssh: Reading...
data.aws_security_group.ssh: Read complete after 1s [id=sg-xxxxx]

No changes. Your infrastructure matches the configuration.
```

### タスク5：キーペアをdataブロックで検索

続けて、キーペアを検索するdataブロックを追加してください：

```hcl
# タスク5：AWSコンソールで作成したキーペアを検索
# ヒント：
# - データソース名は "aws_key_pair"
# - ドキュメントで使用可能な引数を確認
# - key-name フィルターまたは key_name 引数が使用可能
data "aws_key_pair" "quiz" {
  # ここにコードを記述してください



}
```

**ドキュメントから確認すべき情報：**

```hcl
# Example Usage（ドキュメントから）
data "aws_key_pair" "example" {
  key_name = "example-key"
}

# または filter を使用
data "aws_key_pair" "example" {
  filter {
    name   = "key-name"
    values = ["example-key"]
  }
}

# Argument Reference
# key_name - (Optional) Key Pair name
# filter - (Optional) Custom filter block
```

### 質問5：キーペア検索の理解

**質問5-A：** `key_name` 引数と `filter` ブロックのどちらを使用しても同じ結果が得られますか？

**質問5-B：** キーペア名は、AWSコンソールで確認したものと完全に一致している必要がありますか？

**質問5-C：** このdataブロックで取得できる情報（Attribute Reference）には何がありますか？

### 検証

```bash
terraform plan
```

**期待される出力：**

```
data.aws_security_group.ssh: Reading...
data.aws_key_pair.quiz: Reading...
data.aws_security_group.ssh: Read complete after 1s [id=sg-xxxxx]
data.aws_key_pair.quiz: Read complete after 1s [id=terraform-quiz-key]

No changes. Your infrastructure matches the configuration.
```

### タスク6：最新のAMIをdataブロックで検索

最後に、最新のAmazon Linux 2023 AMI（ARM64）を検索するdataブロックを追加してください：

```hcl
# タスク6：最新のAmazon Linux 2023 ARM64 AMIを検索
# ヒント：
# - データソース名は "aws_ami"
# - most_recent = true を必ず指定
# - owners = ["amazon"]
# - 3つのフィルターを使用：
#   1. name: "al2023-ami-*-kernel-6.1-arm64"
#   2. architecture: "arm64"
#   3. virtualization-type: "hvm"
data "aws_ami" "amazon_linux_2023_arm64" {
  # ここにコードを記述してください










}
```

**ドキュメントから確認すべき情報：**

```hcl
# Example Usage（ドキュメントから）
data "aws_ami" "example" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Argument Reference
# most_recent - (Optional) If more than one result, use the most recent AMI
# owners - (Optional) List of AMI owners to limit search
# filter - (Optional) One or more name/value pairs to filter
```

### 質問6：AMI検索の理解

**質問6-A：** なぜ `most_recent = true` が必要なのですか？

**質問6-B：** `owners = ["amazon"]` を指定する理由は何ですか？

**質問6-C：** name フィルターで `*` （ワイルドカード）を使用する理由は何ですか？

**質問6-D：** このdataブロックで取得できる情報（Attribute Reference）の中で、最も重要な属性は何ですか？

### 検証

```bash
terraform plan
```

**期待される出力：**

```
data.aws_ami.amazon_linux_2023_arm64: Reading...
data.aws_security_group.ssh: Reading...
data.aws_key_pair.quiz: Reading...
data.aws_security_group.ssh: Read complete after 1s [id=sg-xxxxx]
data.aws_key_pair.quiz: Read complete after 1s [id=terraform-quiz-key]
data.aws_ami.amazon_linux_2023_arm64: Read complete after 2s [id=ami-xxxxx]

No changes. Your infrastructure matches the configuration.
```

### 質問7：複数dataブロックの実行

**質問7-A：** 3つのdataブロックは、どのような順序で実行されましたか？

**質問7-B：** dataブロック間に依存関係はありますか？

**質問7-C：** もし1つのdataブロックが失敗した場合、他のdataブロックは実行されますか？

## パート3：取得した情報を使ってEC2インスタンスを作成

### タスク7：EC2インスタンスリソースの作成

dataブロックで取得した情報を使用して、EC2インスタンスを作成します。

`main.tf` に以下を追加してください：

```hcl
# =============================================================================
# 演習用：EC2インスタンスの作成
# =============================================================================

# タスク7：dataブロックで取得した情報を使ってEC2インスタンスを作成
# ヒント：
# - リソースタイプは "aws_instance"
# - 以下の情報をdataブロックから取得：
#   - AMI ID
#   - セキュリティグループID
#   - キーペア名
# - データソースの参照方法: data.データソース名.ローカル名.属性
resource "aws_instance" "quiz" {
  # AMI ID（dataブロックから取得）
  # ヒント: data.aws_ami.amazon_linux_2023_arm64.id
  ami = # ここにコードを記述

  # インスタンスタイプ（ARM64対応のt4g.small）
  instance_type = # ここにコードを記述

  # キーペア名（dataブロックから取得）
  # ヒント: data.aws_key_pair.quiz.key_name
  key_name = # ここにコードを記述

  # サブネット（パブリックサブネットの1つ目）
  # ヒント: aws_subnet.public[0].id
  subnet_id = # ここにコードを記述

  # セキュリティグループID（dataブロックから取得、リスト形式）
  # ヒント: [data.aws_security_group.ssh.id]
  vpc_security_group_ids = # ここにコードを記述

  # タグ
  tags = {
    Name    = "${var.project_name}-data-quiz-instance"
    Purpose = "terraform-data-quiz"
  }
}
```

### 質問8：データソースの参照

**質問8-A：** データソースを参照する際、必ず付けるプレフィックスは何ですか？

**質問8-B：** `data.aws_security_group.ssh.id` の各部分は何を意味しますか？

**質問8-C：** なぜセキュリティグループIDをリスト形式 `[]` で指定する必要がありますか？

**質問8-D：** このEC2インスタンスは、どのdataブロックに依存していますか？

### 検証

```bash
terraform validate
terraform plan
```

**期待される出力：**

```
  # aws_instance.quiz will be created
  + resource "aws_instance" "quiz" {
      + ami                         = "ami-xxxxx"
      + instance_type               = "t4g.small"
      + key_name                    = "terraform-quiz-key"
      + subnet_id                   = "subnet-xxxxx"
      + vpc_security_group_ids      = [
          + "sg-xxxxx",
        ]
      + tags                        = {
          + "Name"    = "terraform-practice-data-quiz-instance"
          + "Purpose" = "terraform-data-quiz"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

### 質問9：planの読み取り

**質問9-A：** AMI IDは具体的な値が表示されていますか、それとも "(known after apply)" ですか？なぜですか？

**質問9-B：** セキュリティグループIDとキーペア名も具体的な値が表示されていますか？

**質問9-C：** これらの値がplan時に表示される理由は何ですか？

## パート4：outputで取得した情報を確認

### タスク8：outputブロックの追加

取得した情報と作成したリソースの情報を出力します。

`main.tf` に以下を追加してください：

```hcl
# =============================================================================
# 演習用：出力の定義
# =============================================================================

# インスタンスのパブリックIPアドレス
output "quiz_instance_public_ip" {
  description = "Public IP address of the quiz instance"
  value       = # ここにコードを記述（aws_instance.quiz.public_ip）
}

# 使用したAMI ID
output "quiz_ami_id" {
  description = "AMI ID used for the instance"
  value       = # ここにコードを記述（data.aws_ami.amazon_linux_2023_arm64.id）
}

# 使用したAMI名
output "quiz_ami_name" {
  description = "AMI name used for the instance"
  value       = # ここにコードを記述（data.aws_ami.amazon_linux_2023_arm64.name）
}

# 使用したセキュリティグループID
output "quiz_security_group_id" {
  description = "Security group ID used for the instance"
  value       = # ここにコードを記述（data.aws_security_group.ssh.id）
}

# 使用したキーペア名
output "quiz_key_pair_name" {
  description = "Key pair name used for the instance"
  value       = # ここにコードを記述（data.aws_key_pair.quiz.key_name）
}
```

### 質問10：outputブロックの理解

**質問10-A：** outputブロックはいつ値を表示しますか？

**質問10-B：** dataブロックから取得した値もoutputで表示できますか？

**質問10-C：** `terraform output` コマンドと `terraform show` コマンドの違いは何ですか？

## パート5：リソースの作成と確認

### タスク9：すべてのリソースを作成

```bash
# 最終確認
terraform plan

# リソースの作成
terraform apply
```

**実行中の出力例：**

```
data.aws_ami.amazon_linux_2023_arm64: Reading...
data.aws_security_group.ssh: Reading...
data.aws_key_pair.quiz: Reading...
data.aws_security_group.ssh: Read complete after 1s [id=sg-xxxxx]
data.aws_key_pair.quiz: Read complete after 1s [id=terraform-quiz-key]
data.aws_ami.amazon_linux_2023_arm64: Read complete after 2s [id=ami-xxxxx]

Terraform will perform the following actions:

  # aws_instance.quiz will be created
  + resource "aws_instance" "quiz" {
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Enter a value: yes

aws_instance.quiz: Creating...
aws_instance.quiz: Still creating... [10s elapsed]
aws_instance.quiz: Still creating... [20s elapsed]
aws_instance.quiz: Creation complete after 25s [id=i-xxxxx]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

quiz_ami_id = "ami-xxxxx"
quiz_ami_name = "al2023-ami-2024.x.x-kernel-6.1-arm64"
quiz_instance_public_ip = "xx.xx.xx.xx"
quiz_key_pair_name = "terraform-quiz-key"
quiz_security_group_id = "sg-xxxxx"
```

### 質問11：apply実行の理解

**質問11-A：** dataブロックは apply 実行時に再度実行されましたか？

**質問11-B：** EC2インスタンスが作成されるまでにかかった時間は？

**質問11-C：** 出力された情報の中で、dataブロックから取得したものはどれですか？

### タスク10：outputの確認

```bash
# すべての出力を表示
terraform output

# 特定の出力のみ表示
terraform output quiz_instance_public_ip
terraform output quiz_ami_name
```

### 質問12：outputの活用

**質問12-A：** `terraform output` コマンドはいつでも実行できますか？

**質問12-B：** outputの値は、terraform.tfstate ファイルに保存されていますか？

**質問12-C：** 他のTerraformプロジェクトからこのoutputの値を参照することは可能ですか？

### タスク11：AWSコンソールで確認

1. **EC2インスタンス**

- EC2 → インスタンス
- 作成されたインスタンスを確認
- インスタンスID、パブリックIP、状態を確認

2. **セキュリティグループ**

- インスタンスの詳細で、紐付けられているセキュリティグループを確認
- AWSコンソールで作成したものと同じか確認

3. **キーペア**

- インスタンスの詳細で、設定されているキーペア名を確認

4. **AMI情報**

- インスタンスの詳細で、使用されているAMI IDを確認
- outputで表示されたAMI IDと一致するか確認

### 質問13：リソースの関連性

**質問13-A：** AWSコンソールで確認した内容と、Terraformのoutputは一致していますか？

**質問13-B：** EC2インスタンスに紐付けられているセキュリティグループは、Terraformで作成したものですか、AWSコンソールで作成したものですか？

**質問13-C：** もしAWSコンソールでセキュリティグループを削除しようとした場合、削除できますか？なぜですか？

## パート6：dataブロックとresourceブロックの違いを理解

### タスク12：terraform showで状態を確認

```bash
# EC2インスタンスの状態を表示
terraform state show aws_instance.quiz

# dataブロックの状態を表示
terraform state show data.aws_security_group.ssh
```

### 質問14：状態ファイルの理解

**質問14-A：** `terraform state show` で表示される情報に違いはありますか？

**質問14-B：** dataブロックで取得した情報は、terraform.tfstate に保存されていますか？

**質問14-C：** もしAWSコンソールでセキュリティグループの設定を変更した場合、次回 `terraform plan` を実行すると何が起こりますか？

### タスク13：動作の違いを確認

**実験1：dataブロックで取得したリソースをAWSコンソールで変更**

1. AWSコンソールでセキュリティグループの説明を変更
2. `terraform plan` を実行

```bash
terraform plan
```

**期待される結果：**

```
No changes. Your infrastructure matches the configuration.
```

**実験2：resourceで作成したリソースをAWSコンソールで変更**

1. AWSコンソールでEC2インスタンスのタグを追加（例：`Test = "manual"`）
2. `terraform plan` を実行

```bash
terraform plan
```

**期待される結果：**

```
  # aws_instance.quiz will be updated in-place
  ~ resource "aws_instance" "quiz" {
      ~ tags     = {
          - "Test"    = "manual" -> null
            # ...
        }
    }
```

### 質問15：dataとresourceの動作の違い

**質問15-A：** なぜセキュリティグループの変更はTerraformに検出されなかったのですか？

**質問15-B：** なぜEC2インスタンスのタグ変更はTerraformに検出されたのですか？

**質問15-C：** dataブロックで取得したリソースは、Terraformの「管理対象」ですか？

**質問15-D：** resourceブロックで作成したリソースは、Terraformの「管理対象」ですか？

**質問15-E：** この違いは、どのような場面で重要になりますか？

## パート7：演習用リソースのクリーンアップ

**重要：** この演習で作成したリソース（EC2インスタンスのみ）と、AWSコンソールで作成したリソース（セキュリティグループ、キーペア）を削除します。既存のVPC、サブネット、ルートテーブルなどのネットワークインフラは**削除しません**。

### クリーンアップ対象

**Terraformで削除：**

- `aws_instance.quiz` - EC2インスタンス

**AWSコンソールで削除：**

- セキュリティグループ（terraform-quiz-ssh-sg）
- キーペア（terraform-quiz-key）

**コードから削除：**

- dataブロック3つ
- resourceブロック1つ
- outputブロック5つ

### ステップ1：Terraformで作成したリソースを削除

まず、EC2インスタンスを削除します：

```bash
# EC2インスタンスのみを削除
terraform destroy -target=aws_instance.quiz
```

**実行中の出力例：**

```
aws_instance.quiz: Refreshing state... [id=i-xxxxx]

Terraform will perform the following actions:

  # aws_instance.quiz will be destroyed
  - resource "aws_instance" "quiz" {
      - ami                         = "ami-xxxxx" -> null
      - id                          = "i-xxxxx" -> null
      # ...
    }

Plan: 0 to add, 0 to change, 1 to destroy.

Do you really want to destroy all resources?
  Enter a value: yes

aws_instance.quiz: Destroying... [id=i-xxxxx]
aws_instance.quiz: Still destroying... [10s elapsed]
aws_instance.quiz: Still destroying... [20s elapsed]
aws_instance.quiz: Destruction complete after 30s

Destroy complete! Resources: 1 destroyed.
```

### 質問16：ターゲット削除の理解

**質問16-A：** `-target` オプションを使用する理由は何ですか？

**質問16-B：** このコマンドは、dataブロックで取得したセキュリティグループも削除しますか？

**質問16-C：** EC2インスタンスが削除された後、セキュリティグループはどうなりますか？

### ステップ2：AWSコンソールで作成したリソースを削除

**セキュリティグループの削除：**

1. EC2サービス → セキュリティグループ
2. `terraform-quiz-ssh-sg` を選択
3. 「アクション」→「セキュリティグループを削除」
4. 確認して削除

**キーペアの削除：**

1. EC2サービス → キーペア
2. `terraform-quiz-key` を選択
3. 「アクション」→「削除」
4. 確認して削除

**ローカルのキーファイルも削除：**

```bash
rm terraform-quiz-key.pem
```

### ステップ3：設定ファイルからコードを削除

`main.tf` から以下のブロックを削除してください：

```hcl
# 削除するdataブロック
data "aws_security_group" "ssh" { ... }
data "aws_key_pair" "quiz" { ... }
data "aws_ami" "amazon_linux_2023_arm64" { ... }

# 削除するresourceブロック
resource "aws_instance" "quiz" { ... }

# 削除するoutputブロック
output "quiz_instance_public_ip" { ... }
output "quiz_ami_id" { ... }
output "quiz_ami_name" { ... }
output "quiz_security_group_id" { ... }
output "quiz_key_pair_name" { ... }
```

### ステップ4：削除の確認

```bash
# 構文チェック
terraform validate

# 変更がないことを確認
terraform plan
```

**期待される出力：**

```
No changes. Your infrastructure matches the configuration.
```

### ステップ5：状態ファイルの確認

```bash
terraform state list
```

**期待される出力（演習用リソースは含まれない）：**

```
data.aws_availability_zones.available
aws_internet_gateway.main
aws_route_table.public
aws_route_table_association.public[0]
aws_route_table_association.public[1]
aws_subnet.private[0]
aws_subnet.private[1]
aws_subnet.public[0]
aws_subnet.public[1]
aws_vpc.main
```

### 質問17：クリーンアップの理解

**質問17-A：** なぜEC2インスタンスを先に削除する必要があったのですか？

**質問17-B：** dataブロックで取得したリソース（セキュリティグループ、キーペア）は、Terraformで削除できますか？

**質問17-C：** 設定ファイルからdataブロックを削除しても、AWSのリソースは削除されませんでしたか？なぜですか？

**質問17-D：** もしAWSコンソールで作成したセキュリティグループを削除し忘れた場合、次回どのようなエラーが発生しますか？

### 削除確認チェックリスト

```
✓ EC2インスタンスが削除された（AWSコンソールで確認）
✓ セキュリティグループが削除された
✓ キーペアが削除された
✓ ローカルのキーファイルが削除された
✓ main.tf から演習用コードが削除された
✓ terraform plan で変更がないことを確認
✓ 既存のVPC/サブネット/IGW/ルートテーブルは残っている
```

## 解答例と解説

### タスク4の解答：セキュリティグループの検索

```hcl
data "aws_security_group" "ssh" {
  filter {
    name   = "tag:Purpose"
    values = ["terraform-data-quiz"]
  }

  filter {
    name   = "vpc-id"
    values = [aws_vpc.main.id]
  }
}
```

**質問4の解答：**

**質問4-A：** `"tag:タグのKey名"`（例：`"tag:Purpose"`）

**質問4-B：** 同じ名前のセキュリティグループが異なるVPCに存在する可能性があるため、VPC IDで絞り込むことで確実に目的のセキュリティグループを特定できます。

**質問4-C：** `terraform plan` 時に実行されます。すべてのフィルター条件が固定値（変数やリソース参照を含む既知の値）なので、plan時に検索が完了します。

**質問4-D：** `Error: no matching SecurityGroup found` のようなエラーが表示されます。

### タスク5の解答：キーペアの検索

```hcl
data "aws_key_pair" "quiz" {
  key_name = "terraform-quiz-key"
}

# または filter を使用
data "aws_key_pair" "quiz" {
  filter {
    name   = "key-name"
    values = ["terraform-quiz-key"]
  }
}
```

**質問5の解答：**

**質問5-A：** はい、どちらも同じ結果が得られます。`key_name` 引数の方がシンプルで読みやすいです。

**質問5-B：** はい、完全に一致している必要があります（大文字小文字を含む）。

**質問5-C：** `id`、`arn`、`key_name`、`key_pair_id`、`fingerprint`、`tags` など。

### タスク6の解答：AMIの検索

```hcl
data "aws_ami" "amazon_linux_2023_arm64" {
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
```

**質問6の解答：**

**質問6-A：** 条件に一致するAMIが複数見つかった場合、Terraformはどれを使用すべきか判断できずエラーになります。`most_recent = true` を指定することで、最新のものを自動的に選択します。

**質問6-B：** サードパーティや個人が公開しているAMIではなく、Amazon公式のAMIのみを検索対象にするためです。

**質問6-C：** AMI名には日付やバージョン番号が含まれるため、ワイルドカードを使用することで特定のバージョンに依存せず、最新のAMIを取得できます。

**質問6-D：** `id`（AMI ID）です。これがEC2インスタンスを作成する際に必要な値です。

**質問7の解答：**

**質問7-A：** ほぼ同時に並行して実行されました（依存関係がないため）。

**質問7-B：** いいえ、3つのdataブロックは互いに独立しており、依存関係はありません。

**質問7-C：** はい、独立しているため他のdataブロックは実行されます。ただし、失敗したdataブロックに依存するリソースは作成できません。

### タスク7の解答：EC2インスタンスの作成

```hcl
resource "aws_instance" "quiz" {
  ami                    = data.aws_ami.amazon_linux_2023_arm64.id
  instance_type          = "t4g.small"
  key_name               = data.aws_key_pair.quiz.key_name
  subnet_id              = aws_subnet.public[0].id
  vpc_security_group_ids = [data.aws_security_group.ssh.id]

  tags = {
    Name    = "${var.project_name}-data-quiz-instance"
    Purpose = "terraform-data-quiz"
  }
}
```

**質問8の解答：**

**質問8-A：** `data.`

**質問8-B：**

- `data.` - データソースであることを示すプレフィックス
- `aws_security_group` - データソースのタイプ
- `.ssh` - ローカル名
- `.id` - 属性（セキュリティグループID）

**質問8-C：** `vpc_security_group_ids` は複数のセキュリティグループIDを受け取るリスト型のパラメータだからです。

**質問8-D：** 3つすべて：

- `data.aws_ami.amazon_linux_2023_arm64`（AMI ID）
- `data.aws_security_group.ssh`（セキュリティグループID）
- `data.aws_key_pair.quiz`（キーペア名）

**質問9の解答：**

**質問9-A：** はい、具体的な値が表示されています。dataブロックがplan時に実行され、実際のAMI IDを取得したからです。

**質問9-B：** はい、どちらも具体的な値が表示されています。

**質問9-C：** これらのdataブロックはすべて固定値（またはresource参照の既知の値）のみを使用しているため、plan時に検索が完了し、結果が確定するからです。

### タスク8の解答：outputの定義

```hcl
output "quiz_instance_public_ip" {
  description = "Public IP address of the quiz instance"
  value       = aws_instance.quiz.public_ip
}

output "quiz_ami_id" {
  description = "AMI ID used for the instance"
  value       = data.aws_ami.amazon_linux_2023_arm64.id
}

output "quiz_ami_name" {
  description = "AMI name used for the instance"
  value       = data.aws_ami.amazon_linux_2023_arm64.name
}

output "quiz_security_group_id" {
  description = "Security group ID used for the instance"
  value       = data.aws_security_group.ssh.id
}

output "quiz_key_pair_name" {
  description = "Key pair name used for the instance"
  value       = data.aws_key_pair.quiz.key_name
}
```

**質問10の解答：**

**質問10-A：** `terraform apply` 実行後、または `terraform output` コマンド実行時です。

**質問10-B：** はい、できます。dataブロックから取得した値もoutputで表示できます。

**質問10-C：**

- `terraform output` - outputブロックで定義された値のみを表示
- `terraform show` - すべてのリソースとdataソースの詳細な状態を表示

### その他の質問の解答

**質問1の解答：**

**質問1-A：** いいえ、記録されません。AWSコンソールで作成したリソースは、Terraformの管理対象外です。

**質問1-B：** dataブロックで検索する際のフィルター条件として使用するためです。複数のセキュリティグループの中から、目的のものを特定できます。

**質問1-C：** AWSコンソールのセキュリティグループ詳細ページ、またはセキュリティグループ一覧のID列で確認できます。

**質問2の解答：**

**質問2-A：** 秘密鍵（プライベートキー）が含まれています。

**質問2-B：** 公開鍵です。秘密鍵はAWSには送信されず、ダウンロードしたファイルにのみ含まれます。

**質問2-C：**

- TLSプロバイダー：Terraformが秘密鍵と公開鍵の両方を生成し、terraform.tfstateに保存
- AWSコンソール：AWSが秘密鍵と公開鍵を生成し、公開鍵のみAWSに保存、秘密鍵はダウンロード

**質問3の解答：**

**質問3-A：**

1. 概要説明（データソースの目的）
2. 注意事項（Note/Warning）
3. Example Usage（使用例）
4. Argument Reference（引数一覧）
5. Attribute Reference（参照可能な属性）

**質問3-B：**

- Argument Reference：データソースの「入力」（検索条件）
- Attribute Reference：データソースの「出力」（取得できる情報）

**質問3-C：** Example Usageから読み始めるべきです。実際の使用例を見ることで、全体像が理解しやすくなります。

**質問11の解答：**

**質問11-A：** はい、apply実行時にdataブロックは自動的に再実行（refresh）されます。

**質問11-B：** 約20〜30秒（インスタンスの起動時間）

**質問11-C：** dataブロックから取得したもの：

- `quiz_ami_id`
- `quiz_ami_name`
- `quiz_security_group_id`
- `quiz_key_pair_name`

resourceから取得したもの：

- `quiz_instance_public_ip`

**質問12の解答：**

**質問12-A：** はい、`terraform apply` 実行後であればいつでも実行できます。

**質問12-B：** はい、terraform.tfstate ファイルの "outputs" セクションに保存されています。

**質問12-C：** はい、`terraform_remote_state` データソースを使用すれば可能です（後の章で学習します）。

**質問13の解答：**

**質問13-A：** はい、一致しています。

**質問13-B：** AWSコンソールで作成したものです。Terraformはdataブロックでそれを参照しているだけです。

**質問13-C：** いいえ、削除できません。EC2インスタンスが使用中のため、先にインスタンスを削除する必要があります。

**質問14の解答：**

**質問14-A：** はい、表示される情報に違いがあります：

- resource：Terraformが管理する完全な状態
- data：検索時に取得した読み取り専用の情報

**質問14-B：** はい、dataブロックで取得した情報も terraform.tfstate に保存されます（キャッシュとして）。

**質問14-C：** dataブロックは読み取り専用なので、Terraformは変更を検出しません。次回plan時に最新の情報を再取得しますが、差分は表示されません。

**質問15の解答：**

**質問15-A：** dataブロックは読み取り専用で、Terraformの管理対象ではないからです。dataブロックは情報を取得するだけで、リソースの状態を管理しません。

**質問15-B：** resourceブロックで作成したリソースは、Terraformの管理対象だからです。terraform.tfstate に定義された状態と、実際のAWSの状態を比較して差分を検出します。

**質問15-C：** いいえ、管理対象ではありません。dataブロックは既存リソースを「参照」するだけです。

**質問15-D：** はい、管理対象です。resourceブロックで作成したリソースの状態をTerraformが追跡・管理します。

**質問15-E：** 以下のような場面で重要です：

- 手動で作成した既存インフラをTerraformから利用したい場合（dataブロック使用）
- 他のチームが管理するリソースを参照したい場合（dataブロック使用）
- Terraformで完全に管理したいリソースの場合（resourceブロック使用）

**質問16の解答：**

**質問16-A：** 演習用のEC2インスタンスのみを削除し、既存のネットワークインフラ（VPC、サブネットなど）を残すためです。

**質問16-B：** いいえ、削除できません。dataブロックはリソースを参照しているだけで、管理していないからです。

**質問16-C：** そのままです。EC2インスタンスが削除されたため、どのリソースからも使用されていない状態になります。

**質問17の解答：**

**質問17-A：** EC2インスタンスがセキュリティグループを使用しているため、先にインスタンスを削除しないとセキュリティグループの削除ができないからです（依存関係）。

**質問17-B：** いいえ、できません。dataブロックで取得したリソースは、Terraformの管理対象ではないため、Terraformで削除することはできません。AWSコンソールまたはAWS CLIで手動削除が必要です。

**質問17-C：** いいえ、削除されません。dataブロックはリソースを「参照」しているだけで、「作成」や「管理」はしていないからです。設定ファイルから削除しても、AWS上のリソースには影響しません。

**質問17-D：** 次回演習でdataブロックを追加した際、`Error: no matching SecurityGroup found` のようなエラーが発生します。

## まとめ

### この演習で学んだこと

1. **dataブロックとresourceブロックの本質的な違い**

- data：既存リソースを「参照」（読み取り専用）
- resource：新しいリソースを「作成・管理」
- 管理対象かどうかの違い

2. **プロバイダードキュメントの実践的な読み方**

- Example Usageから始める
- Argument Reference で検索条件を確認
- Attribute Reference で取得可能な情報を確認

3. **dataブロックの実践的な使用方法**

- filterを使った検索条件の指定
- タグを使った特定リソースの絞り込み
- most_recentを使った最新リソースの取得

4. **データソースの参照方法**

- `data.データソース名.ローカル名.属性` の形式
- 必ず `data.` プレフィックスを付ける
- resourceの参照と区別する

5. **dataブロックの実行タイミング**

- 固定値のみ → plan時に実行
- 具体的な値がplan時に表示される
- refreshで最新情報を取得

6. **既存インフラとの連携**

- 手動で作成したリソースをdataブロックで参照
- Terraformの管理対象外のリソースを活用
- 段階的なTerraform導入が可能

7. **リソース管理の責任範囲**

- dataブロック：参照のみ、削除は手動
- resourceブロック：完全管理、Terraformで削除

8. **outputを使った情報の可視化**

- dataブロックから取得した情報もoutputで表示可能
- 実際の値の確認方法
- 他のプロジェクトとの情報共有の準備

### 重要なポイント

**dataブロックの特徴：**

- AWS上に何も作成しない
- 既存の情報を読み取るだけ
- terraform.tfstate に情報をキャッシュ
- 管理対象ではない

**resourceブロックとの違い：**

- resource：作成・更新・削除を管理
- data：読み取りのみ
- 手動削除が必要なのがdata

**ベストプラクティス：**

- 既存インフラを参照する場合はdataブロック
- 新規作成・管理する場合はresourceブロック
- 明確なタグ付けでリソース特定を確実に
- ドキュメントを必ず確認
