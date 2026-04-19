# error-handling.md テンプレート（D2.4 エラーハンドリング）

```markdown
# エラーハンドリング (D2.4)

- 対象: フロント / バック / 境界でのエラー処理
- 作成日: YYYY-MM-DD
- 関連: [architecture.md](./architecture.md) → 本書
- 状態: draft

## 1. 設計原則

- **エラーは3種に分類**: バリデーションエラー / 業務エラー / 予期しないエラー
- バックは RFC 7807 `application/problem+json` で返す
- フロントは原則トーストで通知。致命的エラーのみ画面全体でハンドリング

## 2. エラーレスポンス形式（バック）

```json
{
  "type": "https://example.com/errors/validation",
  "title": "Invalid input",
  "status": 400,
  "detail": "title must be 1..200 characters",
  "errors": {
    "title": ["must be 1..200 characters"]
  }
}
```

| HTTP | 用途 |
|------|------|
| 400 | バリデーション |
| 401 | 未認証 |
| 403 | 権限なし |
| 404 | 未存在 |
| 409 | 衝突（楽観ロック等） |
| 422 | 業務エラー |
| 500 | 予期しないエラー |

## 3. フロントでの扱い

| HTTP | 挙動 |
|------|------|
| 400 / 422 | フォームに inline error 表示 + トースト（任意） |
| 401 | ログイン画面へリダイレクト |
| 403 | トースト + 該当UI無効化 |
| 404 | 画面内の空状態 or 404 画面 |
| 409 | 最新データ再取得のダイアログ |
| 5xx | グローバルトースト、Error Boundary で包む |

## 4. ログ

- request_id をフロント→バック→ログ→エラートラッキングで一貫させる
- PII（個人情報）はログに書かない

## 5. フロント Error Boundary

- ルート直下に1つ配置。予期しない例外時はフォールバック UI と「再読み込み」ボタン

## 6. 引継ぎメモ

- D3.1 api-detail へ: 各エンドポイントでエラー例（400/404/409）を明記
- D1.5 interactions へ: トースト表示時間は danger=5s、info=3s

## 7. 完了判定 (DoD)

- [ ] エラー3分類と HTTP ステータス表が揃っている
- [ ] RFC 7807 形式の例がある
- [ ] Error Boundary / ログ方針が決まっている
```
