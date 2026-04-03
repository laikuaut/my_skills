# ライブラリプロジェクト構成

PyPIパッケージや社内共有ライブラリ向けの構成。`src/`レイアウトを採用する。

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
│       └── py.typed
└── tests/
    ├── __init__.py
    └── conftest.py
```

## タイプ固有の設定

### pyproject.toml 追記

```toml
[project.urls]
Repository = ""
Documentation = ""

[tool.hatch.build.targets.wheel]
packages = ["src/<package_name>"]

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["src"]
```

### src/<package_name>/__init__.py

```python
"""<project-name>パッケージ。"""

__version__ = "0.1.0"
```

### src/<package_name>/py.typed

空ファイル（PEP 561 型情報マーカー）。

### tests/__init__.py

空ファイル。

### Dockerfile CMD

```dockerfile
CMD ["python", "-c", "import <package_name>; print(<package_name>.__version__)"]
```

### docker-compose.yaml

共通テンプレートのままでよい（ポート不要）。

### start.sh 追記

```bash
echo "テストを実行します..."
uv run pytest
```

### README.md 追記

```markdown
## インストール

\```bash
pip install <project-name>
\```

## 使い方

\```python
import <package_name>
\```
```

## 検証項目

- [ ] `src/<package_name>/__init__.py` が存在し、`__version__`が定義されているか
- [ ] `src/<package_name>/py.typed` が存在するか
- [ ] `tests/__init__.py` が存在するか
- [ ] pyproject.tomlに`tool.hatch.build.targets.wheel`が設定されているか
