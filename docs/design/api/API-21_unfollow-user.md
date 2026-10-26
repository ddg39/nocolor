# API-21 フォローを解除する

[API 設計](README.md) / API-21 フォローを解除する

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`DELETE /api/users/:username/follow`（R-21）

**概要**: フォローを解除する（FOL-02）。**冪等**。

**リクエスト**: パス `username`。`X-CSRF-Token` が必要。

**レスポンス**: 200

```json
{ "following": false, "follower_count": 56 }
```

**エラー**（共通: E1、E2）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 404 | `USER_NOT_FOUND` | 存在しない、仮登録のまま、形式不正 |

**備考**: 使用テーブル: `follows`、`users`。影響行数が 0 でも成功とする。自分自身を指定しても、フォローが存在しないので成功する（400 にはしない）。
