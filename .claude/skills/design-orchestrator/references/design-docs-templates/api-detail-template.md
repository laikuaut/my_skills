# api-detail.md テンプレート（D3.1 API詳細）

```markdown
# API詳細 (D3.1)

- 対象: REST エンドポイント一覧と詳細仕様
- 作成日: YYYY-MM-DD
- 関連: [data-layer.md](./data-layer.md) → 本書 → [schemas.md](./schemas.md) → [../openapi.yaml](../openapi.yaml)
- 状態: draft

## 1. 設計原則

- **リソース指向**。動詞は HTTP メソッドで表現
- ページネーションは offset/limit またはカーソル。v1 は offset/limit で統一
- エラーは RFC 7807（error-handling.md 準拠）
- 認証は Cookie セッション（architecture.md 準拠）

## 2. ベースURL & 共通規約

- ベース: `/api`
- 日時: ISO 8601 / UTC
- ID: ULID 文字列
- 一覧レスポンス: `{ items: [...], total: N, offset: N, limit: N }`

## 3. エンドポイント一覧

| メソッド | パス | 概要 | 認証 |
|---------|------|------|------|
| POST | /api/auth/login | ログイン | 不要 |
| POST | /api/auth/logout | ログアウト | 要 |
| GET | /api/me | 自分の情報 | 要 |
| GET | /api/tasks | タスク一覧 | 要 |
| POST | /api/tasks | タスク作成 | 要 |
| GET | /api/tasks/{id} | タスク詳細 | 要 |
| PATCH | /api/tasks/{id} | タスク更新 | 要 |
| DELETE | /api/tasks/{id} | タスク削除 | 要 |
| GET | /api/tags | タグ一覧 | 要 |

## 4. 各エンドポイント詳細（抜粋）

### POST /api/tasks

- 認証: 要
- Request
  ```json
  { "title": "買い物", "status": "todo", "tag_ids": ["01HXX..."] }
  ```
- Response 201
  ```json
  { "id": "01HY...", "title": "買い物", "status": "todo", ... }
  ```
- Errors
  - 400: title が 1..200 文字外
  - 404: tag_ids に未存在ID
  - 422: status が enum 外

### PATCH /api/tasks/{id}
...

（全エンドポイントを同じ粒度で記述）

## 5. エラー例

（error-handling.md の形式に準拠。エンドポイントごとに起こりうるエラーを列挙）

## 6. 引継ぎメモ

- D3.2 schemas へ: ここで示した JSON 形状を Pydantic / Zod スキーマに
- D3.3 openapi.yaml へ: 同じ内容を OpenAPI 3.1 形式で出力

## 7. 完了判定 (DoD)

- [ ] エンドポイント一覧表が揃っている
- [ ] 主要3〜5エンドポイントに Request / Response / Errors が書かれている
- [ ] 共通規約（ID 形式、日時形式、一覧形状）が決まっている
```
