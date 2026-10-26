# API-07 ログアウト

[API 設計](README.md) / API-07 ログアウト

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`POST /api/auth/logout`（R-07）

**概要**: 現在のセッションを破棄する（ACC-07、6.2）。

**リクエスト**: なし（ボディ不要）。`X-CSRF-Token` ヘッダーが必要。

**レスポンス**: 204（ボディなし）。`Set-Cookie: nocolor_session=; Max-Age=0; ...` で Cookie を削除する。

**エラー**（共通: E1、E2）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 401 | `UNAUTHENTICATED` | すでにログアウト済み・期限切れ。画面は成功扱いにしてよい（screens R-07） |

**備考**:
- 使用テーブル: `sessions`（`DeleteSession`）。
- 副作用: 当該セッションの行を削除する。他の端末のセッションは残す。
