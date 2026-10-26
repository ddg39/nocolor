# API-18 プロフィールの取得

[API 設計](README.md) / API-18 プロフィールの取得

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`GET /api/users/:username`（R-18）

**概要**: ユーザーのプロフィールを返す（PRF-01、PRF-02）。SCR-12 の初期値にも使う。

**リクエスト**: パス `username`（1.2。形式不正は 404）。

**レスポンス**: 200。本文は Profile（2 章）。

```json
{
  "username": "alice",
  "display_name": "Alice",
  "bio": "朝の光を撮っています。",
  "icon_url": "/images/avatars/9b2e7c40-1d3a-4f5b-8c6d-2a1e0f9b7d33.jpg",
  "post_count": 12,
  "following_count": 34,
  "follower_count": 56,
  "is_following": true
}
```

**エラー**（共通: E1）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 404 | `USER_NOT_FOUND` | 存在しない、仮登録のまま、形式不正 |

**備考**: 使用テーブル: `users`、`posts`、`follows`（`GetUserProfile`）。副作用なし。自分自身を取得したとき（ユーザー名が閲覧者と同じ）、`is_following` は `false`。メールアドレスは返さない。
