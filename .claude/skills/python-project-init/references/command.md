# コマンドプロジェクト構成

CLIツール向けの構成。`argparse`を使用し、`src/`レイアウトを採用する。

## ディレクトリ構成

```
<project-name>/
├── pyproject.toml
├── Dockerfile
├── docker-compose.yaml
├── .gitignore
├── start.sh
├── README.md
├── src/
│   └── <package_name>/
│       ├── __init__.py
│       ├── __main__.py
│       └── cli.py
└── tests/
    ├── __init__.py
    └── conftest.py
```

## タイプ固有の設定

### pyproject.toml 追記

```toml
[project.scripts]
<project-name> = "<package_name>.cli:main"

[tool.hatch.build.targets.wheel]
packages = ["src/<package_name>"]

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["src"]
```

### src/<package_name>/__init__.py

```python
"""<project-name> CLIツール。"""

__version__ = "0.1.0"
```

### src/<package_name>/__main__.py

```python
"""python -m <package_name> で実行可能にする。"""

from <package_name>.cli import main

main()
```

### src/<package_name>/cli.py

```python
"""コマンドラインインターフェース。"""

import argparse
import sys

from <package_name> import __version__


def build_parser() -> argparse.ArgumentParser:
    """引数パーサーを構築する。"""
    parser = argparse.ArgumentParser(
        prog="<project-name>",
        description="",
    )
    parser.add_argument(
        "-V", "--version",
        action="version",
        version=f"%(prog)s {__version__}",
    )
    return parser


def main(argv: list[str] | None = None) -> None:
    """エントリーポイント。"""
    parser = build_parser()
    args = parser.parse_args(argv)
    print("Hello from <project-name>!")
```

### tests/__init__.py

空ファイル。

### Dockerfile CMD

```dockerfile
CMD ["python", "-m", "<package_name>"]
```

### docker-compose.yaml

共通テンプレートのままでよい（ポート不要）。

### start.sh 追記

```bash
echo "CLIを実行します..."
uv run python -m <package_name> "$@"
```

### README.md 追記

```markdown
## 使い方

\```bash
# 直接実行
bash start.sh

# インストール後
uv run <project-name>

# モジュール実行
uv run python -m <package_name>

# バージョン確認
uv run <project-name> --version
\```

## Docker

\```bash
docker compose run --rm app
\```
```

## 検証項目

- [ ] `src/<package_name>/__init__.py` が存在し、`__version__`が定義されているか
- [ ] `src/<package_name>/__main__.py` が存在するか
- [ ] `src/<package_name>/cli.py` が存在し、`main`関数が定義されているか
- [ ] pyproject.tomlに`project.scripts`が設定されているか
- [ ] `tests/__init__.py` が存在するか
