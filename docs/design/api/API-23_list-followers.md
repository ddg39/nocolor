# API-23 フォロワー一覧

[API 設計](README.md) / API-23 フォロワー一覧

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`GET /api/users/:username/followers`（R-23）

**概要**: そのユーザーをフォローしているユーザーを、フォローされた順（新しい順）に返す（PRF-05）。

**リクエスト・レスポンス・エラー**: API-22 と同じ（`items` は UserSummary）。

**備考**: 使用テーブル: `follows`、`users`（`ListFollowers`）。インデックスは `idx_follows_followee_created`。
