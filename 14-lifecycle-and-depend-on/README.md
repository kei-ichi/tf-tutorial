# Terraformのlifecycleとdepends_on完全ガイド

## この章で学ぶこと

Terraformのリソース管理をより細かく制御する2つの重要なメタ引数について学習します。

**学習内容：**

- lifecycleブロック：リソースのライフサイクル制御
- depends_on：依存関係の明示的な指定
- なぜこれらが必要なのか
- 実践的な使用例
- ベストプラクティスと注意点

## メタ引数とは

**基本的な考え方：**

```
通常の引数：
  リソース固有の設定
  例：ami、instance_type、cidr_block

メタ引数：
  すべてのリソースで使える特別な引数
  Terraformの動作を制御
  例：lifecycle、depends_on、count、for_each
```

**メタ引数の種類：**

```
すでに学習したメタ引数：
- count：複数のリソースを作成
- for_each：複数のリソースを作成（キー付き）

この章で学習するメタ引数：
- lifecycle：リソースのライフサイクルを制御
- depends_on：依存関係を明示的に指定
```

## lifecycleブロック：リソースのライフサイクル制御

### なぜlifecycleが必要なのか

#### 問題1：ダウンタイムが発生する

**シナリオ：EC2インスタンスのAMI変更**

```hcl
# 現在の設定
resource "aws_instance" "web" {
  ami           = "ami-old123"
  instance_type = "t4g.small"
}
```

**AMIを変更：**

```hcl
# 変更後
resource "aws_instance" "web" {
  ami           = "ami-new456"  # AMI変更
  instance_type = "t4g.small"
}
```

**terraform planの結果（lifecycleなし）：**

```bash
terraform plan

# 出力：
Terraform will perform the following actions:

  # aws_instance.web must be replaced
-/+ resource "aws_instance" "web" {
      ~ ami           = "ami-old123" -> "ami-new456" # forces replacement
        instance_type = "t4g.small"
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```

**実際の動作：**

```
ステップ1：旧インスタンス削除
  ↓ サービス停止
ステップ2：新インスタンス作成
  ↓ 起動に数分
ステップ3：サービス再開

結果：数分間のダウンタイム発生
```

**問題点：**

- サービスが停止する
- ユーザーがアクセスできない
- ビジネスへの影響

#### 解決策：create_before_destroy

```hcl
resource "aws_instance" "web" {
  ami           = "ami-new456"
  instance_type = "t4g.small"

  lifecycle {
    create_before_destroy = true
  }
}
```

**実際の動作：**

```
ステップ1：新インスタンス作成（旧インスタンスは稼働中）
  ↓ サービス継続
ステップ2：新インスタンス起動完了
  ↓ 切り替え
ステップ3：旧インスタンス削除

結果：ダウンタイムなし（ゼロダウンタイム）
```

#### 問題2：意図しない削除を防ぎたい

**シナリオ：本番データベース**

```hcl
resource "aws_db_instance" "production" {
  identifier        = "prod-db"
  engine            = "mysql"
  instance_class    = "db.t3.small"
  allocated_storage = 100
}
```

**誤って削除しようとした場合：**

```bash
# 誤ってリソースを削除
terraform destroy

# 通常の動作：
# データベースが削除される
# データが失われる
# 復旧不可能
```

#### 解決策：prevent_destroy

```hcl
resource "aws_db_instance" "production" {
  identifier        = "prod-db"
  engine            = "mysql"
  instance_class    = "db.t3.small"
  allocated_storage = 100

  lifecycle {
    prevent_destroy = true
  }
}
```

**実際の動作：**

```bash
terraform destroy

# 出力：
Error: Instance cannot be destroyed

  on main.tf line 10:
  10: resource "aws_db_instance" "production" {

Resource aws_db_instance.production has lifecycle.prevent_destroy set,
but the plan calls for this resource to be destroyed.

結果：削除が防止される（安全）
```

#### 問題3：頻繁に変わる属性を無視したい

**シナリオ：タグを手動で追加することがある**

```hcl
resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "web-server"
    Team = "platform"
  }
}
```

**運用中に手動でタグ追加：**

```bash
# AWSコンソールで手動追加
# Owner = "john"
```

**terraform planの結果（ignore_changesなし）：**

```bash
terraform plan

# 出力：
Terraform will perform the following actions:

  # aws_instance.web will be updated in-place
  ~ resource "aws_instance" "web" {
      ~ tags = {
          - Owner = "john" -> null  # 手動追加したタグが削除される
            Name  = "web-server"
            Team  = "platform"
        }
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

**問題点：**

- 手動で追加したタグが削除される
- 毎回planに表示される
- 運用が面倒

#### 解決策：ignore_changes

```hcl
resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "web-server"
    Team = "platform"
  }

  lifecycle {
    ignore_changes = [tags]
  }
}
```

**実際の動作：**

```bash
terraform plan

# 出力：
No changes. Your infrastructure matches the configuration.

結果：手動で追加したタグは無視される
```

### lifecycleブロックの基本構文

```hcl
resource "リソースタイプ" "名前" {
  # リソースの設定

  lifecycle {
    create_before_destroy = true/false
    prevent_destroy       = true/false
    ignore_changes        = [属性リスト]
    replace_triggered_by  = [参照リスト]
    precondition {
      condition     = 条件式
      error_message = "エラーメッセージ"
    }
    postcondition {
      condition     = 条件式
      error_message = "エラーメッセージ"
    }
  }
}
```

### lifecycleの各オプション詳細

#### 1. create_before_destroy

**目的：** ダウンタイムを防ぐ

**構文：**

```hcl
lifecycle {
  create_before_destroy = true
}
```

**いつ使うか：**

```
使用する場合：
- Webサーバー（サービス継続が重要）
- アプリケーションサーバー
- ロードバランサー
- DNSレコード

使用しない場合：
- 開発環境のリソース（ダウンタイムOK）
- 一意制約があるリソース（名前が重複する場合）
```

**実例1：EC2インスタンス**

```hcl
resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "web-server"
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

**動作の流れ：**

```
AMIを変更してterraform apply実行：

通常の動作（create_before_destroy = false）：
1. 旧インスタンス削除
2. 新インスタンス作成
→ ダウンタイム発生

create_before_destroy = true：
1. 新インスタンス作成（旧インスタンス稼働中）
2. 新インスタンス起動完了
3. 旧インスタンス削除
→ ダウンタイムなし
```

**実例2：ロードバランサーのターゲットグループ**

```hcl
resource "aws_lb_target_group" "app" {
  name     = "app-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    path = "/health"
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

**理由：**

```
ターゲットグループの変更時：
- 新しいターゲットグループを先に作成
- ロードバランサーの設定を切り替え
- 旧ターゲットグループを削除
→ サービス継続
```

**注意点：**

```hcl
# 問題が発生する例：名前の一意制約

resource "aws_db_instance" "main" {
  identifier = "my-database"  # 一意である必要がある
  # ...

  lifecycle {
    create_before_destroy = true
  }
}

# エラー発生：
# 新しいDBインスタンスを作成しようとする
# しかし同じ名前"my-database"は使えない
# → エラーで失敗

# 解決策：名前に乱数を含める
resource "aws_db_instance" "main" {
  identifier = "my-database-${random_id.db.hex}"
  # ...
}
```

#### 2. prevent_destroy

**目的：** 重要なリソースの誤削除を防ぐ

**構文：**

```hcl
lifecycle {
  prevent_destroy = true
}
```

**いつ使うか：**

```
必ず使用する場合：
- 本番データベース
- S3バケット（重要データ）
- ステートファイル用のS3バケット
- 本番環境のVPC

使用しない場合：
- 開発環境のリソース
- 簡単に再作成できるリソース
- テスト用リソース
```

**実例1：本番データベース**

```hcl
resource "aws_db_instance" "production" {
  identifier           = "prod-db"
  engine               = "mysql"
  engine_version       = "8.0"
  instance_class       = "db.r5.large"
  allocated_storage    = 100
  storage_encrypted    = true
  multi_az             = true

  lifecycle {
    prevent_destroy = true
  }
}
```

**動作確認：**

```bash
# データベースを削除しようとする
terraform destroy

# 出力：
Error: Instance cannot be destroyed

  on main.tf line 1:
   1: resource "aws_db_instance" "production" {

Resource aws_db_instance.production has lifecycle.prevent_destroy set,
but the plan calls for this resource to be destroyed.

# 削除が防止される
```

**実例2：Terraformステートファイル用S3バケット**

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state-bucket"

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

**理由：**

```
ステートファイル用バケットの重要性：
- すべてのインフラ情報を含む
- 削除すると管理不能になる
- 復旧が非常に困難

prevent_destroyで保護：
- 誤削除を防ぐ
- 安全性を確保
```

**prevent_destroyを解除する方法：**

```hcl
# ステップ1：prevent_destroyをfalseに変更
resource "aws_db_instance" "production" {
  # ...

  lifecycle {
    prevent_destroy = false  # falseに変更
  }
}

# ステップ2：terraform apply
terraform apply

# ステップ3：削除可能になる
terraform destroy
```

**重要な注意点：**

```
prevent_destroyの制限：

防げること：
- terraform destroy
- resourceブロックの削除
- リソースを削除する変更

防げないこと：
- AWSコンソールからの手動削除
- AWS CLIでの削除
- アカウント権限での削除

→ 完全な保護ではなく、Terraform操作のみの保護
```

#### 3. ignore_changes

**目的：** 特定の属性の変更を無視する

**構文：**

```hcl
lifecycle {
  ignore_changes = [属性名]
}

# または全ての属性を無視
lifecycle {
  ignore_changes = all
}
```

**いつ使うか：**

```
使用する場合：
- 手動で変更される属性（タグなど）
- 外部システムが管理する属性
- 頻繁に変わる属性（Auto Scalingなど）
- Terraformで管理したくない属性

注意：
過度な使用は避ける（設定のドリフトを隠す）
```

**実例1：タグの無視**

```hcl
resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "web-server"
    Team = "platform"
  }

  lifecycle {
    ignore_changes = [tags]
  }
}
```

**理由：**

```
運用での実情：
- 運用チームが手動でタグを追加
- コスト管理システムがタグを追加
- 監視システムがタグを追加

ignore_changesがない場合：
- terraform planで毎回差分が表示される
- terraform applyで手動タグが削除される

ignore_changesがある場合：
- 手動タグは保持される
- planがクリーンになる
```

**実例2：Auto Scalingのdesired_capacity**

```hcl
resource "aws_autoscaling_group" "app" {
  name                = "app-asg"
  min_size            = 2
  max_size            = 10
  desired_capacity    = 4
  vpc_zone_identifier = aws_subnet.private[*].id

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  lifecycle {
    ignore_changes = [desired_capacity]
  }
}
```

**理由：**

```
Auto Scalingの動作：
- 負荷に応じてdesired_capacityが変化
- 例：2 → 6 → 3 → 4（自動調整）

ignore_changesがない場合：
- terraform planで毎回差分が表示される
- terraform applyで元の値（4）に戻される
- Auto Scalingが機能しない

ignore_changesがある場合：
- 自動調整された値を保持
- Terraformは最小値と最大値のみ管理
```

**実例3：特定のタグのみ無視**

```hcl
resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name        = "web-server"
    Team        = "platform"
    Environment = "production"
  }

  lifecycle {
    # Ownerタグのみ無視（他のタグは管理）
    ignore_changes = [tags["Owner"]]
  }
}
```

**実例4：すべての変更を無視（非推奨）**

```hcl
resource "aws_instance" "legacy" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  lifecycle {
    ignore_changes = all  # すべての属性を無視
  }
}
```

**注意：**

```
ignore_changes = all の使用：

推奨されない理由：
- Terraformで管理する意味がない
- 設定のドリフトが完全に隠される
- インフラの状態が不明確になる

使用してもよい場合：
- レガシーシステムの一時的な取り込み
- 移行期間中のみ
- 将来的に削除予定
```

**ベストプラクティス：**

```hcl
# 良い例：必要最小限の属性のみ無視
lifecycle {
  ignore_changes = [
    tags["Owner"],
    tags["CostCenter"]
  ]
}

# 悪い例：タグ全体を無視
lifecycle {
  ignore_changes = [tags]
}

# より良い例：管理したいタグのみTerraformで定義
resource "aws_instance" "web" {
  # ...

  tags = {
    Name = "web-server"  # Terraformで管理
    # OwnerやCostCenterは手動追加を許可
  }

  lifecycle {
    ignore_changes = [
      tags["Owner"],
      tags["CostCenter"]
    ]
  }
}
```

#### 4. replace_triggered_by

**目的：** 他のリソースが変更された時に再作成をトリガー

**構文：**

```hcl
lifecycle {
  replace_triggered_by = [
    リソース参照
  ]
}
```

**実例：起動テンプレートの変更でEC2を再作成**

```hcl
resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = var.ami_id
  instance_type = "t4g.small"

  # 設定変更時に新しいバージョンが作成される
}

resource "aws_instance" "app" {
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  lifecycle {
    # 起動テンプレートが変わったらインスタンスを再作成
    replace_triggered_by = [
      aws_launch_template.app
    ]
  }
}
```

**理由：**

```
通常の動作：
- 起動テンプレートを変更
- terraform plan
- インスタンスは変更されない（Terraformは検知できない）

replace_triggered_byがある場合：
- 起動テンプレートを変更
- terraform plan
- インスタンスの再作成が計画される
```

#### 5. precondition（事前条件チェック）

**目的：** リソース作成前に条件をチェック

**構文：**

```hcl
lifecycle {
  precondition {
    condition     = 条件式
    error_message = "エラーメッセージ"
  }
}
```

**実例1：AMIのアーキテクチャチェック**

```hcl
data "aws_ami" "app" {
  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["app-ami-*"]
  }
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.app.id
  instance_type = "t4g.small"  # ARM64インスタンス

  lifecycle {
    precondition {
      condition     = data.aws_ami.app.architecture == "arm64"
      error_message = "AMIはARM64アーキテクチャである必要があります。現在: ${data.aws_ami.app.architecture}"
    }
  }
}
```

**実例2：本番環境での最小インスタンスサイズ**

```hcl
variable "environment" {
  type = string
}

variable "instance_type" {
  type = string
}

locals {
  production_instance_types = [
    "t4g.medium",
    "t4g.large",
    "t4g.xlarge"
  ]
}

resource "aws_instance" "app" {
  ami           = "ami-xxxxx"
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition = (
        var.environment != "production" ||
        contains(local.production_instance_types, var.instance_type)
      )
      error_message = "本番環境ではt4g.medium以上のインスタンスタイプを使用してください。"
    }
  }
}
```

**実例3：暗号化の必須チェック**

```hcl
resource "aws_db_instance" "main" {
  identifier        = "app-db"
  engine            = "mysql"
  instance_class    = "db.t3.small"
  storage_encrypted = var.enable_encryption

  lifecycle {
    precondition {
      condition     = var.enable_encryption == true
      error_message = "データベースは必ず暗号化する必要があります。"
    }
  }
}
```

#### 6. postcondition（事後条件チェック）

**目的：** リソース作成後に状態をチェック

**構文：**

```hcl
lifecycle {
  postcondition {
    condition     = 条件式
    error_message = "エラーメッセージ"
  }
}
```

**実例1：パブリックIPの確認**

```hcl
resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"
  subnet_id     = aws_subnet.public.id

  lifecycle {
    postcondition {
      condition     = self.public_ip != ""
      error_message = "WebサーバーにはパブリックIPが必要です。"
    }
  }
}
```

**実例2：マルチAZ構成の確認**

```hcl
resource "aws_db_instance" "production" {
  identifier     = "prod-db"
  engine         = "mysql"
  instance_class = "db.r5.large"
  multi_az       = var.enable_multi_az

  lifecycle {
    postcondition {
      condition     = self.multi_az == true
      error_message = "本番データベースはマルチAZ構成である必要があります。"
    }
  }
}
```

**実例3：セキュリティグループの確認**

```hcl
resource "aws_instance" "web" {
  ami                    = "ami-xxxxx"
  instance_type          = "t4g.small"
  vpc_security_group_ids = var.security_group_ids

  lifecycle {
    postcondition {
      condition     = length(self.vpc_security_group_ids) > 0
      error_message = "インスタンスには少なくとも1つのセキュリティグループが必要です。"
    }
  }
}
```

### lifecycle複数オプションの組み合わせ

```hcl
resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name        = "web-server"
    Environment = "production"
  }

  lifecycle {
    # ダウンタイムを防ぐ
    create_before_destroy = true

    # 手動で追加されるタグを無視
    ignore_changes = [
      tags["Owner"],
      tags["CostCenter"]
    ]

    # 事前条件：本番環境では適切なインスタンスタイプ
    precondition {
      condition = (
        var.environment != "production" ||
        can(regex("^(t4g\\.(medium|large)|m6g\\.)", var.instance_type))
      )
      error_message = "本番環境では t4g.medium 以上または m6g シリーズを使用してください。"
    }

    # 事後条件：パブリックIPが割り当てられている
    postcondition {
      condition     = self.public_ip != null
      error_message = "WebサーバーにはパブリックIPが必要です。"
    }
  }
}
```

## depends_on：依存関係の明示的な指定

### なぜdepends_onが必要なのか

#### 問題：暗黙的な依存関係だけでは不十分

**シナリオ1：IAMロールとインスタンス**

```hcl
# IAMロールの作成
resource "aws_iam_role" "ec2_role" {
  name = "ec2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
}

# IAMロールポリシーのアタッチ
resource "aws_iam_role_policy_attachment" "ec2_s3" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}

# IAMインスタンスプロファイル
resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2-profile"
  role = aws_iam_role.ec2_role.name
}

# EC2インスタンス
resource "aws_instance" "app" {
  ami                  = "ami-xxxxx"
  instance_type        = "t4g.small"
  iam_instance_profile = aws_iam_instance_profile.ec2_profile.name
}
```

**問題点：**

```
terraform applyの実行順序：
1. IAMロール作成
2. インスタンスプロファイル作成
3. EC2インスタンス作成
4. ロールポリシーアタッチ ← これが最後

結果：
- EC2インスタンスが起動
- しかしS3へのアクセス権限がまだない
- アプリケーション起動時にエラー
- 数秒後にポリシーがアタッチされる
- タイミングの問題が発生
```

#### 解決策：depends_onで順序を制御

```hcl
resource "aws_instance" "app" {
  ami                  = "ami-xxxxx"
  instance_type        = "t4g.small"
  iam_instance_profile = aws_iam_instance_profile.ec2_profile.name

  # ポリシーアタッチが完了してからインスタンスを作成
  depends_on = [
    aws_iam_role_policy_attachment.ec2_s3
  ]
}
```

**実際の動作：**

```
terraform applyの実行順序（depends_onあり）：
1. IAMロール作成
2. インスタンスプロファイル作成
3. ロールポリシーアタッチ
4. EC2インスタンス作成 ← ポリシー適用済み

結果：
- EC2インスタンス起動時に権限が完全に設定済み
- アプリケーションが正常に動作
- タイミング問題なし
```

### depends_onの基本構文

```hcl
resource "リソースタイプ" "名前" {
  # リソースの設定

  depends_on = [
    リソース参照1,
    リソース参照2,
    module.モジュール名
  ]
}
```

### 暗黙的依存関係と明示的依存関係の違い

#### 暗黙的依存関係（自動検出）

```hcl
# VPCの作成
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# サブネットの作成
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id  # ← VPC IDを参照
  cidr_block = "10.0.1.0/24"
}

# Terraformが自動的に検出：
# 1. VPCを先に作成
# 2. サブネットを後から作成
# → depends_on 不要
```

**暗黙的依存関係の仕組み：**

```
Terraformは設定ファイルを解析：
- aws_subnet.public.vpc_id が aws_vpc.main.id を参照
- 自動的に依存関係を検出
- 正しい順序で作成

これで十分な場合が多い
```

#### 明示的依存関係（depends_on）

```hcl
# VPCエンドポイントの作成
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.ap-northeast-1.s3"
}

# ルートテーブルの作成
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
}

# VPCエンドポイントとルートテーブルの関連付け
resource "aws_vpc_endpoint_route_table_association" "s3" {
  route_table_id  = aws_route_table.private.id
  vpc_endpoint_id = aws_vpc_endpoint.s3.id

  # エンドポイントが完全に作成されてから関連付け
  depends_on = [
    aws_vpc_endpoint.s3,
    aws_route_table.private
  ]
}
```

**なぜdepends_onが必要か：**

```
このケースの問題：
- リソースIDの参照だけでは不十分
- リソースの「作成完了」を保証したい
- AWS APIの呼び出し順序が重要

depends_onの効果：
- 指定したリソースの作成が完全に終わるまで待つ
- その後で次のリソースを作成
```

### depends_onの実践例

#### 実例1：IAMとEC2の適切な順序

```hcl
# =============================================================================
# IAMロールとポリシー
# =============================================================================

resource "aws_iam_role" "app_role" {
  name = "app-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy" "app_policy" {
  name = "app-policy"
  role = aws_iam_role.app_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "s3:GetObject",
        "s3:ListBucket"
      ]
      Resource = "*"
    }]
  })
}

resource "aws_iam_instance_profile" "app_profile" {
  name = "app-profile"
  role = aws_iam_role.app_role.name
}

# =============================================================================
# EC2インスタンス
# =============================================================================

resource "aws_instance" "app" {
  ami                  = "ami-xxxxx"
  instance_type        = "t4g.small"
  iam_instance_profile = aws_iam_instance_profile.app_profile.name

  user_data = <<-EOF
    #!/bin/bash
    # 起動直後にS3からファイルをダウンロード
    aws s3 cp s3://my-bucket/config.json /etc/app/config.json
  EOF

  # ポリシーが完全に適用されてからインスタンスを起動
  depends_on = [
    aws_iam_role_policy.app_policy
  ]

  tags = {
    Name = "app-server"
  }
}
```

**なぜ必要か：**

```
depends_onがない場合：
1. IAMロール作成
2. インスタンスプロファイル作成
3. EC2インスタンス起動（user_data実行）
4. IAMポリシーアタッチ ← user_data実行後

問題：
- user_dataが実行される時点で権限がない
- S3からのダウンロードが失敗
- アプリケーションが起動しない

depends_onがある場合：
1. IAMロール作成
2. インスタンスプロファイル作成
3. IAMポリシーアタッチ
4. EC2インスタンス起動（権限あり）
→ user_dataが正常に動作
```

#### 実例2：データベースとアプリケーションの順序

```hcl
# =============================================================================
# データベース
# =============================================================================

resource "aws_db_instance" "main" {
  identifier        = "app-db"
  engine            = "mysql"
  instance_class    = "db.t3.small"
  allocated_storage = 20

  db_name  = "appdb"
  username = "admin"
  password = var.db_password

  skip_final_snapshot = true
}

# データベース初期化スクリプト実行
resource "null_resource" "db_init" {
  provisioner "local-exec" {
    command = "mysql -h ${aws_db_instance.main.endpoint} -u admin -p${var.db_password} < init.sql"
  }

  # データベースが利用可能になってから実行
  depends_on = [
    aws_db_instance.main
  ]
}

# =============================================================================
# アプリケーションサーバー
# =============================================================================

resource "aws_instance" "app" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  user_data = <<-EOF
    #!/bin/bash
    echo "DB_HOST=${aws_db_instance.main.endpoint}" > /etc/app/config
    systemctl start app
  EOF

  # データベースの初期化が完了してからアプリケーション起動
  depends_on = [
    null_resource.db_init
  ]

  tags = {
    Name = "app-server"
  }
}
```

**実行順序：**

```
1. aws_db_instance.main 作成
   → データベースインスタンス起動（数分）

2. null_resource.db_init 実行
   → 初期化スクリプト実行（テーブル作成など）

3. aws_instance.app 作成
   → アプリケーション起動（データベース準備済み）

結果：
- アプリケーション起動時にデータベースが完全に準備済み
- 接続エラーなし
```

#### 実例3：モジュール間の依存関係

```hcl
# =============================================================================
# ネットワークモジュール
# =============================================================================

module "network" {
  source = "./modules/network"

  vpc_cidr = "10.0.0.0/16"
}

# =============================================================================
# セキュリティモジュール
# =============================================================================

module "security" {
  source = "./modules/security"

  vpc_id = module.network.vpc_id
}

# =============================================================================
# データベースモジュール
# =============================================================================

module "database" {
  source = "./modules/database"

  vpc_id             = module.network.vpc_id
  subnet_ids         = module.network.private_subnet_ids
  security_group_ids = [module.security.db_security_group_id]

  # セキュリティグループが完全に設定されてから作成
  depends_on = [
    module.security
  ]
}

# =============================================================================
# アプリケーションモジュール
# =============================================================================

module "application" {
  source = "./modules/application"

  vpc_id             = module.network.vpc_id
  subnet_ids         = module.network.public_subnet_ids
  security_group_ids = [module.security.app_security_group_id]
  db_endpoint        = module.database.endpoint

  # データベースが完全に準備されてからアプリケーション起動
  depends_on = [
    module.database
  ]
}
```

#### 実例4：S3バケットポリシーとCloudFront

```hcl
# =============================================================================
# S3バケット
# =============================================================================

resource "aws_s3_bucket" "website" {
  bucket = "my-website-bucket"
}

resource "aws_s3_bucket_public_access_block" "website" {
  bucket = aws_s3_bucket.website.id

  block_public_acls       = true
  block_public_policy     = false  # CloudFrontからのアクセス許可
  ignore_public_acls      = true
  restrict_public_buckets = false
}

# =============================================================================
# CloudFront
# =============================================================================

resource "aws_cloudfront_origin_access_identity" "website" {
  comment = "OAI for website"
}

resource "aws_cloudfront_distribution" "website" {
  enabled = true

  origin {
    domain_name = aws_s3_bucket.website.bucket_regional_domain_name
    origin_id   = "S3-${aws_s3_bucket.website.id}"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.website.cloudfront_access_identity_path
    }
  }

  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-${aws_s3_bucket.website.id}"
    viewer_protocol_policy = "redirect-to-https"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}

# =============================================================================
# S3バケットポリシー
# =============================================================================

resource "aws_s3_bucket_policy" "website" {
  bucket = aws_s3_bucket.website.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid    = "CloudFrontAccess"
      Effect = "Allow"
      Principal = {
        AWS = aws_cloudfront_origin_access_identity.website.iam_arn
      }
      Action   = "s3:GetObject"
      Resource = "${aws_s3_bucket.website.arn}/*"
    }]
  })

  # CloudFrontが完全に作成されてからポリシー適用
  depends_on = [
    aws_cloudfront_distribution.website,
    aws_s3_bucket_public_access_block.website
  ]
}
```

**なぜ必要か：**

```
depends_onがない場合：
1. S3バケット作成
2. CloudFront作成（開始）
3. バケットポリシー適用
4. CloudFront作成（完了）

問題：
- バケットポリシーでCloudFront OAIを参照
- しかしCloudFrontがまだ完全に作成されていない
- ポリシー適用がエラーになる可能性

depends_onがある場合：
1. S3バケット作成
2. CloudFront作成（完了まで待つ）
3. バケットポリシー適用
→ 正常に動作
```

### depends_onを使うべき場合と使わない場合

#### 使うべき場合

```
1. 暗黙的な依存関係が検出されない
   - リソースIDを直接参照していない
   - APIの呼び出し順序が重要

2. 初期化処理の順序が重要
   - データベース初期化 → アプリケーション起動
   - IAMポリシー適用 → リソース作成

3. 外部リソースの準備を待つ必要がある
   - null_resourceのプロビジョナー実行
   - 外部スクリプトの完了待ち

4. モジュール間の順序制御
   - ネットワーク → セキュリティ → アプリケーション
```

#### 使わない場合（暗黙的依存関係で十分）

```hcl
# 暗黙的依存関係で十分な例

# VPC作成
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# サブネット作成
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id  # ← VPC IDを参照
  cidr_block = "10.0.1.0/24"
}

# EC2作成
resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"
  subnet_id     = aws_subnet.public.id  # ← サブネットIDを参照
}

# depends_on不要：
# - Terraformが自動的に順序を決定
# - VPC → サブネット → EC2 の順に作成
```

#### 過度な使用は避ける

```hcl
# 悪い例：不必要なdepends_on

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"

  # これは不要（暗黙的依存関係で十分）
  depends_on = [
    aws_vpc.main
  ]
}
```

**問題点：**

```
不必要なdepends_onの問題：
1. コードが冗長になる
2. 暗黙的依存関係が明確でなくなる
3. メンテナンスが困難に

原則：
- まず暗黙的依存関係を使う
- 不十分な場合のみdepends_onを追加
```

## ベストプラクティス

### 1. lifecycleのベストプラクティス

#### 1-1. create_before_destroyの適切な使用

```hcl
# 良い例：ダウンタイムが許容できないリソース

resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_lb_target_group" "app" {
  name     = "app-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  lifecycle {
    create_before_destroy = true
  }
}
```

**避けるべき：名前の一意制約があるリソース**

```hcl
# 問題がある例

resource "aws_db_instance" "main" {
  identifier = "my-database"  # 一意制約
  # ...

  lifecycle {
    create_before_destroy = true  # エラーになる可能性
  }
}

# 解決策：名前に乱数を含める
resource "random_id" "db" {
  byte_length = 4
}

resource "aws_db_instance" "main" {
  identifier = "my-database-${random_id.db.hex}"
  # ...

  lifecycle {
    create_before_destroy = true  # これでOK
  }
}
```

#### 1-2. prevent_destroyの適切な使用

```hcl
# 良い例：重要なリソースに適用

# 本番データベース
resource "aws_db_instance" "production" {
  identifier = "prod-db"
  # ...

  lifecycle {
    prevent_destroy = true
  }
}

# ステートファイル用S3バケット
resource "aws_s3_bucket" "terraform_state" {
  bucket = "terraform-state-bucket"

  lifecycle {
    prevent_destroy = true
  }
}

# 悪い例：すべてのリソースに適用
resource "aws_instance" "dev" {
  # ...
  lifecycle {
    prevent_destroy = true  # 開発環境では不要
  }
}
```

**環境別の設定：**

```hcl
variable "environment" {
  type = string
}

locals {
  is_production = var.environment == "production"
}

resource "aws_db_instance" "main" {
  identifier = "${var.environment}-db"
  # ...

  lifecycle {
    # 本番環境のみprevent_destroy
    prevent_destroy = local.is_production
  }
}
```

#### 1-3. ignore_changesの最小限の使用

```hcl
# 良い例：必要最小限の属性のみ

resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t4g.small"

  tags = {
    Name = "web-server"
  }

  lifecycle {
    # 特定のタグのみ無視
    ignore_changes = [
      tags["Owner"],
      tags["CostCenter"]
    ]
  }
}

# 悪い例：すべてを無視
resource "aws_instance" "web" {
  # ...

  lifecycle {
    ignore_changes = all  # 避けるべき
  }
}
```

#### 1-4. precondition/postconditionの活用

```hcl
# 良い例：重要な条件を事前チェック

variable "environment" {
  type = string
}

resource "aws_db_instance" "main" {
  identifier        = "${var.environment}-db"
  engine            = "mysql"
  instance_class    = var.db_instance_class
  storage_encrypted = true
  multi_az          = var.enable_multi_az

  lifecycle {
    # 本番環境の要件をチェック
    precondition {
      condition = (
        var.environment != "production" ||
        (var.enable_multi_az == true && can(regex("^db\\.(r5|r6)", var.db_instance_class)))
      )
      error_message = "本番環境ではマルチAZ構成とr5/r6インスタンスクラスが必要です。"
    }

    # 暗号化が有効か確認
    postcondition {
      condition     = self.storage_encrypted == true
      error_message = "データベースストレージは必ず暗号化する必要があります。"
    }
  }
}
```

### 2. depends_onのベストプラクティス

#### 2-1. 暗黙的依存関係を優先

```hcl
# 良い例：IDを参照（暗黙的依存関係）

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id  # これで十分
  cidr_block = "10.0.1.0/24"

  # depends_on不要
}

# 悪い例：不必要なdepends_on
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"

  depends_on = [aws_vpc.main]  # 不要（冗長）
}
```

#### 2-2. depends_onが必要な場合のコメント

```hcl
# 良い例：理由を明記

resource "aws_instance" "app" {
  ami                  = "ami-xxxxx"
  instance_type        = "t4g.small"
  iam_instance_profile = aws_iam_instance_profile.app.name

  # 理由：user_data実行時にIAMポリシーが必要
  # IAMポリシーのアタッチ完了を待つ
  depends_on = [
    aws_iam_role_policy_attachment.app_s3
  ]

  user_data = <<-EOF
    #!/bin/bash
    aws s3 cp s3://my-bucket/config.json /etc/app/
  EOF
}
```

#### 2-3. モジュール間依存関係の明確化

```hcl
# 良い例：依存関係を明確に

module "network" {
  source = "./modules/network"
}

module "database" {
  source = "./modules/database"

  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.private_subnet_ids

  # ネットワークが完全に準備されてから
  depends_on = [module.network]
}

module "application" {
  source = "./modules/application"

  vpc_id      = module.network.vpc_id
  subnet_ids  = module.network.public_subnet_ids
  db_endpoint = module.database.endpoint

  # データベースが利用可能になってから
  depends_on = [module.database]
}
```

### 3. 組み合わせのベストプラクティス

#### 完全な例：本番Webアプリケーション

```hcl
# =============================================================================
# 変数定義
# =============================================================================

variable "environment" {
  type = string
}

variable "instance_type" {
  type = string
}

locals {
  is_production = var.environment == "production"

  production_instance_types = [
    "t4g.medium",
    "t4g.large",
    "m6g.large"
  ]
}

# =============================================================================
# IAMロールとポリシー
# =============================================================================

resource "aws_iam_role" "app_role" {
  name = "${var.environment}-app-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })

  lifecycle {
    # 本番環境では削除防止
    prevent_destroy = local.is_production
  }
}

resource "aws_iam_role_policy" "app_policy" {
  name = "${var.environment}-app-policy"
  role = aws_iam_role.app_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "s3:GetObject",
        "s3:PutObject"
      ]
      Resource = "arn:aws:s3:::${var.environment}-app-data/*"
    }]
  })
}

resource "aws_iam_instance_profile" "app_profile" {
  name = "${var.environment}-app-profile"
  role = aws_iam_role.app_role.name
}

# =============================================================================
# EC2インスタンス
# =============================================================================

resource "aws_instance" "app" {
  ami                  = var.ami_id
  instance_type        = var.instance_type
  iam_instance_profile = aws_iam_instance_profile.app_profile.name
  subnet_id            = var.subnet_id

  tags = {
    Name        = "${var.environment}-app-server"
    Environment = var.environment
  }

  lifecycle {
    # ダウンタイムを防ぐ
    create_before_destroy = true

    # 手動で追加されるタグを無視
    ignore_changes = [
      tags["Owner"],
      tags["Team"]
    ]

    # 本番環境では削除防止
    prevent_destroy = local.is_production

    # 本番環境では適切なインスタンスタイプを要求
    precondition {
      condition = (
        !local.is_production ||
        contains(local.production_instance_types, var.instance_type)
      )
      error_message = "本番環境では ${join(", ", local.production_instance_types)} のいずれかを使用してください。"
    }

    # パブリックIPが割り当てられているか確認
    postcondition {
      condition     = self.public_ip != null && self.public_ip != ""
      error_message = "インスタンスにはパブリックIPが必要です。"
    }
  }

  # IAMポリシーが適用されてからインスタンス起動
  depends_on = [
    aws_iam_role_policy.app_policy
  ]
}

# =============================================================================
# データベース
# =============================================================================

resource "aws_db_instance" "main" {
  identifier        = "${var.environment}-db"
  engine            = "mysql"
  instance_class    = var.db_instance_class
  allocated_storage = 100
  storage_encrypted = true
  multi_az          = local.is_production

  lifecycle {
    # 本番環境では削除防止
    prevent_destroy = local.is_production

    # 本番環境ではマルチAZ必須
    precondition {
      condition = (
        !local.is_production ||
        var.enable_multi_az == true
      )
      error_message = "本番環境ではマルチAZ構成が必要です。"
    }

    # 暗号化確認
    postcondition {
      condition     = self.storage_encrypted == true
      error_message = "データベースは必ず暗号化する必要があります。"
    }
  }
}
```

## トラブルシューティング

### エラー1：create_before_destroyでの名前衝突

```
Error: Error creating DB Instance: DBInstanceAlreadyExists:
Database instance "my-database" already exists.
```

**原因：**

```hcl
resource "aws_db_instance" "main" {
  identifier = "my-database"  # 固定名

  lifecycle {
    create_before_destroy = true
  }
}

# 動作：
# 1. 新しいDBインスタンスを"my-database"で作成しようとする
# 2. しかし既に"my-database"が存在
# 3. エラー
```

**解決方法：**

```hcl
resource "random_id" "db" {
  byte_length = 4

  keepers = {
    # 設定が変わったら新しいIDを生成
    instance_class = var.db_instance_class
  }
}

resource "aws_db_instance" "main" {
  identifier = "my-database-${random_id.db.hex}"
  # ...

  lifecycle {
    create_before_destroy = true
  }
}
```

### エラー2：prevent_destroyで削除できない

```
Error: Instance cannot be destroyed

Resource aws_db_instance.production has lifecycle.prevent_destroy set,
but the plan calls for this resource to be destroyed.
```

**解決方法：**

```hcl
# ステップ1：prevent_destroyを無効化
resource "aws_db_instance" "production" {
  # ...

  lifecycle {
    prevent_destroy = false  # 一時的に無効化
  }
}

# ステップ2：apply
terraform apply

# ステップ3：削除可能
terraform destroy
```

### エラー3：循環依存

```
Error: Cycle: aws_instance.a, aws_instance.b
```

**原因：**

```hcl
resource "aws_instance" "a" {
  # ...
  depends_on = [aws_instance.b]
}

resource "aws_instance" "b" {
  # ...
  depends_on = [aws_instance.a]
}

# AがBに依存、BがAに依存 → 循環
```

**解決方法：**

```hcl
# 依存関係を見直す
resource "aws_instance" "a" {
  # ...
  # depends_on不要
}

resource "aws_instance" "b" {
  # ...
  depends_on = [aws_instance.a]  # 一方向のみ
}
```

## まとめ

### lifecycleブロックの使用

```
create_before_destroy：
- ダウンタイムを防ぐ
- Webサーバー、ロードバランサーに使用
- 名前の一意制約に注意

prevent_destroy：
- 重要なリソースの削除防止
- 本番データベース、ステートファイルに使用
- 環境別に設定を分ける

ignore_changes：
- 特定の属性の変更を無視
- 必要最小限の使用
- 過度な使用は避ける

precondition/postcondition：
- 重要な条件をチェック
- 本番環境の要件を保証
- エラーの早期発見
```

### depends_onの使用

```
暗黙的依存関係を優先：
- リソースIDを参照すればTerraformが自動検出
- 多くの場合はこれで十分

depends_onが必要な場合：
- IAMポリシーの適用待ち
- 初期化処理の順序制御
- モジュール間の依存関係
- 外部リソースの準備待ち

コメントで理由を明記：
- なぜdepends_onが必要か
- どの順序を保証するか
```

### ベストプラクティス

```
1. 必要最小限の使用
   - lifecycleもdepends_onも過度に使わない
   - 本当に必要な場合のみ

2. 環境別の設定
   - 本番環境と開発環境で使い分け
   - 変数と条件式を活用

3. コメントの追加
   - 理由を明記
   - チームメンバーが理解できるように

4. テストの実施
   - planで動作確認
   - 開発環境で先にテスト
   - 段階的に本番環境へ
```
