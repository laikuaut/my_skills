# FastAPI実装ガイド

FastAPIを使う場合、以下のルールを必ず適用する。

## 1. バリデーション（必須）

すべてのリクエスト・レスポンスに`pydantic.BaseModel`でスキーマを定義する。直接`dict`や生の値を受け取らない。

```python
from pydantic import BaseModel, Field


class CreateUserRequest(BaseModel):
    """ユーザー作成リクエスト。"""

    name: str = Field(..., min_length=1, max_length=100, examples=["田中太郎"])
    email: str = Field(..., pattern=r"^[\w\.-]+@[\w\.-]+\.\w+$", examples=["taro@example.com"])
    age: int = Field(..., ge=0, le=150)


class UserResponse(BaseModel):
    """ユーザーレスポンス。"""

    id: int
    name: str
    email: str
    age: int

    model_config = {"from_attributes": True}


@app.post("/users", response_model=UserResponse, status_code=201)
async def create_user(body: CreateUserRequest) -> UserResponse:
    """ユーザーを作成する。"""
    ...
```

**ルール**:
- リクエストボディは必ず`BaseModel`サブクラスで受け取る
- `Field`で制約（`min_length`, `ge`, `pattern`等）を明示する
- パスパラメータ・クエリパラメータにも`Path()`, `Query()`で制約をつける
- レスポンスも`response_model`で型を明示する
- カスタムバリデーションが必要なら`@field_validator`を使う

```python
from fastapi import Path, Query


@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int = Path(..., gt=0, description="ユーザーID"),
    include_deleted: bool = Query(False, description="削除済みを含むか"),
) -> UserResponse:
    """指定IDのユーザーを取得する。"""
    ...
```

エラーレスポンスもスキーマを定義し、`responses`パラメータで明示する:

```python
class ErrorResponse(BaseModel):
    """エラーレスポンス。"""

    detail: str


@app.get(
    "/users/{user_id}",
    response_model=UserResponse,
    responses={404: {"model": ErrorResponse, "description": "ユーザーが見つからない"}},
)
async def get_user(user_id: int = Path(..., gt=0)) -> UserResponse:
    ...
```

## 2. APIドキュメント（必須）

Swagger UI（`/docs`）が常に正確で使いやすい状態を維持する。

**アプリケーション設定**:

```python
from fastapi import FastAPI

app = FastAPI(
    title="プロジェクト名 API",
    description="APIの概要説明",
    version="1.0.0",
)
```

**エンドポイントごとのドキュメント**:

- `summary`: 一覧に表示される短い説明（省略時は関数名から自動生成）
- `description`: 詳細説明（省略時はdocstringから自動生成）
- `response_model`: レスポンスのスキーマ
- `responses`: エラーケースのレスポンス定義
- `tags`: グループ分け

```python
@app.post(
    "/orders",
    response_model=OrderResponse,
    status_code=201,
    summary="注文を作成する",
    tags=["orders"],
    responses={
        400: {"model": ErrorResponse, "description": "バリデーションエラー"},
        409: {"model": ErrorResponse, "description": "在庫不足"},
    },
)
async def create_order(body: CreateOrderRequest) -> OrderResponse:
    """注文を作成し、在庫を引き当てる。

    決済は別途 POST /orders/{order_id}/payment で行う。
    """
    ...
```

**スキーマの`examples`**:

Swagger UIで「Try it out」したときに分かりやすいよう、`Field`や`model_config`でexamplesを設定する:

```python
class CreateUserRequest(BaseModel):
    """ユーザー作成リクエスト。"""

    name: str = Field(..., min_length=1, max_length=100, examples=["田中太郎"])
    email: str = Field(..., examples=["taro@example.com"])
    age: int = Field(..., ge=0, le=150, examples=[25])
```

**ルーターのタグ分け**:

```python
from fastapi import APIRouter

router = APIRouter(prefix="/users", tags=["users"])


@router.get("", response_model=list[UserResponse], summary="ユーザー一覧を取得する")
async def list_users() -> list[UserResponse]:
    ...
```
