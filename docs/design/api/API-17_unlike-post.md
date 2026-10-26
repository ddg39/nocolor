# API-17 いいねを取り消す

[API 設計](README.md) / API-17 いいねを取り消す

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`DELETE /api/posts/:post_id/like`（R-17）

**概要**: いいねを取り消す（LIK-02）。**冪等**で、いいねしていなくても成功する。

**リクエスト**: パス `post_id`。`X-CSRF-Token` が必要。

**レスポンス**: 200

```json
{ "liked": false, "like_count": 12 }
```

**エラー**（共通: E1、E2）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 404 | `POST_NOT_FOUND` | 存在しない、削除済み、形式不正 |

**備考**: 使用テーブル: `likes`、`posts`。`DELETE FROM likes WHERE post_id = ? AND user_id = ?`。影響行数が 0 でも成功とする。投稿の存在は別に確認する（存在しなければ 404）。
