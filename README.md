# my_skills

Claude Code用カスタムスキル / エージェントの管理リポジトリ。

## スキル一覧

### オーケストレーション

| スキル | 説明 |
|---|---|
| [fullstack-pipeline](.claude/skills/fullstack-pipeline/) | FastAPI + React のフルスタック開発を一気通貫で進める手順管理。各フェーズは個別スキルへ委譲 |

### 実装品質基準

| スキル | 説明 |
|---|---|
| [python-impl](.claude/skills/python-impl/) | Pythonの実装品質基準（docstring・型ヒント・ruff・クラス設計） |
| [react-impl](.claude/skills/react-impl/) | React+TSの実装品質基準（TSDoc・型・分割・カスタムフック・Tailwind） |

### プロジェクト初期化

| スキル | 説明 |
|---|---|
| [python-project-init](.claude/skills/python-project-init/) | Python雛形生成（uv+ruff+pytest、library/backend/command の3タイプ） |
| [react-project-init](.claude/skills/react-project-init/) | React雛形生成（Vite+TS+Tailwind+Vitest、spa/dashboard の2タイプ） |

### ドキュメント生成

| スキル | 説明 |
|---|---|
| [openapi-docs](.claude/skills/openapi-docs/) | バックエンド実装からOpenAPI 3.1 YAMLを生成・差分更新 |
| [screen-spec-reverse](.claude/skills/screen-spec-reverse/) | React/Next.js実装から画面仕様書をリバース生成（キャプチャはオプション） |

### テスト

| スキル | 説明 |
|---|---|
| [rest-api-test](.claude/skills/rest-api-test/) | OpenAPIから網羅的なcurl結合試験スイートを生成 |
| [playwright-test](.claude/skills/playwright-test/) | E2Eテストの生成→実行→バグ修正を一括 |

### Git運用

| スキル | 説明 |
|---|---|
| [git-commit-push](.claude/skills/git-commit-push/) | Git差分確認・コミット分割・接頭辞付き2行メッセージ生成・プッシュ |

## 関連エージェント（user-level: `~/.claude/agents/`）

| エージェント | 役割 |
|---|---|
| skill-quality-reviewer | SKILL.md品質をA-Fでレビュー（frontmatter・構造・具体性・責務・アンチパターン） |
| skill-description-optimizer | SKILL.md frontmatter `description` を1024字以内・トリガー条件付きに最適化 |

## ディレクトリ構成

```
my_skills/
├── README.md
├── CLAUDE.md
└── .claude/
    └── skills/
        └── <skill-name>/
            ├── SKILL.md
            ├── references/   ← 詳細ガイド・テンプレ（必要時のみ参照）
            ├── scripts/      ← 補助スクリプト
            └── evals/        ← トリガー精度テスト
```

各スキルは `.claude/skills/<skill-name>/SKILL.md` に定義する。新規作成・改善は `/skill-creator` を使う。
