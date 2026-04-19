# schemas.md テンプレート（D3.2 Pydantic/Zodスキーマ）

```markdown
# Pydantic / Zod スキーマ (D3.2)

- 対象: バック（Pydantic）とフロント（Zod or TS型）のスキーマ定義
- 作成日: YYYY-MM-DD
- 関連: [api-detail.md](./api-detail.md) → 本書 → [../openapi.yaml](../openapi.yaml)
- 状態: draft

## 1. 設計原則

- **バックが真実の源**。フロントはそのコピーを持つ（OpenAPI 自動生成で差分ゼロが理想）
- 日時: `datetime`（UTC）/ フロントは ISO 8601 文字列
- Enum は `Literal`（Python）と union 型（TS）で揃える

## 2. Pydantic（バック）

```python
from typing import Literal
from datetime import datetime
from pydantic import BaseModel, Field

TaskStatus = Literal["todo", "doing", "done"]

class TaskCreate(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    status: TaskStatus = "todo"
    tag_ids: list[str] = Field(default_factory=list)

class TaskResponse(BaseModel):
    id: str
    title: str
    status: TaskStatus
    tag_ids: list[str]
    created_at: datetime
```

## 3. Zod / TypeScript（フロント）

```ts
import { z } from "zod";

export const TaskStatus = z.enum(["todo", "doing", "done"]);
export type TaskStatus = z.infer<typeof TaskStatus>;

export const TaskCreate = z.object({
  title: z.string().min(1).max(200),
  status: TaskStatus.default("todo"),
  tag_ids: z.array(z.string()).default([]),
});
export type TaskCreate = z.infer<typeof TaskCreate>;

export const TaskResponse = z.object({
  id: z.string(),
  title: z.string(),
  status: TaskStatus,
  tag_ids: z.array(z.string()),
  created_at: z.string().datetime(), // ISO 8601
});
export type TaskResponse = z.infer<typeof TaskResponse>;
```

## 4. 命名規約

- DTO 名は `{Entity}{Action}`（例: `TaskCreate`, `TaskUpdate`, `TaskResponse`）
- Enum 名は単数形 + 状態分類（例: `TaskStatus`）

## 5. バリデーションの一次責任

- **バックで最終チェック**。フロントはユーザー体験用の先行バリデーション
- Pydantic の制約（`min_length`, `Field(...)`）を正とし、フロントはそれを写す

## 6. 引継ぎメモ

- D3.3 openapi.yaml へ: Pydantic スキーマをそのまま OpenAPI にマッピング
- 実装フェーズ（fullstack-pipeline）: フロントの型は OpenAPI から自動生成する戦略も検討

## 7. 完了判定 (DoD)

- [ ] 主要エンティティの Pydantic スキーマが書かれている
- [ ] 同エンティティの Zod / TS 型が対応付けて書かれている
- [ ] Enum と日時の表現が揃っている
```
