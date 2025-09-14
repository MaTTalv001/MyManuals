AWS EC2 インスタンスを作成して VS Code から操作する手順をまとめます。

# AWS EC2 + VS Code リモート開発環境構築手順

## 1. 前提準備

### 必要なもの

- AWS アカウント
- VS Code
- SSH クライアント（Windows の場合は OpenSSH）

## 2. AWS キーペアの作成

### 手順

1. AWS マネジメントコンソールにログイン
2. **EC2** サービスに移動
3. 左メニューから「**キーペア**」を選択
4. 「**キーペアを作成**」をクリック
5. 設定項目：
   - **名前**: `my-ec2-key`（任意の名前）
   - **キーペアのタイプ**: RSA
   - **プライベートキーファイル形式**: .pem
6. 「**キーペアを作成**」をクリック
7. **重要**: ダウンロードされる `.pem` ファイルを安全な場所に保存

## 3. セキュリティグループの作成

### 手順

1. EC2 ダッシュボードで「**セキュリティグループ**」を選択
2. 「**セキュリティグループを作成**」をクリック
3. 基本設定：
   - **セキュリティグループ名**: `my-dev-sg`
   - **説明**: `Development environment security group`
   - **VPC**: デフォルト VPC を選択
4. インバウンドルール：
   - **タイプ**: SSH
   - **プロトコル**: TCP
   - **ポート範囲**: 22
   - **ソース**: マイ IP（自分の IP アドレスが自動設定される）
5. 「**セキュリティグループを作成**」をクリック

## 4. EC2 インスタンスの作成

### 手順

1. EC2 ダッシュボードで「**インスタンス**」を選択
2. 「**インスタンスを起動**」をクリック
3. **名前とタグ**:
   - **名前**: `my-development-server`
4. **アプリケーションおよび OS イメージ**:
   - **Amazon Linux 2023 AMI** を選択（推奨）
   - または **Ubuntu Server 22.04 LTS** も可
5. **インスタンスタイプ**:
   - `t2.micro`（無料利用枠対象）または `t3.micro`
6. **キーペア**:
   - 先ほど作成した `my-ec2-key` を選択
7. **ネットワーク設定**:
   - **VPC**: デフォルト VPC のまま
   - **サブネット**: デフォルトのまま
   - **パブリック IP の自動割り当て**: 有効
   - **セキュリティグループ**: 「既存のセキュリティグループを選択」→ `my-dev-sg`
8. **ストレージを設定**:
   - **サイズ**: 8 GiB（デフォルト、無料利用枠内）
   - **タイプ**: gp3
9. 「**インスタンスを起動**」をクリック

## 5. VS Code の設定

### Remote-SSH 拡張機能のインストール

1. VS Code を開く
2. 拡張機能タブ（Ctrl+Shift+X）を開く
3. "Remote - SSH" を検索してインストール
4. "Remote Development" 拡張機能パックもおすすめ

### SSH 設定ファイルの作成

1. VS Code で `Ctrl+Shift+P` を押してコマンドパレットを開く
2. `Remote-SSH: Open SSH Configuration File` を検索・選択
3. ユーザー設定ファイル（`~/.ssh/config`）を選択
4. 以下の設定を追加：

```
Host my-ec2
    HostName YOUR_EC2_PUBLIC_IP
    User ec2-user
    IdentityFile /path/to/your/my-ec2-key.pem
    ServerAliveInterval 60
```

**注意点**:

- `YOUR_EC2_PUBLIC_IP`: EC2 インスタンスのパブリック IP アドレスに置き換え
- `/path/to/your/my-ec2-key.pem`: ダウンロードした.pem ファイルの実際のパスに置き換え
- Ubuntu を使用した場合は `User ubuntu` に変更

## 6. SSH キーファイルの権限設定

### Windows の場合

PowerShell で以下を実行：

```powershell
# キーファイルの権限を設定
icacls "C:\path\to\your\my-ec2-key.pem" /inheritance:r
icacls "C:\path\to\your\my-ec2-key.pem" /grant:r "%USERNAME%:R"
```

### macOS/Linux の場合

ターミナルで以下を実行：

```bash
chmod 400 /path/to/your/my-ec2-key.pem
```

## 7. VS Code でリモート接続

### 接続手順

1. VS Code で `Ctrl+Shift+P`
2. `Remote-SSH: Connect to Host` を検索・選択
3. `my-ec2` を選択
4. 新しい VS Code ウィンドウが開き、EC2 に接続される
5. 左下に `SSH: my-ec2` と表示されれば成功

### 初回接続時の注意

- フィンガープリントの確認ダイアログが出たら「Continue」を選択
- 初回は少し時間がかかる場合があります

## 8. 開発環境のセットアップ

EC2 に接続後、必要な開発ツールをインストール：

### 基本ツール（Amazon Linux 2023 の場合）

```bash
# システム更新
sudo dnf update -y

# 基本的な開発ツール
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y git curl wget

# Node.js（例）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc
nvm install node

# Python（例）
sudo dnf install -y python3 python3-pip
```

## 9. セキュリティ設定の確認

### 重要なセキュリティポイント

- **キーファイルの管理**: .pem ファイルは絶対に他人に渡さない
- **セキュリティグループ**: SSH 接続は自分の IP アドレスからのみ許可
- **定期的な更新**: EC2 インスタンスの OS を定期的に更新
- **不要時の停止**: 使わない時はインスタンスを停止してコスト削減

### インスタンスの操作

- **停止**: EC2 コンソールでインスタンスを選択 → アクション → インスタンスの状態 → 停止
- **開始**: EC2 コンソールでインスタンスを選択 → アクション → インスタンスの状態 → 開始
- **注意**: 停止/開始するとパブリック IP が変わるため、SSH 設定の更新が必要
