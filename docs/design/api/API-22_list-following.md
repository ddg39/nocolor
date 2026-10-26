# API-22 フォロー一覧

[API 設計](README.md) / API-22 フォロー一覧

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`GET /api/users/:username/following`（R-22）

**概要**: そのユーザーがフォローしているユーザーを、フォローした順（新しい順）に返す（PRF-05）。

**リクエスト**: パス `username`、クエリ `cursor`・`limit`（1.7）。

**レスポンス**: 200（1.7 の形式。`items` は UserSummary）

```json
{
  "items": [
    {
      "username": "bob",
      "display_name": "Bob",
      "icon_url": null,
      "bio": "白黒写真が好きです。",
      "is_following": true
    }
  ],
  "next_cursor": null
}
```

**エラー**（共通: E1）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 404 | `USER_NOT_FOUND` | 存在しない、仮登録のまま、形式不正 |
| 400 | `INVALID_CURSOR` | カーソルが不正、または別のユーザー・別の一覧のカーソル |
| 400 | `VALIDATION_ERROR` | `fields.limit` = `INVALID_VALUE` |

**備考**: 使用テーブル: `follows`、`users`（`ListFollowing`）。並びは `(follows.created_at, 相手の内部 ID)` の降順。副作用なし。`is_following` は閲覧者から見た値。
