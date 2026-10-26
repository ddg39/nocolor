# API-06 ログイン

[API 設計](README.md) / API-06 ログイン

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`POST /api/auth/login`（R-06）

**概要**: ユーザー名またはメールアドレスとパスワードでログインし、セッションを開始する（ACC-06）。本登録前のアカウントはログインできず、確認メールの再送を案内する（ACC-04）。

**リクエスト**: JSON

| 項目 | 型 | 必須 | 規則 |
| --- | --- | --- | --- |
| `identifier` | string | 要 | ユーザー名またはメールアドレス。前後の空白を除き小文字にする。`@` を含めばメールアドレス、含まなければユーザー名として照合する（ユーザー名に `@` は使えない。db.md 4.5）。形式は検査しない |
| `password` | string | 要 | 空でないこと。文字数・文字種は検査しない |

どちらかが空のときは 400 `VALIDATION_ERROR`（`fields` に `REQUIRED`）。

**レスポンス**: 200

```json
{
  "user": { "username": "alice", "display_name": "Alice", "icon_url": null },
  "csrf_token": "q8Yx0m2k...（43 文字）"
}
```

`Set-Cookie: nocolor_session=...; HttpOnly; SameSite=Lax; Path=/; Max-Age=2592000`（HTTPS では `Secure` も）

**エラー**（共通: E1、E3、E4）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 400 | `VALIDATION_ERROR` | `identifier` / `password` が空 |
| 401 | `INVALID_CREDENTIALS` | ユーザーが存在しない、パスワードが違う、または期限切れ（7 日超）の仮登録。区別しない |
| 403 | `EMAIL_NOT_VERIFIED` | **パスワードが正しく**、かつ `email_verified_at` が `NULL`（ACC-04） |
| 429 | `RATE_LIMITED` | `login_ip` / `login_account` |

**備考**:
- 使用テーブル: `users`、`sessions`、`rate_limits`。
- **ACC-04 の応答**: パスワードの検証に成功した後でだけ `email_verified_at` を調べる（db.md 4.5 の末尾）。パスワードが違う場合は、確認状態にかかわらず `INVALID_CREDENTIALS` を返す。これにより、パスワードを知らない第三者は「このアカウントは仮登録中」という情報を得られない。`EMAIL_NOT_VERIFIED` の応答にメールアドレスを含めない（画面は入力された識別子から再送フォームに補う。screens SCR-05）。
- **列挙対策**: ユーザーが存在しないときも、ダミーのハッシュに対して bcrypt の照合を行い、処理時間を揃える。`INVALID_CREDENTIALS` のメッセージも共通にする。
- 副作用: 成功時に `sessions` へ行を追加し（有効期限 30 日）、`Set-Cookie` を返す。リクエストに有効な別のセッションの Cookie が付いていた場合は、そのセッションを削除してから新しいものを発行する。同じユーザーの他のセッション（他の端末）は残す。
- 失敗の応答では `Set-Cookie` を返さない。
- セッションの有効期限は**ログインからの固定の 30 日**で、アクセスしても延長しない（スライディングしない。設計判断 D-26）。`sessions.expires_at` と Cookie の `Max-Age` は、ともにログイン時刻から 30 日にする。
