# バックエンドプロジェクト構成

FastAPI WebAPI向けの構成。`app/`レイアウトを採用する。

## ディレクトリ構成

```
<project-name>/
├── pyproject.toml
├── Dockerfile
├── docker-compose.yaml
├── .gitignore
├── start.sh
├── README.md
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── api/
│   │   ├── __init__.py
│   │   └── routes/
│   │       ├── __init__.py
│   │       └── health.py
│   ├── models/
│   │   └── __init__.py
│   └── services/
│       └── __init__.py
└── tests/
    ├── __init__.py
    └── conftest.py
```

## タイプ固有の設定

### pyproject.toml 追記

`dependencies`に以下を追加:

```toml
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.34",
]
```

`dev-dependencies`に以下を追加:

```toml
[tool.uv]
dev-dependencies = [
    "pytest>=8.0",
    "pytest-cov>=5.0",
    "httpx>=0.28",
]
```

### app/__init__.py

空ファイル。

### app/main.py

```python
"""アプリケーションエントリーポイント。"""

from fastapi import FastAPI

from app.api.routes import health

app = FastAPI(
    title="<project-name> API",
    description="",
    version="0.1.0",
)

app.include_router(health.router)
```

### app/config.py

```python
"""アプリケーション設定。"""

from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    """環境変数から読み込む設定。"""

    app_env: str = "development"
    debug: bool = False

    model_config = {"env_prefix": "APP_"}


settings = Settings()
```

`pydantic-settings`を`dependencies`に追加すること:

```toml
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.34",
    "pydantic-settings>=2.0",
]
```

### app/api/__init__.py

空ファイル。

### app/api/routes/__init__.py

空ファイル。

### app/api/routes/health.py

```python
"""ヘルスチェックエンドポイント。"""

from fastapi import APIRouter

router = APIRouter(tags=["health"])


@router.get("/health", summary="ヘルスチェック")
async def health_check() -> dict[str, str]:
    """アプリケーションの稼働状態を返す。"""
    return {"status": "ok"}
```

### app/models/__init__.py

空ファイル。

### app/services/__init__.py

空ファイル。

### tests/__init__.py

空ファイル。

### Dockerfile CMD

```dockerfile
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### docker-compose.yaml

```yaml
services:
  app:
    build: .
    container_name: <project-name>
    ports:
      - "8000:8000"
    environment:
      - APP_ENV=development
      - APP_DEBUG=true
    restart: unless-stopped
```

### start.sh 追記

```bash
echo "開発サーバーを起動します..."
uv run uvicorn app.main:app --reload --port 8000
```

### README.md 追記

```markdown
## 起動

\```bash
bash start.sh
\```

API ドキュメント: http://localhost:8000/docs

## Docker

\```bash
docker compose up --build
\```
```

## 検証項目

- [ ] `app/main.py` が存在し、FastAPIインスタンスが定義されているか
- [ ] `app/config.py` が存在するか
- [ ] `app/api/routes/health.py` が存在するか
- [ ] pyproject.tomlの`dependencies`に`fastapi`と`uvicorn`が含まれるか
- [ ] docker-compose.yamlにポート`8000`が設定されているか
- [ ] `tests/__init__.py` が存在するか
