# API-11 全体タイムライン

[API 設計](README.md) / API-11 全体タイムライン

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`GET /api/timelines/global`（R-11）

**概要**: 全ユーザーの投稿を新着順に返す（TML-02）。未ログインでも使える（TML-04）。

**リクエスト**: クエリ `cursor`、`limit`（1.7）。

**レスポンス**: 200（1.7 の形式。`items` は Post）

```json
{
  "items": [
    {
      "id": "01JC8ZQ4V6S2R9T3M7X5N1B0KD",
      "image_url": "/images/posts/3f1c2a9e-7b0d-4c8a-9d52-1e6f0a7b4c11.jpg",
      "image_width": 2400,
      "image_height": 1600,
      "thumbnail_url": "/images/thumbs/a81d5e07-93c2-4b6a-8f10-5c7d2e9b0a44.jpg",
      "thumbnail_width": 600,
      "thumbnail_height": 400,
      "caption": "朝の駅\n人はまだ少ない",
      "created_at": "2026-10-26T05:30:12.345Z",
      "like_count": 12,
      "liked_by_me": false,
      "author": {
        "username": "alice",
        "display_name": "Alice",
        "icon_url": null
      }
    }
  ],
  "next_cursor": "AQ3v1o...（不透明な文字列）"
}
```

**エラー**（共通: E1）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 400 | `INVALID_CURSOR` | カーソルが不正 |
| 400 | `VALIDATION_ERROR` | `fields.limit` = `INVALID_VALUE` |

**備考**: 使用テーブル: `posts`、`users`、`likes`（`ListGlobalTimeline`）。副作用なし。未ログインは閲覧者 ID を 0 として `liked_by_me` を `false` にする。並びは `posts.id` の降順（1.7）。
