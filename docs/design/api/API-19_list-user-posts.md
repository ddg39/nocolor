# API-19 ユーザーの投稿一覧

[API 設計](README.md) / API-19 ユーザーの投稿一覧

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`GET /api/users/:username/posts`（R-19）

**概要**: ユーザーの投稿を新着順に返す（PRF-02）。

**リクエスト**: パス `username`、クエリ `cursor`・`limit`（1.7）。

**レスポンス**: 200（1.7 の形式。`items` は Post）。`author` は全項目で同じユーザーになるが、形を揃えるため含める。

**エラー**（共通: E1）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 404 | `USER_NOT_FOUND` | 存在しない、仮登録のまま、形式不正 |
| 400 | `INVALID_CURSOR` | カーソルが不正、または別のユーザーのカーソル |
| 400 | `VALIDATION_ERROR` | `fields.limit` = `INVALID_VALUE` |

**備考**: 使用テーブル: `posts`、`users`、`likes`（`ListPostsByUser`）。副作用なし。
