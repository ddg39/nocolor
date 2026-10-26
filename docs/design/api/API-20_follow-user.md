# API-20 フォローする

[API 設計](README.md) / API-20 フォローする

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`PUT /api/users/:username/follow`（R-20）

**概要**: 閲覧者が、対象のユーザーをフォローする（FOL-01）。相手の承認は要らない（FOL-03）。**冪等**。

**リクエスト**: パス `username`。ボディなし。`X-CSRF-Token` が必要。

**レスポンス**: 200

```json
{ "following": true, "follower_count": 57 }
```

| キー | 型 | 説明 |
| --- | --- | --- |
| `following` | boolean | 操作後の状態（常に `true`） |
| `follower_count` | integer | 操作後の、**対象のユーザー**のフォロワー数 |

**エラー**（共通: E1、E2）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 400 | `CANNOT_FOLLOW_SELF` | 対象が閲覧者自身（FOL-01） |
| 404 | `USER_NOT_FOUND` | 存在しない、仮登録のまま、形式不正 |

**備考**: 使用テーブル: `follows`、`users`。`INSERT ... ON DUPLICATE KEY UPDATE`（db.md 4.4）。自分自身の検査はアプリで先に行い 400 を返す（DB の `CHECK` は最後の防御）。すでにフォロー済みなら `created_at` を更新せず成功とする。
