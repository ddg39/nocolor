# API-10 パスワードの再設定

[API 設計](README.md) / API-10 パスワードの再設定

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`POST /api/auth/password/reset`（R-10）

**概要**: 再設定トークンと新しいパスワードで、パスワードを変更する（ACC-08）。成功時にトークンを使用済みにし、**そのユーザーの全セッションを破棄する**（6.2）。

**リクエスト**: JSON

| 項目 | 型 | 必須 | 規則 |
| --- | --- | --- | --- |
| `token` | string | 要 | 43 文字の Base64URL |
| `password` | string | 要 | 1.9 のパスワード |

**レスポンス**: 204（ボディなし）

**エラー**（共通: E1、E3、E4）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 400 | `TOKEN_INVALID` | トークンが存在しない・期限切れ・使用済み・形式不正（`password` の検証より先に判定する。1.5 の例外） |
| 400 | `VALIDATION_ERROR` | `fields.password` = `REQUIRED` / `TOO_SHORT` / `TOO_LONG` / `NEED_LETTER_AND_DIGIT`。トークンは消費されない |
| 429 | `RATE_LIMITED` | `token_ip` |

**備考**:
- 使用テーブル: `password_reset_tokens`、`users`、`sessions`、`rate_limits`。
- 処理（1 つのトランザクション）: (1) `UPDATE password_reset_tokens SET used_at = now WHERE token_hash = ? AND used_at IS NULL AND expires_at > now` の影響行数が 1 であること（並行して 2 回使われても片方だけ成功する。db.md 3.3）、(2) `users.password_hash` を更新、(3) そのユーザーの全セッションを削除（`DeleteSessionsByUser`）、(4) そのユーザーの他の未使用の再設定トークンも使用済みにする。リクエストの処理順は、(0) `token` の形式と有効性を読み取りで確認（無効なら `TOKEN_INVALID`。`password` が不正でもこちらを返す）、(0') `password` の検証（失敗は `VALIDATION_ERROR`）、そのあと上の (1)〜(4) である。パスワードの検証失敗（400）のときは (1) の前に返すので、トークンは消費されず残る。(0) と (1) の間に期限切れ等が起きても、(1) の条件付き更新が最終的に判定し、失敗なら `TOKEN_INVALID` を返す。
- 副作用: **全セッションの破棄**。リクエストに有効なセッションの Cookie が付いていた場合は、そのセッションも破棄されるため、応答で Cookie を削除する。自動ログインはしない（screens 3.4.3）。
- 対象は本登録済みのユーザーのトークンだけである（API-08 が仮登録には発行しない）。
