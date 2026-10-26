# API-14 投稿の取得

[API 設計](README.md) / API-14 投稿の取得

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`GET /api/posts/:post_id`（R-14）

**概要**: 投稿 1 件を返す（PST-07）。

**リクエスト**: パス `post_id`（1.2 の ULID）。

**レスポンス**: 200。本文は Post（API-11 の項目と同じ）。

**エラー**（共通: E1）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 404 | `POST_NOT_FOUND` | 存在しない、削除済み、ID の形式が不正 |

**備考**: 使用テーブル: `posts`、`users`、`likes`（`GetPost`）。副作用なし。未ログインでも使える。
