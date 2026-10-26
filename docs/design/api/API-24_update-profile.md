# API-24 プロフィールの更新

[API 設計](README.md) / API-24 プロフィールの更新

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`PUT /api/me/profile`（R-24）

**概要**: 自分の表示名・自己紹介・アイコンを 1 回の呼び出しで更新する（PRF-03、PRF-04）。**テキストとアイコンを原子的に更新し**、途中まで成功した状態を作らない。

**リクエスト**: `multipart/form-data`。`X-CSRF-Token` が必要。

| 項目 | 型 | 必須 | 規則 |
| --- | --- | --- | --- |
| `display_name` | text | 要 | 1.9 の表示名（1〜30 文字） |
| `bio` | text | 要 | 1.9 の自己紹介（0〜160 文字）。空文字は「空にする」の意味 |
| `icon_action` | text | 任意 | `keep`（既定）/ `replace` / `remove`。それ以外は `fields.icon_action` = `INVALID_VALUE` |
| `icon` | file | `icon_action` が `replace` のとき要 | API-13 の `image` と同じ規則（形式、サイズ `max_image_bytes` 以下、アニメーション不可、白黒、解像度の安全弁）。`keep`・`remove` のときに付いていたら `fields.icon` = `UNEXPECTED`、`replace` で無いなら `REQUIRED` |

PUT なので、表示名と自己紹介は変更がなくても常に送る（現在の値を API-18 で取得してフォームに入れているため、画面にとって負担はない）。

**レスポンス**: 200。本文は更新後の Profile（API-18 と同じ形。`is_following` は `false`）。

**エラー**（共通: E1、E2、E5）

| HTTP | コード | 条件 |
| --- | --- | --- |
| 400 | `VALIDATION_ERROR` | `fields` に `display_name` / `bio` / `icon` / `icon_action` の項目別コード |
| 400 | `NOT_GRAYSCALE` | アイコンが白黒でない（PRF-04） |
| 400 | `INVALID_IMAGE` | アイコンが壊れている |
| 400 | `ANIMATED_IMAGE` | アイコンがアニメーション |
| 400 | `IMAGE_RESOLUTION_TOO_LARGE` | アイコンの寸法が、いずれかの辺で 65535px 超、または総画素数 1 億超（固定の安全弁。API-13 の処理順 (4)）。要件 8 章の上限が決まったら、それを超える場合も同じ |
| 413 | `IMAGE_TOO_LARGE` | アイコンのサイズ超過 |
| 415 | `UNSUPPORTED_IMAGE_TYPE` | アイコンの中身が JPEG・PNG・WebP でない |
| 429 | `RATE_LIMITED` | `profile_user` |
| 503 | `SERVICE_UNAVAILABLE` | MinIO に保存できない |

**備考**:
- 使用テーブル: `users`、`object_deletions`、`rate_limits`。
- **アイコンの画像処理**（`replace` のとき。サーバーで縮小して保存する。設計判断 D-27）: API-13 の処理順 (2)〜(8)（サイズ・形式・解像度の安全弁・アニメーション・復号・白黒判定・メタデータの除去と向きの焼き込み）を行ったあと、**長辺 256px に縮小**する。縦横比を保ち、切り抜きはしない。元の長辺が 256px 以下なら拡大せずそのままの寸法にする。**元画像は保存しない**（MinIO に置くのは縮小後の 1 枚だけ）。縮小は画素の補間だけで色の変換はしないので、要件 2.2（バックエンドでの白黒変換をしない）に抵触しない。縮小後の形式はサムネイルと同じで、要件 8 章の未決（Q-3）に従う。保存後の寸法は DB に持たず、`icon_url` だけを返す（db.md の `avatar_key` のみ）。
- 処理: テキストと画像のすべての検証を先に終える。`replace` のときは新しいキー（`avatars/{UUIDv4}.{拡張子}`。ADR 001 のとおり同じキーを上書きしない）に、上の縮小後の画像を MinIO に保存し、**1 つのトランザクション**で `users` の `display_name`・`bio`・`avatar_key` を更新して、旧キーを `object_deletions` に追加する。`remove` のときは `avatar_key` を `NULL` にして旧キーを予約する。DB の更新に失敗したら、新しく保存したキーを削除する。検証に 1 つでも失敗したら、何も変更しない。
- アイコンが未設定のときの `remove` は何もせず成功する。
- 副作用: MinIO への保存・削除（旧画像の削除は予約方式。db.md 5 章）。
- ユーザー名・メールアドレスは変更できない（4.1、2.2）。
