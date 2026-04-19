# components.md テンプレート（D1.3 コンポーネントカタログ）

```markdown
# コンポーネントカタログ (D1.3)

- 対象: 全画面共通のUI部品
- 作成日: YYYY-MM-DD
- 関連: [tokens.md](./tokens.md) → 本書 → [interactions.md](./interactions.md)
- 状態: draft

## 1. 設計原則

- **1コンポーネント = 1責務**。複数責務を持たせない
- Props は型で表現する。boolean のスイッチを増やすより variant 文字列で分岐
- 50行を超えたら分割を検討（react-impl スキルに準拠）

## 2. コンポーネント一覧

| 名前 | 責務 | variant | サイズ | 状態 |
|------|------|---------|--------|------|
| Button | 汎用ボタン | primary / secondary / danger / ghost | sm / md | default / hover / disabled / loading |
| IconButton | アイコンのみボタン | — | sm / md | default / hover / disabled |
| Input | テキスト入力 | — | md | default / focus / error / disabled |
| Select | ドロップダウン | — | md | default / open / disabled |
| Checkbox | チェックボックス | — | sm | default / checked / indeterminate / disabled |
| Badge | ラベル | default / info / success / warn / danger | sm | — |
| Card | 情報グループ | — | — | default / interactive |
| Modal | オーバーレイ | — | sm / md / lg | open / closed |
| Toast | 一時通知 | info / success / warn / danger | — | — |
| Table | データ一覧 | — | — | default / loading / empty |
| EmptyState | 空状態 | — | — | — |

## 3. 代表コンポーネントの仕様

### Button
```ts
type ButtonProps = {
  variant?: "primary" | "secondary" | "danger" | "ghost";
  size?: "sm" | "md";
  loading?: boolean;
  disabled?: boolean;
  leftIcon?: React.ReactNode;
  onClick?: () => void;
  children: React.ReactNode;
};
```
- 高さ: sm=28px / md=32px
- loading 中は spinner を出し、クリックを無効化
- disabled と loading は同時指定可

### Input
```ts
type InputProps = {
  value: string;
  onChange: (v: string) => void;
  placeholder?: string;
  error?: string;     // undefined なら正常状態
  disabled?: boolean;
  autoFocus?: boolean;
};
```
- error 時: 枠線 danger + 下部に error メッセージ
- 文字数上限は呼び出し側でバリデーションし、error に渡す

（他コンポーネントも同様の粒度で書く）

## 4. 依存の向き

```
Button / Input / Badge （atomic）
   ↓
Card / Table / Modal （combined）
   ↓
画面 (features/*)
```
atomic → combined → 画面 の一方向依存。逆参照禁止。

## 5. 引継ぎメモ

- D1.5 interactions へ: hover/focus/disabled の状態遷移を定義
- D2.3 state-management へ: Modal の open/close は URL or ローカル state のどちらで管理するか

## 6. 完了判定 (DoD)

- [ ] コンポーネント一覧が列挙されている
- [ ] 主要3〜5コンポーネントに Props 型と挙動が記述されている
- [ ] 依存の向きが図示されている
```
