# ログ設計ガイド

大規模Pythonプロジェクト（10ファイル以上）向けのログ設計リファレンス。

## 目次

1. [基本設定](#基本設定)
2. [ログレベル設計](#ログレベル設計)
3. [構造化ログ](#構造化ログ)
4. [ログ設定の一元管理](#ログ設定の一元管理)
5. [パフォーマンス考慮](#パフォーマンス考慮)

## 基本設定

### ロガーの取得

モジュールごとに`__name__`でロガーを取得する。これによりログの出所が自動的に記録される。

```python
import logging

logger = logging.getLogger(__name__)
```

`logging.basicConfig()`はエントリーポイント（`main.py`）でのみ呼ぶ。ライブラリコードでは呼ばない。

### エントリーポイントでの設定

```python
# main.py
import logging
import sys

def setup_logging(level: str = "INFO") -> None:
    """アプリケーション全体のログ設定を行う。

    Args:
        level: ログレベル文字列。環境変数LOG_LEVELで上書き可能。
    """
    log_level = getattr(logging, level.upper(), logging.INFO)
    
    logging.basicConfig(
        level=log_level,
        format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
        handlers=[
            logging.StreamHandler(sys.stdout),
        ],
    )
    
    # サードパーティのログレベルを抑制
    logging.getLogger("httpx").setLevel(logging.WARNING)
    logging.getLogger("urllib3").setLevel(logging.WARNING)
```

## ログレベル設計

| レベル | 用途 | 例 |
|--------|------|-----|
| DEBUG | 開発・デバッグ用の詳細情報 | SQL文、リクエスト/レスポンスの中身 |
| INFO | 正常な業務イベント | ユーザー登録、注文完了、バッチ開始/終了 |
| WARNING | 異常だが処理続行可能 | レート制限接近、非推奨APIの使用、リトライ |
| ERROR | エラー発生、該当処理は失敗 | 決済失敗、外部API応答エラー |
| CRITICAL | システム全体に影響 | DB接続不可、設定ファイル読み込み失敗 |

### 使い分けの指針

- **INFO以上**が本番で出力される前提で書く
- DEBUGは「このログがなくても本番運用に支障はない」もの
- WARNINGは「今は大丈夫だが、放置すると問題になる」もの
- ERRORは「この処理は失敗した」もの（システム全体は動いている）
- CRITICALは「アプリケーションの継続が困難」もの

## 構造化ログ

本番環境ではJSON形式のログを使う。ログ集約サービス（CloudWatch、Datadog等）での検索・集計が容易になる。

```python
import json
import logging
from typing import Any


class JsonFormatter(logging.Formatter):
    """JSON形式でログを出力するフォーマッター。

    ログ集約サービスでのパース・検索を容易にする。
    """

    def format(self, record: logging.LogRecord) -> str:
        """ログレコードをJSON文字列に変換する。

        Args:
            record: ログレコード。

        Returns:
            JSON形式のログ文字列。
        """
        log_data: dict[str, Any] = {
            "timestamp": self.formatTime(record, self.datefmt),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
        }
        
        # 例外情報があれば追加
        if record.exc_info and record.exc_info[0] is not None:
            log_data["exception"] = self.formatException(record.exc_info)
        
        # extra フィールドの追加
        for key, value in record.__dict__.items():
            if key not in logging.LogRecord(
                "", 0, "", 0, "", (), None
            ).__dict__ and key != "message":
                log_data[key] = value
        
        return json.dumps(log_data, ensure_ascii=False, default=str)
```

### コンテキスト情報の付与

リクエストIDやユーザーIDなど、追跡に必要な情報をログに含める。

```python
import logging
from contextvars import ContextVar

# リクエストスコープのコンテキスト
request_id_var: ContextVar[str] = ContextVar("request_id", default="")


class ContextFilter(logging.Filter):
    """コンテキスト変数からリクエスト情報をログに付与するフィルター。"""

    def filter(self, record: logging.LogRecord) -> bool:
        """ログレコードにリクエストIDを付与する。

        Args:
            record: ログレコード。

        Returns:
            常にTrue（フィルタリングはしない）。
        """
        record.request_id = request_id_var.get("")  # type: ignore[attr-defined]
        return True
```

## ログ設定の一元管理

`logging.config.dictConfig`で設定を一元管理する。

```python
import logging.config

LOGGING_CONFIG: dict = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "standard": {
            "format": "%(asctime)s [%(levelname)s] %(name)s: %(message)s",
        },
        "json": {
            "()": "app.logging.JsonFormatter",
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "standard",
            "stream": "ext://sys.stdout",
        },
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "formatter": "json",
            "filename": "app.log",
            "maxBytes": 10_485_760,  # 10MB
            "backupCount": 5,
        },
    },
    "loggers": {
        "app": {
            "level": "DEBUG",
            "handlers": ["console", "file"],
            "propagate": False,
        },
    },
    "root": {
        "level": "WARNING",
        "handlers": ["console"],
    },
}
```

## パフォーマンス考慮

### 遅延評価

ログメッセージの文字列フォーマットは`%`記法を使う。ログレベルが無効な場合、文字列生成がスキップされる。

```python
# 良い：ログレベルが無効なら文字列生成されない
logger.debug("処理結果: count=%d, items=%s", count, items)

# 悪い：ログレベルに関係なくf-stringが評価される
logger.debug(f"処理結果: count={count}, items={items}")
```

### 重い処理のガード

```python
if logger.isEnabledFor(logging.DEBUG):
    # この中だけで重い処理を実行
    detailed_info = expensive_debug_calculation()
    logger.debug("詳細情報: %s", detailed_info)
```

### ログの量に注意

- ループ内のログは`DEBUG`レベルに留める
- 大量データのログは件数やサマリーだけにする
- 外部APIのレスポンス全文はDEBUGレベルで出す
