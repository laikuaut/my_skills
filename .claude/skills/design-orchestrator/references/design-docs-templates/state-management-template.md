# state-management.md テンプレート（D2.3 フロント状態管理）

```markdown
# フロント状態管理 (D2.3)

- 対象: React での状態分類と管理手法
- 作成日: YYYY-MM-DD
- 関連: [architecture.md](./architecture.md) → 本書
- 状態: draft

## 1. 設計原則

- **サーバー状態は TanStack Query、UI 状態は Zustand**。両者を混ぜない
- サーバー状態を Zustand に入れない（キャッシュ二重管理で不整合を起こす）
- URL で表現できるものは URL に乗せる（検索条件・タブ・ページ番号）

## 2. 状態分類

| 分類 | 例 | 管理手法 |
|------|------|---------|
| サーバー状態 | タスク一覧、ユーザー情報 | TanStack Query |
| URL 状態 | 検索文字列、タブ選択 | search params |
| UI 状態（アプリ横断） | ダークモード、Toast | Zustand |
| UI 状態（ローカル） | モーダル open/close | useState |
| フォーム状態 | 入力中の値 | useState or react-hook-form |

## 3. TanStack Query 設定

```ts
// default
{
  staleTime: 0,
  refetchOnWindowFocus: true,
  retry: 1,
}
```
- `staleTime=0` = 必要時には常に再取得。ただし `placeholderData` で UX を保つ
- ミューテーション成功時は `invalidateQueries` で関連キャッシュを無効化

## 4. クエリキー規約

```
["tasks"]                 -- 一覧
["tasks", { status }]     -- 条件付き一覧
["task", id]              -- 単一
["user", "me"]            -- 自分のユーザー情報
```

## 5. Zustand ストア設計

```ts
type UIStore = {
  theme: "light" | "dark";
  toasts: Toast[];
  setTheme: (t: "light" | "dark") => void;
  pushToast: (t: Toast) => void;
};
```
- ストア1つに肥大化させず、ドメイン別に分ける
- サーバー由来データを置かない

## 6. 引継ぎメモ

- D2.4 error-handling へ: Query のエラーは onError でトーストに流す共通化
- D3.1 api-detail へ: エンドポイントは Query キーと1対1で対応させると運用が楽

## 7. 完了判定 (DoD)

- [ ] 状態分類表が埋まっている
- [ ] TanStack Query のデフォルト設定が決まっている
- [ ] Zustand ストアの責務範囲が書かれている
```
