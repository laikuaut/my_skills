---
name: react-impl
description: |
  React + TypeScript実装の品質基準を適用するスキル。新規React/TSXコードの作成・既存コードのリファクタリング時に使用する。
  TSDoc、厳密な型定義、ESLint + Prettier、コンポーネント設計、カスタムフック抽出、テスト設計を自動的に適用する。
  Reactコンポーネントの作成・編集・リファクタリング・レビューを依頼されたとき、
  「Reactで実装して」「コンポーネント作って」「TSXを書いて」「リファクタリングして」「フック作って」「TypeScriptで型つけて」
  などReact/TypeScriptの実装に関する指示があったときにこのスキルを使用すること。
  react-project-initとは異なり、既存プロジェクト内でのコード実装に適用する。プロジェクト新規作成には使わない。
---

# React実装スキル

React + TypeScriptのコードを書くとき、このスキルの基準に従って実装する。新規作成でもリファクタリングでも同じ基準を適用する。

## コーディング規約

### 1. TSDoc

すべての公開コンポーネント・フック・ユーティリティ関数にTSDocを書く。

```tsx
/**
 * 商品カードコンポーネント。
 *
 * 商品画像・名前・価格を表示し、カートへの追加操作を提供する。
 * 在庫切れの場合はボタンを無効化し、視覚的に区別する。
 *
 * @param props - コンポーネントのプロパティ
 * @param props.product - 表示する商品データ
 * @param props.onAddToCart - カート追加時のコールバック。商品IDを引数に呼ばれる
 *
 * @example
 * ```tsx
 * <ProductCard
 *   product={{ id: "1", name: "Book", price: 1500, inStock: true }}
 *   onAddToCart={(id) => console.log(`Added: ${id}`)}
 * />
 * ```
 */
```

**判断基準**: 「このコンポーネント/フックを初めて使う人が、TSDocだけで正しく使えるか？」を基準にする。Props名から明らかなことは省略してよいが、コールバックのタイミング・副作用・エッジケースは必ず書く。

### 2. コメント

コードの「なぜ（Why）」を説明するコメントを書く。「何をしているか（What）」はコード自体が語るべき。

```tsx
// 良い例：なぜそうするのかを説明
// リスト項目が1000件を超えるケースがあり、再レンダリングで
// 目に見えるカクつきが出るため仮想化で描画を制限する
const virtualizedItems = useVirtualizer({ count: items.length, ... });

// 悪い例：コードの直訳
// アイテムを仮想化する
const virtualizedItems = useVirtualizer({ count: items.length, ... });
```

**書くべきコメント**:
- ビジネスロジックの背景（なぜこの表示条件なのか）
- パフォーマンス上の理由による設計判断（メモ化・仮想化・遅延読み込み）
- ブラウザ互換性やライブラリ制約によるワークアラウンド
- TODOには担当・期限・チケット番号を含める（`// TODO(user): #1234 期限2024-06`）

### 3. TypeScript型定義

厳密な型定義でコンポーネントの契約を明確にする。

```tsx
// Propsはinterfaceで定義する（拡張性のため）
interface UserProfileProps {
  /** ユーザーデータ */
  user: User;
  /** 編集モードの有効/無効 */
  isEditable?: boolean;
  /** プロフィール更新時のコールバック */
  onUpdate?: (updated: User) => void;
}

// データ型はtypeで定義する（合成しやすい）
type User = {
  id: string;
  name: string;
  email: string;
  role: "admin" | "member" | "guest";
};

// 判別共用体型で非同期状態を表現する（不正な状態を型レベルで排除）
// `isLoading: boolean` + `data: T | null` + `error: string | null` ではなく、
// statusフィールドで判別する形にする。これにより「loading中なのにdataがある」
// といった矛盾した状態をTypeScriptコンパイラが検出してくれる。
type AsyncState<T> =
  | { status: "idle"; data: undefined; error: undefined }
  | { status: "loading"; data: undefined; error: undefined }
  | { status: "success"; data: T; error: undefined }
  | { status: "error"; data: undefined; error: Error };
```

**ルール**:
- Propsは`interface`、データ型は`type`を基本にする（`interface`はextends可能で宣言マージが効く）
- `any`は使わない。不明な型は`unknown`で受けてナローイングする
- **非同期状態は判別共用体型**で表現する。`status`フィールドで状態を判別し、各状態で有効なプロパティを型レベルで制約する
- イベントハンドラは`React.MouseEvent<HTMLButtonElement>`のように要素型を明示する
- 文字列リテラルのユニオン型で列挙値を表現する（enumは避ける）
- コンポーネントの戻り値型は明示しない（`React.FC`も使わない）。TypeScriptのJSX推論に任せる

### 4. Import順序

```tsx
// 1. React本体
import { useState, useCallback, useMemo } from "react";

// 2. サードパーティライブラリ
import { useQuery } from "@tanstack/react-query";
import clsx from "clsx";

// 3. プロジェクト内の共通モジュール（エイリアス `@/` 使用）
import { Button } from "@/components/ui/Button";
import { useAuth } from "@/hooks/useAuth";
import type { User } from "@/types/user";

// 4. 相対パスのローカルモジュール
import { ProfileAvatar } from "./ProfileAvatar";
import { useProfileForm } from "./useProfileForm";
```

グループ間は1行空行。グループ内はアルファベット順。`type`インポートは通常のインポートと分離する（`import type`）。

### 5. Lint・Format（ESLint + Prettier）

コードを書いたら、以下のコマンドで品質をチェックする：

```bash
# lint チェック（自動修正つき）
npx eslint . --fix

# フォーマット
npx prettier --write "src/**/*.{ts,tsx,css,json}"
```

プロジェクトに設定がなければ、`react-project-init`スキルで定義されている設定（`eslint.config.js` / `.prettierrc`）を追加する。

### 6. コンポーネント分割

1つのコンポーネントは**1つの役割**を持つ。コンポーネント関数の本体（`function`の`{`から`}`まで）が**50行を超えたら分割を検討**する。

ここで言う「本体」はロジック + JSXの関数本体のみを指す。import文・Props型定義・TSDocは含めない。

**分割の判断基準**:
- 異なる関心事が混在している → 関心事ごとに分割（表示 / ロジック / データ取得）
- JSX内に条件分岐が3つ以上ネスト → 条件ごとにサブコンポーネント化
- 同じUI構造が2箇所以上に出現 → 共通コンポーネントとして抽出
- フォームのフィールドが繰り返しで肥大化 → `FormField`コンポーネントに切り出し

**ページ（オーケストレータ）コンポーネント**はフック呼び出しと子コンポーネントの配置が主な責務になるため、本体がやや長くなりやすい。その場合でも以下を徹底する：
- ロジックはすべてカスタムフックに移す。ページコンポーネント内に`useCallback`や`useState`を3つ以上書いているなら、専用フック（`usePageName`）への抽出を検討する
- ページ本体はフックの呼び出し + JSXの組み立てのみにする

```tsx
// 悪い例：1コンポーネントに複数の関心事
function OrderPage() {
  // データ取得（20行）...
  // フォームバリデーション（30行）...
  // カート計算ロジック（15行）...
  // 巨大なJSX（100行）...
}

// 良い例：関心事ごとに分割し、ページはフック+JSX配置のみ
function OrderPage() {
  const { order, isLoading } = useOrder(orderId);
  if (isLoading) return <OrderSkeleton />;
  return (
    <div className="space-y-6">
      <OrderSummary order={order} />
      <ShippingForm orderId={order.id} />
      <PaymentSection amount={order.total} />
    </div>
  );
}
```

**フォームの繰り返しパターン**: label + input + errorが同じ構造で複数回出現する場合は`FormField`を抽出する。

```tsx
// FormFieldで繰り返しを解消する
interface FormFieldProps {
  label: string;
  value: string;
  error?: string;
  onChange: (value: string) => void;
  type?: "text" | "textarea";
}

function FormField({ label, value, error, onChange, type = "text" }: FormFieldProps) {
  const Component = type === "textarea" ? "textarea" : "input";
  return (
    <div>
      <label className="mb-1 block text-sm font-medium text-gray-700">{label}</label>
      <Component
        value={value}
        onChange={(e) => onChange(e.target.value)}
        className={clsx("w-full rounded-lg border px-3 py-2", error && "border-red-500")}
      />
      {error && <p className="mt-1 text-sm text-red-500">{error}</p>}
    </div>
  );
}
```

### 7. ファイル・ディレクトリ構成

**1ファイル1コンポーネント**が原則。ファイルが300行を超えたら分割を検討する。

機能（Feature）ベースのディレクトリ構成を推奨する：

```
src/
├── components/           # 汎用UIコンポーネント
│   └── ui/
│       ├── Button.tsx
│       ├── Input.tsx
│       └── Modal.tsx
├── features/             # 機能単位のモジュール
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   └── SignupForm.tsx
│   │   ├── hooks/
│   │   │   └── useAuth.ts
│   │   ├── stores/
│   │   │   └── authStore.ts
│   │   └── types.ts
│   └── products/
│       ├── components/
│       │   ├── ProductCard.tsx
│       │   └── ProductList.tsx
│       ├── hooks/
│       │   └── useProducts.ts
│       └── types.ts
├── hooks/                # アプリ全体で使う汎用フック
│   ├── useLocalStorage.ts
│   └── useMediaQuery.ts
├── stores/               # グローバルストア
│   └── uiStore.ts
├── types/                # 共通型定義
│   └── api.ts
├── lib/                  # ユーティリティ関数
│   ├── api-client.ts
│   └── format.ts
├── App.tsx
├── main.tsx
└── index.css
```

小規模（コンポーネント10個以下）の場合は`features/`を省略してフラットにしてよい。

### 8. コンポーネント設計

- **Propsを小さく保つ**: 5つ以上のPropsがあれば、オブジェクトにまとめるか、コンポーネントの責務を見直す
- **合成（Composition）パターン**: `children`や`Slot`パターンで柔軟にレイアウトを組む
- **制御/非制御を意識する**: フォーム要素は制御コンポーネント（`value` + `onChange`）を基本とする
- **表示と取得を分離する**: データ取得はフックやコンテナコンポーネントに、表示は純粋なコンポーネントに分ける

```tsx
// 良い例：合成パターンでレイアウトを柔軟にする
function Card({ children }: { children: React.ReactNode }) {
  return <div className="rounded-lg border bg-white p-4 shadow-sm">{children}</div>;
}

function CardHeader({ children }: { children: React.ReactNode }) {
  return <div className="mb-3 border-b pb-2 font-bold">{children}</div>;
}

function CardBody({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>;
}

// 使用側で自由に組み合わせられる
<Card>
  <CardHeader>注文履歴</CardHeader>
  <CardBody><OrderList orders={orders} /></CardBody>
</Card>
```

### 9. カスタムフックの抽出

**3回以上**出現するロジックパターン、または1つのコンポーネント内で複雑化したロジックはカスタムフックとして抽出する。

抽出すべきもの：
- データ取得とキャッシュ（`useFetchUser`、`useProducts`）
- フォーム状態管理（`useLoginForm`、`useSearchFilter`）
- ブラウザAPIのラッパー（`useLocalStorage`、`useMediaQuery`）
- タイマー・インターバル（`useDebounce`、`useInterval`）

```tsx
// hooks/useLocalStorage.ts
import { useState, useCallback } from "react";

/**
 * localStorageと同期するstate管理フック。
 *
 * 値の読み書きはJSON.parse/stringifyで自動変換される。
 * localStorageにアクセスできない環境ではinitialValueのみで動作する。
 *
 * @param key - localStorageのキー
 * @param initialValue - 初期値
 * @returns [現在の値, セッター関数] のタプル
 */
function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T | ((prev: T) => T)) => void] {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? (JSON.parse(item) as T) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = useCallback(
    (value: T | ((prev: T) => T)) => {
      setStoredValue((prev) => {
        const nextValue = value instanceof Function ? value(prev) : value;
        localStorage.setItem(key, JSON.stringify(nextValue));
        return nextValue;
      });
    },
    [key],
  );

  return [storedValue, setValue];
}

export { useLocalStorage };
```

### 10. パフォーマンス最適化

過度な最適化は避けるが、以下のケースでは積極的に適用する：

- **`useMemo`**: 計算コストが高い派生データ（フィルタ・ソート・集計）
- **`useCallback`**: 子コンポーネントにPropsとして渡すコールバック関数
- **`React.memo`**: リストのアイテムコンポーネントなど、親の再レンダリングで不要に再描画されるもの
- **`React.lazy` + `Suspense`**: ルート単位のコード分割

```tsx
// リストアイテムはmemo化する（リストが長いとき効果大）
const ProductCard = React.memo(function ProductCard({ product, onSelect }: ProductCardProps) {
  return (
    <div onClick={() => onSelect(product.id)}>
      <h3>{product.name}</h3>
      <p>{product.price.toLocaleString()}円</p>
    </div>
  );
});
```

**やらないこと**:
- すべてのコンポーネントを`React.memo`で囲む（計測なしの最適化は複雑さだけ増やす）
- プリミティブ値や安定した参照の`useMemo` / `useCallback`（効果がない）

### 11. テスト（Vitest + React Testing Library）

テストはユーザーの操作と振る舞いを検証する。実装の詳細（state・内部メソッド）はテストしない。

```tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { describe, it, expect, vi } from "vitest";
import { Counter } from "./Counter";

describe("Counter", () => {
  it("incrementボタンでカウントが1増える", async () => {
    const user = userEvent.setup();
    render(<Counter initialCount={0} />);

    await user.click(screen.getByRole("button", { name: "増やす" }));

    expect(screen.getByText("1")).toBeInTheDocument();
  });

  it("上限に達するとincrementボタンが無効になる", () => {
    render(<Counter initialCount={10} max={10} />);

    expect(screen.getByRole("button", { name: "増やす" })).toBeDisabled();
  });
});
```

**テストの方針**:
- `getByRole` > `getByLabelText` > `getByText` の優先順でクエリする（アクセシビリティ順）
- ユーザーイベントは`@testing-library/user-event`を使う（`fireEvent`より実際の操作に近い）
- 非同期処理は`findBy`クエリか`waitFor`で待つ
- モックは外部依存（API・ストレージ）に限定し、コンポーネント間のモックは避ける

### 12. Tailwind CSSの使用

Tailwind CSSでスタイリングする。条件付きクラスは`clsx`（または`cn`ユーティリティ）で合成する。

```tsx
import clsx from "clsx";

interface ButtonProps {
  variant?: "primary" | "secondary" | "danger";
  size?: "sm" | "md" | "lg";
  disabled?: boolean;
  children: React.ReactNode;
  onClick?: () => void;
}

function Button({ variant = "primary", size = "md", disabled, children, onClick }: ButtonProps) {
  return (
    <button
      className={clsx(
        "rounded font-medium transition-colors focus:outline-none focus:ring-2",
        {
          "bg-blue-600 text-white hover:bg-blue-700": variant === "primary",
          "bg-gray-200 text-gray-800 hover:bg-gray-300": variant === "secondary",
          "bg-red-600 text-white hover:bg-red-700": variant === "danger",
        },
        {
          "px-2 py-1 text-sm": size === "sm",
          "px-4 py-2 text-base": size === "md",
          "px-6 py-3 text-lg": size === "lg",
        },
        disabled && "cursor-not-allowed opacity-50",
      )}
      disabled={disabled}
      onClick={onClick}
    >
      {children}
    </button>
  );
}
```

**ルール**:
- インラインスタイル（`style`属性）は使わない。Tailwindのクラスで表現する
- マジックナンバーの繰り返しはTailwindの設定（`tailwind.config.ts`）で定義する
- レスポンシブはモバイルファーストで`sm:` `md:` `lg:`のブレークポイントを使う

## リファクタリング

既存コードのリファクタリングを依頼された場合も、上記すべての基準を適用する。加えて：

1. **現状把握を先に行う**: コンポーネントツリーとデータフローを把握する
2. **段階的に進める**: 1つの改善を適用→動作確認→次の改善
3. **テストを先に書く**: リファクタリング前に既存の振る舞いをテストで固定する
4. **振る舞いを変えない**: UIの見た目と操作結果を変えない。機能追加と混ぜない

リファクタリングの優先順位：
1. 型定義の追加・厳密化（`any`排除、Props型定義）
2. コンポーネント分割（関数本体50行超のコンポーネントを分解）
3. カスタムフック抽出（ロジックとUIの分離）
4. 共通コンポーネントの抽出（重複UIの統一）
5. TSDoc・コメントの追加（保守性向上）
6. lint/formatの適用（一貫性の確保）

## 品質チェックリスト

コードを書き終えたら、以下を確認する：

- [ ] すべての公開コンポーネント・フックにTSDocがあるか
- [ ] Props型がinterfaceで定義されているか
- [ ] `any`を使っていないか
- [ ] importが規約順に整理されているか
- [ ] `npx eslint .`がエラーなしで通るか
- [ ] `npx prettier --check "src/**/*.{ts,tsx}"`が差分なしで通るか
- [ ] コンポーネント関数本体が50行以内か（超えている場合は分割を検討したか）
- [ ] ページコンポーネントのロジックがカスタムフックに抽出されているか
- [ ] 1ファイルが300行以内か（超えている場合は分割を検討したか）
- [ ] イベントハンドラの型が明示されているか
- [ ] `useMemo` / `useCallback`が必要な箇所にのみ使われているか
- [ ] テストがユーザー操作ベースで書かれているか
