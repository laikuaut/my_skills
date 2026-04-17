---
name: python-impl
description: |
  Pythonコードの実装品質基準（Google Style docstring・型ヒント・ruff lint/format・クラス設計・ファイル分割）を自動適用するスキル。既存プロジェクト内でのコード作成・編集・リファクタリングが対象。
  使用するケース: 「Pythonで実装して」「型ヒントつけて」「docstring書いて」「リファクタリングして」「PEP8準拠で」「ruffで整えて」など、Pythonコードの作成・改修依頼。
  使わないケース: プロジェクト新規作成（python-project-initを使う）、Pythonの実行・デバッグだけ、テストの実行のみ。
  FastAPI実装やログ設計の詳細は references/fastapi_guide.md / references/logging_guide.md に分離（必要時のみ参照）。
---

# Python実装スキル

Pythonコードを書くとき、このスキルの基準に従って実装する。新規作成でもリファクタリングでも同じ基準を適用する。

## コーディング規約

### 1. Docstring（Google Style）

すべての公開モジュール・クラス・関数にdocstringを書く。Google Styleに従う。

```python
def calculate_total_price(
    items: list[CartItem],
    discount_rate: float = 0.0,
) -> Decimal:
    """カート内商品の合計金額を計算する。

    割引率が指定された場合、合計金額に割引を適用する。
    税込み価格はCartItem側で保持している前提。

    Args:
        items: カート内の商品リスト。空リストの場合は0を返す。
        discount_rate: 割引率（0.0〜1.0）。デフォルトは割引なし。

    Returns:
        割引適用後の合計金額。小数点以下2桁で丸める。

    Raises:
        ValueError: discount_rateが0.0〜1.0の範囲外の場合。

    Examples:
        >>> items = [CartItem(name="Book", price=Decimal("1500"))]
        >>> calculate_total_price(items, discount_rate=0.1)
        Decimal('1350.00')
    """
```

**判断基準**: 「この関数を初めて読む人が、docstringだけで正しく使えるか？」を基準にする。引数名から明らかなことは省略してよいが、制約条件・副作用・エッジケースは必ず書く。

### 2. コメント

コードの「なぜ（Why）」を説明するコメントを書く。「何をしているか（What）」はコード自体が語るべき。

```python
# 良い例：なぜそうするのかを説明
# 在庫データはキャッシュから取得する。DBへの直接アクセスは
# ピーク時にレイテンシが急増するため回避する（2024-03 障害対応で判明）
cached_stock = cache.get(f"stock:{product_id}")

# 悪い例：コードの直訳
# キャッシュから在庫を取得する
cached_stock = cache.get(f"stock:{product_id}")
```

**書くべきコメント**:
- ビジネスロジックの背景（なぜこの計算式なのか）
- パフォーマンス上の理由による設計判断
- 外部仕様への準拠（「RFC 7231 Section 6.5.1 に従い...」など）
- ワークアラウンドとその理由（将来削除できる条件も書く）
- TODOには担当・期限・チケット番号を含める（`# TODO(user): #1234 期限2024-06`）

### 3. タイプヒント

すべての関数の引数・戻り値にタイプヒントをつける。Python 3.10+のUnion記法（`X | None`）を使う。

```python
from collections.abc import Sequence

def find_users(
    query: str,
    *,
    limit: int = 50,
    include_inactive: bool = False,
) -> list[User]:
    ...

def get_config(key: str) -> str | None:
    ...

# コレクション型はcollections.abcの抽象型を引数に使う
def process_items(items: Sequence[Item]) -> list[Result]:
    ...
```

**ルール**:
- `Any`は最終手段。まず`Protocol`や`TypeVar`で表現できないか検討する
- `dict`の中身が複雑なら`TypedDict`を使う
- コールバックには`Callable`ではなく`Protocol`を優先する（引数名が自己文書化される）

### 4. Import順序（isort互換）

```python
# 1. 標準ライブラリ
import logging
import os
from collections.abc import Sequence
from pathlib import Path

# 2. サードパーティ
import httpx
from pydantic import BaseModel, Field

# 3. ローカル（プロジェクト内）
from app.core.config import Settings
from app.models.user import User
```

グループ間は1行空行。グループ内はアルファベット順。`from`インポートは`import`の後に置く。

### 5. Lint・Format（ruff）

コードを書いたら、以下のコマンドで品質をチェックする：

```bash
# lint チェック
ruff check <対象パス> --fix

# フォーマット
ruff format <対象パス>

# import順序の整理（ruffに統合済み）
ruff check <対象パス> --select I --fix
```

ruffが未インストールの場合は`pip install ruff`を実行する。`pyproject.toml`にruff設定がなければ以下を追加する：

```toml
[tool.ruff]
target-version = "py312"
line-length = 88

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort
    "N",    # pep8-naming
    "UP",   # pyupgrade
    "B",    # flake8-bugbear
    "SIM",  # flake8-simplify
    "ANN",  # flake8-annotations
    "D",    # pydocstyle
]
ignore = ["ANN101", "ANN102", "D100"]

[tool.ruff.lint.pydocstyle]
convention = "google"
```

### 6. メソッド分割

1つのメソッドは**1つの責務**を持つ。目安として20行を超えたら分割を検討する。

**分割の判断基準**:
- 異なる抽象度の処理が混在している → 抽象度ごとに分割
- 同じ処理が2箇所以上に出現 → メソッドとして抽出
- コメントで「ここから〇〇処理」と区切っている → その単位で分割

```python
# 悪い例：1メソッドに複数の責務
def process_order(order: Order) -> Receipt:
    # バリデーション（20行）...
    # 在庫チェック（15行）...
    # 決済処理（25行）...
    # レシート生成（10行）...
    pass

# 良い例：責務ごとに分割
def process_order(order: Order) -> Receipt:
    """注文を処理してレシートを返す。"""
    self._validate_order(order)
    self._reserve_stock(order.items)
    payment_result = self._execute_payment(order)
    return self._create_receipt(order, payment_result)
```

### 7. ファイル分割

**1ファイル1モジュール**が原則。ファイルが300行を超えたら分割を検討する。

推奨ディレクトリ構成（中〜大規模）：

```
project/
├── pyproject.toml
├── src/
│   └── package_name/
│       ├── __init__.py
│       ├── main.py              # エントリーポイント
│       ├── config.py            # 設定管理
│       ├── models/              # データモデル
│       │   ├── __init__.py
│       │   ├── user.py
│       │   └── order.py
│       ├── services/            # ビジネスロジック
│       │   ├── __init__.py
│       │   ├── user_service.py
│       │   └── order_service.py
│       ├── repositories/        # データアクセス
│       │   ├── __init__.py
│       │   └── user_repository.py
│       ├── api/                 # APIエンドポイント
│       │   ├── __init__.py
│       │   └── routes/
│       ├── utils/               # ユーティリティ
│       │   ├── __init__.py
│       │   └── date_helpers.py
│       └── exceptions.py        # カスタム例外
├── tests/
│   ├── conftest.py
│   ├── test_user_service.py
│   └── test_order_service.py
└── scripts/                     # 運用スクリプト
```

小規模（〜5ファイル）の場合は`src/`レイヤーなしのフラット構成でよい。

### 8. クラス設計

- **単一責任の原則**: 1クラス1責務
- **継承よりコンポジション**: 共通処理はミックスインか委譲で解決
- **データクラス**: 振る舞いのないデータ構造には`dataclass`か`pydantic.BaseModel`を使う
- **プロトコル**: ダックタイピングを型安全にするため`Protocol`を活用する

```python
from dataclasses import dataclass
from typing import Protocol


class PaymentProcessor(Protocol):
    """決済処理のインターフェース。"""

    def charge(self, amount: Decimal, token: str) -> PaymentResult:
        """決済を実行する。"""
        ...


@dataclass(frozen=True)
class Money:
    """金額を表す値オブジェクト。

    通貨単位を明確にし、計算の安全性を保証する。
    """

    amount: Decimal
    currency: str = "JPY"

    def add(self, other: "Money") -> "Money":
        """同一通貨の加算。"""
        if self.currency != other.currency:
            raise ValueError(f"通貨が異なります: {self.currency} != {other.currency}")
        return Money(amount=self.amount + other.amount, currency=self.currency)
```

### 9. 共通処理の抽出

**3回以上**出現するパターンは共通化を検討する。ただし、過度な抽象化は避ける。

共通化すべきもの：
- エラーハンドリングパターン（リトライ、フォールバック）
- バリデーションロジック
- データ変換（DTO ↔ Entity）
- 外部API呼び出しのラッパー

```python
# utils/retry.py
import asyncio
import logging
from collections.abc import Callable
from typing import TypeVar

logger = logging.getLogger(__name__)
T = TypeVar("T")


async def retry_async(
    func: Callable[..., T],
    *args: object,
    max_retries: int = 3,
    backoff_factor: float = 1.0,
) -> T:
    """非同期関数をリトライ付きで実行する。

    指数バックオフで間隔を空けながらリトライする。
    すべてのリトライが失敗した場合は最後の例外をraiseする。

    Args:
        func: 実行する非同期関数。
        *args: funcに渡す引数。
        max_retries: 最大リトライ回数。
        backoff_factor: バックオフの基準秒数。

    Returns:
        funcの戻り値。

    Raises:
        Exception: すべてのリトライが失敗した場合、最後の例外。
    """
    last_exception: Exception | None = None
    for attempt in range(max_retries + 1):
        try:
            return await func(*args)
        except Exception as e:
            last_exception = e
            if attempt < max_retries:
                wait = backoff_factor * (2 ** attempt)
                logger.warning(
                    "リトライ %d/%d: %s（%s秒後に再試行）",
                    attempt + 1, max_retries, e, wait,
                )
                await asyncio.sleep(wait)
    raise last_exception  # type: ignore[misc]
```

### 10. ログ設計

標準`logging`モジュールを使う。基本原則:

- ロガーは`__name__`で取得: `logger = logging.getLogger(__name__)`
- f-string でなく `%s` プレースホルダで渡す（フォーマットコストの遅延評価のため）
- 例外時は `logger.exception()` でスタックトレースを残す
- 個人情報・パスワード・トークンはログに出さない

ログレベルの使い分け:

```python
import logging

logger = logging.getLogger(__name__)

logger.debug("クエリ実行: %s params=%s", query, params)        # 開発時のみ
logger.info("ユーザー登録完了: user_id=%s", user.id)            # 正常な業務イベント
logger.warning("APIレート制限に接近: remaining=%d", remaining)  # 注意が必要
logger.error("決済処理失敗: order_id=%s error=%s", order_id, e) # エラーだが継続可能
logger.critical("DB接続不可: %s", connection_string)            # システム停止レベル
```

複数チーム・マイクロサービス・本番運用が前提のプロジェクトでは構造化ログ（JSON形式）・ハンドラ設定・相関IDなどが必要になる。詳細は `references/logging_guide.md` を参照する。

### 11. FastAPI実装（API開発時）

FastAPIを使う場合は、バリデーション（Pydantic Field）と Swagger UI ドキュメントを必須とする。詳細とテンプレートは `references/fastapi_guide.md` を参照する。

## リファクタリング

既存コードのリファクタリングを依頼された場合も、上記すべての基準を適用する。加えて：

1. **現状把握を先に行う**: まずコードを読み、問題点を洗い出す
2. **段階的に進める**: 一度に全部変えず、1つの改善を適用→動作確認→次の改善
3. **テストを先に書く**: リファクタリング前に既存の振る舞いをテストで固定する
4. **振る舞いを変えない**: リファクタリングは外部から見た振る舞いを保つ。機能追加と混ぜない

リファクタリングの優先順位：
1. タイプヒントの追加（型安全性の確保が最優先）
2. 関数・メソッドの分割（可読性向上）
3. クラス構造の整理（責務の明確化）
4. 共通処理の抽出（DRY原則）
5. docstring・コメントの追加（保守性向上）
6. lint/formatの適用（一貫性の確保）

## 品質チェックリスト

コードを書き終えたら、以下を確認する：

- [ ] すべての公開関数にGoogle Styleのdocstringがあるか
- [ ] すべての関数にタイプヒントがあるか
- [ ] importがisort順になっているか
- [ ] `ruff check`がエラーなしで通るか
- [ ] `ruff format --check`が差分なしで通るか
- [ ] 1メソッドが20行以内か（超えている場合は分割を検討したか）
- [ ] 1ファイルが300行以内か（超えている場合は分割を検討したか）
- [ ] 大規模プロジェクトならログ設計がされているか
- [ ] コメントが「なぜ」を説明しているか
