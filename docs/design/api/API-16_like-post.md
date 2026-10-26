# API-16 いいねを付ける

[API 設計](README.md) / API-16 いいねを付ける

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`PUT /api/posts/:post_id/like`（R-16）

**概要**: 閲覧者の、その投稿へのいいねを付ける。自分の投稿にも付けられる（LIK-01）。**冪等**で、すでにいいね済みでも成功する。

**リクエスト**: パス `post_id`。ボディなし。`X-CSRF-Token` が必要。

**レスポンス**: 200

```json
{ "liked": true, "like_count": 13 }
```

| キー | 型 | 説明 |
| --- | --- | --- |
| `liked` | boolean | 操作後の状態（常に `true`） |
| `like_count` | integer | 操作後のいいね数 |

**エラー**（共通: E1、E2）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 404 | `POST_NOT_FOUND` | 存在しない、削除済み、形式不正 |

**備考**:
- 使用テーブル: `likes`、`posts`。
- `INSERT ... ON DUPLICATE KEY UPDATE`（db.md 4.4）。1 ユーザー 1 投稿 1 回は主キー `(post_id, user_id)` が保証する（LIK-01）。存在しない投稿は外部キー違反（1452）で検知して 404 にする。
- `like_count` は操作の直後に数え直した値。他の人の同時の操作で画面の表示とずれることがあるが、次の取得で直る。
