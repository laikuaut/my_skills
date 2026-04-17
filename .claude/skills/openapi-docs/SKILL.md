---
name: openapi-docs
description: |
  バックエンド実装コードからOpenAPI 3.1準拠のYAMLドキュメントを生成・更新するスキル。スタンドアロンで使えるし、fullstack-pipelineの一部としても呼ばれる。
  対応フレームワーク: FastAPI / Express / Hono / NestJS / Gin / Echo / Spring Boot。Pydantic・Zod・struct tag・DTOからスキーマ抽出、認証方式（Bearer/APIKey/OAuth2）も自動検出する。
  使用するケース: 「APIドキュメントを作って」「OpenAPIを生成」「Swagger定義を書いて」「openapi.yamlを更新」「エンドポイント一覧をまとめて」など、コードからAPI仕様を生成・更新する依頼全般。
  使わないケース: ドキュメントから実装を生成する逆方向の作業、AsyncAPI/GraphQL等のREST以外のAPI、手動で書かれた仕様書のレビューのみ。
  既存openapi.yamlがある場合は差分更新し、手書きされたdescription/exampleは保持する。
---

# OpenAPI Document Generator

バックエンド実装コードを読み取り、OpenAPI 3.1準拠のYAMLドキュメントを生成する。

## 全体の流れ

1. プロジェクト構造を調査し、フレームワークと言語を特定する
2. ルーティング定義・コントローラー・モデルを解析する
3. OpenAPI 3.1 YAMLを生成する
4. バリデーションして `docs/openapi.yaml` に出力する

## Step 1: プロジェクト調査

まず対象プロジェクトの構造を把握する。以下を確認すること:

- **使用言語**: Python / TypeScript / Go / Java / Kotlin など
- **フレームワーク**: FastAPI / Express / Hono / NestJS / Gin / Echo / Spring Boot など
- **ルーティング定義の場所**: ルーターファイル、コントローラークラス、デコレータなど
- **モデル/スキーマの場所**: Pydantic, Zod, struct, DTO クラスなど
- **認証方式**: Bearer token, API key, OAuth2, Cookie など
- **既存のOpenAPIファイル**: すでに `openapi.yaml` や `swagger.json` があるか

フレームワークの特定には以下のファイルを確認する:
- `package.json` → Express / Hono / NestJS
- `pyproject.toml` / `requirements.txt` → FastAPI / Django REST
- `go.mod` → Gin / Echo
- `pom.xml` / `build.gradle` → Spring Boot

## Step 2: エンドポイント抽出

フレームワークごとのルーティングパターンからエンドポイントを抽出する。

### 抽出すべき情報

各エンドポイントについて以下を収集する:

| 項目 | 説明 |
|------|------|
| パス | `/api/v1/users/{id}` のようなURLパス |
| HTTPメソッド | GET, POST, PUT, PATCH, DELETE |
| 概要 (summary) | エンドポイントの短い説明 |
| タグ | リソースやドメインによるグループ分け |
| パスパラメータ | `{id}` のようなURL内パラメータ |
| クエリパラメータ | `?page=1&limit=10` のようなクエリ文字列 |
| リクエストボディ | POST/PUT/PATCHのリクエスト本文スキーマ |
| レスポンス | ステータスコード別のレスポンススキーマ |
| 認証要否 | 認証が必要かどうか |

### フレームワーク別の読み取りポイント

**FastAPI (Python)**
- `@app.get()`, `@router.post()` などのデコレータからパスとメソッドを取得
- 関数の引数の型ヒントからパラメータとリクエストボディを推定
- `response_model` からレスポンススキーマを特定
- Pydanticモデルの `Field()` から description や example を取得
- `Depends()` から認証ミドルウェアを検出

**Express / Hono (TypeScript/JavaScript)**
- `router.get()`, `app.post()` からパスとメソッドを取得
- ミドルウェアチェーンから認証要否を判定
- Zodスキーマやバリデーションライブラリからリクエスト/レスポンスの型を推定
- JSDocコメントがあれば説明文として活用

**Gin / Echo (Go)**
- `r.GET()`, `e.POST()` からパスとメソッドを取得
- ハンドラ関数内の `c.Bind()`, `c.JSON()` から入出力の型を推定
- structタグ（`json:`, `binding:`）からフィールド情報を取得
- コメントから説明文を抽出

**Spring Boot (Java/Kotlin)**
- `@GetMapping`, `@PostMapping` などのアノテーションからパスとメソッドを取得
- `@RequestBody`, `@PathVariable`, `@RequestParam` から入力パラメータを特定
- DTOクラスからスキーマを構築
- `@PreAuthorize`, `@Secured` から認証要件を検出

## Step 3: OpenAPI YAML生成

### ドキュメント構造

以下の構造でOpenAPI 3.1準拠のYAMLを生成する:

```yaml
openapi: "3.1.0"
info:
  title: API Title
  description: APIの概要説明
  version: "1.0.0"

servers:
  - url: http://localhost:8000
    description: Local development

paths:
  /api/v1/resource:
    get:
      summary: リソース一覧取得
      tags:
        - Resource
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
      responses:
        "200":
          description: 成功
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ResourceList"

components:
  schemas:
    Resource:
      type: object
      properties:
        id:
          type: integer
          description: リソースID
        name:
          type: string
          description: リソース名
      required:
        - id
        - name

  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

### 生成ルール

- **info.title**: プロジェクト名やパッケージ名から推定する。不明な場合はディレクトリ名を使う
- **info.version**: `package.json` や `pyproject.toml` のバージョン情報があれば使う
- **servers**: 設定ファイルからポート番号を読み取り、ローカルサーバーURLを設定する
- **tags**: エンドポイントのパスプレフィックスやコントローラー名からグループ化する
- **operationId**: `メソッド名` または `動詞_リソース名` の形式で自動生成する（例: `listUsers`, `createUser`）
- **$ref によるスキーマ再利用**: 同じモデルが複数のエンドポイントで使われる場合は `components/schemas` に定義して `$ref` で参照する。インラインスキーマの重複を避ける
- **required フィールド**: モデル定義でデフォルト値がないフィールド、またはバリデーションで必須とされているフィールドを `required` に含める
- **description**: コード内のコメント、docstring、JSDocから可能な限り抽出する。ない場合は関数名やパスから簡潔な説明を生成する
- **example**: モデルにexampleが定義されていれば含める。なくても型から妥当なexampleを生成してもよい

### エラーレスポンス

一般的なエラーレスポンスも含める:

```yaml
responses:
  "400":
    description: リクエスト不正
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/ErrorResponse"
  "401":
    description: 認証エラー
  "404":
    description: リソースが見つからない
  "500":
    description: サーバーエラー
```

エラーレスポンスのスキーマは、コード内にエラーハンドリングの実装があればそこから抽出する。なければ一般的な `ErrorResponse` スキーマを定義する。

## Step 4: バリデーションと出力

### 出力前の確認事項

生成したYAMLが以下を満たすことを確認する:

- OpenAPI 3.1の仕様に準拠している
- `$ref` の参照先がすべて存在する
- 全エンドポイントに少なくとも1つのレスポンスが定義されている
- required に列挙されたプロパティがすべて properties に存在する
- パスパラメータ `{param}` に対応する parameters 定義がある
- YAMLとして正しい構文である（インデントやクォーティング）

### 出力

ファイルを `docs/openapi.yaml` に書き出す。`docs/` ディレクトリがなければ作成する。

既存の `openapi.yaml` がある場合は、差分を確認してユーザーに報告してから上書きする。

### 出力後のサマリー

生成完了後、以下の情報をユーザーに報告する:

- 検出したエンドポイント数
- タグ（グループ）一覧
- 定義したスキーマ一覧
- 認証方式
- 注意点や手動確認が必要な箇所（型が推定できなかったエンドポイントなど）

## 既存ファイルの更新

既存の `openapi.yaml` がある場合:

1. 既存ファイルを読み込む
2. 実装コードから現在のエンドポイントを抽出する
3. 差分を特定する（追加・変更・削除されたエンドポイント）
4. 差分をユーザーに提示し、確認を得てから更新する
5. 既存の description や example など、手動で追記された情報は保持する
