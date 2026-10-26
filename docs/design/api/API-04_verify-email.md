# API-04 メールアドレスの確認

[API 設計](README.md) / API-04 メールアドレスの確認

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`POST /api/auth/verify`（R-04）

**概要**: 確認メールのリンクのトークンで本登録を完了する（ACC-02、ACC-03）。メールの先読みでトークンが消費されないよう、GET にしない（screens D-06）。

**リクエスト**: JSON

| 項目 | 型 | 必須 | 規則 |
| --- | --- | --- | --- |
| `token` | string | 要 | 43 文字の Base64URL（`^[A-Za-z0-9_-]{43}$`）。形式が違えば DB を引かず `TOKEN_INVALID` |

**レスポンス**: 200

```json
{ "verified": true }
```

**エラー**（共通: E1、E3、E4）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 400 | `TOKEN_INVALID` | トークンが存在しない、期限切れ（24 時間）、形式が不正。区別しない |
| 409 | `ALREADY_VERIFIED` | すでに本登録済みのユーザーのトークン（使用済みトークンを残す。D-8） |
| 429 | `RATE_LIMITED` | `token_ip` |

- 期限切れと不正を `TOKEN_INVALID` にまとめる。画面設計 SCR-04 のとおり利用者に区別して見せず、DB の期限切れ削除でトークンの行が先に消えている場合とも区別できないため。画面設計で想定していた `TOKEN_EXPIRED` は返さない。

**備考**:
- 使用テーブル: `email_verification_tokens`、`users`、`rate_limits`。
- 副作用: `users.email_verified_at` を現在時刻にする（`email_verified_at IS NULL` のときだけ更新。並行して 2 回呼ばれても 1 回だけ成功する）。**セッションは作らない**（トークン付き URL でログイン状態にしないため。screens 3.4.3）。
- 本登録が完了しても確認トークンは削除せず、期限切れの定期削除に任せる（`ALREADY_VERIFIED` を返せるようにするため。D-8）。
