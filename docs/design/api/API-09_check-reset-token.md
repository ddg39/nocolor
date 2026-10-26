# API-09 再設定リンクの確認

[API 設計](README.md) / API-09 再設定リンクの確認

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`POST /api/auth/password/reset/check`（R-09）

**概要**: 再設定リンクのトークンが有効かを確認する。**トークンを消費しない**（読み取りのみ）。入力後に期限切れが分かるのを避ける（screens D-07）。

**リクエスト**: JSON

| 項目 | 型 | 必須 | 規則 |
| --- | --- | --- | --- |
| `token` | string | 要 | 43 文字の Base64URL。形式違反は DB を引かず `TOKEN_INVALID` |

トークンを URL ではなくボディで送るのは、URL がアクセスログやリファラに残るのを避けるためである（読み取りだが POST にした）。

**レスポンス**: 200

```json
{ "valid": true }
```

**エラー**（共通: E1、E3、E4）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 400 | `TOKEN_INVALID` | 存在しない、期限切れ（1 時間）、使用済み、形式不正。区別しない |
| 429 | `RATE_LIMITED` | `token_ip` |

**備考**: 使用テーブル: `password_reset_tokens`、`users`、`rate_limits`。副作用なし（`used_at` も更新しない）。
