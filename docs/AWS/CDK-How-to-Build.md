# AWS CDK Python チュートリアル

## 概要

AWS CDK（Cloud Development Kit）を使用して Python で AWS インフラを構築する完全ガイド。
EC2、S3、Lambda、IAM を組み合わせたシンプルな Web アプリケーション環境を構築します。

## 学習目標

- CDK の基本概念と仕組みを理解
- Python を使ったインフラストラクチャ as Code
- AWS サービス間の連携（EC2→S3、EC2→Lambda）
- IAM ロールと権限管理
- CDK のデプロイ・削除サイクル

## 前提条件

- AWS アカウント（無料利用枠で十分）
- macOS/Linux/Windows（本ガイドは macOS/Linux 想定）
- 基本的なターミナル操作の知識

---

## 1. 環境準備

### 1.1 必要ツールのインストール

#### Python 3.8 以上

```bash
# バージョン確認
python3 --version

# macOSの場合（Homebrewでインストール）
brew install python

# Ubuntuの場合
sudo apt update && sudo apt install -y python3 python3-pip python3-venv
```

**なぜ必要？**: CDK の Python ライブラリを使用するため

#### Node.js 18 以上

```bash
# バージョン確認
node --version
npm --version

# macOSの場合
brew install node

# Ubuntuの場合
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs
```

**なぜ Node.js？**: CDK CLI が Node.js 製のため、Python でコードを書く場合でも必要

#### Git

```bash
# バージョン確認
git --version

# macOSの場合
brew install git

# Ubuntuの場合
sudo apt install -y git
```

---

## 2. AWS CLI 設定

### 2.1 AWS CLI インストール

#### macOS

```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

#### Linux

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

#### インストール確認

```bash
aws --version
# 例: aws-cli/2.x.x Python/3.x.x Linux/5.x.x botocore/2.x.x
```

### 2.2 AWS 認証情報の取得

1. AWS コンソールにログイン
2. 右上のアカウント名 → 「Security credentials」
3. 「Access keys」セクションで「Create access key」
4. 「Command Line Interface (CLI)」を選択
5. Access Key ID と Secret Access Key をメモ

**セキュリティ注意**: アクセスキーは他人に見せない、GitHub にプッシュしない

### 2.3 AWS CLI 設定

```bash
aws configure
```

対話形式で以下を入力：

- `AWS Access Key ID`: 取得した Access Key ID
- `AWS Secret Access Key`: 取得した Secret Access Key
- `Default region name`: `us-east-1` (推奨、料金が安い)
- `Default output format`: `json`

### 2.4 認証確認

```bash
aws sts get-caller-identity
```

レスポンス例:

```json
{
  "UserId": "AIDACKCEVSQ6C2EXAMPLE",
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:user/username"
}
```

**意味**: AWS アカウントへの接続が正常に行われている

---

## 3. CDK CLI インストール

### 3.1 CDK CLI インストール

```bash
npm install -g aws-cdk
```

**なぜグローバルインストール？**: CDK コマンドをどのディレクトリからでも実行可能にするため

### 3.2 バージョン確認

```bash
cdk --version
# 例: 2.100.0 (build abcd123)
```

### 3.3 テレメトリー無効化（オプション）

```bash
cdk acknowledge 34892
```

**意味**: CDK の使用統計収集通知を非表示にする

---

## 4. CDK プロジェクト作成

### 4.1 プロジェクトディレクトリ作成

```bash
mkdir simple-cdk-app
cd simple-cdk-app
```

### 4.2 CDK 初期化

```bash
cdk init app --language python
```

**何が起こる？**:

- `app.py`: エントリーポイント（CDK アプリケーションの開始点）
- `cdk.json`: CDK 設定ファイル
- `requirements.txt`: Python 依存関係
- `simple_cdk_app/`: メインパッケージディレクトリ
- `.venv/`: Python 仮想環境

### 4.3 生成されたファイル確認

```bash
ls -la
```

期待される構造:

```
├── README.md
├── app.py                    # エントリーポイント
├── cdk.json                  # CDK設定
├── requirements.txt          # Python依存関係
├── simple_cdk_app/          # メインモジュール
│   ├── __init__.py
│   └── simple_cdk_app_stack.py  # スタック定義
├── tests/                   # テストディレクトリ
└── .venv/                   # Python仮想環境
```

---

## 5. Python 環境設定

### 5.1 仮想環境有効化

```bash
source .venv/bin/activate
```

**プロンプト変化**: `(.venv)` が先頭に表示される

**なぜ仮想環境？**: プロジェクト固有の依存関係を分離し、他の Python プロジェクトと衝突を避けるため

### 5.2 依存関係インストール

```bash
pip install -r requirements.txt
```

### 5.3 インストール確認

```bash
pip list | grep aws-cdk
```

期待される出力:

```
aws-cdk-lib      2.100.0
```

---

## 6. CDK 環境初期化（Bootstrap）

### 6.1 Bootstrap 実行

```bash
cdk bootstrap
```

**何が起こる？**:

- S3 バケット作成（CDK の成果物保存用）
- IAM ロール作成（CDK 実行用）
- CloudFormation スタック作成（CDKToolkit）

期待される出力:

```
⏳  Bootstrapping environment aws://123456789012/us-east-1...
✅  Environment aws://123456789012/us-east-1 bootstrapped.
```

**一度だけ実行**: 同じ AWS アカウント・リージョンでは初回のみ

---

## 7. スタック実装

### 7.1 スタックファイルの編集

`simple_cdk_app/simple_cdk_app_stack.py` を以下の内容に置き換える:

```python
import aws_cdk as cdk
from aws_cdk import (
    Stack,
    aws_ec2 as ec2,
    aws_s3 as s3,
    aws_lambda as _lambda,
    aws_iam as iam,
    CfnOutput, Duration
)
from constructs import Construct

class SimpleCdkAppStack(Stack):

    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        # ==== VPC設定（シンプル） ====
        vpc = ec2.Vpc(
            self, "SimpleVPC",
            max_azs=1,                    # 1つのAZのみ使用
            cidr="10.0.0.0/24",          # 狭いネットワーク（256個のIP）
            subnet_configuration=[
                ec2.SubnetConfiguration(
                    name="Public",
                    subnet_type=ec2.SubnetType.PUBLIC,
                    cidr_mask=26          # /26 = 64個のIPアドレス
                )
            ],
            nat_gateways=0                # NATゲートウェイ不要（コスト削減）
        )

        # ==== S3バケット（静的ファイル用） ====
        static_bucket = s3.Bucket(
            self, "StaticBucket",
            bucket_name=f"simple-app-static-{self.account}-{self.region}",
            versioned=False,              # シンプルにバージョニングなし
            encryption=s3.BucketEncryption.S3_MANAGED,
            block_public_access=s3.BlockPublicAccess.BLOCK_ALL,
            removal_policy=cdk.RemovalPolicy.DESTROY  # 開発用（削除可能）
        )

        # ==== IAMロール設定 ====
        # EC2用ロール
        ec2_role = iam.Role(
            self, "EC2Role",
            assumed_by=iam.ServicePrincipal("ec2.amazonaws.com"),
            managed_policies=[
                # EC2がSSMで管理できるように
                iam.ManagedPolicy.from_aws_managed_policy_name("AmazonSSMManagedInstanceCore")
            ]
        )

        # EC2にS3読み取り権限を付与
        static_bucket.grant_read(ec2_role)

        # Lambda用ロール
        lambda_role = iam.Role(
            self, "LambdaRole",
            assumed_by=iam.ServicePrincipal("lambda.amazonaws.com"),
            managed_policies=[
                iam.ManagedPolicy.from_aws_managed_policy_name("service-role/AWSLambdaBasicExecutionRole")
            ]
        )

        # LambdaにS3書き込み権限を付与
        static_bucket.grant_write(lambda_role)

        # ==== Lambda関数 ====
        hello_lambda = _lambda.Function(
            self, "HelloLambda",
            runtime=_lambda.Runtime.PYTHON_3_11,
            handler="index.handler",
            code=_lambda.Code.from_inline("""
import json
import boto3
from datetime import datetime

def handler(event, context):
    s3 = boto3.client('s3')
    bucket_name = event.get('bucket_name', 'default-bucket')

    # 現在時刻をファイルに保存
    content = f"Hello from Lambda! Time: {datetime.now().isoformat()}"

    try:
        s3.put_object(
            Bucket=bucket_name,
            Key='lambda-output.txt',
            Body=content,
            ContentType='text/plain'
        )

        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'File uploaded successfully!',
                'content': content
            })
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({
                'error': str(e)
            })
        }
            """),
            role=lambda_role,
            timeout=Duration.seconds(30),
            memory_size=128               # 最小メモリ（コスト削減）
        )

        # ==== セキュリティグループ ====
        web_security_group = ec2.SecurityGroup(
            self, "WebSecurityGroup",
            vpc=vpc,
            description="Security group for web server",
            allow_all_outbound=True
        )

        # HTTP（80）とSSH（22）を許可
        web_security_group.add_ingress_rule(
            ec2.Peer.any_ipv4(),
            ec2.Port.tcp(80),
            "Allow HTTP access"
        )

        web_security_group.add_ingress_rule(
            ec2.Peer.any_ipv4(),
            ec2.Port.tcp(22),
            "Allow SSH access"
        )

        # ==== EC2インスタンス ====
        # 起動時スクリプト
        user_data = ec2.UserData.for_linux()
        user_data.add_commands(
            "yum update -y",
            "yum install -y httpd python3 python3-pip",
            "pip3 install boto3",

            # 簡単なWebページを作成
            "systemctl start httpd",
            "systemctl enable httpd",

            # HTMLファイル作成
            "cat > /var/www/html/index.html << 'HTML_EOF'",
            "<!DOCTYPE html>",
            "<html>",
            "<head><title>Simple CDK App</title></head>",
            "<body>",
            "<h1>Hello from CDK!</h1>",
            "<p>EC2 instance is running!</p>",
            f"<p>S3 Bucket: {static_bucket.bucket_name}</p>",
            f"<p>Lambda Function: {hello_lambda.function_name}</p>",
            "<p>Check the Lambda function to see S3 integration!</p>",
            "</body>",
            "</html>",
            "HTML_EOF",

            # Lambdaを呼び出すPythonスクリプト
            "cat > /home/ec2-user/call_lambda.py << 'PYTHON_EOF'",
            "import boto3",
            "import json",
            "",
            "def call_lambda():",
            "    client = boto3.client('lambda')",
            "    response = client.invoke(",
            f"        FunctionName='{hello_lambda.function_name}',",
            "        Payload=json.dumps({",
            f"            'bucket_name': '{static_bucket.bucket_name}'",
            "        })",
            "    )",
            "    result = json.loads(response['Payload'].read())",
            "    print('Lambda response:', result)",
            "",
            "if __name__ == '__main__':",
            "    call_lambda()",
            "PYTHON_EOF",

            "chown ec2-user:ec2-user /home/ec2-user/call_lambda.py",
            "chmod +x /home/ec2-user/call_lambda.py"
        )

        # EC2インスタンス作成
        instance = ec2.Instance(
            self, "WebServer",
            vpc=vpc,
            instance_type=ec2.InstanceType.of(
                ec2.InstanceClass.T3,
                ec2.InstanceSize.MICRO     # t3.micro（無料利用枠対象）
            ),
            machine_image=ec2.MachineImage.latest_amazon_linux2(),
            security_group=web_security_group,
            role=ec2_role,
            user_data=user_data,
            vpc_subnets=ec2.SubnetSelection(
                subnet_type=ec2.SubnetType.PUBLIC
            )
        )

        # EC2にLambda実行権限を追加
        lambda_invoke_policy = iam.PolicyStatement(
            effect=iam.Effect.ALLOW,
            actions=["lambda:InvokeFunction"],
            resources=[hello_lambda.function_arn]
        )
        ec2_role.add_to_policy(lambda_invoke_policy)

        # ==== 出力 ====
        CfnOutput(self, "InstancePublicIP",
            value=instance.instance_public_ip,
            description="EC2 Public IP Address"
        )

        CfnOutput(self, "WebsiteURL",
            value=f"http://{instance.instance_public_ip}",
            description="Website URL"
        )

        CfnOutput(self, "S3BucketName",
            value=static_bucket.bucket_name,
            description="S3 Bucket Name"
        )

        CfnOutput(self, "LambdaFunctionName",
            value=hello_lambda.function_name,
            description="Lambda Function Name"
        )

        CfnOutput(self, "SSHCommand",
            value=f"aws ssm start-session --target {instance.instance_id}",
            description="SSH command using SSM (no key pair needed)"
        )
```

### 7.2 コードの解説

**主要コンポーネント**:

1. **VPC**: 仮想ネットワーク（10.0.0.0/24、パブリックサブネットのみ）
2. **S3 バケット**: 静的ファイル保存用、一意な名前で作成
3. **IAM ロール**: EC2 と Lambda が必要な AWS サービスにアクセスするための権限
4. **Lambda 関数**: 現在時刻を S3 に保存するシンプルな処理
5. **セキュリティグループ**: HTTP(80)と SSH(22)の通信を許可
6. **EC2 インスタンス**: Web サーバー（Apache）と Lambda 呼び出しスクリプト

**権限設計**:

- EC2 → S3 読み取り、Lambda 実行、SSM 管理
- Lambda → S3 書き込み

---

## 8. デプロイ

### 8.1 構文チェック

```bash
python -m py_compile simple_cdk_app/simple_cdk_app_stack.py
```

**エラーなしの場合**: 何も出力されない

### 8.2 CloudFormation テンプレート生成確認

```bash
cdk synth
```

**意味**: CDK コードを CloudFormation テンプレート（JSON）に変換
**大量の JSON が出力されれば成功**

### 8.3 差分確認

```bash
cdk diff
```

**何が表示される？**:

- 作成される AWS リソース一覧
- IAM ロールと権限の変更
- セキュリティグループのルール

### 8.4 デプロイ実行

```bash
cdk deploy
```

**確認プロンプトで 'y' を入力**

**デプロイ時間**: 約 5-10 分（EC2 の起動に時間がかかるため）

成功時の出力例:

```
✅  SimpleCdkAppStack

Outputs:
SimpleCdkAppStack.InstancePublicIP = 3.80.123.45
SimpleCdkAppStack.WebsiteURL = http://3.80.123.45
SimpleCdkAppStack.S3BucketName = simple-app-static-123456789012-us-east-1
SimpleCdkAppStack.LambdaFunctionName = SimpleCdkAppStack-HelloLambda12345-abcdef
SimpleCdkAppStack.SSHCommand = aws ssm start-session --target i-1234567890abcdef0
```

---

## 9. 動作確認

### 9.1 Web サイト確認

出力された`WebsiteURL`をブラウザで開く（EC2 初期化に 2-3 分要する場合がある）

期待される表示:

```
Hello from CDK!
EC2 instance is running!
S3 Bucket: simple-app-static-123456789012-us-east-1
Lambda Function: SimpleCdkAppStack-HelloLambda12345-abcdef
Check the Lambda function to see S3 integration!
```

### 9.2 Lambda 関数の動作確認

#### SSM で EC2 にログイン

```bash
aws ssm start-session --target YOUR_INSTANCE_ID
```

**SSM の利点**: キーペア不要でセキュアな接続

#### Lambda 関数実行

```bash
python3 /home/ec2-user/call_lambda.py
```

期待される出力:

```
Lambda response: {'statusCode': 200, 'body': '{"message": "File uploaded successfully!", "content": "Hello from Lambda! Time: 2024-01-15T10:30:00"}'}
```

#### セッション終了

```bash
exit
```

### 9.3 S3 確認

#### バケット内容確認

```bash
aws s3 ls s3://YOUR_BUCKET_NAME/
```

期待される出力:

```
2024-01-15 10:30:00         50 lambda-output.txt
```

#### ファイル内容確認

```bash
aws s3 cp s3://YOUR_BUCKET_NAME/lambda-output.txt -
```

期待される出力:

```
Hello from Lambda! Time: 2024-01-15T10:30:00
```

---

## 10. リソース削除

### 10.1 削除実行

```bash
cdk destroy
```

**確認プロンプトで 'y' を入力**

### 10.2 削除完了確認

```bash
aws cloudformation list-stacks --stack-status-filter DELETE_COMPLETE
```

### 10.3 料金確認

AWS コンソール → Billing → Bills で削除後の料金を確認
（削除から 24 時間後に反映）

---

## 11. トラブルシューティング

### 11.1 よくあるエラー

#### 権限不足

```
User: arn:aws:iam::xxx:user/xxx is not authorized to perform: sts:AssumeRole
```

**解決策**: IAM ユーザーに PowerUserAccess ポリシーを付与

#### S3 バケット名重複

```
BucketAlreadyExists: The requested bucket name is not available
```

**解決策**: CDK が自動生成する名前を使用（通常は発生しない）

#### Node.js バージョンエラー

```
Node.js version must be >= 18.0.0
```

**解決策**: Node.js 最新版をインストール

### 11.2 デバッグ用コマンド

#### AWS 認証情報確認

```bash
aws sts get-caller-identity
```

#### CDK 設定確認

```bash
cat cdk.json
```

#### CloudFormation スタック状況確認

```bash
aws cloudformation describe-stacks --stack-name SimpleCdkAppStack
```

---

## 12. ポイント

### 12.1 CDK の利点

- **型安全性**: IDE 補完とコンパイル時エラー検出
- **再利用性**: 関数・クラスによるコンポーネント化
- **バージョン管理**: Git でインフラの履歴管理
- **テスト可能**: ユニットテストでインフラをテスト

### 12.2 アーキテクチャ設計

- **最小権限の原則**: IAM ロールは必要最小限の権限のみ
- **セキュリティ**: セキュリティグループで通信制御
- **コスト最適化**: 不要なリソース（NAT Gateway）は作成しない
- **可観測性**: CloudWatch Logs 自動設定

### 12.3 本番環境への発展

- 複数環境（dev/staging/prod）の分離
- CI/CD パイプラインとの統合
- 監視・アラート設定
- データベース追加（RDS）
- ロードバランサー（ALB）の追加

---

## 13. 参考リンク

- [AWS CDK Developer Guide](https://docs.aws.amazon.com/cdk/v2/guide/)
- [CDK API Reference](https://docs.aws.amazon.com/cdk/api/v2/)
- [CDK Examples Repository](https://github.com/aws-samples/aws-cdk-examples)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
