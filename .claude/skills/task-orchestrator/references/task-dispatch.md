# タスク → 子スキル マッピング表

タスクの文言・成果物から担当スキルを推定する早見表。複数該当する場合は一番上に書いてある方を優先。

## 主要マッピング

| タスクの特徴 | 担当スキル | 備考 |
|---|---|---|
| 「設計して」「要求整理」「requirements.md」「concept」「architecture」「api-detail」「schemas」 | `design-orchestrator` | 複数の設計ドキュメントに跨る |
| 個別の設計ドキュメント 1 本だけ書く（例: "concept.md 書き直して"） | `design-orchestrator` | 内部で references を使う |
| 「1 機能フルスタックで作って」「バックエンドとフロント両方」 | `fullstack-pipeline` | 複数フェーズを束ねる |
| 新規 Python プロジェクト / FastAPI プロジェクト / CLI 雛形 | `python-project-init` | 空ディレクトリから |
| 既存 Python コードの追加・修正・リファクタ | `python-impl` | docstring / 型ヒント / ruff |
| 新規 React プロジェクト / Vite / SPA 雛形 | `react-project-init` | 空ディレクトリから |
| 既存 React コードの追加・修正・リファクタ | `react-impl` | TSDoc / 型 / コンポーネント分割 |
| 「OpenAPI 生成」「Swagger 作って」「openapi.yaml 更新」 | `openapi-docs` | 実装コードから生成 |
| 「API テスト」「curl テスト」「結合試験」 | `rest-api-test` | OpenAPI から生成 |
| 「画面仕様書」「リバース」「UI ドキュメント」 | `screen-spec-reverse` | React 実装から |
| 「E2E」「Playwright」「ブラウザテスト」 | `playwright-test` | 画面仕様書 or 実装から |
| 「コミット」「プッシュ」「変更まとめて」 | `git-commit-push` | 区切りごと |
| 「画面モック」「ビジュアル UI」「デザイン性の高い UI」 | `frontend-design` プラグイン | **別セッション推奨** |

## 迷ったときの判断基準

1. **タスクが複数の成果物に跨るか？** → Yes なら上位オーケストレータ（`design-orchestrator` / `fullstack-pipeline`）
2. **新規ディレクトリ作成か既存編集か？** → 新規なら `*-project-init`、既存なら `*-impl`
3. **成果物が docs/ 配下か実装コードか？** → docs/ なら設計系 or openapi-docs / screen-spec-reverse、実装は impl 系
4. **どれにも該当しない** → メインで自分が対応せず、ユーザに「このタスク、どのスキルに投げますか？」と聞く

## スキル未存在のタスク

子スキルが用意されていない領域（例: データベースマイグレーション特化、CI/CD 設定、インフラ構築）は:

- 小さな作業ならメインセッションで直接対応してよい（本スキルの "実行範囲" 例外として明示）
- 大きな作業なら「この領域は専用スキルが無いので、別途スキルを作るか手動で進めるかを検討してください」とユーザに伝える

## 多段委譲の例

タスク「ユーザー管理機能を追加」を受けたときの典型的な委譲:

```
task-orchestrator
  └─ fullstack-pipeline
       ├─ (Phase 2) API 設計 → 自力 or design-orchestrator
       ├─ (Phase 3) python-impl で FastAPI 実装
       ├─ (Phase 4) openapi-docs で仕様書生成
       ├─ (Phase 5) rest-api-test でテスト
       ├─ (Phase 6) react-impl で画面実装
       ├─ (Phase 7) screen-spec-reverse で仕様書
       ├─ (Phase 8) playwright-test で E2E
       └─ (Phase 9) git-commit-push でコミット
```

本スキルが `fullstack-pipeline` を起動したあとは、内部の Phase 遷移は `fullstack-pipeline` 側に任せる（口出ししない）。
