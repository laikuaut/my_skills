# tokens.md テンプレート（D1.2 デザイントークン）

```markdown
# デザイントークン (D1.2)

- 対象: Tailwind config / CSS 変数
- 作成日: YYYY-MM-DD
- 関連: [concept.md](./concept.md) → 本書 → [components.md](./components.md)
- 状態: draft

## 1. 設計原則

- **Tailwind の標準スケールから逸脱しすぎない**。独自値は最小限
- **セマンティックトークン** を用意（`bg-surface` / `text-muted` 等）。生の色は使わない
- **ダークモード対応可否** をここで決める（対応する / v1 は見送り）

## 2. カラートークン

| 名前 | Light | Dark | 用途 |
|------|-------|------|------|
| surface | #ffffff | #0b0b0c | ページ背景 |
| surface-elevated | #f7f7f8 | #18181b | カード・モーダル |
| text | #0f172a | #e5e7eb | 本文 |
| text-muted | #64748b | #9ca3af | 補助テキスト |
| border | #e5e7eb | #27272a | 枠線 |
| accent | #2563eb | #3b82f6 | CTA・リンク |
| danger | #dc2626 | #ef4444 | エラー・削除 |

## 3. スペーシング

| 名前 | 値 (px) | 用途 |
|------|---------|------|
| xs | 4 | アイコン間隔 |
| sm | 8 | タイト余白 |
| md | 16 | 標準ブロック間 |
| lg | 24 | セクション間 |
| xl | 48 | ページ余白 |

## 4. タイポグラフィ

| 名前 | サイズ | 行間 | weight | 用途 |
|------|--------|------|--------|------|
| xs | 12 | 16 | 400 | メタ情報 |
| sm | 14 | 20 | 400 | 本文 |
| base | 14 | 20 | 500 | 強調本文 |
| lg | 18 | 24 | 600 | セクション見出し |
| xl | 24 | 32 | 700 | ページタイトル |

## 5. 角丸・影

| 名前 | 値 | 用途 |
|------|-----|------|
| radius-sm | 4px | ボタン |
| radius-md | 8px | カード |
| shadow-1 | 0 1px 2px rgba(0,0,0,.06) | 浮き要素 |
| shadow-2 | 0 4px 16px rgba(0,0,0,.08) | モーダル |

## 6. Tailwind 設定への反映

```js
// tailwind.config.ts の theme.extend に反映する想定
// colors / spacing / fontSize / borderRadius / boxShadow
```

## 7. 引継ぎメモ

- D1.3 components へ: ボタン＝accent/danger、カード＝surface-elevated + shadow-1 を基本に
- ダークモード: v1 は {対応する / 対応しない}

## 8. 完了判定 (DoD)

- [ ] カラー・スペーシング・タイポ・角丸・影の5カテゴリが埋まっている
- [ ] Tailwind config に反映可能な形式
- [ ] ダークモード対応可否が明示されている
```
