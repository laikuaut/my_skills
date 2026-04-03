---
name: rest-api-test
description: |
  REST APIの結合試験（インテグレーションテスト）環境を構築し、curlベースのテストケースをシェルスクリプトとして生成するスキル。
  OpenAPI/Swagger定義を読み込んで、正常系・異常系を網羅的にカバーするテストケースを自動生成する。
  各テストケースは独立した.shファイルとして作成され、入力（リクエスト）と出力（レスポンス）を保存する。
  このスキルは、ユーザが「APIテスト」「結合試験」「curl テスト」「エンドポイントのテスト」「APIの動作確認」
  「テストケース生成」「REST テスト」などに言及したときに使用する。
  既にサーバが起動している前提で、curlによるブラックボックステストを行う場面で積極的に使うこと。
---

# REST API 結合試験スキル

REST APIの結合試験環境を構築し、curlベースのテストスクリプトを生成する。
OpenAPI定義からエンドポイントを読み取り、正常系・異常系を網羅的にテストする。

## 前提条件

- テスト対象のAPIサーバは事前に起動済みであること
- curlがインストールされていること
- bash環境で実行すること（Windows の場合は Git Bash / WSL を使用）
- jqがインストールされていること（レスポンスのJSON比較に使用）

## ワークフロー

### Step 1: API仕様の取得

ユーザにOpenAPI/Swagger定義ファイルのパスを聞く。ファイルを読み込み、以下を抽出する：

- 各エンドポイントのパス、HTTPメソッド、パラメータ
- リクエストボディのスキーマ
- レスポンスのスキーマとステータスコード定義
- 認証方式（Bearer, API Key など）

OpenAPI定義がない場合は、ユーザにエンドポイント一覧を聞いて手動で情報を収集する。

### Step 2: フォルダ構成の生成

以下のフォルダ構成を作成する：

```
api-tests/
├── README.md               # テスト一覧・実行方法・前提条件の説明
├── run_all.sh              # 全テスト一括実行スクリプト
├── common.sh               # 共通設定（BASE_URL, 認証トークンなど）
├── .gitignore              # results/ を除外
├── results/                # テスト実行結果の出力先
│   └── .gitkeep
├── 01_users/               # リソース単位でグループ化
│   ├── 01_create_user_success.sh
│   ├── 01_create_user_success.input.json
│   ├── 01_create_user_success.expected.json
│   ├── 02_create_user_missing_field.sh
│   ├── 02_create_user_missing_field.input.json
│   ├── 02_create_user_missing_field.expected.json
│   ├── 03_get_user_success.sh
│   ├── 03_get_user_success.expected.json
│   └── ...
├── 02_products/
│   └── ...
└── ...
```

**フォルダ命名規則**:
- リソース名で番号付きディレクトリを作る（`01_users/`, `02_products/`）
- テストケースのファイル名: `{番号}_{メソッド}_{リソース}_{シナリオ}.sh`
- 入力データ: 同名で `.input.json` 拡張子
- 期待値データ: 同名で `.expected.json` 拡張子

### Step 3: common.sh の作成

共通設定ファイルを生成する。テスト環境に合わせてユーザが編集できるようにする。

```bash
#!/bin/bash
# === 共通設定 ===
BASE_URL="http://localhost:8080"
AUTH_TOKEN=""
CONTENT_TYPE="application/json"
RESULTS_DIR="$(cd "$(dirname "$0")" && pwd)/results"

# 色付き出力
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# テスト結果カウンタ
PASS_COUNT=0
FAIL_COUNT=0

# === 共通関数 ===

# レスポンスのステータスコードを検証
assert_status() {
  local actual="$1"
  local expected="$2"
  local test_name="$3"
  if [ "$actual" -eq "$expected" ]; then
    echo -e "${GREEN}[PASS]${NC} ${test_name}: status=${actual}"
    PASS_COUNT=$((PASS_COUNT + 1))
    return 0
  else
    echo -e "${RED}[FAIL]${NC} ${test_name}: expected=${expected}, actual=${actual}"
    FAIL_COUNT=$((FAIL_COUNT + 1))
    return 1
  fi
}

# レスポンスボディのJSONキー・値を検証
assert_json_value() {
  local response_body="$1"
  local jq_filter="$2"
  local expected_value="$3"
  local test_name="$4"
  local actual_value
  actual_value=$(echo "$response_body" | jq -r "$jq_filter")
  if [ "$actual_value" = "$expected_value" ]; then
    echo -e "${GREEN}[PASS]${NC} ${test_name}: ${jq_filter}=${actual_value}"
    PASS_COUNT=$((PASS_COUNT + 1))
    return 0
  else
    echo -e "${RED}[FAIL]${NC} ${test_name}: ${jq_filter} expected=${expected_value}, actual=${actual_value}"
    FAIL_COUNT=$((FAIL_COUNT + 1))
    return 1
  fi
}

# レスポンスボディに特定キーが存在するか検証
assert_json_has_key() {
  local response_body="$1"
  local jq_filter="$2"
  local test_name="$3"
  local value
  value=$(echo "$response_body" | jq -e "$jq_filter" 2>/dev/null)
  if [ $? -eq 0 ]; then
    echo -e "${GREEN}[PASS]${NC} ${test_name}: key exists (${jq_filter})"
    PASS_COUNT=$((PASS_COUNT + 1))
    return 0
  else
    echo -e "${RED}[FAIL]${NC} ${test_name}: key not found (${jq_filter})"
    FAIL_COUNT=$((FAIL_COUNT + 1))
    return 1
  fi
}

# レスポンス全体を期待値ファイルと比較（キー単位の部分一致）
assert_json_match() {
  local response_body="$1"
  local expected_file="$2"
  local test_name="$3"
  if [ ! -f "$expected_file" ]; then
    echo -e "${YELLOW}[SKIP]${NC} ${test_name}: expected file not found (${expected_file})"
    return 1
  fi
  # 期待値JSONの各キーについて、レスポンスの値と一致するか検証
  local keys
  keys=$(jq -r 'keys[]' "$expected_file")
  local all_pass=true
  for key in $keys; do
    local expected_val actual_val
    expected_val=$(jq -r ".${key}" "$expected_file")
    actual_val=$(echo "$response_body" | jq -r ".${key}")
    if [ "$expected_val" != "$actual_val" ]; then
      echo -e "${RED}[FAIL]${NC} ${test_name}: .${key} expected=${expected_val}, actual=${actual_val}"
      FAIL_COUNT=$((FAIL_COUNT + 1))
      all_pass=false
    fi
  done
  if $all_pass; then
    echo -e "${GREEN}[PASS]${NC} ${test_name}: response matches expected"
    PASS_COUNT=$((PASS_COUNT + 1))
  fi
}

# テスト結果サマリを出力
print_summary() {
  echo ""
  echo "================================"
  echo "  Test Summary"
  echo "================================"
  echo -e "  ${GREEN}PASS${NC}: ${PASS_COUNT}"
  echo -e "  ${RED}FAIL${NC}: ${FAIL_COUNT}"
  echo "  TOTAL: $((PASS_COUNT + FAIL_COUNT))"
  echo "================================"
  if [ "$FAIL_COUNT" -gt 0 ]; then
    return 1
  fi
  return 0
}
```

### Step 4: 個別テストケースの作成

各エンドポイントに対して、以下の観点でテストケースを作成する。

#### 正常系テストケース

| 観点 | 説明 |
|------|------|
| 基本CRUD | GET/POST/PUT/PATCH/DELETE の正常動作 |
| パラメータ指定 | クエリパラメータ、パスパラメータの正常値 |
| ページネーション | limit, offset, page 等のパラメータ |
| フィルタリング | 検索条件やソート指定 |
| 認証あり | 正しいトークンでのアクセス |
| デフォルト値確認 | 任意パラメータ省略時にデフォルト値が適用されるか |
| 境界値（正常側） | 文字数上限ちょうど、数値の最小/最大有効値 |

#### 異常系テストケース

| 観点 | 説明 |
|------|------|
| 必須パラメータ欠落 | required フィールドを1つずつ省略（フィールドごとに個別テスト） |
| 不正な型 | 数値に文字列、文字列に数値など型の不一致 |
| 存在しないリソース | 存在しないIDへのアクセス（404） |
| 認証エラー | トークンなし・不正トークン（401） |
| 権限エラー | 権限不足（403） |
| バリデーションエラー | 文字数超過、不正フォーマット、範囲外の値（400/422） |
| メソッド不許可 | 許可されていないHTTPメソッド（405） |
| 重複登録 | 一意制約違反（409） |
| 空ボディ | リクエストボディが空のJSON `{}` |
| 不正なIDフォーマット | パスパラメータに文字列や負値を指定 |

#### 境界値・特殊値テストケース

境界値テストはバグの温床になりやすいため、以下の観点を意識してテストケースを作成する：

| 観点 | 例 |
|------|------|
| 文字列長の境界 | maxLength ちょうど（正常）、maxLength+1（異常） |
| 数値の境界 | minimum ちょうど（正常）、minimum-1（異常）、0、負値 |
| 空文字列 | `""` を必須フィールドに送信 |
| 特殊文字 | 日本語、絵文字、HTMLタグ `<script>`、SQLインジェクション `' OR 1=1` |
| 極端な値 | 非常に長い文字列（10000文字）、非常に大きい数値 |
| null値 | `"field": null` を送信 |

#### CRUD後の状態確認テストケース

リソースの操作後に、状態が正しく変化しているか確認するテストも追加する：

| 観点 | 例 |
|------|------|
| 作成後の取得 | POSTで作成 → GETで取得して内容一致を確認 |
| 更新後の取得 | PUTで更新 → GETで取得して更新値を確認 |
| 削除後の取得 | DELETEで削除 → GETで404を確認 |

#### テストスクリプトのテンプレート

各.shファイルは以下のパターンで生成する：

```bash
#!/bin/bash
# テスト: [テスト名]
# エンドポイント: [METHOD] [PATH]
# 期待ステータス: [STATUS_CODE]
# 観点: [正常系|異常系] - [具体的な観点]

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "${SCRIPT_DIR}/../common.sh"

TEST_NAME="[テスト名]"
ENDPOINT="${BASE_URL}/api/v1/users"
METHOD="POST"
EXPECTED_STATUS=201

# リクエスト実行
RESPONSE=$(curl -s -w "\n%{http_code}" \
  -X ${METHOD} \
  -H "Content-Type: ${CONTENT_TYPE}" \
  -H "Authorization: Bearer ${AUTH_TOKEN}" \
  -d @"${SCRIPT_DIR}/[ファイル名].input.json" \
  "${ENDPOINT}")

# レスポンス分離
HTTP_STATUS=$(echo "$RESPONSE" | tail -1)
RESPONSE_BODY=$(echo "$RESPONSE" | sed '$d')

# 結果をファイルに保存
RESULT_DIR="${RESULTS_DIR}/[グループ名]"
mkdir -p "${RESULT_DIR}"
echo "$RESPONSE_BODY" | jq '.' > "${RESULT_DIR}/${TEST_NAME}.output.json" 2>/dev/null || \
  echo "$RESPONSE_BODY" > "${RESULT_DIR}/${TEST_NAME}.output.txt"

# アサーション
assert_status "$HTTP_STATUS" "$EXPECTED_STATUS" "$TEST_NAME"

# レスポンスボディの検証（全テストケースで必須）
EXPECTED_FILE="${SCRIPT_DIR}/[ファイル名].expected.json"
assert_json_match "$RESPONSE_BODY" "$EXPECTED_FILE" "${TEST_NAME} body"

print_summary
```

**重要なポイント**:
- `curl -s -w "\n%{http_code}"` でレスポンスボディとステータスコードを両方取得する
- レスポンスは `results/` 配下に `.output.json` として保存する（実際のレスポンス記録）
- GETリクエストなど入力データがない場合は `.input.json` は作成しない
- `.expected.json` にはレスポンスで検証したいキーと値だけを書く（部分一致）

### .expected.json の作成ルール

`.expected.json` は全テストケースに必ず作成する。テスト結果の信頼性を担保するために、ステータスコードだけでなくレスポンスボディの内容も検証することが重要である。

#### 正常系の .expected.json 例

動的に変わる値（id, created_at など）は省略し、固定値のみ記述する：

```json
{
  "status": "pending",
  "priority": "medium"
}
```

#### 異常系の .expected.json 例

エラーレスポンスでも、エラーの種類が判別できるキーを検証する：

```json
{
  "error": "validation_error",
  "message": "title is required"
}
```

エラーレスポンスの形式がAPI仕様に明記されていない場合は、一般的なパターンで記述する：

- 400 Bad Request: `{"error": "bad_request"}` や `{"error": "validation_error"}`
- 401 Unauthorized: `{"error": "unauthorized"}`
- 404 Not Found: `{"error": "not_found"}`
- 409 Conflict: `{"error": "conflict"}`

GETリクエストの一覧取得では、レスポンス構造の検証を行う：

```json
{
  "items": [],
  "total": 0
}
```

この場合 `items` の中身は空配列でよい（存在確認のみ）。実際のテストスクリプトでは `assert_json_has_key` で配列キーの存在を確認し、`assert_json_match` で `.expected.json` の固定値部分を検証する。

### Step 5: run_all.sh の生成

全テストケースを一括実行するスクリプトを生成する。
リソース名やキーワードでフィルタ実行できる機能を持たせ、部分的なテスト実行も可能にする。

```bash
#!/bin/bash
# 全テストケース一括実行スクリプト
# 使い方:
#   bash run_all.sh              # 全テスト実行
#   bash run_all.sh users        # "users" を含むテストのみ実行
#   bash run_all.sh --dir 01     # 01_xxx ディレクトリのテストのみ実行

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "${SCRIPT_DIR}/common.sh"

FILTER=""
DIR_FILTER=""

# 引数の解析
while [[ $# -gt 0 ]]; do
  case "$1" in
    --dir)
      DIR_FILTER="$2"
      shift 2
      ;;
    *)
      FILTER="$1"
      shift
      ;;
  esac
done

echo "================================"
echo "  REST API Integration Tests"
echo "  $(date '+%Y-%m-%d %H:%M:%S')"
if [ -n "$FILTER" ]; then
  echo "  Filter: $FILTER"
fi
if [ -n "$DIR_FILTER" ]; then
  echo "  Directory: $DIR_FILTER"
fi
echo "================================"
echo ""

# 結果ディレクトリの初期化
rm -rf "${RESULTS_DIR:?}"/*

# 全テストスクリプトを番号順に実行
TOTAL_PASS=0
TOTAL_FAIL=0
TOTAL_SKIP=0
TEST_FILES=$(find "${SCRIPT_DIR}" -name "*.sh" \
  ! -name "run_all.sh" \
  ! -name "common.sh" \
  | sort)

for test_file in $TEST_FILES; do
  # フィルタ条件に一致しなければスキップ
  if [ -n "$FILTER" ] && [[ "$(basename "$test_file")" != *"$FILTER"* ]]; then
    TOTAL_SKIP=$((TOTAL_SKIP + 1))
    continue
  fi
  if [ -n "$DIR_FILTER" ] && [[ "$(dirname "$test_file")" != *"$DIR_FILTER"* ]]; then
    TOTAL_SKIP=$((TOTAL_SKIP + 1))
    continue
  fi

  echo "--- $(basename "$test_file") ---"
  output=$(bash "$test_file" 2>&1)
  echo "$output"
  pass=$(echo "$output" | grep -c '\[PASS\]')
  fail=$(echo "$output" | grep -c '\[FAIL\]')
  TOTAL_PASS=$((TOTAL_PASS + pass))
  TOTAL_FAIL=$((TOTAL_FAIL + fail))
  echo ""
done

# 全体サマリ
echo "========================================"
echo "  Overall Summary"
echo "========================================"
echo -e "  ${GREEN}PASS${NC}: ${TOTAL_PASS}"
echo -e "  ${RED}FAIL${NC}: ${TOTAL_FAIL}"
if [ "$TOTAL_SKIP" -gt 0 ]; then
  echo -e "  ${YELLOW}SKIP${NC}: ${TOTAL_SKIP}"
fi
echo "  TOTAL: $((TOTAL_PASS + TOTAL_FAIL))"
echo "========================================"

if [ "$TOTAL_FAIL" -gt 0 ]; then
  exit 1
fi
exit 0
```

### Step 6: テスト依存関係の考慮

APIテストでは実行順序が重要な場合がある（例: ユーザ作成 → そのユーザの取得）。
依存関係がある場合は以下のように対処する：

- ファイル名の番号で実行順序を制御する
- 前のテストで取得したIDなどを一時ファイル経由で受け渡す
- `results/` 配下に `.id` ファイルなどで中間データを保存する

例：
```bash
# 01_create_user_success.sh の末尾で
echo "$RESPONSE_BODY" | jq -r '.id' > "${RESULT_DIR}/created_user_id.txt"

# 03_get_user_success.sh の冒頭で
USER_ID=$(cat "${RESULTS_DIR}/01_users/created_user_id.txt")
ENDPOINT="${BASE_URL}/api/v1/users/${USER_ID}"
```

### Step 7: README.md と .gitignore の生成

テスト環境のセットアップを簡単にするため、README.md と .gitignore を自動生成する。

#### README.md

以下の情報を含むREADME.mdを生成する：

```markdown
# API結合試験

## 前提条件
- bash, curl, jq がインストールされていること
- テスト対象サーバが起動していること

## セットアップ
1. `common.sh` の `BASE_URL` と認証情報を環境に合わせて編集
2. テストスクリプトに実行権限を付与: `chmod +x *.sh **/*.sh`

## 実行方法
```
bash run_all.sh              # 全テスト実行
bash run_all.sh users        # "users" を含むテストのみ
bash run_all.sh --dir 01     # 特定ディレクトリのテストのみ
bash 01_users/01_create_user_success.sh  # 個別実行
```

## テストケース一覧
| # | ファイル | 観点 | メソッド | 期待ステータス |
|---|---------|------|---------|--------------|
| ... | ... | ... | ... | ... |
```

テストケース一覧テーブルには、生成した全テストケースを番号順にリストアップする。

#### .gitignore

```
results/*
!results/.gitkeep
```

## テストケース生成時の注意

- OpenAPIの `required` フィールドを参照して、必須パラメータの欠落テストを漏れなく作る
- `enum` 定義がある場合は、有効値と無効値の両方をテストする
- レスポンスの `example` があればそれを `.expected.json` の参考にする
- 認証が必要なエンドポイントには、認証なし・不正トークンのテストを必ず追加する
- テストデータ内にタイムスタンプやランダムIDが含まれる場合は、`.expected.json` で該当キーを省略して部分一致にする
- 各テストスクリプトには冒頭コメントでテストの観点を明記する
