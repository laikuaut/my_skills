---
name: playwright-test
description: |
  画面仕様書とフロントエンド実装からPlaywrightのE2Eテストケースを網羅的に生成し、テスト実行・バグ修正まで行うスキル。
  画面仕様書（Markdown）がある場合はそこからテストケースを導出し、なければ実装コードから直接テストを生成する。
  「E2Eテストを書いて」「Playwrightテスト作って」「画面テスト」「結合テスト」「ブラウザテスト」
  「画面仕様書からテストを作って」「テストケースを網羅して」「テストして直して」「テスト実行してバグ修正」
  「playwright」「e2e」「エンドツーエンド」などテスト作成・実行に関する指示があったときにこのスキルを使用すること。
  単体テスト（Vitest/Jest）の場合は使わない。ブラウザ上での画面操作テストに特化する。
---

# Playwright E2Eテスト生成・実行スキル

画面仕様書とフロントエンド実装を解析し、Playwrightテストを網羅的に生成→実行→バグ修正まで行う。

## 全体フロー

1. **環境確認** — Playwrightがセットアップされているか確認、なければ初期化
2. **テスト対象の把握** — 画面仕様書と実装コードから対象画面を特定
3. **テストケース設計** — ユースケースからテストケースを網羅的に導出
4. **テストコード生成** — Playwrightテストファイルを生成
5. **テスト実行** — テストを実行し結果を分析
6. **バグ修正** — 失敗したテストの原因を特定し、実装を修正
7. **再テスト** — 修正後に全テストを再実行して確認
8. **レポート** — 結果をユーザに報告

## Step 1: 環境確認

### Playwrightセットアップの確認

```bash
# package.jsonにplaywrightがあるか確認
cat package.json | grep -i playwright

# playwright.config.tsが存在するか
ls playwright.config.ts 2>/dev/null || ls playwright.config.js 2>/dev/null
```

### 未セットアップの場合

```bash
npm init playwright@latest -- --yes --quiet
```

生成される`playwright.config.ts`で以下を確認・調整する:
- `baseURL`: 開発サーバのURL（通常`http://localhost:3000`）
- `webServer`: 開発サーバの自動起動設定
- `projects`: テスト対象ブラウザ（初期は`chromium`のみで十分）

```typescript
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: "./e2e",
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: "html",
  use: {
    baseURL: "http://localhost:3000",
    trace: "on-first-retry",
    screenshot: "only-on-failure",
  },
  webServer: {
    command: "npm run dev",
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI,
  },
  projects: [
    { name: "chromium", use: { browserName: "chromium" } },
  ],
});
```

## Step 2: テスト対象の把握

### 画面仕様書がある場合

`docs/screens/`等にあるMarkdown仕様書を読み、テスト対象の画面を一覧化する。仕様書の以下のセクションがテストの主な入力になる：
- **入力要素テーブル** (I-N) → 入力操作のテスト
- **アクション** (A-N) → ユーザ操作のテスト
- **ユースケース** (UC-N) → テストシナリオの骨格
- **API通信** (API-N) → モック対象の特定
- **画面遷移** → ナビゲーションテスト

### 画面仕様書がない場合

実装コードを直接読み、以下を特定する：
- ページコンポーネント（ルーティング定義から）
- フォーム要素とバリデーション
- API呼び出し
- 画面遷移（router.push, Link等）
- 条件分岐による表示切替

いずれの場合も、対象画面リストをユーザに提示して確認を取る。

## Step 3: テストケース設計

各画面のユースケースからテストケースを導出する。以下のカテゴリで網羅する。

### テストカテゴリ

**表示テスト**: 画面の初期表示が正しいか
- 主要な要素が表示されるか（見出し、ボタン、入力欄）
- 初期データが正しく読み込まれるか
- ローディング状態が表示されるか

**操作テスト**: ユーザ操作が正しく動作するか
- フォーム入力と送信
- ボタンクリックの結果
- 選択操作（ドロップダウン、チェックボックス等）
- ページネーション、ソート、フィルタ

**バリデーションテスト**: 入力チェックが正しく動作するか
- 必須項目の空送信
- 形式不正（メール、電話番号等）
- 文字数制限
- 型不正（数値フィールドに文字等）

**遷移テスト**: 画面遷移が正しいか
- リンクの遷移先
- フォーム送信後のリダイレクト
- 認証状態による遷移制御
- ブラウザの戻る/進む

**エラーテスト**: エラー時の挙動
- APIエラー時の表示
- ネットワークエラー
- 404ページ

**境界テスト**: エッジケース
- データ0件時の表示
- 大量データ時の表示
- 長文入力
- 特殊文字

### テストケース一覧の作成

テストケースを以下の形式でまとめ、ユーザに提示する：

```markdown
## テストケース一覧: ログイン画面

| # | カテゴリ | テストケース | 元UC |
|---|---------|------------|------|
| T-1 | 表示 | ログインフォームが表示される | - |
| T-2 | 操作 | 正しい認証情報でログインできる | UC-1 |
| T-3 | バリデーション | メール未入力で送信するとエラー表示 | UC-3 |
| T-4 | エラー | 不正な認証情報でエラーメッセージ表示 | UC-2 |
| T-5 | 遷移 | ログイン成功後に商品一覧へ遷移 | UC-1 |
```

## Step 4: テストコード生成

### ディレクトリ構成

```
e2e/
├── fixtures/        # テスト用フィクスチャ
│   └── test-data.ts
├── helpers/         # 共通ヘルパー
│   ├── auth.ts      # ログインヘルパー
│   └── api-mock.ts  # APIモック共通処理
├── login.spec.ts
├── product-list.spec.ts
├── product-detail.spec.ts
└── user-management.spec.ts
```

### コーディング規約

テストコードは以下の方針で書く：

**テスト構造**: `describe` → `test` の2階層。テスト名は日本語で操作と期待結果を明記する。

```typescript
import { test, expect } from "@playwright/test";

test.describe("ログイン画面", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("/login");
  });

  test("メールアドレスとパスワードの入力フォームが表示される", async ({ page }) => {
    await expect(page.getByLabel("メールアドレス")).toBeVisible();
    await expect(page.getByLabel("パスワード")).toBeVisible();
    await expect(page.getByRole("button", { name: "ログイン" })).toBeVisible();
  });

  test("正しい認証情報でログインするとダッシュボードへ遷移する", async ({ page }) => {
    await page.getByLabel("メールアドレス").fill("user@example.com");
    await page.getByLabel("パスワード").fill("password123");
    await page.getByRole("button", { name: "ログイン" }).click();
    await expect(page).toHaveURL("/dashboard");
  });
});
```

**要素の取得**: アクセシビリティベースのロケータを優先する。
1. `getByRole` — ボタン、リンク、見出し等（最優先）
2. `getByLabel` — フォーム入力
3. `getByText` — テキスト表示の確認
4. `getByTestId` — 上記で特定できない場合の最終手段

`getByTestId`を使う場合は、実装側に`data-testid`属性の追加が必要になる。その場合は実装の修正もセットで行う。

**APIモック**: 外部APIに依存するテストは`page.route`でモックする。

```typescript
test("商品一覧が表示される", async ({ page }) => {
  await page.route("**/api/products*", async (route) => {
    await route.fulfill({
      status: 200,
      contentType: "application/json",
      body: JSON.stringify({
        products: [
          { id: "1", name: "テスト商品", price: 1000, category: "electronics", stock: 5 },
        ],
        total: 1,
        page: 1,
        totalPages: 1,
      }),
    });
  });
  await page.goto("/products");
  await expect(page.getByText("テスト商品")).toBeVisible();
  await expect(page.getByText("¥1,000")).toBeVisible();
});
```

**認証が必要なページ**: ログイン状態のセットアップをヘルパーに切り出す。

```typescript
// e2e/helpers/auth.ts
import { Page } from "@playwright/test";

export async function loginAs(page: Page, email = "test@example.com", password = "password123") {
  await page.route("**/api/auth/login", async (route) => {
    await route.fulfill({
      status: 200,
      contentType: "application/json",
      body: JSON.stringify({
        token: "mock-token",
        user: { id: "1", name: "テストユーザ", email, role: "admin" },
      }),
    });
  });
  await page.goto("/login");
  await page.getByLabel("メールアドレス").fill(email);
  await page.getByLabel("パスワード").fill(password);
  await page.getByRole("button", { name: "ログイン" }).click();
}
```

**待機**: 明示的な`waitForTimeout`は使わない。`expect`のauto-retryingか`waitForResponse`、`waitForURL`を使う。

### 生成時の注意

- テストは画面ごとに1ファイル。ファイル名は`<画面名>.spec.ts`
- 共通のモック定義やログインヘルパーは`helpers/`に切り出す
- テストデータは`fixtures/`にまとめる
- 1テスト = 1つの確認事項。複数の独立した確認を1テストに詰め込まない
- テスト間の依存を作らない。各テストは独立して実行可能にする

## Step 5: テスト実行

```bash
# 全テスト実行
npx playwright test

# 特定のファイルだけ実行
npx playwright test e2e/login.spec.ts

# UIモードで実行（デバッグ時）
npx playwright test --ui

# 失敗時のトレースを確認
npx playwright show-report
```

### 結果の分析

テスト結果を3つに分類する：

1. **テストコードのバグ** — ロケータの間違い、モックの不足、タイミングの問題
   → テストコードを修正する
2. **実装のバグ** — 仕様通りに動作していない、エラーハンドリングの不備
   → 実装コードを修正する（Step 6）
3. **仕様と実装の乖離** — 仕様書の記載と実装が異なる
   → ユーザに確認し、仕様かテストのどちらを修正するか判断を仰ぐ

## Step 6: バグ修正

テスト失敗の原因が実装のバグである場合、以下の手順で修正する。

1. **失敗したテストのエラーメッセージとスクリーンショットを確認**する
2. **原因箇所を特定**する（テストのトレース、実装コードの読解）
3. **最小限の修正**を行う。テストを通すために不必要な変更を加えない
4. **修正後に該当テストを単体実行**して通ることを確認する

修正の種類：
- **表示の修正**: テキスト、ラベル、aria属性の追加・修正
- **ロジックの修正**: バリデーション、状態遷移、API呼び出しの修正
- **アクセシビリティの改善**: `data-testid`の追加、`aria-label`の追加（テストで`getByRole`/`getByLabel`が使えるようにするため）

## Step 7: 再テスト

全修正が完了したら、全テストを再実行する。

```bash
npx playwright test
```

全テストがパスするまでStep 5-6を繰り返す。

## Step 8: レポート

最終結果をユーザに報告する：

```markdown
## テスト結果レポート

### サマリ
- テスト数: XX件
- 成功: XX件
- 修正後に成功: XX件
- スキップ: XX件

### 修正した実装バグ
| # | 画面 | バグ内容 | 修正ファイル |
|---|------|---------|------------|
| 1 | ログイン | バリデーションメッセージが表示されない | app/login/page.tsx |

### テストコードの修正
| # | 原因 | 対応 |
|---|------|------|
| 1 | ロケータがlabel属性と不一致 | getByLabelからgetByPlaceholderに変更 |

### 追加したdata-testid
| 要素 | testid | ファイル |
|------|--------|---------|
| 商品グリッド | product-grid | app/products/page.tsx |
```
