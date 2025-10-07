# Terraformブロック理解度チェック

## このクイズについて

先ほど学習したTerraformの7つの基本ブロックについて、理解度を確認しましょう。実際のコードを見ながら答えてください。

---

## 問題1：terraformブロック

以下のコードを見て答えてください：

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

**質問：** `version = "~> 6.0"`の意味として正しいものはどれですか？

A) バージョン6.0のみを使用  
B) バージョン6.0以上であれば何でも使用  
C) バージョン6.x.xシリーズの最新を使用（7.0は使用しない）  
D) バージョン6.0未満を使用

---

## 問題2：providerブロック

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

**質問：** `default_tags`を使用する最大の利点は何ですか？

A) AWS料金を削減できる  
B) 全てのAWSリソースに自動的にタグが適用され、タグの付け忘れを防げる  
C) リソースの作成速度が向上する  
D) セキュリティが強化される

---

## 問題3：variableブロック

```hcl
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.88.0.0/16"
}
```

**質問：** このvariableブロックで、`default`を指定する理由として最も適切なものはどれですか？

A) 必須項目だから  
B) 値が指定されなかった場合の初期値を提供するため  
C) セキュリティを向上させるため  
D) エラーを防ぐため

---

## 問題4：dataブロック

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}
```

**質問：** このdataブロックの役割として正しいものはどれですか？

A) 新しいアベイラビリティゾーンを作成する  
B) 既存の利用可能なアベイラビリティゾーン情報をAWSから取得する  
C) アベイラビリティゾーンを削除する  
D) アベイラビリティゾーンの設定を変更する

---

## 問題5：localsブロック

```hcl
locals {
  availability_zones = slice(data.aws_availability_zones.available.names, 0, 2)
  public_subnet_cidrs  = ["10.88.1.0/24", "10.88.2.0/24"]
  private_subnet_cidrs = ["10.88.11.0/24", "10.88.12.0/24"]
}
```

**質問：** なぜ今回のコードでは`common_tags`をlocalsから削除したのですか？

A) common_tagsは不要だから  
B) providerの`default_tags`が自動的にタグを適用するため、重複を避けるため  
C) エラーが発生するから  
D) AWSでサポートされていないから

---

## 問題6：resourceブロック

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}
```

**質問：** このresourceブロックのtagsで`Environment`や`Project`を指定していない理由は何ですか？

A) 不要だから  
B) エラーになるから  
C) `default_tags`で自動的に適用されるから  
D) AWSが自動で設定するから

---

## 問題7：outputブロック

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}
```

**質問：** outputブロックを使用する主な目的として正しいものはどれですか？

A) リソースを作成するため  
B) 作成したリソースの重要な情報を他のモジュールや人が使用できるように出力するため  
C) 設定を保存するため  
D) エラーをチェックするため

---

## 問題8：ブロックの順序

**質問：** 以下のTerraformブロックを、コード内で記述する推奨順序に並べてください：

1. resource
2. terraform
3. output
4. locals
5. variable
6. data
7. provider

正しい順序は？

A) 2→7→5→6→4→1→3  
B) 5→2→7→6→4→1→3  
C) 2→7→6→5→4→1→3  
D) 7→2→5→6→4→1→3

---

## 問題9：実践的理解

あなたが今回のコードを`terraform apply`で実行した結果、以下のリソースが作成されます：

**質問：** 作成されるサブネットの合計数はいくつですか？

A) 2個  
B) 4個  
C) 6個  
D) 8個

---

## 問題10：ベストプラクティス確認

**質問：** 今回学んだコードで実装されているベストプラクティスとして正しいものを全て選んでください（複数回答）：

A) バージョン制約の指定  
B) default_tagsの使用  
C) 変数の活用  
D) データソースの動的取得  
E) countを使用した繰り返し処理  
F) 適切なタグ付け

---

## 解答と解説

### 問題1：正解 C

**解説：** `~> 6.0`は「悲観的バージョン制約」と呼ばれ、6.0以上7.0未満（6.x.xシリーズ）の最新版を使用することを意味します。メジャーバージョンアップによる破壊的変更を避けながら、バグ修正や機能追加の恩恵を受けられます。

### 問題2：正解 B

**解説：** `default_tags`は全てのAWSリソースに自動的にタグを適用する機能です。手動でタグを付け忘れることを防ぎ、一貫したリソース管理を実現します。コスト分析や環境識別に重要な役割を果たします。

### 問題3：正解 B

**解説：** `default`値は、変数に値が指定されなかった場合に使用される初期値です。これにより、毎回値を指定する必要がなく、コードの再利用性が向上します。

### 問題4：正解 B

**解説：** dataブロックは既存のAWS情報を取得するために使用します。この例では、利用可能なアベイラビリティゾーンの一覧をAWSから動的に取得しています。

### 問題5：正解 B

**解説：** `default_tags`で`Environment`、`Project`、`ManagedBy`タグが自動適用されるため、localsで`common_tags`を定義してmergeする必要がありません。DRY原則に従い、重複を避けています。

### 問題6：正解 C

**解説：** providerの`default_tags`設定により、`Environment`、`Project`、`ManagedBy`タグは自動的に全てのAWSリソースに適用されます。そのため、個別に指定する必要がありません。

### 問題7：正解 B

**解説：** outputブロックは、作成したリソースの重要な情報（IDやARNなど）を出力し、他のTerraformモジュールや外部システムから参照できるようにします。

### 問題8：正解 A

**解説：** 推奨順序は：terraform→provider→variable→data→locals→resource→output です。設定から始まり、データ取得、計算、リソース作成、結果出力の流れになります。

### 問題9：正解 B

**解説：** パブリックサブネット2個（count = 2）+ プライベートサブネット2個（count = 2）= 合計4個のサブネットが作成されます。

### 問題10：正解 A、B、C、D、E、F（全て）

**解説：**

- A: `required_version`と`version`制約
- B: `default_tags`の使用
- C: variableブロックでの設定の外部化
- D: `aws_availability_zones`データソース
- E: サブネット作成でのcount使用
- F: 適切なName、Typeタグの設定
