---
name: python-project-init
description: |
  Pythonプロジェクト雛形を新規生成するスキル。uv + ruff + pytest を共通仕様、library / backend(FastAPI) / command(CLI) の3タイプから選択する。Dockerはbackend以外ではオプション。
  使用するケース: 「Pythonプロジェクトを作って」「新規プロジェクト」「雛形を作って」「FastAPIプロジェクト」「CLIツールを作りたい」「ライブラリを作りたい」「uvで初期化」など、空ディレクトリからのスケルトン作成。
  使わないケース: 既存コードの編集・追加・リファクタリング（python-impl を使う）、依存追加だけ、設定ファイルのみの修正。
---

# Python Project Init

Pythonプロジェクトの雛形を生成するスキル。ユーザの指定に応じて3つの構成から選択し、必要なファイルをすべて生成する。

## フロー

1. ユーザにプロジェクト名と構成タイプを確認する
2. 対応するリファレンスを読み、ファイルを生成する
3. 構成検証を実行する

## プロジェクトタイプ

| タイプ | 用途 | リファレンス |
|---|---|---|
| library | PyPIパッケージ、社内ライブラリ | `references/library.md` |
| backend | FastAPI WebAPI | `references/backend.md` |
| command | CLIツール（argparse） | `references/command.md` |

ユーザが明示しない場合は用途をヒアリングして判断する。迷ったら`backend`を提案する。

## 共通仕様

すべてのタイプで以下を生成する。Dockerfile/docker-compose.yamlは`backend`タイプではデフォルト生成、`library`/`command`では「Dockerが必要」とユーザが明示した場合のみ生成する。

`pyproject.toml`の`version`は初期値`0.1.0`としてユーザ確認を取る（ユーザが指定すればそれを使う）。

### pyproject.toml

```toml
[project]
name = "<project-name>"
version = "0.1.0"
description = ""
readme = "README.md"
requires-python = ">=3.12"
dependencies = []

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.backends"

[tool.uv]
dev-dependencies = [
    "pytest>=8.0",
    "pytest-cov>=5.0",
]

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

[tool.pytest.ini_options]
testpaths = ["tests"]
```

### Dockerfile（マルチステージビルド、backend向けデフォルト / library・commandは任意）

```dockerfile
FROM python:3.12-slim AS builder

WORKDIR /app

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

COPY pyproject.toml uv.lock* ./
RUN uv sync --frozen --no-install-project --no-dev

COPY . .
RUN uv sync --frozen --no-dev

FROM python:3.12-slim AS runtime

WORKDIR /app

COPY --from=builder /app/.venv /app/.venv
COPY --from=builder /app .

ENV PATH="/app/.venv/bin:$PATH"
```

`runtime`ステージの`CMD`はタイプ別リファレンスで定義する。

### docker-compose.yaml

```yaml
services:
  app:
    build: .
    container_name: <project-name>
    restart: unless-stopped
```

ポート設定等はタイプ別リファレンスで追記する。

### .gitignore

```
__pycache__/
*.py[cod]
*.egg-info/
dist/
build/
.venv/
.env
*.lock
.pytest_cache/
.ruff_cache/
.coverage
htmlcov/
```

### start.sh

```bash
#!/bin/bash
set -euo pipefail

# uvがなければインストール
if ! command -v uv &> /dev/null; then
    echo "uvをインストールします..."
    curl -LsSf https://astral.sh/uv/install.sh | sh
    source "$HOME/.local/bin/env"
fi

# 依存関係のインストール
uv sync

echo "セットアップ完了"
```

起動コマンドはタイプ別リファレンスで追記する。

### README.md

```markdown
# <project-name>

## セットアップ

\```bash
bash start.sh
\```

## 開発

\```bash
# lint
uv run ruff check . --fix

# format
uv run ruff format .

# test
uv run pytest
\```
```

### tests/conftest.py

空ファイルで生成する。

## 生成手順

1. 共通ファイルを上記テンプレートに従って生成する
2. タイプ別リファレンス（`references/<type>.md`）を読み、タイプ固有のファイルを生成する
3. プレースホルダ（`<project-name>`等）をすべて実際の値に置換する
4. パッケージ名はプロジェクト名のハイフンをアンダースコアに変換する（例: `my-app` → `my_app`）

## 構成検証

生成後、以下をチェックする:

- [ ] pyproject.tomlが存在し、project.nameが正しいか
- [ ] Dockerfileが存在するか
- [ ] docker-compose.yamlが存在するか
- [ ] .gitignoreが存在するか
- [ ] start.shが存在し、実行権限があるか
- [ ] テストディレクトリ（tests/）が存在するか
- [ ] タイプ固有のエントリーポイントが存在するか（リファレンス参照）
- [ ] README.mdが存在するか

検証結果を一覧でユーザに報告する。
