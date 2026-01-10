# 環境一覧

このドキュメントでは、tvs-videometa-api の各デプロイ環境とその作成方法を説明します。

## 環境概要

| 環境 | スタック名 | 用途 | デプロイコマンド |
|------|-----------|------|-----------------|
| **V1（本番）** | `tvs-videometa-api` | 本番環境 | `sam deploy` |
| **V2（開発）** | `tvs-videometa-api-v2` | 開発・テスト | `sam deploy --config-env v2` |
| **V3（テスト）** | `tvs-videometa-api-v3` | 新規テスト用 | `sam deploy --config-env v3` |

---

## 環境別URL一覧

### V1 環境（本番）

| リソース | URL / ID |
|----------|----------|
| **フロントエンド** | https://dca9zycmu3jpc.cloudfront.net |
| **API Gateway** | https://0zfucmjrz3.execute-api.ap-northeast-1.amazonaws.com/v1 |
| **Lambda Function URL** | https://37xzweykiig6zoknwf5eohvby40wlnsb.lambda-url.ap-northeast-1.on.aws/ |
| **Cognito ドメイン** | tvs-videometa-api-093667080892.auth.ap-northeast-1.amazoncognito.com |
| **UserPoolId** | ap-northeast-1_w3dwLi2pZ |
| **UserPoolClientId** | 2uborad8sca5jshkf0c104hmpa |

### V2 環境（開発）

| リソース | URL / ID |
|----------|----------|
| **フロントエンド** | https://d3gqyu7bldmqhq.cloudfront.net |
| **API Gateway** | https://d1hzqer8x9.execute-api.ap-northeast-1.amazonaws.com/v1 |
| **Lambda Function URL** | https://cxw4ve3sbk2rzsi3azic2ln7py0ecnrp.lambda-url.ap-northeast-1.on.aws/ |
| **Cognito ドメイン** | tvs-videometa-api-v2-093667080892.auth.ap-northeast-1.amazoncognito.com |
| **UserPoolId** | ap-northeast-1_rdcIXkPFL |
| **UserPoolClientId** | 6r3i2lpkmf851cgfit1i5a8uio |

### V3 環境（テスト）

| リソース | URL / ID |
|----------|----------|
| **フロントエンド** | https://d2a99j7usymjcz.cloudfront.net |
| **API Gateway** | https://m1hwncouuf.execute-api.ap-northeast-1.amazonaws.com/v1 |
| **Lambda Function URL** | https://vt5xhe4gje4tcs6r7x7ikyndp40vtotb.lambda-url.ap-northeast-1.on.aws/ |
| **Cognito ドメイン** | tvs-videometa-api-v3-093667080892.auth.ap-northeast-1.amazoncognito.com |
| **UserPoolId** | ap-northeast-1_g2BB2INqR |
| **UserPoolClientId** | 7404rgchd51pgm3o9mc5cfr79s |

---

## 新しい環境の作成方法

### 1. samconfig.toml の設定

`infra/samconfig.toml` に新しい環境セクションを追加します。

```toml
# ========================================
# 新環境の例（v4）
# Deploy: sam deploy --config-env v4
# ========================================
[v4.deploy.parameters]
stack_name = "tvs-videometa-api-v4"
resolve_s3 = true
s3_prefix = "tvs-videometa-api-v4"
confirm_changeset = true
capabilities = "CAPABILITY_IAM CAPABILITY_NAMED_IAM"
parameter_overrides = "GeminiModelName=\"gemini-2.5-flash\" WebACLArn=\"\""
resolve_image_repos = true

[v4.global.parameters]
region = "ap-northeast-1"
```

**注意**: `samconfig.toml` は `.gitignore` に含まれています。初期設定は `samconfig.toml.example` を参照してください。

### 2. SAM ビルド＆デプロイ

```bash
cd infra

# ビルド
sam build --use-container

# デプロイ（初回は --guided を付けてGeminiApiKeyを入力）
sam deploy --config-env v3 --parameter-overrides "GeminiApiKey=YOUR_API_KEY"

# 2回目以降（APIキーがsamconfig.tomlに保存されている場合）
sam deploy --config-env v3
```

### 3. デプロイ出力の確認

デプロイ完了後、以下のコマンドで出力値を確認：

```bash
aws cloudformation describe-stacks \
  --stack-name tvs-videometa-api-v3 \
  --query 'Stacks[0].Outputs' \
  --output table
```

### 4. フロントエンド設定の更新

`frontend/config.js` を新環境用にコピーして設定を更新：

```javascript
const APP_CONFIG = {
    API_BASE_URL: '<ApiUrl の値>',
    LAMBDA_FUNCTION_URL: '<LambdaFunctionUrl の値>',
    COGNITO: {
        domain: '<CognitoDomain の値（https://を除く）>',
        clientId: '<UserPoolClientId の値>',
        redirectUri: '<FrontendUrl の値>',
        scope: 'email openid profile'
    }
};
```

### 5. フロントエンドのデプロイ

```bash
# S3にアップロード
aws s3 sync frontend/ s3://tvs-videometa-api-v3-frontend-<ACCOUNT_ID>/

# CloudFrontキャッシュ無効化
aws cloudfront create-invalidation \
  --distribution-id <DISTRIBUTION_ID> \
  --paths "/*"
```

### 6. Cognitoユーザーの作成

```bash
aws cognito-idp admin-create-user \
  --user-pool-id <UserPoolId> \
  --username user@example.com \
  --temporary-password TempPass123! \
  --user-attributes Name=email,Value=user@example.com Name=email_verified,Value=true
```

---

## 環境の削除

```bash
# スタックの削除
aws cloudformation delete-stack --stack-name tvs-videometa-api-v3

# 削除完了を待機
aws cloudformation wait stack-delete-complete --stack-name tvs-videometa-api-v3
```

**注意**: S3バケットにオブジェクトが残っている場合、削除に失敗します。先にバケットを空にしてください：

```bash
aws s3 rm s3://tvs-videometa-api-v3-videos-<ACCOUNT_ID> --recursive
aws s3 rm s3://tvs-videometa-api-v3-proxy-<ACCOUNT_ID> --recursive
aws s3 rm s3://tvs-videometa-api-v3-frontend-<ACCOUNT_ID> --recursive
```

---

## 環境間の違い

各環境は完全に独立しており、以下のリソースが環境ごとに作成されます：

- Cognito User Pool（ユーザーは環境ごとに別管理）
- S3 バケット（動画、プロキシ、フロントエンド）
- DynamoDB テーブル
- Lambda 関数
- API Gateway
- Step Functions
- ECS Cluster（Fargate）
- CloudFront Distribution

**共有リソース**:
- WAF WebACL（us-east-1 にデプロイ、全環境で共有可能）
- Gemini API Key（同じキーを使用可能）
