# API定義書

> **更新日**: 2025-12-30
> **対象ブランチ**: `aws-deploy-multiple-csv`

本ドキュメントでは、動画メタデータ自動生成APIの仕様を記載します。

## 1. 共通仕様

### 1.1 認証方式

| 方式 | ヘッダー | 用途 |
|------|---------|------|
| Cognito JWT | `Authorization: Bearer <token>` | フロントエンドUI |
| APIキー | `Authorization: ApiKey <key>` | 外部システム連携 |

全てのAPIエンドポイントで認証が必要です（Lambda Authorizer）。

### 1.2 ベースURL

```
https://{api-gateway-id}.execute-api.ap-northeast-1.amazonaws.com/v1
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

## 3. 黒動画分析API（通常動画）

黒動画とは、テロップあり・BGMありの完成版動画です。

### 3.1 POST /analyze

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
    }
  ],
  "total_jobs": 1
}
```

---

### 3.2 GET /jobs

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
  }
}
```

#### ジョブステータス一覧

| ステータス | 説明 |
|-----------|------|
| PENDING | 処理待機中 |
| QUEUED | キュー登録済み |
| CONVERTING | FFmpeg変換中（Fargate） |
| ANALYZING | Gemini分析中 |
| COMPLETED | 完了 |
| FAILED | エラー |
| CANCELLED | キャンセル済み |

---

### 3.3 GET /jobs/{job_id}

ジョブ詳細・タイムテーブルCSVを取得します。

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
  "requires_conversion": true,
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
    "program_summary": "## ファイルID\n...",
    "format_corrected": false
  }
}
```

#### レスポンス（200 OK）- CSV形式（format=csv）

```
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="番組A-metadata.csv"

"タイムスタンプ","画面構成","出演者","内容要約","テロップ","備考"
"00:00:00","スタジオ","田中アナ(男)","オープニング","タイトル",""
"00:02:30","VTR","","事件概要","事件発生日: 12月15日",""
```

---

### 3.4 GET /jobs/{job_id}/summary

番組要約CSVを取得します。

#### パスパラメータ

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|:----:|------|
| job_id | string | ○ | ジョブID |

#### レスポンス（200 OK）

```
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="番組A-summary.csv"

"ファイルID","タイトル","エリア","キーワード","取材収録日","放送日","種類"
"","番組タイトル","番組の概要説明","キーワード1, キーワード2","","","黒"
```

#### 番組要約CSVの列説明

| 列名 | 説明 |
|------|------|
| ファイルID | ファイル識別子（通常は空） |
| タイトル | 番組名 |
| エリア | 番組全体の内容要約 |
| キーワード | カンマ区切りのキーワード |
| 取材収録日 | 取材・収録日（判明時のみ） |
| 放送日 | 放送日（判明時のみ） |
| 種類 | 「黒」（テロップあり）または「白」（テロップなし） |

---

### 3.5 DELETE /jobs/{job_id}

ジョブをキャンセルします。

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
- 実行中のFargateタスク・Step Functions実行も停止される
- S3上の一時ファイルは自動的にクリーンアップされる

---

## 4. 白動画分析API（コンテキスト分析）

白動画とは、テロップなし・BGMなしの素材動画です。黒動画（完成版）のメタデータを参照しながら分析します。

### 4.1 POST /context/analyze

白動画の分析ジョブを開始します。

#### リクエスト

```json
{
  "files": [
    {
      "s3_key": "uploads/uuid/white-video.mp4",
      "file_size": 1073741824,
      "content_type": "video/mp4",
      "original_filename": "素材A.mp4"
    }
  ],
  "context_csv": "タイムスタンプ,画面構成,出演者,内容要約,テロップ,備考\n00:00:00,スタジオ,田中アナ(男),オープニング,番組タイトル,"
}
```

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|:----:|------|
| files | array | ○ | ファイル情報の配列 |
| context_csv | string | ○ | 黒動画から抽出したメタデータCSV |

#### レスポンス（202 Accepted）

```json
{
  "batch_id": "batch-uuid",
  "jobs": [
    {
      "job_id": "job-uuid-1",
      "original_filename": "素材A.mp4",
      "queue_position": 0
    }
  ],
  "total_jobs": 1
}
```

---

### 4.2 GET /context/jobs

白動画ジョブ一覧を取得します。

#### クエリパラメータ

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|:----:|------|
| batch_id | string | - | バッチIDでフィルタ |
| status | string | - | ステータスでフィルタ |
| limit | integer | - | 取得件数（デフォルト/最大: 100） |

#### レスポンス

`GET /jobs` と同一形式

---

### 4.3 GET /context/jobs/{job_id}

白動画ジョブ詳細・タイムテーブルCSVを取得します。

#### クエリパラメータ

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|:----:|------|
| format | string | - | 出力形式（json/csv） |

#### レスポンス

`GET /jobs/{job_id}` と同一形式

---

### 4.4 GET /context/jobs/{job_id}/summary

白動画の番組要約CSVを取得します。種類は自動的に「白」が設定されます。

#### レスポンス（200 OK）

```
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="素材A-summary.csv"

"ファイルID","タイトル","エリア","キーワード","取材収録日","放送日","種類"
"","番組タイトル","番組の概要説明","キーワード1, キーワード2","","","白"
```

---

## 5. 分析結果フォーマット

### 5.1 タイムテーブルCSV

| 列名 | 説明 |
|------|------|
| タイムスタンプ | HH:MM:SS形式 |
| 画面構成 | スタジオ/VTR/インタビュー/中継など |
| 出演者 | カンマ区切り（「氏名(性別)」形式） |
| 内容要約 | セグメントの内容説明 |
| テロップ | 画面表示テキスト |
| 備考 | 補足情報 |

### 5.2 番組要約CSV

| 列名 | 説明 |
|------|------|
| ファイルID | ファイル識別子 |
| タイトル | 番組名 |
| エリア | 番組全体の内容要約 |
| キーワード | カンマ区切りのキーワード |
| 取材収録日 | 取材・収録日 |
| 放送日 | 放送日 |
| 種類 | 「黒」または「白」 |

### 5.3 VideoSegment（JSON）

| フィールド | 型 | 説明 |
|-----------|-----|------|
| timestamp | string | タイムスタンプ（HH:MM:SS形式） |
| screen_type | string | 画面構成 |
| performers | array[string] | 出演者リスト |
| summary | string | 内容要約 |
| telop | string | テロップテキスト |
| remarks | string | 備考 |

### 5.4 AnalysisResult（JSON）

| フィールド | 型 | 説明 |
|-----------|-----|------|
| segments | array[VideoSegment] | セグメントリスト |
| total_segments | integer | 総セグメント数 |
| program_summary | string | 番組要約（Markdown形式） |
| format_corrected | boolean | フォーマット修正が行われたか |
| analysis_started_at | string | 分析開始日時（ISO 8601形式） |
| analysis_completed_at | string | 分析完了日時（ISO 8601形式） |
| analysis_duration_ms | integer | 分析所要時間（ミリ秒） |

---

## 6. 制限値

| 項目 | 値 | 備考 |
|------|-----|------|
| 最大同時ファイル数 | 50件/バッチ | |
| 最大ファイルサイズ | 300GB/ファイル | |
| 直接分析可能サイズ | 2GB | MP4かつ2GB以下なら変換不要 |
| 最大動画長 | 12時間 | |
| セグメント長 | 15分 | FFmpeg変換時のセグメント単位 |
| 並列FFmpegタスク数 | 最大24 | Fargate |
| 並列Gemini分析数 | 最大5 | Step Functions Map state |
| ジョブ保持期間 | 7日（TTL） | DynamoDB |
| S3ファイル保持期間 | 1日（TTL） | Lifecycle Policy |
| 署名付きURL有効期限 | 1時間 | |

---

## 7. 処理フロー

### 7.1 黒動画処理フロー

```
1. POST /upload/start → マルチパートアップロード開始
2. GET /upload/part-url → 各パートの署名付きURL取得
3. （クライアント）S3に直接アップロード
4. POST /upload/complete → アップロード完了
5. POST /analyze → 分析ジョブ開始
6. （Step Functions）
   a. ファイル形式判定（MXF/大容量MP4は変換必要）
   b. FFmpeg変換（15分セグメントに分割、1fps化）
   c. Gemini分析（セグメント並列処理）
   d. 結果マージ・番組要約生成
7. GET /jobs/{job_id} → タイムテーブル取得
8. GET /jobs/{job_id}/summary → 番組要約取得
```

### 7.2 白動画処理フロー

```
1. 黒動画を先に分析し、CSVを取得
2. POST /context/analyze → 白動画分析開始（黒動画CSVを添付）
3. （Step Functions）
   a. FFmpeg変換
   b. Gemini分析（黒動画メタデータを参照）
   c. 結果マージ・番組要約生成（種類=白）
4. GET /context/jobs/{job_id} → タイムテーブル取得
5. GET /context/jobs/{job_id}/summary → 番組要約取得
```
