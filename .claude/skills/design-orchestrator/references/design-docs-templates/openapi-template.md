# openapi.yaml テンプレート（D3.3 OpenAPI 3.1 ドラフト）

このひな形は `docs/openapi.yaml` に配置する。手書きドラフトで、実装後は `openapi-docs` スキルで自動生成版と突合する。

```yaml
openapi: 3.1.0
info:
  title: {プロダクト名} API
  version: 0.1.0
  description: |
    設計フェーズの手書きドラフト。実装後に openapi-docs スキルで更新する。
servers:
  - url: http://localhost:8000
    description: 開発環境
  - url: https://api.example.com
    description: 本番環境

tags:
  - name: auth
  - name: tasks
  - name: tags

paths:
  /api/auth/login:
    post:
      tags: [auth]
      summary: ログイン
      security: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/LoginRequest"
      responses:
        "200":
          description: 成功
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/MeResponse"
        "401":
          $ref: "#/components/responses/Unauthorized"

  /api/tasks:
    get:
      tags: [tasks]
      summary: タスク一覧
      parameters:
        - in: query
          name: status
          schema:
            $ref: "#/components/schemas/TaskStatus"
        - in: query
          name: offset
          schema: { type: integer, minimum: 0, default: 0 }
        - in: query
          name: limit
          schema: { type: integer, minimum: 1, maximum: 100, default: 50 }
      responses:
        "200":
          description: 成功
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/TaskListResponse"
    post:
      tags: [tasks]
      summary: タスク作成
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/TaskCreate"
      responses:
        "201":
          description: 作成成功
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/TaskResponse"
        "400":
          $ref: "#/components/responses/BadRequest"

components:
  securitySchemes:
    cookieSession:
      type: apiKey
      in: cookie
      name: session

  schemas:
    TaskStatus:
      type: string
      enum: [todo, doing, done]

    TaskCreate:
      type: object
      required: [title]
      properties:
        title:
          type: string
          minLength: 1
          maxLength: 200
        status:
          $ref: "#/components/schemas/TaskStatus"
        tag_ids:
          type: array
          items: { type: string }

    TaskResponse:
      type: object
      required: [id, title, status, tag_ids, created_at]
      properties:
        id: { type: string }
        title: { type: string }
        status: { $ref: "#/components/schemas/TaskStatus" }
        tag_ids:
          type: array
          items: { type: string }
        created_at:
          type: string
          format: date-time

    TaskListResponse:
      type: object
      required: [items, total, offset, limit]
      properties:
        items:
          type: array
          items: { $ref: "#/components/schemas/TaskResponse" }
        total: { type: integer }
        offset: { type: integer }
        limit: { type: integer }

    LoginRequest:
      type: object
      required: [email, password]
      properties:
        email: { type: string, format: email }
        password: { type: string }

    MeResponse:
      type: object
      required: [id, email, name]
      properties:
        id: { type: string }
        email: { type: string }
        name: { type: string }

    Problem:
      type: object
      description: RFC 7807
      properties:
        type: { type: string, format: uri }
        title: { type: string }
        status: { type: integer }
        detail: { type: string }
        errors:
          type: object
          additionalProperties:
            type: array
            items: { type: string }

  responses:
    BadRequest:
      description: バリデーションエラー
      content:
        application/problem+json:
          schema: { $ref: "#/components/schemas/Problem" }
    Unauthorized:
      description: 未認証
      content:
        application/problem+json:
          schema: { $ref: "#/components/schemas/Problem" }

security:
  - cookieSession: []
```

## 完了判定 (DoD)

- [ ] OpenAPI 3.1 として lint 通過（`npx @redocly/cli lint`）
- [ ] 全エンドポイント（api-detail.md と1対1）が含まれる
- [ ] エラーレスポンスが `application/problem+json` で定義されている
- [ ] 認証方式が明示されている
