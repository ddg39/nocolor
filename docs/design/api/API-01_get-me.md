# API-01 現在のユーザーの取得

[API 設計](README.md) / API-01 現在のユーザーの取得

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`GET /api/me`（R-01）

**概要**: ログイン状態を判定する。アプリ起動時に呼ぶ。ログイン中は CSRF トークン（1.4）も返す。未ログインでも 200 を返す。

**リクエスト**: なし。

**レスポンス**: 200

```json
{
  "user": {
    "username": "alice",
    "display_name": "Alice",
    "icon_url": "/images/avatars/9b2e7c40-1d3a-4f5b-8c6d-2a1e0f9b7d33.jpg"
  },
  "csrf_token": "q8Yx0m2k...（43 文字）"
}
```

未ログイン:

```json
{ "user": null, "csrf_token": null }
```

| キー | 型 | 説明 |
| --- | --- | --- |
| `user` | UserBrief \| null | ログイン中のユーザー |
| `csrf_token` | string \| null | 1.4 のトークン。ログイン中のみ |

**エラー**: E1 のみ。

**備考**:
- 使用テーブル: `sessions`、`users`（`GetSessionWithUser`）。
- 副作用なし。Cookie が付いていたのに無効だった場合は、応答で Cookie を削除する。
- `Cache-Control: no-store`（トークンを含むため）。
