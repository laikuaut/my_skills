# SPAプロジェクト構成

シンプルなSingle Page Application向けの構成。React Router でページ遷移を実現する。

## 追加パッケージ

```bash
npm install react-router-dom
```

## ディレクトリ構成

```
<project-name>/
├── public/
├── src/
│   ├── components/
│   │   └── ui/
│   │       └── Button.tsx
│   ├── hooks/
│   │   └── .gitkeep
│   ├── stores/
│   │   └── useCounterStore.ts
│   ├── types/
│   │   └── .gitkeep
│   ├── lib/
│   │   └── .gitkeep
│   ├── pages/
│   │   ├── Home.tsx
│   │   └── About.tsx
│   ├── layouts/
│   │   └── RootLayout.tsx
│   ├── test/
│   │   └── setup.ts
│   ├── App.tsx
│   ├── App.test.tsx
│   ├── main.tsx
│   └── index.css
├── vite.config.ts
├── eslint.config.js
├── .prettierrc
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
├── Dockerfile
├── docker-compose.yaml
├── nginx.conf
├── package.json
└── .gitignore
```

## タイプ固有ファイル

### src/pages/Home.tsx

```tsx
import { useCounterStore } from "@/stores/useCounterStore";
import { Button } from "@/components/ui/Button";

export default function Home() {
  const { count, increment, decrement, reset } = useCounterStore();

  return (
    <div className="flex flex-col items-center justify-center min-h-[60vh] gap-6">
      <h1 className="text-4xl font-bold text-gray-900">Welcome</h1>
      <p className="text-lg text-gray-600">
        Vite + React + TypeScript + Tailwind CSS
      </p>
      <div className="flex items-center gap-4">
        <Button onClick={decrement}>-</Button>
        <span className="text-2xl font-mono w-12 text-center">{count}</span>
        <Button onClick={increment}>+</Button>
      </div>
      <button
        onClick={reset}
        className="text-sm text-gray-500 hover:text-gray-700 underline"
      >
        Reset
      </button>
    </div>
  );
}
```

### src/pages/About.tsx

```tsx
export default function About() {
  return (
    <div className="flex flex-col items-center justify-center min-h-[60vh] gap-4">
      <h1 className="text-4xl font-bold text-gray-900">About</h1>
      <p className="text-gray-600">This is a sample SPA project.</p>
    </div>
  );
}
```

### src/layouts/RootLayout.tsx

```tsx
import { Link, Outlet } from "react-router-dom";

export default function RootLayout() {
  return (
    <div className="min-h-screen bg-gray-50">
      <nav className="bg-white shadow-sm border-b border-gray-200">
        <div className="max-w-5xl mx-auto px-4 py-3 flex gap-6">
          <Link
            to="/"
            className="text-gray-700 hover:text-gray-900 font-medium"
          >
            Home
          </Link>
          <Link
            to="/about"
            className="text-gray-700 hover:text-gray-900 font-medium"
          >
            About
          </Link>
        </div>
      </nav>
      <main className="max-w-5xl mx-auto px-4 py-8">
        <Outlet />
      </main>
    </div>
  );
}
```

### src/components/ui/Button.tsx

```tsx
import { type ButtonHTMLAttributes } from "react";

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary";
}

export function Button({
  variant = "primary",
  className = "",
  children,
  ...props
}: ButtonProps) {
  const base =
    "px-4 py-2 rounded-lg font-medium transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2";
  const variants = {
    primary:
      "bg-blue-600 text-white hover:bg-blue-700 focus:ring-blue-500",
    secondary:
      "bg-gray-200 text-gray-800 hover:bg-gray-300 focus:ring-gray-400",
  };

  return (
    <button className={`${base} ${variants[variant]} ${className}`} {...props}>
      {children}
    </button>
  );
}
```

### src/App.tsx

```tsx
import { BrowserRouter, Routes, Route } from "react-router-dom";
import RootLayout from "@/layouts/RootLayout";
import Home from "@/pages/Home";
import About from "@/pages/About";

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route element={<RootLayout />}>
          <Route path="/" element={<Home />} />
          <Route path="/about" element={<About />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}
```

### src/App.test.tsx

```tsx
import { render, screen } from "@testing-library/react";
import { describe, it, expect } from "vitest";
import App from "./App";

describe("App", () => {
  it("renders the home page", () => {
    render(<App />);
    expect(screen.getByText("Welcome")).toBeInTheDocument();
  });
});
```

## 検証項目

- [ ] `src/pages/Home.tsx`と`src/pages/About.tsx`が存在するか
- [ ] `src/layouts/RootLayout.tsx`が存在するか
- [ ] `src/components/ui/Button.tsx`が存在するか
- [ ] `react-router-dom`が`package.json`のdependenciesに含まれるか
- [ ] App.tsxでBrowserRouterによるルーティングが設定されているか
