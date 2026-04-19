# mcp-spec.md テンプレート（D4.1 MCP仕様・任意）

MCP 連携をするプロダクトのみ作成する。

```markdown
# MCP 仕様 (D4.1)

- 対象: Model Context Protocol サーバの Tools / Resources / Prompts
- 作成日: YYYY-MM-DD
- 関連: [architecture.md](./architecture.md) / [api-detail.md](./api-detail.md) → 本書 → [mcp-integration.md](./mcp-integration.md)
- 状態: draft

## 1. 設計原則

- **Tools = 副作用のある操作**（書き込み・状態変更）
- **Resources = 読み取り可能な情報**（キャッシュ可能・冪等）
- **Prompts = 定型プロンプト**（ユーザーに提示する起動テンプレ）
- 呼び出し元は Claude Code / Claude Desktop などのホスト

## 2. サーバ構成

- 名前: `{プロダクト名}-mcp`
- トランスポート: stdio / HTTP SSE
- 起動コマンド例: `uvx run {パッケージ名}`

## 3. Tools 一覧

| 名前 | 概要 | 入力スキーマ | 副作用 |
|------|------|-------------|--------|
| create_task | タスク作成 | {title, status, tag_ids} | DB 書き込み |
| update_task | タスク更新 | {id, title?, status?, tag_ids?} | DB 更新 |
| delete_task | タスク削除 | {id} | DB 削除 |

### 各 Tool の詳細（例: create_task）

```json
{
  "name": "create_task",
  "description": "新しいタスクを作成する",
  "inputSchema": {
    "type": "object",
    "required": ["title"],
    "properties": {
      "title": {"type": "string", "maxLength": 200},
      "status": {"type": "string", "enum": ["todo","doing","done"]},
      "tag_ids": {"type": "array", "items": {"type": "string"}}
    }
  }
}
```

- 成功時の戻り値: 作成したタスクの JSON
- エラー時: `content[].type = "text"` にエラーメッセージ（RFC 7807 の detail）

## 4. Resources 一覧

| URI | 概要 | MIME |
|-----|------|------|
| `task://list` | タスク一覧 | application/json |
| `task://{id}` | タスク詳細 | application/json |
| `tag://list` | タグ一覧 | application/json |

## 5. Prompts 一覧

| 名前 | 概要 | 引数 |
|------|------|------|
| daily_review | 今日のタスクレビュー | {date} |
| weekly_plan | 週次計画作成 | {week} |

## 6. 認証・権限

- MCP サーバは**バックエンド API と同じ認証**を使う（Cookie / API Key）
- ホスト側の設定に API キー or セッション情報を持たせる

## 7. 引継ぎメモ

- D4.2 mcp-integration へ: Tools の内部実装は httpx で内部 API を叩く
- エラーの扱い: 全て JSON-RPC error ではなく `content[].type = "text"` で表現（ホスト側 UX 優先）

## 8. 完了判定 (DoD)

- [ ] Tools / Resources / Prompts の一覧が揃っている
- [ ] 主要 Tool に入力スキーマが書かれている
- [ ] 認証方式が明示されている
```
