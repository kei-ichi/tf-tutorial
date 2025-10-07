# Terraformのインストールとプロジェクト作成

## この章で学ぶこと

このレッスンでは、以下のことを学びます：

- 自分のコンピュータにTerraformをインストールする方法
- 初めてのTerraformプロジェクトを作成する方法
- 基本的なコマンドの使い方

## 準備するもの

- インターネットに接続されたコンピュータ
- コマンドライン（ターミナル）の基本的な使い方
- テキストエディタ（メモ帳でも大丈夫です）

## Terraformのインストール

新しいツールを使い始める時は、まずそのツールを自分のコンピュータに入れる必要があります。これを「インストール」と言います。

### macOSでのインストール

#### 方法1：Homebrewを使う（推奨）

Homebrewは、macOSでソフトウェアを簡単にインストールするためのツールです。

1. ターミナルを開く
2. 以下のコマンドを入力する：

```bash
brew install terraform
```

3. インストールが完了したら確認：

```bash
terraform --version
```

#### 方法2：公式サイトからダウンロード

1. https://developer.hashicorp.com/terraform/downloads にアクセス
2. macOS版をダウンロード
3. ダウンロードしたファイルを解凍
4. terraformファイルを `/usr/local/bin/` フォルダに移動

### Windowsでのインストール

#### 方法1：Chocolateyを使う（推奨）

Chocolateyは、Windowsでソフトウェアをコマンドでインストールするツールです。

1. 管理者権限でPowerShellを開く
2. 以下のコマンドを入力：

```powershell
choco install terraform
```

3. インストールが完了したら確認：

```powershell
terraform --version
```

#### 方法2：公式サイトからダウンロード

1. https://developer.hashicorp.com/terraform/downloads にアクセス
2. Windows版をダウンロード
3. ダウンロードしたzipファイルを解凍
4. terraformファイルを適当なフォルダに置く（例：`C:\terraform\`）
5. 環境変数PATHにそのフォルダを追加

**環境変数の設定方法：**

1. 「設定」→「システム」→「詳細情報」→「システムの詳細設定」
2. 「環境変数」ボタンをクリック
3. 「システム環境変数」の「Path」を選択して「編集」
4. 「新規」をクリックしてterraformファイルがあるフォルダのパスを入力

### Linuxでのインストール

#### Ubuntu/Debianの場合

1. パッケージリストを更新：

```bash
sudo apt-get update
```

2. 必要なパッケージをインストール：

```bash
sudo apt-get install -y gnupg software-properties-common
```

3. HashiCorpのGPGキーを追加：

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
```

4. HashiCorpのリポジトリを追加：

```bash
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
```

5. パッケージリストを再度更新：

```bash
sudo apt-get update
```

6. Terraformをインストール：

```bash
sudo apt-get install terraform
```

7. インストール確認：

```bash
terraform --version
```

#### CentOS/RHELの場合

1. HashiCorpのリポジトリを追加：

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
```

2. Terraformをインストール：

```bash
sudo yum -y install terraform
```

3. インストール確認：

```bash
terraform --version
```

### インストールの確認

どのOSでも、最後に以下のコマンドでTerraformが正しくインストールされたか確認しましょう：

```bash
terraform --version
```

以下のような出力が表示されれば成功です：

```
Terraform v1.6.0
on darwin_amd64
```

## 初めてのTerraformプロジェクトを作る

図書館で新しい本棚を作ることを想像してください。まず、どこに作るかを決めて、必要な材料を用意する必要があります。Terraformプロジェクトも同じです。

### ステップ1：プロジェクト用フォルダを作る

1. 新しいフォルダを作りましょう：

**Windows:**

```powershell
mkdir my-first-terraform
cd my-first-terraform
```

**macOS/Linux:**

```bash
mkdir my-first-terraform
cd my-first-terraform
```

### ステップ2：設定ファイルを作る

Terraformの設定は、`.tf`という拡張子のファイルに書きます。

`main.tf`という名前でファイルを作成し、以下の内容を入力してください：

```hcl
# これは私の最初のTerraformプロジェクトです
terraform {
  required_version = ">= 1.0"
  required_providers {
    random = {
      source  = "hashicorp/random"
      version = "~> 3.1"
    }
  }
}

# ランダムな文字列を作るリソース
resource "random_string" "example" {
  length  = 8
  special = false
  upper   = false
}

# 結果を表示する
output "random_string_value" {
  value = random_string.example.result
}
```

**このコードの説明：**

- `terraform {}`：Terraformの基本設定
- `resource`：作りたいもの（今回はランダムな文字列）
- `output`：結果を表示するための設定

### ステップ3：プロジェクトを初期化する

新しいプロジェクトを始める時は、まず「初期化」が必要です。これは、必要な準備を整えることです。

```bash
terraform init
```

以下のような出力が表示されます：

```
Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/random versions matching "~> 3.1"...
- Installing hashicorp/random v3.4.3...
- Installed hashicorp/random v3.4.3

Terraform has been successfully initialized!
```

この出力が表示されれば成功です！

### ステップ4：プランを確認する

実際に何かを作る前に、「何が作られるのか」を確認しましょう。これを「プラン」と言います。

```bash
terraform plan
```

以下のような出力が表示されます：

```
Terraform will perform the following actions:

  # random_string.example will be created
  + resource "random_string" "example" {
      + id      = (known after apply)
      + length  = 8
      + lower   = true
      + number  = true
      + result  = (known after apply)
      + special = false
      + upper   = false
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

これは「8文字のランダム文字列を1つ作る」という意味です。

### ステップ5：実際に作る

プランが問題なければ、実際に作ってみましょう：

```bash
terraform apply
```

確認メッセージが表示されます：

```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```

`yes`と入力してEnterを押してください。

成功すると以下のような出力が表示されます：

```
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

random_string_value = "abc123de"
```

おめでとうございます！初めてのTerraformリソースが作成されました！

### ステップ6：作ったものを削除する

テストが終わったら、作ったリソースを削除しましょう：

```bash
terraform destroy
```

確認メッセージで`yes`と入力すると、リソースが削除されます。

## プロジェクトの構造を理解する

現在のフォルダには以下のファイルができています：

```
my-first-terraform/
├── main.tf                    # 設定ファイル
├── .terraform/               # Terraformが使う作業フォルダ
├── terraform.tfstate         # 現在の状態を記録するファイル
└── terraform.tfstate.backup  # 状態ファイルのバックアップ
```

**重要なファイル：**

- `main.tf`：あなたが書いた設定
- `terraform.tfstate`：現在何が作られているかの記録（とても重要！）

## よくある問題と解決方法

### 問題1：「terraform command not found」エラー

**原因：**Terraformが正しくインストールされていない
**解決方法：**インストール手順をもう一度確認してください

### 問題2：「permission denied」エラー

**原因：**ファイルやフォルダにアクセスする権限がない
**解決方法：**管理者権限でコマンドを実行するか、別の場所にプロジェクトを作ってください

### 問題3：プランが表示されない

**原因：**`terraform init`を実行していない
**解決方法：**まず`terraform init`を実行してください

## 次のステップのために

次のレッスンでは、AWSを使って実際のクラウドリソースを作ります。そのために：

1. AWSアカウントを準備してください（無料で作成できます）
2. このプロジェクトのフォルダは残しておいてください
3. 基本的なコマンド（init、plan、apply、destroy）を覚えておいてください

## まとめ

- Terraformのインストール方法はOSによって違います
- 新しいプロジェクトは`terraform init`で始める
- `terraform plan`で実行内容を確認できる
- `terraform apply`で実際にリソースを作る
- `terraform destroy`でリソースを削除できる
- `.tf`ファイルに設定を書く
- `terraform.tfstate`ファイルは絶対に削除しないでください

次回は、AWSでサーバーを作ってみましょう！

---

**English Commentary:**

**Pedagogical Approach:**

1. **OS-Inclusive Design**: Provided installation instructions for all major operating systems with both package manager and manual installation options. This ensures accessibility regardless of the learner's environment.

2. **Progressive Complexity**: Started with a simple Random provider example instead of cloud resources to avoid overwhelming beginners with AWS concepts before they understand Terraform basics.

3. **Step-by-Step Verification**: Each major step includes verification commands and expected output, allowing learners to confirm success before proceeding.

4. **Error Prevention**: Included common troubleshooting scenarios that beginners frequently encounter, with clear explanations and solutions.

**Technical Considerations:**

- Used the Random provider for the first example as it requires no external dependencies or credentials
- Emphasized the importance of the state file without overwhelming beginners with technical details
- Included proper cleanup procedures to establish good habits early

**Cultural Adaptations:**

- Used library organization analogy which resonates with Japanese systematic thinking
- Structured instructions in clear, numbered steps following Japanese educational preferences
- Included comprehensive troubleshooting section, which Japanese learners particularly appreciate

**Safety and Best Practices:**

- Emphasized state file importance
- Introduced destroy command early to prevent resource accumulation
- Used version constraints in the example code
- Established proper project structure from the beginning

The tutorial builds confidence through immediate success while preparing learners for more complex AWS scenarios in the next lesson.
