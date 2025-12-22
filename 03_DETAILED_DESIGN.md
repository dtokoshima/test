# コード詳細設計書

> **更新日**: 2025-12-20
> **対象ブランチ**: `aws-deploy-multiple-csv`

## 1. 型定義（Pydanticモデル）

### 1.1 models/metadata.py

```python
from datetime import datetime
from pydantic import BaseModel


class VideoSegment(BaseModel):
    """動画の1セグメント（時間区間）のメタデータ"""
    timestamp: str           # "00:00:00" 形式（HH:MM:SS）
    screen_type: str         # スタジオ / 中継 / VTR / CM 等
    performers: list[str]    # ["田中アナ(男)", "佐藤記者(女)"]
    topic: str = ""          # トピック（後方互換性維持、通常は未使用）
    summary: str             # 内容要約
    telop: str = ""          # テロップテキスト
    remarks: str = ""        # 備考（空の場合あり）


class AnalysisResult(BaseModel):
    """動画分析の結果"""
    segments: list[VideoSegment]
    total_segments: int
    analysis_started_at: datetime    # 分析開始時刻
    analysis_completed_at: datetime  # 分析完了時刻
    analysis_duration_ms: int        # 分析にかかった時間（ミリ秒）
    format_corrected: bool           # フォーマット修正が行われたかどうか
```

### 1.2 models/errors.py

```python
class VideoMetaError(Exception):
    """基底例外クラス"""
    def __init__(self, code: str, message: str):
        self.code = code
        self.message = message


class GeminiApiError(VideoMetaError):
    """Gemini API呼び出しエラー"""

class ParseError(VideoMetaError):
    """AI出力のパースエラー"""
    raw_output: str | None = None

class VideoMetaTimeoutError(VideoMetaError):
    """処理タイムアウト"""
```

---

## 2. 設定管理

### 2.1 config.py

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    # Gemini API
    gemini_api_key: str
    gemini_model_name: str = "gemini-2.5-pro"
    request_timeout_seconds: int = 1200

    # ログレベル
    log_level: str = "INFO"

    # S3設定
    video_bucket: str | None = None
    proxy_bucket: str | None = None

    # DynamoDB設定
    jobs_table: str | None = None

    # MediaConvert設定
    mediaconvert_role: str | None = None

    # Step Functions設定
    state_machine_arn: str | None = None

    # フォーマット修正設定
    format_correction_max_retries: int = 3
    format_correction_model_name: str = "gemini-2.0-flash"

    # プロキシ動画設定
    proxy_video_width: int = 640
    proxy_video_height: int = 360
    proxy_video_framerate: int = 1

    # 長時間動画分割設定
    segment_duration_seconds: int = 5400      # 1.5時間
    max_video_duration_seconds: int = 86340   # 23時間59分

    model_config = SettingsConfigDict(
        env_prefix="VIDEOMETA_",
        env_file=".env",
        env_file_encoding="utf-8",
    )
```

### 2.2 環境変数一覧

| 変数名 | 必須 | デフォルト | 説明 |
|--------|:----:|-----------|------|
| VIDEOMETA_GEMINI_API_KEY | ○ | - | Gemini APIキー |
| VIDEOMETA_GEMINI_MODEL_NAME | - | gemini-2.5-pro-preview-05-06 | 動画分析モデル |
| VIDEOMETA_REQUEST_TIMEOUT_SECONDS | - | 1200 | タイムアウト秒数 |
| VIDEOMETA_LOG_LEVEL | - | INFO | ログレベル |
| VIDEOMETA_FORMAT_CORRECTION_MAX_RETRIES | - | 3 | フォーマット修正最大リトライ |
| VIDEOMETA_FORMAT_CORRECTION_MODEL_NAME | - | gemini-2.0-flash | フォーマット修正用モデル |
| VIDEOMETA_SEGMENT_DURATION_SECONDS | - | 5400 | セグメント長（秒） |
| VIDEOMETA_MAX_VIDEO_DURATION_SECONDS | - | 86340 | 最大対応動画長（秒） |

---

## 3. モジュールインターフェース

### 3.1 handler.py - APIエンドポイント

```python
MAX_BATCH_FILES = 50  # バッチ処理の最大ファイル数

def lambda_handler(event: dict, context: Any) -> dict:
    """Lambdaエントリポイント（ルーティング）"""

def _handle_upload_start(event: dict) -> dict:
    """マルチパートアップロード開始"""

def _handle_analyze_batch(event: dict) -> dict:
    """バッチジョブ開始"""

def _handle_list_jobs(event: dict) -> dict:
    """ジョブ一覧取得"""

def _handle_cancel_job(job_id: str | None) -> dict:
    """ジョブキャンセル"""

def _handle_analyze(event: dict) -> dict:
    """同期分析（小さいMP4用）"""
```

### 3.2 handler.py - Step Functions内部アクション

```python
def _handle_internal_action(event: dict, action: str) -> dict:
    """Step Functionsからの内部アクション処理"""
    # 対応アクション:
    # - start_mediaconvert: MediaConvertジョブ開始
    # - check_mediaconvert: 複数ジョブの状態確認
    # - list_segments: セグメントファイルリスト
    # - analyze_single_segment: 単一セグメント分析
    # - merge_segment_results: 結果マージ
    # - cleanup: ファイル削除

def _start_mediaconvert_job(event: dict) -> dict:
    """MediaConvertジョブを開始（クリッピング複数MP4出力）"""
    # 戻り値: mediaconvert_job_ids, proxy_prefix, num_segments

def _check_mediaconvert_status(event: dict) -> dict:
    """MediaConvertジョブの状態を確認（複数ジョブ対応）"""
    # StartTimecode超過エラーはスキップ扱い

def _list_segments_for_analysis(event: dict) -> dict:
    """分析対象のセグメントファイルをリスト（Map state用）"""

def _analyze_single_segment(event: dict) -> dict:
    """単一セグメントを分析（Map state内で呼ばれる）"""

def _merge_segment_results(event: dict) -> dict:
    """Map stateの結果をマージ"""
```

### 3.3 services/gemini_client.py

```python
class GeminiClient:
    def __init__(self, api_key: str, model_name: str = "gemini-2.5-pro"):
        pass

    def analyze_video(
        self, file_data: bytes, file_name: str, prompt: str, timeout: int = 1200
    ) -> str:
        """動画を分析してテキストを返す"""

    def generate_text(self, prompt: str) -> str:
        """テキストのみの生成（フォーマット修正用）"""
```

### 3.4 parsers/markdown_parser.py

```python
def parse_markdown_table(markdown_text: str) -> list[VideoSegment]:
    """Markdownテーブルをパースしてセグメントリストを返す"""

def _parse_header(header_line: str) -> list[str]:
    """ヘッダー行をパース"""

def _parse_row(row_line: str, columns: list[str]) -> VideoSegment:
    """1行をパースしてVideoSegmentを返す"""
```

---

## 4. 長時間動画処理

### 4.1 クリッピング方式

MediaConvertのInputClippingsを使用して、長時間動画を複数のMP4ファイルに分割します。

```python
def _seconds_to_timecode(seconds: int) -> str:
    """秒数をタイムコード形式（HH:MM:SS:00）に変換"""

def _estimate_video_duration_seconds(file_size_bytes: int) -> int:
    """ファイルサイズから動画の長さを推定（秒）"""
    # MXFの一般的なビットレート（25Mbps）で保守的に推定
```

### 4.2 セグメント分析の並列化

Step Functions Map stateを使用して、各セグメントを並列分析します。

```
ListSegments → AnalyzeSegmentsParallel (Map) → MergeSegmentResults
```

| 設定 | 値 |
|------|-----|
| MaxConcurrency | 5 |
| タイムスタンプ調整 | セグメント番号 × segment_duration_seconds |

### 4.3 エラーハンドリング

| エラー種別 | 対応 |
|-----------|------|
| StartTimecode超過 | スキップ（正常終了扱い） |
| MediaConvert失敗 | 即時エラー |
| Gemini API失敗 | リトライ3回後エラー |

---

## 5. エラーハンドリング

### 5.1 エラーコード体系

| コード | HTTP Status | 意味 |
|--------|-------------|------|
| INVALID_REQUEST | 400 | リクエスト形式不正 |
| UNAUTHORIZED | 401 | 認証エラー |
| GEMINI_API_ERROR | 502 | Gemini API呼び出しエラー |
| PARSE_ERROR | 500 | AI出力のパースエラー |
| TIMEOUT_ERROR | 504 | 処理タイムアウト |
| INTERNAL_ERROR | 500 | 予期しない内部エラー |

### 5.2 エラーハンドリングパターン

```python
def lambda_handler(event, context):
    try:
        return success_response(result)
    except VideoMetaError as e:
        logger.error(f"[{e.code}] {e.message}")
        return error_response(e.code, e.message)
    except Exception as e:
        logger.exception("Unexpected error")
        return error_response("INTERNAL_ERROR", "内部エラーが発生しました")
```

---

## 6. コーディング規約

### 6.1 命名規則

| 対象 | 規則 | 例 |
|-----|------|-----|
| クラス | PascalCase | `VideoSegment`, `GeminiClient` |
| 関数・メソッド | snake_case | `analyze_video`, `parse_markdown_table` |
| 変数 | snake_case | `video_url`, `api_key` |
| 定数 | UPPER_SNAKE_CASE | `SYSTEM_PROMPT_JA`, `MAX_BATCH_FILES` |
| プライベート | 先頭アンダースコア | `_validate_url`, `_parse_row` |

### 6.2 型ヒント

- 全ての関数に型ヒントを付ける
- `Optional[X]`より`X | None`を使用（Python 3.10+）
- `Any`は外部ライブラリとの境界のみ

---

## 7. テスト方針

### 7.1 テストファイル構成

```
tests/
├── __init__.py
├── conftest.py
└── parsers/
    └── test_markdown_parser.py
```

### 7.2 テスト実行

```bash
uv run pytest tests/parsers/test_markdown_parser.py -v
uv run pytest tests/parsers/test_markdown_parser.py::test_function_name -v
```

---

## 8. 依存ライブラリ

### 8.1 本番依存

| ライブラリ | 用途 |
|-----------|------|
| pydantic | データバリデーション |
| pydantic-settings | 設定管理 |
| google-genai | Gemini API SDK |
| boto3 | AWS SDK |

### 8.2 開発依存

| ライブラリ | 用途 |
|-----------|------|
| pytest | テストフレームワーク |
| pytest-cov | カバレッジ計測 |
| mypy | 静的型チェック |
| ruff | リンター・フォーマッター |
