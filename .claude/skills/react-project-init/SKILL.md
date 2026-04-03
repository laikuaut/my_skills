---
name: react-project-init
description: |
  Reactプロジェクトの初期構築スキル。Vite・TypeScript・Tailwind CSS・ESLint・Prettier・Vitest・Dockerを含むプロジェクト雛形を生成する。
  SPA・ダッシュボード・ランディングページなど用途に応じた構成に対応。
  「Reactプロジェクトを作って」「Viteで新規プロジェクト」「フロントエンドの雛形」「SPAを作りたい」
  「Reactアプリ作成」「ダッシュボード作りたい」「TypeScript+Reactで始めたい」「Tailwind使って画面作りたい」
  などReact/フロントエンドプロジェクトの新規作成を求められたときに使用する。既存コードの編集には使わない。
---

# React Project Init

Vite + React + TypeScriptベースのプロジェクト雛形を生成するスキル。ユーザの指定に応じてプロジェクトタイプを選択し、必要なファイルをすべて生成する。

## フロー

1. ユーザにプロジェクト名とプロジェクトタイプを確認する
2. `npm create vite@latest`でプロジェクトを作成する（テンプレート: `react-ts`）
3. 追加パッケージをインストールし、設定ファイルを生成する
4. タイプ別リファレンスに従ってディレクトリ構成とサンプルコードを生成する
5. 構成検証を実行する

## プロジェクトタイプ

| タイプ | 用途 | リファレンス |
|---|---|---|
| spa | シンプルなSPA、小〜中規模アプリ | `references/spa.md` |
| dashboard | 管理画面・ダッシュボード | `references/dashboard.md` |

ユーザが明示しない場合は用途をヒアリングして判断する。迷ったら`spa`を提案する。

## 共通仕様

すべてのタイプで以下のセットアップを行う。

### Step 1: Viteプロジェクト作成

```bash
npm create vite@latest <project-name> -- --template react-ts
cd <project-name>
```

### Step 2: 追加パッケージのインストール

```bash
# Tailwind CSS
npm install -D tailwindcss @tailwindcss/vite

# 状態管理
npm install zustand

# テスト
npm install -D vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom @types/jest

# ESLint + Prettier
npm install -D prettier eslint-config-prettier eslint-plugin-prettier
```

### Step 3: Tailwind CSS設定

`src/index.css`を以下に置き換える:

```css
@import "tailwindcss";
```

`vite.config.ts`にTailwindプラグインを追加:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react-swc";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: {
      "@": "/src",
    },
  },
});
```

### Step 4: ESLint設定

Viteが生成する`eslint.config.js`に追記する:

```js
import js from "@eslint/js";
import globals from "globals";
import reactHooks from "eslint-plugin-react-hooks";
import reactRefresh from "eslint-plugin-react-refresh";
import tseslint from "typescript-eslint";
import prettier from "eslint-plugin-prettier";

export default tseslint.config(
  { ignores: ["dist"] },
  {
    extends: [js.configs.recommended, ...tseslint.configs.recommended],
    files: ["**/*.{ts,tsx}"],
    languageOptions: {
      ecmaVersion: 2020,
      globals: globals.browser,
    },
    plugins: {
      "react-hooks": reactHooks,
      "react-refresh": reactRefresh,
      prettier,
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      "react-refresh/only-export-components": [
        "warn",
        { allowConstantExport: true },
      ],
      "prettier/prettier": "warn",
    },
  }
);
```

### Step 5: Prettier設定

`.prettierrc`を作成:

```json
{
  "semi": true,
  "singleQuote": false,
  "tabWidth": 2,
  "trailingComma": "all",
  "printWidth": 80
}
```

### Step 6: Vitest設定

Vitest 4.x以降では`vite.config.ts`内の`test`プロパティがサポートされない。テスト設定は別ファイル`vitest.config.ts`に分離する。

`vitest.config.ts`を作成:

```ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react-swc";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      "@": "/src",
    },
  },
  test: {
    globals: true,
    environment: "jsdom",
    setupFiles: "./src/test/setup.ts",
    css: true,
  },
});
```

`vite.config.ts`はStep 3の内容のまま変更しない（Tailwindプラグイン + パスエイリアスのみ）。

`src/test/setup.ts`を作成:

```ts
import "@testing-library/jest-dom";
```

`tsconfig.app.json`の`compilerOptions`に追加:

```json
{
  "compilerOptions": {
    "types": ["vitest/globals", "@testing-library/jest-dom"],
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

`package.json`の`scripts`に追加:

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "lint": "eslint .",
    "format": "prettier --write \"src/**/*.{ts,tsx,css,json}\""
  }
}
```

### Step 7: Dockerfile（マルチステージビルド）

```dockerfile
FROM node:22-slim AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM nginx:alpine AS runtime

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Step 8: nginx.conf

```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### Step 9: docker-compose.yaml

```yaml
services:
  app:
    build: .
    container_name: <project-name>
    ports:
      - "3000:80"
    restart: unless-stopped
```

### Step 10: .gitignore

Viteが生成するものに以下を追記:

```
.env.local
.env.*.local
coverage/
```

### Step 11: 共通ディレクトリ構成

```
src/
├── components/     # 共通UIコンポーネント
│   └── ui/         # ボタン・入力等の基本コンポーネント
├── hooks/          # カスタムフック
├── stores/         # Zustandストア
├── types/          # 型定義
├── lib/            # ユーティリティ関数
├── test/           # テスト設定
│   └── setup.ts
├── App.tsx
├── main.tsx
└── index.css
```

これらの空ディレクトリは`.gitkeep`で保持する。

### Step 12: サンプルストア

`src/stores/useCounterStore.ts`を作成（Zustandの使い方の参考として）:

```ts
import { create } from "zustand";

interface CounterState {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
}

export const useCounterStore = create<CounterState>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}));
```

### Step 13: App.tsxの置き換え

Viteが生成するApp.tsxを、Tailwind CSSとZustandを使ったシンプルな内容に置き換える。具体的な内容はタイプ別リファレンスで定義する。

### Step 14: サンプルテスト

`src/App.test.tsx`を作成:

```tsx
import { render, screen } from "@testing-library/react";
import { describe, it, expect } from "vitest";
import App from "./App";

describe("App", () => {
  it("renders without crashing", () => {
    render(<App />);
    expect(document.querySelector("#root, [data-testid]")).toBeDefined();
  });
});
```

## 生成手順

1. Viteでプロジェクトを作成する
2. 追加パッケージをインストールする
3. 共通設定ファイルを生成する
4. タイプ別リファレンス（`references/<type>.md`）を読み、タイプ固有のファイルを生成する
5. プレースホルダ（`<project-name>`等）をすべて実際の値に置換する
6. `npm run build`で正常にビルドできることを確認する

## 構成検証

生成後、以下をチェックする:

- [ ] `npm run build`がエラーなく完了するか
- [ ] `package.json`にすべてのスクリプト（dev, build, test, lint, format）があるか
- [ ] `vite.config.ts`にTailwindプラグインとパスエイリアスがあるか
- [ ] `vitest.config.ts`にテスト設定があるか
- [ ] `eslint.config.js`が存在するか
- [ ] `.prettierrc`が存在するか
- [ ] `src/test/setup.ts`が存在するか
- [ ] `Dockerfile`と`docker-compose.yaml`が存在するか
- [ ] `nginx.conf`が存在するか
- [ ] Zustandストアのサンプルが存在するか
- [ ] タイプ固有のディレクトリ構成とファイルが存在するか（リファレンス参照）

検証結果を一覧でユーザに報告する。
