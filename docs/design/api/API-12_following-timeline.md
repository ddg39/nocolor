# API-12 フォロー中タイムライン

[API 設計](README.md) / API-12 フォロー中タイムライン

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`GET /api/timelines/following`（R-12）

**概要**: フォローしているユーザーと自分の投稿を新着順に返す（TML-03）。

**リクエスト**: クエリ `cursor`、`limit`（1.7）。

**レスポンス**: 200（API-11 と同じ形式）

**エラー**（共通: E1）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 401 | `UNAUTHENTICATED` | 未ログイン |
| 400 | `INVALID_CURSOR` | カーソルが不正 |
| 400 | `VALIDATION_ERROR` | `fields.limit` = `INVALID_VALUE` |

**備考**: 使用テーブル: `posts`、`users`、`follows`、`likes`（`ListFollowingTimeline`）。副作用なし。GET なので CSRF トークンは不要。
