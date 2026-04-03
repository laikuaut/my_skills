# ダッシュボードプロジェクト構成

管理画面・ダッシュボード向けの構成。サイドバーナビゲーションとReact Routerによるページ遷移を実現する。

## 追加パッケージ

```bash
npm install react-router-dom lucide-react
```

`lucide-react`はアイコンライブラリ。サイドバーのナビゲーションアイコンに使用する。

## ディレクトリ構成

```
<project-name>/
├── public/
├── src/
│   ├── components/
│   │   ├── ui/
│   │   │   ├── Button.tsx
│   │   │   └── Card.tsx
│   │   └── layout/
│   │       ├── Sidebar.tsx
│   │       └── Header.tsx
│   ├── hooks/
│   │   └── .gitkeep
│   ├── stores/
│   │   ├── useCounterStore.ts
│   │   └── useSidebarStore.ts
│   ├── types/
│   │   └── .gitkeep
│   ├── lib/
│   │   └── .gitkeep
│   ├── pages/
│   │   ├── Dashboard.tsx
│   │   └── Settings.tsx
│   ├── layouts/
│   │   └── DashboardLayout.tsx
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

### src/stores/useSidebarStore.ts

```ts
import { create } from "zustand";

interface SidebarState {
  isOpen: boolean;
  toggle: () => void;
}

export const useSidebarStore = create<SidebarState>((set) => ({
  isOpen: true,
  toggle: () => set((state) => ({ isOpen: !state.isOpen })),
}));
```

### src/components/ui/Button.tsx

SPAタイプと同じ`Button.tsx`を使用する（`references/spa.md`参照）。

### src/components/ui/Card.tsx

```tsx
import { type ReactNode } from "react";

interface CardProps {
  title: string;
  children: ReactNode;
  className?: string;
}

export function Card({ title, children, className = "" }: CardProps) {
  return (
    <div
      className={`bg-white rounded-xl shadow-sm border border-gray-200 p-6 ${className}`}
    >
      <h3 className="text-sm font-medium text-gray-500 mb-2">{title}</h3>
      <div>{children}</div>
    </div>
  );
}
```

### src/components/layout/Sidebar.tsx

```tsx
import { Link, useLocation } from "react-router-dom";
import { LayoutDashboard, Settings } from "lucide-react";
import { useSidebarStore } from "@/stores/useSidebarStore";

const navItems = [
  { to: "/", label: "Dashboard", icon: LayoutDashboard },
  { to: "/settings", label: "Settings", icon: Settings },
];

export default function Sidebar() {
  const location = useLocation();
  const { isOpen } = useSidebarStore();

  return (
    <aside
      className={`bg-gray-900 text-white transition-all duration-300 ${
        isOpen ? "w-64" : "w-16"
      }`}
    >
      <div className="p-4">
        <h1
          className={`font-bold text-lg ${isOpen ? "block" : "hidden"}`}
        >
          Admin
        </h1>
      </div>
      <nav className="mt-4">
        {navItems.map((item) => {
          const isActive = location.pathname === item.to;
          return (
            <Link
              key={item.to}
              to={item.to}
              className={`flex items-center gap-3 px-4 py-3 transition-colors ${
                isActive
                  ? "bg-gray-800 text-white"
                  : "text-gray-400 hover:text-white hover:bg-gray-800"
              }`}
            >
              <item.icon size={20} />
              {isOpen && <span>{item.label}</span>}
            </Link>
          );
        })}
      </nav>
    </aside>
  );
}
```

### src/components/layout/Header.tsx

```tsx
import { Menu } from "lucide-react";
import { useSidebarStore } from "@/stores/useSidebarStore";

export default function Header() {
  const { toggle } = useSidebarStore();

  return (
    <header className="bg-white border-b border-gray-200 px-6 py-4 flex items-center justify-between">
      <button
        onClick={toggle}
        className="p-1 rounded-lg hover:bg-gray-100 transition-colors"
      >
        <Menu size={20} />
      </button>
      <div className="text-sm text-gray-500">Admin Panel</div>
    </header>
  );
}
```

### src/layouts/DashboardLayout.tsx

```tsx
import { Outlet } from "react-router-dom";
import Sidebar from "@/components/layout/Sidebar";
import Header from "@/components/layout/Header";

export default function DashboardLayout() {
  return (
    <div className="flex h-screen bg-gray-50">
      <Sidebar />
      <div className="flex-1 flex flex-col overflow-hidden">
        <Header />
        <main className="flex-1 overflow-y-auto p-6">
          <Outlet />
        </main>
      </div>
    </div>
  );
}
```

### src/pages/Dashboard.tsx

```tsx
import { Card } from "@/components/ui/Card";

export default function Dashboard() {
  return (
    <div>
      <h1 className="text-2xl font-bold text-gray-900 mb-6">Dashboard</h1>
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
        <Card title="Total Users">
          <p className="text-3xl font-bold text-gray-900">1,234</p>
        </Card>
        <Card title="Revenue">
          <p className="text-3xl font-bold text-gray-900">$12,345</p>
        </Card>
        <Card title="Orders">
          <p className="text-3xl font-bold text-gray-900">567</p>
        </Card>
        <Card title="Conversion">
          <p className="text-3xl font-bold text-gray-900">3.2%</p>
        </Card>
      </div>
    </div>
  );
}
```

### src/pages/Settings.tsx

```tsx
export default function Settings() {
  return (
    <div>
      <h1 className="text-2xl font-bold text-gray-900 mb-6">Settings</h1>
      <div className="bg-white rounded-xl shadow-sm border border-gray-200 p-6">
        <p className="text-gray-600">Settings page content goes here.</p>
      </div>
    </div>
  );
}
```

### src/App.tsx

```tsx
import { BrowserRouter, Routes, Route } from "react-router-dom";
import DashboardLayout from "@/layouts/DashboardLayout";
import Dashboard from "@/pages/Dashboard";
import Settings from "@/pages/Settings";

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route element={<DashboardLayout />}>
          <Route path="/" element={<Dashboard />} />
          <Route path="/settings" element={<Settings />} />
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
  it("renders the dashboard page", () => {
    render(<App />);
    expect(screen.getByText("Dashboard")).toBeInTheDocument();
  });
});
```

## 検証項目

- [ ] `src/pages/Dashboard.tsx`と`src/pages/Settings.tsx`が存在するか
- [ ] `src/layouts/DashboardLayout.tsx`が存在するか
- [ ] `src/components/layout/Sidebar.tsx`と`src/components/layout/Header.tsx`が存在するか
- [ ] `src/components/ui/Card.tsx`が存在するか
- [ ] `src/stores/useSidebarStore.ts`が存在するか
- [ ] `react-router-dom`と`lucide-react`が`package.json`のdependenciesに含まれるか
- [ ] App.tsxでBrowserRouterによるルーティングが設定されているか
