# セットアップガイド

新規クローンからAWSデプロイ、動作確認までの手順です。

## 目次

1. [前提条件](#1-前提条件)
2. [Gemini API Keyの取得](#2-gemini-api-keyの取得)
3. [AWS認証の設定](#3-aws認証の設定)
4. [SAMビルド＆デプロイ](#4-samビルドデプロイ)
5. [フロントエンド設定](#5-フロントエンド設定)
6. [フロントエンドデプロイ](#6-フロントエンドデプロイ)
7. [Cognitoユーザー作成](#7-cognitoユーザー作成)
8. [動作確認](#8-動作確認)
9. [WAF設定（オプション）](#9-waf設定オプション)
10. [トラブルシューティング](#10-トラブルシューティング)

---

## 1. 前提条件

以下がインストールされていること：

| ツール | バージョン | インストール |
|--------|-----------|-------------|
| Python | 3.11+ | [python.org](https://www.python.org/) |
| uv | 最新 | `pip install uv` または [公式](https://github.com/astral-sh/uv) |
| AWS CLI | v2 | [AWS CLI](https://aws.amazon.com/cli/) |
| AWS SAM CLI | 最新 | [SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html) |
| Docker | 最新 | [Docker Desktop](https://www.docker.com/products/docker-desktop/) |

---

## 2. Gemini API Keyの取得

1. [Google AI Studio](https://makersuite.google.com/app/apikey) にアクセス
2. Googleアカウントでログイン
3. 「Create API Key」をクリック
4. 生成されたAPI Keyをコピーして安全に保管

---

## 3. AWS認証の設定

```bash
aws configure
```

| 設定項目 | 値 |
|----------|-----|
| AWS Access Key ID | IAMユーザーのアクセスキー |
| AWS Secret Access Key | IAMユーザーのシークレットキー |
| Default region name | `ap-northeast-1` |
| Default output format | `json` |

**必要なIAM権限**: AdministratorAccess（または個別にS3, Lambda, API Gateway, DynamoDB, Step Functions, ECS, Cognito, CloudFront, ECR, CloudWatch, SNS, IAM, VPC）

---

## 4. SAMビルド＆デプロイ

### 4.1 リポジトリのクローン

```bash
git clone https://github.com/athenatech-jp/tvs-videometa-api.git
cd tvs-videometa-api
```

### 4.2 依存関係のインストール

```bash
uv sync --all-extras
```

### 4.3 SAMビルド

```bash
cd infra
sam build
```

初回ビルドは5〜10分かかります（Dockerイメージのビルド）。

### 4.4 SAMデプロイ

```bash
sam deploy --guided
```

**入力パラメータ**:

| パラメータ | 入力値 | 説明 |
|-----------|--------|------|
| Stack Name | `tvs-videometa-api` | 任意のスタック名 |
| AWS Region | `ap-northeast-1` | デプロイリージョン |
| GeminiApiKey | `AIzaSy...` | 手順2で取得したAPI Key |
| GeminiModelName | Enter（デフォルト） | `gemini-2.5-flash` |
| WebACLArn | Enter（空欄） | WAF未使用時は空欄 |
| Confirm changes | `y` | |
| Allow SAM CLI IAM role creation | `y` | |
| Disable rollback | `n` | |
| Save arguments to configuration file | `y` | |

初回デプロイは10〜15分かかります。

### 4.5 デプロイ出力の確認

デプロイ完了後、以下の出力値をメモ：

```
CloudFormation outputs:
------------------------------------------------------------
Key                   ApiUrl
Value                 https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com

Key                   LambdaFunctionUrl
Value                 https://xxxxxxxxxx.lambda-url.ap-northeast-1.on.aws/

Key                   FrontendUrl
Value                 https://xxxxxxxxxx.cloudfront.net

Key                   FrontendBucketName
Value                 tvs-videometa-api-frontend-xxxxxxxxxxxx

Key                   CloudFrontDistributionId
Value                 EXXXXXXXXXXXX

Key                   CognitoDomain
Value                 tvs-videometa-api-xxxxxxxxxxxx.auth.ap-northeast-1.amazoncognito.com

Key                   UserPoolId
Value                 ap-northeast-1_xxxxxxxxx

Key                   UserPoolClientId
Value                 xxxxxxxxxxxxxxxxxxxxxxxxxx
------------------------------------------------------------
```

---

## 5. フロントエンド設定

`frontend/config.js` を編集し、手順4.5の出力値を設定：

```javascript
const APP_CONFIG = {
    // SAM出力: ApiUrl
    API_BASE_URL: 'https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com',

    // SAM出力: LambdaFunctionUrl
    LAMBDA_FUNCTION_URL: 'https://xxxxxxxxxx.lambda-url.ap-northeast-1.on.aws/',

    // Cognito認証設定
    COGNITO: {
        // SAM出力: CognitoDomain
        domain: 'tvs-videometa-api-xxxxxxxxxxxx.auth.ap-northeast-1.amazoncognito.com',
        // SAM出力: UserPoolClientId
        clientId: 'xxxxxxxxxxxxxxxxxxxxxxxxxx',
        // SAM出力: FrontendUrl
        redirectUri: 'https://xxxxxxxxxx.cloudfront.net',
        scope: 'email openid profile'
    }
};
```

---

## 6. フロントエンドデプロイ

```bash
# S3にアップロード（FrontendBucketNameを使用）
aws s3 sync frontend/ s3://tvs-videometa-api-frontend-xxxxxxxxxxxx/

# CloudFrontキャッシュ無効化（CloudFrontDistributionIdを使用）
aws cloudfront create-invalidation --distribution-id EXXXXXXXXXXXX --paths "/*"
```

---

## 7. Cognitoユーザー作成

**注意**: セキュリティのため、セルフサインアップは無効化されています（`AdminCreateUserOnly`）。ユーザーは管理者が作成する必要があります。

### AWS CLIでユーザーを作成

```bash
# ユーザー作成（UserPoolIdを使用）
aws cognito-idp admin-create-user \
  --user-pool-id ap-northeast-1_xxxxxxxxx \
  --username your-email@example.com \
  --temporary-password TempPass123! \
  --user-attributes Name=email,Value=your-email@example.com Name=email_verified,Value=true
```

### AWSマネジメントコンソールでユーザーを作成

1. AWSコンソール → Cognito → ユーザープール → 該当プールを選択
2. 「ユーザー」タブ → 「ユーザーを作成」
3. メールアドレスと仮パスワードを入力
4. 「ユーザーを作成」をクリック

初回ログイン時にパスワード変更を求められます。

---

## 8. 動作確認

1. ブラウザで `FrontendUrl`（CloudFront URL）にアクセス
2. Cognitoログイン画面でメールアドレス・仮パスワードを入力
3. 新しいパスワードを設定
4. 動画ファイル（MP4/MXF）をアップロード
5. 分析結果が表示されることを確認
6. CSVでダウンロードできることを確認

---

## 9. WAF設定（オプション）

CloudFrontにWAF（レート制限・脅威対策）を設定する場合：

### 9.1 WAFスタックのデプロイ（us-east-1）

**Windows (PowerShell)**:
```powershell
cd infra
.\deploy-waf.ps1
```

**macOS/Linux**:
```bash
cd infra
./deploy-waf.sh
```

### 9.2 WebACL ARNの取得

デプロイ完了後に出力される `WebACLArn` をメモ：
```
WebACLArn: arn:aws:wafv2:us-east-1:xxxxxxxxxxxx:global/webacl/...
```

### 9.3 メインスタックの再デプロイ

`infra/samconfig.toml` を編集してWebACLArnを追加：

```toml
[default.deploy.parameters]
parameter_overrides = "GeminiApiKey=\"AIzaSy...\" WebACLArn=\"arn:aws:wafv2:us-east-1:...\""
```

再デプロイ：
```bash
cd infra
sam deploy
```

---

## 10. トラブルシューティング

### デプロイエラー

```bash
# スタック状態を確認
aws cloudformation describe-stacks --stack-name tvs-videometa-api --query 'Stacks[0].StackStatus'

# イベントログを確認
aws cloudformation describe-stack-events --stack-name tvs-videometa-api --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'
```

### 認証エラー（401/403）

- Cognitoユーザーが作成されているか確認
- `config.js` の設定値が正しいか確認
- CloudFrontキャッシュが無効化されているか確認

### 分析が失敗する

```bash
# Lambda ログを確認
aws logs tail /aws/lambda/tvs-videometa-api-GeminiAnalyzeFunction --follow

# DynamoDBのジョブ状態を確認
aws dynamodb get-item --table-name tvs-videometa-api-jobs --key '{"job_id":{"S":"job-id-here"}}'
```

### Gemini APIエラー

- API Keyが正しく設定されているか確認
- Gemini APIの利用制限に達していないか確認
- [Google AI Studio](https://makersuite.google.com/) でAPI Keyのステータスを確認

---

## 設定ファイル一覧

| ファイル | 用途 | 編集タイミング |
|----------|------|---------------|
| `frontend/config.js` | フロントエンド設定 | SAMデプロイ後 |
| `infra/samconfig.toml` | SAMデプロイ設定 | 自動生成（初回デプロイ後） |
| `.env` | ローカル開発用 | ローカル開発時のみ |

---

## 2回目以降のデプロイ

```bash
# バックエンドのみ
cd infra
sam build && sam deploy

# フロントエンドのみ
aws s3 sync frontend/ s3://tvs-videometa-api-frontend-xxxxxxxxxxxx/
aws cloudfront create-invalidation --distribution-id EXXXXXXXXXXXX --paths "/*"
```
