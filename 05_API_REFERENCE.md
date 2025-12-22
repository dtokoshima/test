# API定義書

> **更新日**: 2025-12-20
> **対象ブランチ**: `aws-deploy-multiple-csv`

本ドキュメントでは、動画メタデータ自動生成APIの仕様を記載します。

## 1. 共通仕様

### 1.1 認証方式

| 方式 | ヘッダー | 用途 |
|------|---------|------|
| Cognito JWT | `Authorization: Bearer <token>` | フロントエンドUI |
| APIキー | `x-api-key: <key>` | 外部システム連携 |

全てのAPIエンドポイントで認証が必要です（Lambda Authorizer）。

### 1.2 ベースURL

```
https://{api-gateway-id}.execute-api.ap-northeast-1.amazonaws.com/{stage}
```

### 1.3 共通レスポンスヘッダー

```
Content-Type: application/json
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization, x-api-key
```

### 1.4 エラーレスポンス形式

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "エラーの説明"
  }
}
```

### 1.5 エラーコード一覧

| コード | HTTP Status | 説明 |
|--------|-------------|------|
| INVALID_REQUEST | 400 | リクエスト形式不正 |
| UNSUPPORTED_FORMAT | 400 | サポートされていないファイル形式 |
| TOO_MANY_FILES | 400 | ファイル数超過（最大50件） |
| ALREADY_FINISHED | 400 | ジョブは既に終了している |
| UNAUTHORIZED | 401 | 認証エラー |
| FORBIDDEN | 403 | アクセス拒否 |
| NOT_FOUND | 404 | リソースが見つからない |
| CONFIG_ERROR | 500 | サーバー設定エラー |
| INTERNAL_ERROR | 500 | 内部エラー |
| GEMINI_API_ERROR | 502 | Gemini API呼び出しエラー |
| PARSE_ERROR | 500 | AI出力のパースエラー |
| TIMEOUT_ERROR | 504 | 処理タイムアウト |

---

## 2. アップロードAPI

### 2.1 POST /upload/start

マルチパートアップロードを開始します。

#### リクエスト

```json
{
  "filename": "video.mp4",
  "content_type": "video/mp4",
  "file_size": 1073741824
}
```

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|:----:|------|
| filename | string | ○ | ファイル名 |
| content_type | string | - | MIMEタイプ（デフォルト: video/mp4） |
| file_size | integer | - | ファイルサイズ（バイト） |

#### レスポンス（200 OK）

```json
{
  "upload_id": "upload-uuid",
  "s3_key": "uploads/uuid/video.mp4",
  "bucket": "video-bucket-name"
}
```

#### サポートファイル形式

| MIMEタイプ | 拡張子 | 変換要否 |
|-----------|--------|---------|
| video/mp4 | .mp4 | 2GB超の場合のみ |
| video/mxf | .mxf | 常に必要 |
| application/mxf | .mxf | 常に必要 |

---

### 2.2 GET /upload/part-url

マルチパートアップロードのパート用署名付きURLを取得します。

#### クエリパラメータ

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|:----:|------|
| key | string | ○ | S3キー |
| upload_id | string | ○ | アップロードID |
| part_number | integer | ○ | パート番号（1〜10000） |

#### レスポンス（200 OK）

```json
{
  "presigned_url": "https://bucket.s3.amazonaws.com/..."
}
```

#### 使用方法

```javascript
// 1. 署名付きURLを取得
const response = await fetch(`/upload/part-url?key=${key}&upload_id=${uploadId}&part_number=${partNumber}`);
const { presigned_url } = await response.json();

// 2. パートをアップロード
const partResponse = await fetch(presigned_url, {
  method: 'PUT',
  body: chunk  // 100MBチャンク
});
const etag = partResponse.headers.get('ETag');
```

---

### 2.3 POST /upload/complete

マルチパートアップロードを完了します。

#### リクエスト

```json
{
  "bucket": "video-bucket-name",
  "key": "uploads/uuid/video.mp4",
  "upload_id": "upload-uuid",
  "parts": [
    {"PartNumber": 1, "ETag": "\"etag1\""},
    {"PartNumber": 2, "ETag": "\"etag2\""}
  ]
}
```

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|:----:|------|
| bucket | string | - | S3バケット名 |
| key | string | ○ | S3キー |
| upload_id | string | ○ | アップロードID |
| parts | array | ○ | パート情報の配列 |

#### レスポンス（200 OK）

```json
{
  "location": "https://bucket.s3.amazonaws.com/uploads/uuid/video.mp4",
  "bucket": "video-bucket-name",
  "key": "uploads/uuid/video.mp4",
  "etag": "\"final-etag\""
}
```

---

## 3. 分析API

### 3.1 POST /analyze（バッチジョブ開始）

複数ファイルの非同期分析ジョブを開始します。

#### リクエスト

```json
{
  "files": [
    {
      "s3_key": "uploads/uuid/video1.mp4",
      "file_size": 1073741824,
      "content_type": "video/mp4",
      "original_filename": "番組A.mp4"
    },
    {
      "s3_key": "uploads/uuid/video2.mxf",
      "file_size": 5368709120,
      "content_type": "video/mxf",
      "original_filename": "番組B.mxf"
    }
  ]
}
```

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|:----:|------|
| files | array | ○ | ファイル情報の配列（最大50件） |
| files[].s3_key | string | ○ | S3キー |
| files[].file_size | integer | - | ファイルサイズ（バイト） |
| files[].content_type | string | - | MIMEタイプ |
| files[].original_filename | string | - | 元のファイル名 |

#### レスポンス（202 Accepted）

```json
{
  "batch_id": "batch-uuid",
  "jobs": [
    {
      "job_id": "job-uuid-1",
      "original_filename": "番組A.mp4",
      "queue_position": 0
    },
    {
      "job_id": "job-uuid-2",
      "original_filename": "番組B.mxf",
      "queue_position": 1
    }
  ],
  "total_jobs": 2
}
```

#### 処理の流れ

1. 各ファイルに対してジョブIDを発行
2. DynamoDBにジョブ情報を登録
3. Step Functionsワークフローを開始
4. ファイル形式・サイズに応じて変換要否を自動判定

---

### 3.2 POST /analyze（同期分析）

単一ファイルを同期的に分析します（小さいMP4専用、レガシーAPI）。

#### リクエスト

```json
{
  "s3_key": "uploads/uuid/video.mp4"
}
```

#### レスポンス（200 OK）

```json
{
  "segments": [
    {
      "timestamp": "00:00:00",
      "screen_type": "スタジオ",
      "performers": ["田中アナ(男)", "佐藤記者(女)"],
      "summary": "オープニング",
      "telop": "ニュース番組タイトル",
      "remarks": ""
    }
  ],
  "total_segments": 10,
  "analyzed_at": "2025-12-16T10:00:00"
}
```

---

## 4. ジョブ管理API

### 4.1 GET /jobs

ジョブ一覧を取得します。

#### クエリパラメータ

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|:----:|------|
| batch_id | string | - | バッチIDでフィルタ |
| status | string | - | ステータスでフィルタ |
| limit | integer | - | 取得件数（デフォルト/最大: 100） |

#### レスポンス（200 OK）

```json
{
  "jobs": [
    {
      "job_id": "job-uuid",
      "batch_id": "batch-uuid",
      "original_filename": "番組A.mp4",
      "queue_position": 0,
      "status": "ANALYZING",
      "created_at": "2025-12-16T10:00:00"
    }
  ],
  "summary": {
    "total": 10,
    "completed": 5,
    "in_progress": 3,
    "pending": 1,
    "failed": 1,
    "cancelled": 0
  },
  "next_token": "next-job-id"
}
```

#### ジョブステータス一覧

| ステータス | 説明 |
|-----------|------|
| PENDING | 処理待機中 |
| QUEUED | キュー登録済み |
| CONVERTING | MediaConvert変換中 |
| ANALYZING | Gemini分析中 |
| COMPLETED | 完了 |
| FAILED | エラー |
| CANCELLED | キャンセル済み |

---

### 4.2 GET /jobs/{job_id}

ジョブ詳細を取得します。

#### パスパラメータ

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|:----:|------|
| job_id | string | ○ | ジョブID |

#### クエリパラメータ

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|:----:|------|
| format | string | - | 出力形式（json/csv） |

#### レスポンス（200 OK）- JSON形式

```json
{
  "job_id": "job-uuid",
  "status": "COMPLETED",
  "created_at": "2025-12-16T10:00:00",
  "completed_at": "2025-12-16T10:15:00",
  "requires_conversion": false,
  "result": {
    "segments": [
      {
        "timestamp": "00:00:00",
        "screen_type": "スタジオ",
        "performers": ["田中アナ(男)"],
        "summary": "オープニング",
        "telop": "タイトル",
        "remarks": ""
      }
    ],
    "total_segments": 10,
    "analyzed_at": "2025-12-16T10:15:00"
  }
}
```

#### レスポンス（200 OK）- CSV形式（format=csv）

```
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="番組A-metadata.csv"

タイムスタンプ,画面種別,出演者,内容要約,テロップ,備考
00:00:00,スタジオ,田中アナ(男),オープニング,タイトル,
00:02:30,VTR,ナレーション,事件概要,事件発生日: 12月15日,
```

---

### 4.3 DELETE /jobs/{job_id}

ジョブをキャンセルします。

#### パスパラメータ

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|:----:|------|
| job_id | string | ○ | ジョブID |

#### レスポンス（200 OK）

```json
{
  "job_id": "job-uuid",
  "status": "CANCELLED",
  "message": "ジョブをキャンセルしました"
}
```

#### 注意事項

- 終了済みのジョブ（COMPLETED/FAILED/CANCELLED）はキャンセル不可
- 実行中のMediaConvertジョブ・Step Functions実行も停止される
- S3上の一時ファイルは自動的にクリーンアップされる

---

## 5. レガシーAPI（後方互換性維持）

以下のAPIは後方互換性のために維持されています。新規開発では新APIを使用してください。

### 5.1 GET /upload-url

署名付きアップロードURLを取得します。

#### クエリパラメータ

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|:----:|------|
| filename | string | - | ファイル名 |
| content_type | string | - | MIMEタイプ |
| upload_type | string | - | single/multipart |

#### レスポンス（200 OK）- 単一アップロード

```json
{
  "upload_type": "single",
  "upload_url": "https://...",
  "s3_key": "uploads/uuid/video.mp4",
  "expires_in": 3600
}
```

### 5.2 GET /upload-url/part

→ GET /upload/part-url と同一

### 5.3 POST /upload-url/complete

→ POST /upload/complete と同一

### 5.4 POST /jobs

単一ジョブを開始します。

```json
// Request
{
  "s3_key": "uploads/uuid/video.mp4",
  "file_size": 1073741824,
  "content_type": "video/mp4"
}

// Response (202 Accepted)
{
  "job_id": "job-uuid",
  "status": "PENDING",
  "requires_conversion": false,
  "message": "ジョブを開始しました"
}
```

---

## 6. 分析結果フォーマット

### 6.1 VideoSegment

| フィールド | 型 | 説明 |
|-----------|-----|------|
| timestamp | string | タイムスタンプ（HH:MM:SS形式） |
| screen_type | string | 画面種別（スタジオ/中継/VTR/CM等） |
| performers | array[string] | 出演者リスト（「氏名(性別)」形式） |
| summary | string | 内容要約 |
| telop | string | テロップテキスト |
| remarks | string | 備考（空の場合あり） |

### 6.2 AnalysisResult

| フィールド | 型 | 説明 |
|-----------|-----|------|
| segments | array[VideoSegment] | セグメントリスト |
| total_segments | integer | 総セグメント数 |
| analyzed_at | string | 分析完了日時（ISO 8601形式） |

---

## 7. 制限値

| 項目 | 値 | 備考 |
|------|-----|------|
| 最大同時ファイル数 | 50件/バッチ | |
| 最大ファイルサイズ | 100GB/ファイル | |
| 直接分析可能サイズ | 2GB | MP4かつ2GB以下なら変換不要 |
| 最大動画長 | 15時間（デフォルト） | AWS Quotas引き上げで23時間59分まで対応可 |
| セグメント長 | 1.5時間 | 1セグメント=1 MediaConvertジョブ |
| 1動画あたりの最大MediaConvertジョブ数 | 10 | 環境変数で変更可能 |
| 並列分析数 | 最大5 | Step Functions Map state |
| ジョブ保持期間 | 7日（TTL） | DynamoDB |
| S3ファイル保持期間 | 1日（TTL） | Lifecycle Policy |
| 署名付きURL有効期限 | 1時間 | |

### 注意: 長時間動画の処理

デフォルト設定では最大15時間（10セグメント × 1.5時間）の動画まで対応しています。
23時間59分までの動画を処理するには：

1. AWS Service Quotasで「MediaConvert concurrent jobs」を増やす（デフォルト20→40推奨）
2. 環境変数 `VIDEOMETA_MAX_MEDIACONVERT_JOBS_PER_VIDEO` を16に設定
