---
name: fullstack-pipeline
description: |
  FastAPI（バックエンド）+ React（フロントエンド）のフルスタック開発パイプラインスキル。
  「この機能を作って」と言われたら、設計→バックエンド実装→APIドキュメント→フロントエンド実装→画面仕様書→E2Eテスト→コミットの順で開発を進める。
  既存のpython-impl, react-impl, openapi-docs, screen-spec-reverse, playwright-test, rest-api-test等のスキルと連携する。
  「機能を実装して」「バックエンドとフロントエンドを作って」「APIと画面を作って」「フルスタックで開発して」
  「FastAPIとReactで作って」「バックエンドからフロントまで一気に」「この機能を一通り作って」
  などフルスタック開発の指示があったときにこのスキルを使用すること。
  バックエンドのみ、フロントエンドのみの場合は個別のスキルを使う。両方にまたがる開発に使う。
---

# フルスタック開発パイプライン

FastAPI + Reactのフルスタック開発を、設計からテストまで一気通貫で進めるためのガイド。
各フェーズで専門スキルを活用し、品質基準を統一する。

## 開発フロー概要

```
1. 要件整理     → ユーザと対話して要件を明確化
2. API設計      → エンドポイント・データモデルの設計
3. バックエンド  → FastAPIで実装（python-implスキル適用）
4. APIドキュメント → OpenAPI仕様書を生成（openapi-docsスキル）
5. APIテスト     → curlベースの結合試験（rest-api-testスキル）
6. フロントエンド → Reactで実装（react-implスキル適用）
7. 画面仕様書    → 実装から仕様書を生成（screen-spec-reverseスキル）
8. E2Eテスト    → Playwrightテスト生成・実行（playwright-testスキル）
9. コミット      → 変更をコミット・プッシュ（git-commit-pushスキル）
```

ユーザの指示に応じてフェーズを飛ばしたり、特定のフェーズから開始できる。
たとえばバックエンドが既にある場合はフェーズ6から、APIだけ必要な場合はフェーズ2-5だけ実行する。

## Phase 1: 要件整理

ユーザの指示から以下を明確にする。曖昧な場合はユーザに質問する。

### 確認事項

| 項目 | 質問例 |
|------|--------|
| 機能の目的 | 何を実現したいか？誰が使うか？ |
| データモデル | どんなデータを扱うか？主要なエンティティは？ |
| 画面構成 | いくつの画面が必要か？どんな操作をするか？ |
| 認証 | ログインは必要か？権限管理は？ |
| 既存コード | 既にあるコードはあるか？新規プロジェクトか？ |

### プロジェクト構成の確認

既存プロジェクトの場合はディレクトリ構成を確認する。新規の場合は以下の構成を提案する。

```
project/
├── backend/          ← FastAPIプロジェクト
│   ├── src/
│   │   └── app/
│   │       ├── main.py
│   │       ├── routers/
│   │       ├── models/
│   │       ├── schemas/
│   │       └── services/
│   ├── tests/
│   ├── pyproject.toml
│   └── Dockerfile
├── frontend/         ← Reactプロジェクト
│   ├── src/
│   │   ├── features/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── lib/
│   │   └── types/
│   ├── e2e/
│   ├── package.json
│   └── Dockerfile
├── docs/
│   ├── api/          ← OpenAPI仕様書
│   └── screens/      ← 画面仕様書
└── docker-compose.yaml
```

新規プロジェクトの場合は `python-project-init` と `react-project-init` スキルでそれぞれ初期化する。

## Phase 2: API設計

実装に入る前に、APIのエンドポイントとデータモデルを設計する。この段階で決めることで、バックエンドとフロントエンドの開発を明確に分離できる。

### 設計の進め方

1. **エンティティを列挙**する（User, Product, Order等）
2. **各エンティティのCRUD**を洗い出す
3. **エンドポイント一覧**をテーブルで整理する
4. **リクエスト/レスポンスの型**を定義する
5. ユーザに設計を提示し、合意を得る

### エンドポイント設計テンプレート

```markdown
| メソッド | パス | 説明 | 認証 |
|---------|------|------|------|
| POST | /api/auth/login | ログイン | 不要 |
| GET | /api/users | ユーザ一覧 | 要（admin） |
| POST | /api/users | ユーザ作成 | 要（admin） |
| GET | /api/users/{id} | ユーザ詳細 | 要 |
| PUT | /api/users/{id} | ユーザ更新 | 要 |
| DELETE | /api/users/{id} | ユーザ削除 | 要（admin） |
```

### データモデル設計

```python
# Pydanticスキーマとして型を定義する
class UserCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=50)
    email: EmailStr
    role: Literal["admin", "editor", "viewer"]
    password: str = Field(..., min_length=8)

class UserResponse(BaseModel):
    id: str
    name: str
    email: str
    role: str
    created_at: datetime
```

## Phase 3: バックエンド実装

FastAPIで実装する。**python-implスキル**の品質基準に従う。

### 実装順序

バックエンドは以下の順序で実装する。各ステップが前のステップに依存する。

1. **データモデル** (`models/`) — SQLAlchemy / Pydantic モデル
2. **スキーマ** (`schemas/`) — リクエスト/レスポンスの型定義
3. **サービス層** (`services/`) — ビジネスロジック
4. **ルーター** (`routers/`) — エンドポイント定義
5. **メインアプリ** (`main.py`) — ルーターの登録、ミドルウェア設定

### 実装時の注意

- エラーハンドリングは`HTTPException`で統一する
- バリデーションはPydanticの`Field`で定義する（ルーター層でバリデーションしない）
- 認証は依存注入（`Depends`）で実装する
- CORSの設定を忘れない（フロントエンドとの通信に必要）

```python
# CORS設定例
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## Phase 4: APIドキュメント

バックエンド実装が完了したら、**openapi-docsスキル**でOpenAPI仕様書を生成する。

FastAPIは自動でOpenAPIスキーマを提供するが、スキルを使って以下を補完する：
- エンドポイントの説明文
- リクエスト/レスポンスの例
- エラーレスポンスの定義

生成した仕様書は`docs/api/`に配置する。

## Phase 5: APIテスト

**rest-api-testスキル**でcurlベースの結合試験を生成・実行する。

このフェーズでバックエンドのバグを早期に発見する。フロントエンド開発に入る前にAPIが正しく動作することを確認しておくことで、後工程の手戻りを防ぐ。

## Phase 6: フロントエンド実装

Reactで実装する。**react-implスキル**の品質基準に従う。

### 実装順序

1. **型定義** (`types/`) — Phase 2で設計したデータモデルのTypeScript版
2. **APIクライアント** (`lib/api-client.ts`) — fetchラッパー
3. **カスタムフック** (`hooks/` または `features/*/hooks/`) — データ取得・操作ロジック
4. **UIコンポーネント** (`components/` または `features/*/components/`) — 表示
5. **ページコンポーネント** — フック + コンポーネントの組み立て
6. **ルーティング** — ページの登録

### バックエンドとの型共有

Phase 2で設計したデータモデルを、TypeScript型として正確に転記する。型名はPythonスキーマに合わせると混乱が少ない。

```typescript
// backend: UserResponse → frontend: UserResponse
type UserResponse = {
  id: string;
  name: string;
  email: string;
  role: "admin" | "editor" | "viewer";
  created_at: string;  // ISO 8601
};
```

Pydanticの`Literal`型はTypeScriptのユニオン型に、`Optional`は`| undefined`に対応する。

## Phase 7: 画面仕様書

フロントエンド実装が完了したら、**screen-spec-reverseスキル**で画面仕様書を生成する。

実装からリバースで仕様書を作ることで、実装と仕様の乖離がない状態を保つ。生成した仕様書は`docs/screens/`に配置する。

## Phase 8: E2Eテスト

**playwright-testスキル**でE2Eテストを生成・実行する。

Phase 7で生成した画面仕様書をインプットとして使う。仕様書のユースケース（UC-N）がテストケースの骨格になる。

テスト実行でバグが見つかった場合は、このフェーズでバックエンド/フロントエンドの修正も行う。

## Phase 9: コミット

全フェーズが完了したら、**git-commit-pushスキル**で変更をコミット・プッシュする。

変更が大きい場合はフェーズごとにコミットを分割する（バックエンド、フロントエンド、テスト、ドキュメントの4つに分けるなど）。

## フェーズの省略・カスタマイズ

ユーザの状況に応じてフェーズを柔軟に調整する。

| 状況 | 省略/調整 |
|------|----------|
| バックエンドが既にある | Phase 3-5を飛ばし、既存APIを確認してPhase 6から |
| フロントエンドが既にある | Phase 6を飛ばし、Phase 7-8で仕様書とテスト |
| プロトタイプ段階 | Phase 4,5,7を飛ばし、最小限で動くものを作る |
| API設計だけ相談したい | Phase 1-2のみ実行 |
| テストだけ追加したい | Phase 7-8のみ実行 |

ユーザが「まずバックエンドだけ」「画面だけ作って」と言った場合は、該当フェーズのみ実行する。ただし、前後のフェーズで必要な情報が不足している場合は補足を求める。

## スキル連携の呼び出し方

このスキルは他スキルの「使い方ガイド」として機能する。各フェーズで実際のコード生成は専門スキルの基準に委ねる。

明示的に「python-implスキルに従って」等と書く必要はない。各スキルは対応する作業（Pythonコード作成、Reactコンポーネント作成等）で自動的にトリガーされる。

ただし以下のスキルはユーザの明示的な指示で呼ぶ方が確実：
- **openapi-docs** — 「APIドキュメントを生成して」
- **screen-spec-reverse** — 「画面仕様書を作って」
- **playwright-test** — 「E2Eテストを作って」
- **rest-api-test** — 「API結合テストを作って」
- **git-commit-push** — 「コミットして」

各フェーズの完了時に、次フェーズに進むかユーザに確認を取る。一気に全フェーズを自動実行しない。
