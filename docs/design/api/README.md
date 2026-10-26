# API 設計

[要件定義書](../../requirements.md)、[ADR 001](../../adr/001_delivery-architecture.md)、[画面設計](../screens/README.md)、[DB 設計](../db.md) をもとにした REST API の設計である。実装（Echo のハンドラ、sqlc のクエリ）はこの文書にもとづいて別途作る。

**API の仕様の正本は OpenAPI 3.1.0（[openapi.yaml](openapi.yaml)）である。** この README には、OpenAPI で表せないもの（処理の共通規則、CSRF・レート制限などの運用、カーソルの意味、環境変数、メール送信キュー、画面操作との対応、設計判断、未決事項）だけを残す。章・節の番号（1.1〜1.11、2〜6）は、画面設計・DB 設計から参照されているので維持する。内容が OpenAPI に移った節も、見出しを残して移動先を書く。

- 画面設計の付録 A「API 担当への要求」R-01〜R-24 に 1 対 1 で対応させ、**API ID の番号を R 番号と同じにした**（API-07 は R-07）。
- 画面設計・DB 設計への変更は、この文書では反映済みの前提として書く。

## OpenAPI の書き方の規約

OpenAPI を書く・直す担当者は、次の規約に従う。見本は [API-01](paths/API-01_get-me.yaml)（ログイン任意の参照系）、[API-03](paths/API-03_signup.yaml)（ログイン不要の変更系・JSON）、[API-13](paths/API-13_create-post.yaml)（ログイン必須の変更系・multipart）である。

### ファイル構成

```
docs/design/api/
├── openapi.yaml                # ルート（info・servers・tags・paths・components の列挙）
├── redocly.yaml                # lint 設定（extends: recommended。ルールを緩める場合は理由をコメントで書く）
├── README.md                   # OpenAPI で表せない共通規則・一覧・設計判断・未決事項
├── paths/API-xx_<slug>.yaml    # 1 ファイル = 1 Operation Object
└── components/
    ├── schemas/<Name>.yaml     # 1 ファイル 1 スキーマ
    ├── responses/<Name>.yaml   # 共通のエラー応答
    └── parameters/<Name>.yaml  # 共通のパラメータ
```

- 1 API 1 ファイルとする。OpenAPI は同じパスの複数メソッドを 1 つの Path Item に書く必要があるため、ルートの `paths` で、メソッドごとに Operation を `$ref: ./paths/API-xx_<slug>.yaml` で参照する。
- `components` の各要素（`schemas`・`responses`・`parameters`）は、ルートの `components` にファイルへの `$ref` で列挙する。バンドル時に名前付きの型として残るようにするためである。新しい共通部品を足したら、ルートの列挙にも足す。
- `paths/` のファイルの slug は、旧 Markdown と同じ（例: `API-13_create-post.yaml`）。
- コード生成は、`npx -y @redocly/cli@2 bundle docs/design/api/openapi.yaml --config docs/design/api/redocly.yaml -o <出力先>` で作った 1 ファイルを使って行う。lint は `npx -y @redocly/cli@2 lint docs/design/api/openapi.yaml --config docs/design/api/redocly.yaml`（リポジトリのルートで実行。`--config` を省くと `redocly.yaml` が読まれず、無効にしたルールの警告が出る）。

### 書き方

- `servers: [{url: /}]`。`paths` は `/api/...` から始まる完全なパスで書く（Echo のルートと一致させるため）。Echo の `:post_id` は `{post_id}` と書く。
- 各 Operation には次を書く。
  - `operationId`: lowerCamelCase（例: `createPost`）。
  - `summary`: `API-13 投稿の作成` の形。`x-api-id: API-13` も付ける。
  - `tags`: ルートの `tags`（Me・Config・Auth・Timelines・Posts・Users）から選ぶ。
  - `description`: Markdown。概要、処理順、備考、関連要件 ID、利用画面 ID、使用する DB クエリ名を、旧 Markdown から落とさずに書く。
  - `security`、`parameters`、`requestBody`、`responses`。
  - `responses`: 成功例とエラーを網羅する。エラーはステータスごとに 1 つの応答とし、起こりうる `code` を description に列挙する。共通の応答で足りるものは `components/responses` を `$ref` し、API 固有の `code` を含むものはその場に書く。ただし 429（`RateLimited`）は常に `$ref` する（`Retry-After` ヘッダーを持つため）。500 は全 API に付ける。
- 認証は `components.securitySchemes` の `sessionCookie`（apiKey、cookie、`nocolor_session`）と `csrfToken`（apiKey、header、`X-CSRF-Token`）を使う。

  | API の種類 | `security` |
  | --- | --- |
  | ログイン必須の参照系 | `[{sessionCookie: []}]` |
  | ログイン必須の変更系 | `[{sessionCookie: [], csrfToken: []}]` |
  | ログイン任意 | `[{}, {sessionCookie: []}]` |
  | ログイン不要 | `[]` |

  Origin の検査（1.4 (1)）は `security` では表せないので、各 Operation の description と README で説明する。
- OpenAPI 3.1 の書き方を使う。null を許す項目は `type: [string, "null"]`（`nullable` は使わない）。`examples` は Media Type の `examples`、または Schema の `examples`（配列）で書く。
- 入力規則（文字数、文字種、形式、サイズ）は `minLength`・`maxLength`・`pattern`・`format`・`minimum` などで表す。表せない規則（コードポイントで数える、制御文字の禁止、前処理、項目別コードの判定順など）は `description` に書く。`format`・`minLength`・`maxLength` など前処理後の形を前提にする制約は、前処理（前後の空白の除去、小文字化、改行の正規化など）の前の値には付けない（上限は description に「前処理後に N 文字」と書く。`minLength: 1` は付けてよい）。
- `pattern` は Go の `regexp`（RE2）と ECMA-262 の両方で使える書き方に限る（先読み・後読み・後方参照は使わない）。表せない規則（例: パスワードの「英字と数字を 1 文字以上ずつ含む」）は `pattern` に書かず、description に書く。
- スキーマによる自動の要求検証（`OapiRequestValidator` など）は使わない。検証は 1.5・1.9 の順序と項目別コードで実装が行う。パラメータの束縛エラーは 1.6 の形式・コードに変換する（パスの形式不正は 404 `POST_NOT_FOUND` / `USER_NOT_FOUND`、`limit` の不正は 400 `VALIDATION_ERROR`（`fields.limit` = `INVALID_VALUE`）、型の不一致は 400 `BAD_REQUEST` など）。
- パスパラメータの形式が不正なときは、400 ではなく 404（`POST_NOT_FOUND` / `USER_NOT_FOUND`）を返す（1.1）。スキーマの `pattern` は形式を示すためのもので、不一致の応答は 404 である。description に書いておく。
- エラーコードは `components/schemas/ErrorCode.yaml` の `enum` にすべて列挙し、各値の意味を description に書く。項目別コードは `FieldErrorCode.yaml`。
- 画像ファイルを受ける multipart の項目は `type: string` と `format: binary` で書き、`encoding` で許可する形式（`image/jpeg, image/png, image/webp`）を示す。
- 日時は `type: string` と `format: date-time`。ID は `PostId`、ユーザー名は `Username` のスキーマを使う。画像 URL は `/images/` で始まる相対パス（`pattern: '^/images/'`）。

### 共通部品

| 種類 | 名前 | 用途 |
| --- | --- | --- |
| schemas | `Error` | エラー応答の本文（`{error: {code, message, fields, request_id}}`） |
| schemas | `ErrorCode` | エラーコードの列挙と意味 |
| schemas | `FieldErrorCode` | 項目別コード（`fields` の値）の列挙と意味 |
| schemas | `PostId` | 投稿 ID（ULID） |
| schemas | `Username` | ユーザー名 |
| schemas | `UserBrief` | 投稿の `author`、API-01・06 の `user` |
| schemas | `Post` | 投稿 |
| schemas | `UserSummary` | フォロー／フォロワー一覧の項目 |
| schemas | `Profile` | プロフィール（`UserSummary` ＋ 件数） |
| schemas | `LikeState` | いいねの操作後の状態（API-16・17。`liked` と `like_count`） |
| schemas | `FollowState` | フォローの操作後の状態（API-20・21。`following` と `follower_count`） |
| schemas | `NextCursor` | `next_cursor`（`string \| null`） |
| schemas | `PostPage` | 投稿の一覧の応答（`items: Post[]` ＋ `next_cursor`） |
| schemas | `UserSummaryPage` | ユーザーの一覧の応答（`items: UserSummary[]` ＋ `next_cursor`） |
| parameters | `cursor`・`limit` | 一覧のクエリ（1.7） |
| parameters | `post_id`・`username` | パスパラメータ |
| responses | `Unauthenticated` | 401 `UNAUTHENTICATED` |
| responses | `CsrfInvalid` | 403 `CSRF_INVALID` |
| responses | `RateLimited` | 429 `RATE_LIMITED`（`Retry-After` ヘッダー付き） |
| responses | `ValidationError` | 400 `VALIDATION_ERROR` / `BAD_REQUEST` |
| responses | `InvalidPagingParams` | 400 `INVALID_CURSOR` / `VALIDATION_ERROR`（`limit`）。一覧 API 用 |
| responses | `PostNotFound`・`UserNotFound` | 404 `POST_NOT_FOUND` / `USER_NOT_FOUND` |
| responses | `RequestTooLarge` | 413 `REQUEST_TOO_LARGE`（JSON ボディ） |
| responses | `UnsupportedMediaType` | 415 `UNSUPPORTED_MEDIA_TYPE`（JSON を受ける API） |
| responses | `InternalError` | 500 `INTERNAL_ERROR` |
| responses | `ServiceUnavailable` | 503 `SERVICE_UNAVAILABLE`（API 固有の条件が無いもの） |
| responses | `ServiceUnavailableQueueFull` | 503 `SERVICE_UNAVAILABLE`（メール送信キュー満杯。`Retry-After` 付き。API-03・05・08） |
| responses | `TokenInvalid` | 400 `TOKEN_INVALID` / `BAD_REQUEST`（API-04・09。API-10 は `VALIDATION_ERROR` を含むので個別に定義） |
| responses | `ImageTooLarge` | 413 `IMAGE_TOO_LARGE` / `REQUEST_TOO_LARGE`（API-13・24） |
| responses | `UnsupportedImageType` | 415 `UNSUPPORTED_IMAGE_TYPE` / `UNSUPPORTED_MEDIA_TYPE`（API-13・24） |

## 1. 共通仕様

### 1.1 基本

| 項目 | 仕様 |
| --- | --- |
| ベースパス | `/api`。ADR 001 のとおり nginx が `/api/*` を backend に転送し、Echo のルートも `/api` から始める。画像は `/images/*`（MinIO）で、API の対象外 |
| オリジン | フロントエンドと同一オリジン。CORS は設定しない（`Access-Control-*` を返さない）。クライアントは相対パスで呼ぶ |
| 形式 | リクエスト・レスポンスとも JSON（`Content-Type: application/json`）。画像を含む API-13・API-24 のリクエストだけ `multipart/form-data` |
| 文字コード | UTF-8。不正な UTF-8 を含む JSON は 400 `BAD_REQUEST` |
| キーの命名 | JSON のキーは `snake_case`。クエリパラメータも同じ |
| 省略と null | レスポンスのキーは省略しない。値が無いものは `null`（例: `icon_url`）。リクエストの未知のキーは無視する |
| メソッド | `GET`（取得）、`POST`（作成・操作）、`PUT`（置換・冪等な設定）、`DELETE`。`PATCH` は使わない。定義のないメソッドは 405 `METHOD_NOT_ALLOWED`、定義のないパスは 404 `NOT_FOUND`（いずれも JSON のエラー形式。OpenAPI の各 Operation には載せない） |
| キャッシュ | すべての `/api` の応答に `Cache-Control: no-store` を付ける（閲覧者ごとに内容が変わるため、およびトークンを含むため）。API-02 だけ `public, max-age=300` としてよい |
| 応答ヘッダー | `X-Content-Type-Options: nosniff`、`X-Request-Id`（1.6） |
| リクエストの `Content-Type` | JSON を受ける API に `application/json` 以外が来たら 415 `UNSUPPORTED_MEDIA_TYPE`。API-13・API-24 に `multipart/form-data` 以外が来たら同じく 415 |
| ボディの大きさ | JSON を受ける API は 16KiB まで（超えたら 413 `REQUEST_TOO_LARGE`）。multipart の API は「画像の上限（API-02 の `max_image_bytes`）+ 64KiB」まで。nginx の `client_max_body_size` はそれより少し大きくし、画像の超過はアプリが 413 `IMAGE_TOO_LARGE` で返す。nginx 自身が 413 を返す場合に備え、`/api/` では `error_page 413` で同じ JSON 形式を返す設定にする（インフラ側の作業） |
| ID の入力 | パスの ID・ユーザー名の形式が 1.2 に合わないときは、DB を引かずに 404（`POST_NOT_FOUND` / `USER_NOT_FOUND`）を返す（screens D-25 と同じ扱い） |

### 1.2 ID・日時・画像 URL・文字列

形式は OpenAPI のスキーマ（`PostId`・`Username`・`Post` など）にも書いてある。ここには全文を残す。

| 項目 | 仕様 |
| --- | --- |
| 投稿 ID | ULID の 26 文字（Crockford Base32、大文字）。形式は `^[0-7][0-9A-HJKMNP-TV-Z]{25}$`。応答は常に大文字。入力は小文字も受け付けて大文字に直す。DB は `BINARY(16)` で、変換は backend が行う（db.md D-1）。OpenAPI の `PostId` の `pattern` は、入力で受ける小文字も含めて書いてある |
| ユーザーの識別 | `username`（`^[a-z0-9_]{3,20}$`）。DB の連番 ID は API・URL・カーソルの平文に出さない（db.md D-2） |
| 日時 | RFC 3339 の UTC、ミリ秒付き、末尾 `Z`。例: `2026-10-26T05:30:12.345Z`。DB の `DATETIME(3)` の UTC をそのまま整形する |
| 画像の URL | `/images/{オブジェクトキー}` の相対パス（ADR 001）。例: `/images/posts/3f1c2a9e-7b0d-4c8a-9d52-1e6f0a7b4c11.jpg`。ホスト名は含めない。拡張子は例であり、変換後の形式（要件 8 章・未決）で決まる。アイコンが未設定のときは `null` |
| 画像の寸法 | `image_width`・`image_height`・`thumbnail_width`・`thumbnail_height`（ピクセル、正の整数）。EXIF の向きを反映した後の表示上の幅・高さ（保存時に向きを焼き込むため、保存した画像の寸法と一致する） |
| 文字数 | Unicode のコードポイント数で数える（Go の `rune`。screens D-24）。改行は LF に正規化してから数える（CRLF・CR は LF にする） |
| メールアドレス | どの API の応答にも含めない（API-03 の応答が、入力を正規化して返す場合を除く） |

### 1.3 認証（セッション Cookie）

要件 6.2 のとおり、セッション + Cookie で行う。OpenAPI では `components.securitySchemes.sessionCookie` に対応する。

| 項目 | 仕様 |
| --- | --- |
| Cookie 名 | `nocolor_session` |
| 値 | 32 バイトの乱数を Base64URL（パディングなし、43 文字）にした文字列。DB には SHA-256 を `sessions.id_hash` として保存する（db.md 3.4）。ログインのたびに新しい値を発行し、既存の値は再利用しない（セッション固定化の対策） |
| 属性 | `HttpOnly`、`SameSite=Lax`、`Path=/`、`Max-Age=2592000`（30 日）。`Domain` は付けない（ホスト限定）。リクエストが HTTPS（`X-Forwarded-Proto: https` または TLS 接続）のときは `Secure` を付ける |
| 名前に `__Host-` を付けない理由 | `__Host-` は `Secure` を必須にするため、HTTP の開発環境（`docker compose up` の既定）で Cookie が保存されなくなる |
| 有効期限 | 作成から 30 日の固定。アクセスで延長しない（db.md 3.4）。判定は毎回 `expires_at > now` で行う |
| 破棄 | API-07（ログアウト）で当該セッションを、API-10（パスワード再設定）で当該ユーザーの全セッションを DB から削除する。いずれも応答で同名の Cookie を `Max-Age=0` で削除する |
| 未ログイン時の応答 | ログイン必須の API は 401 `UNAUTHENTICATED`。Cookie が無い、値が不正、期限切れ、のいずれも同じ。Cookie が付いていたのに無効だった場合は、応答で Cookie を削除する |
| ログイン不要の API | Cookie が無効でも 401 にせず、未ログインとして処理する（閲覧者 ID は 0。db.md 1 章）。全体タイムラインなどが、期限切れの Cookie を持つブラウザで失敗しないようにするため |

### 1.4 CSRF 対策

画面設計 3.6.2 の「403 `CSRF_INVALID`」に対応する。次の二段で防ぐ。OpenAPI では (2) が `components.securitySchemes.csrfToken` に対応し、(1) は `security` で表せないので各 Operation の description に書く。

**(1) Origin の検査（状態を変更するすべてのリクエスト）**

- 対象メソッド: `POST`・`PUT`・`PATCH`・`DELETE`。`GET`・`HEAD` は状態を変更しない設計にする（検査しない）。
- `Origin` ヘッダーがあるとき、その値（スキーム・ホスト・ポート）が環境変数 `APP_BASE_URL`（1.10）のスキーム・ホスト・ポートと**完全に一致**しなければ 403 `CSRF_INVALID`。`Origin: null` も拒否する。比較相手を `Host` ヘッダーにしないのは、nginx の `proxy_set_header Host` や TLS 終端でホスト・ポート・スキームが変わりうるためである。既定ポート（http の 80、https の 443）は省略した形に正規化して比べる。
- `Origin` が無いときは `Sec-Fetch-Site` を見て、`same-origin` か `none` 以外なら 403。両方無い場合（ブラウザ以外のクライアント）は通す。次の (2) が別に守る。
- ログイン不要の変更系 API（API-03〜06、08〜10）は (1) だけで守る。根拠は次のとおり。(a) Cookie を持たないリクエストにはトークンを結び付ける相手がなく、Cookie が付かない状態の CSRF では被害が出ない。(b) リクエストは `Content-Type: application/json` が必須（1.1）なので、HTML フォームからは送れない。(c) 他オリジンからの `fetch` で JSON を送るとプリフライトになり、CORS を許可していない（1.1）ので送信されない。(d) 残る実害はログインを強制する攻撃（ログイン CSRF）だけで、(1) で止める。つまり **`Content-Type` が JSON 必須であることも防御の一部**であり、この制約を緩める（`text/plain` を受けるなど）ときは本節を見直すこと。

**(2) CSRF トークン（ログイン必須の変更系 API）**

| 項目 | 仕様 |
| --- | --- |
| 対象 | ログイン必須の `POST`・`PUT`・`DELETE`: API-07、13、15、16、17、20、21、24 |
| 方式 | セッションに結び付いた決定的なトークン。`csrf_token = Base64URL(HMAC-SHA256(csrf_key, Cookie の値))`。`csrf_key` は環境変数 `APP_SECRET` から用途ラベル付きで導出した鍵。DB に保存しない（db.md 3.4 の「セッション ID から HMAC で導出する」案を採用） |
| 取得 | API-01（`GET /api/me`）の応答 `csrf_token` と、API-06（ログイン）の応答 `csrf_token`。未ログインのときは `null` |
| 送信 | リクエストヘッダー `X-CSRF-Token: {csrf_token}`。JSON でも multipart でも同じ |
| 検証 | 定数時間で比較する。ヘッダーが無い、または一致しなければ 403 `CSRF_INVALID` |
| 失効 | セッションが破棄されると導出元が消えるので自動的に無効になる。ログインし直すと値が変わる |
| クライアントの動作 | 起動時の API-01 で保持し、403 `CSRF_INVALID` を受けたら API-01 を取り直して 1 回だけ再試行してよい。それでも 403 なら画面設計 3.6.2 の文言を出す |

トークンを JSON の応答本文で渡す（Cookie に入れない）ので、ダブルサブミットクッキーのように Cookie 値を JavaScript が読む必要がなく、`nocolor_session` を `HttpOnly` のままにできる。

### 1.5 リクエストの処理順

エラーが複数あたるときの優先順を固定する（画面側の分岐を単純にし、情報の漏れを防ぐため）。

| 順 | 処理 | 失敗時 |
| --- | --- | --- |
| 1 | メソッド・パスの解決、パスパラメータの形式 | 405 / 404 |
| 2 | Origin の検査（変更系） | 403 `CSRF_INVALID` |
| 3 | セッションの確認（ログイン必須の API） | 401 `UNAUTHENTICATED` |
| 4 | CSRF トークンの検査（ログイン必須の変更系） | 403 `CSRF_INVALID` |
| 5 | レート制限のうち、**キーがリクエストの外にあるスコープ**（IP 単位: `*_ip`。ユーザー単位: `post_user`・`profile_user`。1.8。対象の API のみ） | 429 `RATE_LIMITED` |
| 6 | `Content-Type`・ボディの大きさ・構文（ボディの解析） | 415 / 413 / 400 |
| 7 | 入力の検証（形式・文字数など。DB を引かない検査） | 400 `VALIDATION_ERROR` ほか |
| 8 | レート制限のうち、**キーがボディの値から得られるスコープ**（`login_account`・`signup_mail_addr`・`reset_mail_addr`） | 429 `RATE_LIMITED` |
| 9 | 業務処理（DB の検索、存在・権限・重複など） | 404 / 403 / 409 ほか |

- ログイン不要の API は 3・4 を飛ばす。
- 5 と 8 を分けるのは、アカウント・アドレス単位のキーがボディを解析して正規化した後でないと得られないためである。8 は 9 の DB 検索（パスワードの照合やアカウントの検索）より前に行い、制限中は存在の有無に関する情報を返さない。
- **形式不正の入力の数え方**: 6・7 で弾かれたリクエストは、IP 単位・ユーザー単位のスコープ（5）では 1 回と数えるが、アカウント・アドレス単位のスコープ（8）では数えない（キーが得られない、または有効なアドレス・識別子でないため）。アドレス単位の枠は、形式が正しい入力だけを数える。
- **例外（API-10）**: `token` の形式と有効性の確認（読み取りのみ。トークンは消費しない）を、7 のうち `password` の検証より**先に**行う。トークンが無効なら `password` が不正でも `TOKEN_INVALID` を返す（API-10 の処理順）。

### 1.6 エラー応答

エラー応答の形式、項目別コード、エラーコードの一覧は OpenAPI に移した。

| 内容 | 移動先 |
| --- | --- |
| 形式（`error.code`・`message`・`fields`・`request_id`、`X-Request-Id` ヘッダー） | [components/schemas/Error.yaml](components/schemas/Error.yaml) |
| 項目別コード（`fields` の値） | [components/schemas/FieldErrorCode.yaml](components/schemas/FieldErrorCode.yaml) |
| エラーコード一覧（HTTP ステータス・発生条件・主な API） | [components/schemas/ErrorCode.yaml](components/schemas/ErrorCode.yaml) |
| 共通のエラー応答 | [components/responses/](components/responses/) |

- 403 は `CSRF_INVALID`・`EMAIL_NOT_VERIFIED`・`FORBIDDEN` の 3 種類があるので、画面は HTTP ステータスではなく `code` で分岐する。
- 401 も同様に `UNAUTHENTICATED`（セッション切れ）と `INVALID_CREDENTIALS`（ログイン失敗）を `code` で分ける。

#### 各 API の「共通エラー」

旧 Markdown の各 API は、次の略称で共通のエラーを省略していた。OpenAPI の各 Operation は、該当するエラーを `responses` に明記する（定義のないパス・メソッドに対する 404・405 を除く。これらは個別の Operation には載せず、1.1 の規則とする）。

| 略称 | 内容 |
| --- | --- |
| E1（全 API） | 404 `NOT_FOUND` / 405 `METHOD_NOT_ALLOWED`（パス・メソッド違い）、500 `INTERNAL_ERROR`、503 `SERVICE_UNAVAILABLE` |
| E2（ログイン必須の変更系） | 401 `UNAUTHENTICATED`、403 `CSRF_INVALID`（Origin またはトークン） |
| E3（変更系） | 403 `CSRF_INVALID`（Origin の検査） |
| E4（JSON を受ける API） | 400 `BAD_REQUEST`、413 `REQUEST_TOO_LARGE`、415 `UNSUPPORTED_MEDIA_TYPE` |
| E5（multipart を受ける API） | 400 `BAD_REQUEST`、413 `REQUEST_TOO_LARGE`、415 `UNSUPPORTED_MEDIA_TYPE` |

### 1.7 ページング（カーソル方式）

TML-05 の「読み込み中に新しい投稿が増えても重複・欠落しない」を満たすため、`OFFSET` を使わないカーソル方式にする。対象: API-11、12、19（投稿）、API-22、23（ユーザー）。

- リクエストのパラメータ（`cursor`・`limit`）は [components/parameters/cursor.yaml](components/parameters/cursor.yaml)・[limit.yaml](components/parameters/limit.yaml)。`limit` は 1〜50、既定 20、範囲外や整数でない値は 400 `VALIDATION_ERROR`（`fields.limit` = `INVALID_VALUE`）。画面は指定せず既定の 20 件を使う（6.3）。
- レスポンスの共通形（`items` と `next_cursor`）は [PostPage.yaml](components/schemas/PostPage.yaml)・[UserSummaryPage.yaml](components/schemas/UserSummaryPage.yaml)・[NextCursor.yaml](components/schemas/NextCursor.yaml)。
- **終端**: 次のページが無いときは `next_cursor` が `null`。0 件のときは `items: []` かつ `next_cursor: null`。
- 次のページの有無は、`limit + 1` 件を取得して判定する（db.md 4.1）。`limit` 件ちょうどで終わる場合も、余りの 1 件が無ければ `null` になる（空のページを 1 回余計に返さない）。`next_cursor` は返した最後の項目から作る。

#### カーソルの中身

`next_cursor` は**クライアントにとって不透明な文字列**で、中身を解釈・加工してはならない。

| 項目 | 仕様 |
| --- | --- |
| 形式 | `AES-256-GCM` で暗号化した JSON を Base64URL（パディングなし）にしたもの。鍵は `APP_SECRET` から用途ラベル付きで導出する。先頭に 1 バイトの版を持つ |
| 平文の中身（投稿の一覧: API-11、12、19） | 一覧の種類、最後の項目の投稿 ID（ULID）。API-19 は対象ユーザーも含む |
| 平文の中身（ユーザーの一覧: API-22、23） | 一覧の種類、対象ユーザー、最後の項目の `follows.created_at`（ミリ秒）と相手ユーザーの内部 ID |
| 暗号化する理由 | ユーザーの内部連番 ID を API の外に出さない方針（db.md D-2）を、カーソルでも守るため。同時に改ざんを検出でき、別の一覧のカーソルの流用を拒否できる |
| 検証 | 復号・認証に失敗、版が未知、一覧の種類や対象ユーザーが一致しない場合は 400 `INVALID_CURSOR`。有効期限は設けない |
| 閲覧者 | カーソルは閲覧者に依存しない（誰が使っても同じ位置を指す） |

#### 重複・欠落しない理由

- **投稿の一覧（API-11、12、19）**: 並びのキーは `posts.id`（ULID）だけで、降順である。キーは一意で、作成後に変わらない。次ページは `WHERE id < {カーソルの投稿 ID} ORDER BY id DESC LIMIT limit + 1` で取る。読み込み中に投稿が増えても、新しい投稿は常にカーソルより大きい側（先頭側）に入るので、すでに読んだ範囲と次に読む範囲の境界は動かない。投稿が削除されても、残りの投稿の順序は変わらない。`created_at` は並び順に使わない。
- **ユーザーの一覧（API-22、23）**: キーは `(follows.created_at, 相手ユーザー ID)` の降順で、次ページは db.md 4.4 と同じ展開形 `created_at < カーソルの時刻 OR (created_at = カーソルの時刻 AND 相手ユーザー ID < カーソルの ID)` で取る（行比較は使わない）。`created_at` が同じ行があっても相手 ID で全順序が決まる。読み込み中の新しいフォローは先頭側に入る。解除して再度フォローすると行が先頭側へ移るだけで、同じ一覧の中で二重には現れない。
- **フォロー中タイムライン（API-12）の補足**: 閲覧中にフォロー関係が変わると（新たにフォローした）、その人の古い投稿が次ページ以降に入る。これは重複ではなく、フォロー先の集合が変わったことによる意図した結果である。
- **残るリスク**: ULID の生成から `INSERT` の完了までの数ミリ秒の間に、後から生成された投稿が先に完了すると、その投稿は読み込み中のクライアントに見えないことがある（再読み込みで見える）。db.md 4.1 のとおり許容する。

### 1.8 レート制限（要件 6.2）

「ログインと各種メール送信には試行回数の制限を設ける」を実現する。ストアは MySQL の `rate_limits`（db.md 3.8・D-4）で、固定ウィンドウ方式である。超過時の応答（429 `RATE_LIMITED`、`Retry-After`）は [components/responses/RateLimited.yaml](components/responses/RateLimited.yaml)。

| スコープ | 数える対象（キー） | 上限 | ウィンドウ | 適用する API |
| --- | --- | --- | --- | --- |
| `login_ip` | クライアント IP | 20 回 | 15 分 | API-06 |
| `login_account` | `identifier`（前後の空白を除き小文字にした値） | 5 回 | 15 分 | API-06 |
| `signup_mail_ip` | クライアント IP | 10 回 | 1 時間 | API-03、API-05 |
| `signup_mail_addr` | 正規化したメールアドレス | 3 回 | 1 時間 | API-03、API-05 |
| `reset_mail_ip` | クライアント IP | 10 回 | 1 時間 | API-08 |
| `reset_mail_addr` | 正規化したメールアドレス | 3 回 | 1 時間 | API-08 |
| `token_ip` | クライアント IP | 30 回 | 15 分 | API-04、09、10 |
| `post_user` | ログイン中のユーザー | 30 回 | 1 時間 | API-13 |
| `profile_user` | ログイン中のユーザー | 30 回 | 1 時間 | API-24 |

- 上限とウィンドウ幅は環境変数で変えられる（1.10）。値は db.md Q-6 の例を基本にし、メール系以外を追加した。
- **数え方**: 対象の API が呼ばれるたびに、結果（成功・失敗、アカウントの有無）にかかわらず 1 回と数える。1 つのリクエストで複数のスコープを数え、どれか 1 つでも上限を超えたら 429 にする。キーはハッシュにして保存する（db.md 3.8）。
- **ACC-09 との関係**: メールアドレス単位の制限は、そのアドレスが登録済みかどうかと無関係に数える。したがって 429 になる条件からアカウントの存在は推測できない。制限中は、パスワードの照合やアカウントの検索を行わずに 429 を返す。
- **クライアント IP**: nginx が `X-Real-IP` に接続元アドレスを入れ、backend（Echo の `IPExtractor`）はそのヘッダーだけを信頼する。クライアントが付けた `X-Forwarded-For` を鵜呑みにしない。nginx の前段にロードバランサーなどを置く場合は、nginx 側で real IP の復元（`set_real_ip_from`）を設定する（インフラ側の作業）。IPv6 は上位 64 ビットで数える。
- **超過時の応答**: 429 `RATE_LIMITED` に、秒数の整数を `Retry-After` ヘッダーで付ける（現在のウィンドウの終わりまでの秒数。複数のスコープが超過したときは最大のもの。最小 1）。画面はこれを「約 N 分後に」と表示し、ボタンを無効にする。
- 欠点の許容: 固定ウィンドウなので境界をまたぐと最大で上限の 2 倍まで通る。`login_account` と `*_mail_addr` は、第三者が他人のアカウント（アドレス）の枠を使い切らせることができる。ブルートフォースの防止を優先して許容する（設計判断 D-12）。
- 上記以外の API（いいね、フォロー、一覧など）には、MVP では制限を設けない。

### 1.9 入力検証の共通規則

要件定義書 4 章の入力規則と一致させる。文字数はコードポイントで数える。文字数・文字種・形式は OpenAPI のスキーマ（`minLength`・`maxLength`・`pattern`）にも書いてあるが、前処理と項目別コードの判定順は表せないので、ここに全文を残す。

| 項目 | 規則 | 前処理 | 項目別コード |
| --- | --- | --- | --- |
| ユーザー名 `username`（4.1） | 3〜20 文字、`[a-z0-9_]` のみ。一意 | 加工しない（大文字や空白はエラー）。登録後は変更不可 | 空: `REQUIRED`。範囲外の文字: `INVALID_CHARS`。短い: `TOO_SHORT`。長い: `TOO_LONG`。この順に判定する |
| メールアドレス `email`（4.1） | 形式チェック。254 文字以内。一意（`@` 以降も含め、大文字小文字を区別しない） | 前後の空白を除き、小文字にして保存・照合する（db.md 3.1） | 空: `REQUIRED`。長い: `TOO_LONG`。形式: `INVALID_FORMAT` |
| パスワード `password`（4.1） | 8〜32 文字。英字（A-Z、a-z）と数字（0-9）をそれぞれ 1 文字以上含む | 加工しない（空白も含めて保存対象）。Unicode の正規化もしない | 空: `REQUIRED`。短い: `TOO_SHORT`。長い: `TOO_LONG`。文字種: `NEED_LETTER_AND_DIGIT` |
| 表示名 `display_name`（PRF-03） | 1〜30 文字 | 前後の空白（Unicode の空白）を除いてから数える。改行・制御文字は不可 | 空（空白のみを含む）: `REQUIRED`。長い: `TOO_LONG`。制御文字: `INVALID_CHARS` |
| 自己紹介 `bio`（PRF-03） | 0〜160 文字 | 改行は LF に正規化。改行は保持する。前後の空白は除かない | 長い: `TOO_LONG`。LF 以外の制御文字: `INVALID_CHARS` |
| キャプション `caption`（PST-01） | 0〜500 文字。未指定は空文字 | bio と同じ | bio と同じ |

- **メールアドレスの形式**: `local@domain` の形で、表示名やコメントを含まず、`@` の後ろ（ドメイン部）に `.` を含み、ローカル部は 64 文字以内。実装は Go の `net/mail` で解析し、解析結果のアドレス部が入力と一致することを確認すればよい（MVP では国際化メールアドレスは考慮しない）。
- **制御文字**: Unicode の制御文字（U+0000〜U+001F、U+007F〜U+009F）。ただし bio・caption は LF（U+000A）とタブ（U+0009）を許す。
- ログインの `identifier` と `password` には、形式・文字数の規則を適用しない（画面設計 SCR-05 のとおり。規則を教えないため）。
- **bcrypt の 72 バイト制限**: 32 文字でも UTF-8 では 72 バイトを超えうる。bcrypt が黙って切り詰めると、先頭 72 バイトが同じ別のパスワードでもログインできてしまう。実装は、SHA-256 を Base64 にした値を bcrypt に渡すなど、72 バイトの制限に影響されない方法を取ること（設計判断 D-15）。

### 1.10 前提とする環境変数（案）

要件 6.1 のとおり、値は環境変数で与え、`.env.example` に載せる。名前は案である。

| 変数 | 用途 | 既定値（案） |
| --- | --- | --- |
| `APP_BASE_URL` | メール内のリンクの作成（`{APP_BASE_URL}/verify?token=...`、`/password/reset?token=...`） | 開発: `http://localhost` |
| `APP_SECRET` | CSRF トークンの HMAC 鍵、カーソルの暗号鍵の元になる秘密。32 バイト以上のランダムな値。リポジトリに含めない | 開発用の例を `.env.example` に |
| `MAX_IMAGE_BYTES` | 画像の最大サイズ（PST-02） | 10485760（10MB） |
| `MAX_IMAGE_SIDE`・`MAX_IMAGE_PIXELS` | 画像の最大解像度（要件 8 章・未決。未設定は、下の固定の安全弁だけが働く） | 未設定 |
| （固定値）65535px・1 億画素 | 実装上の安全弁。ヘッダーで読んだ寸法が各辺 65535px 以下かつ総画素数 1 億以下であること。要件 8 章の未決とは別で、環境変数では変えない。`MAX_IMAGE_SIDE`・`MAX_IMAGE_PIXELS` が設定されたら、より厳しい側が効く | 固定 |
| `GRAYSCALE_TOLERANCE` | 白黒判定の許容値（要件 8 章・未決） | 実装時の検証で決める |
| `RATE_LIMIT_{スコープ}_MAX`・`RATE_LIMIT_{スコープ}_WINDOW` | 1.8 の各スコープの上限とウィンドウ幅 | 1.8 の表 |
| `MAIL_QUEUE_SIZE`・`MAIL_WORKERS` | メール送信キューの容量とワーカー数（1.11） | 100・2 |
| `IMAGE_MEMORY_BUDGET` | 画像の復号・変換で同時に使うメモリの見積もりの上限（API-13 の実装の注意） | 2GB |

### 1.11 メール送信キュー（非同期ジョブ）

API-03・05・08 の「応答後の非同期処理」の実体を次のとおり定める。

| 項目 | 仕様 |
| --- | --- |
| 実体 | backend プロセス内の**容量付きキュー**（Go の buffered channel 相当）と、それを読む固定数のワーカー。外部のキューやジョブ基盤は使わない |
| 容量・ワーカー数 | 既定は容量 100・ワーカー 2（`MAIL_QUEUE_SIZE`・`MAIL_WORKERS`） |
| 積む内容 | ジョブの種類と、正規化したメールアドレス。API-05・08 はアドレスの検索以降をワーカーが行う。API-03 は 2 種類ある。(a) 新規登録に成功したとき: トランザクションのコミット後に、送信内容（宛先・本文に入れるトークン）を積む（確認メール）。(b) メールアドレスの重複のとき: 「登録済みメールの処理ジョブ」として正規化したメールアドレスだけを積み、**ワーカーが `GetUserByEmail`（有効な仮登録を含む）で相手を引いて**、本登録済みなら案内メール、有効な仮登録なら旧トークンの削除・新トークンの発行・確認メールの再送、見つからなければ何もしない（db.md 3.1 のワーカーの手順と同じ）。トークンの再発行を含め、これらの DB アクセスとメール送信は同期では行わない |
| 満杯のとき | 待たずに 503 `SERVICE_UNAVAILABLE`。キューの状態はアドレスの登録の有無と無関係なので、ACC-09 に反しない。`Retry-After` は付けてよい（秒数は固定の案: 30） |
| ワーカーの失敗 | SMTP の失敗・DB の失敗は応答に影響しない。ログ（`request_id` 付き）に残し、自動の再試行はしない。利用者は API-05・08 を呼び直す |
| 停止・異常終了 | 正常停止（SIGTERM）では、ワーカーが残りのジョブを一定時間（例: 10 秒）処理してから終了する。異常終了ではキューの内容が失われる。これは設計判断 D-9 のとおり MVP では許容する（利用者は再送できる） |
| 複数プロセス | 前提としない（MVP は backend 1 プロセス）。複数にする場合はキューの共有が必要で、その際に見直す |

ジョブの実行時間はアドレスによって差が出るが、API の応答はキューに積んだ時点で返るので、応答時間には現れない。

## 2. エンドポイント一覧

パスは Echo のルート定義の書式ではなく、OpenAPI の書式（`{名前}` がパスパラメータ）で書く。Echo のルートに書くときは `:post_id` のようにする。`post_id` は投稿 ID（1.2）、`username` はユーザー名である。「ログイン」の列は、「要」がログイン必須（未ログインは 401）、「任意」がログインしていれば閲覧者の状態（`liked_by_me` など）を付ける、「不要」がログインと無関係を表す。「R」の列は screens/README.md 付録 A の要求 ID である。API ID のリンク先は、その API の OpenAPI 定義（Operation Object）である。

| API ID | R | メソッド | パス | ログイン | 関連要件 ID | 利用画面 |
| --- | --- | --- | --- | --- | --- | --- |
| [API-01](paths/API-01_get-me.yaml) | R-01 | GET | `/api/me` | 任意 | 3 章, 6.2 | 全画面（ヘッダー・ガード）。SCR-02, 05, 06 のログイン済み判定 |
| [API-02](paths/API-02_get-config.yaml) | R-02 | GET | `/api/config` | 不要 | PST-02, PRF-04, 6.1 | SCR-08, SCR-12 |
| [API-03](paths/API-03_signup.yaml) | R-03 | POST | `/api/auth/signup` | 不要 | ACC-01, ACC-02, ACC-05, 6.2 | SCR-02 |
| [API-04](paths/API-04_verify-email.yaml) | R-04 | POST | `/api/auth/verify` | 不要 | ACC-02, ACC-03 | SCR-04 |
| [API-05](paths/API-05_resend-verification.yaml) | R-05 | POST | `/api/auth/verify/resend` | 不要 | ACC-03, ACC-04, 6.2 | SCR-03, SCR-04, SCR-05 |
| [API-06](paths/API-06_login.yaml) | R-06 | POST | `/api/auth/login` | 不要 | ACC-04, ACC-06, 6.2 | SCR-05 |
| [API-07](paths/API-07_logout.yaml) | R-07 | POST | `/api/auth/logout` | 要 | ACC-07, 6.2 | 全画面（ヘッダーのメニュー）。主に SCR-01 |
| [API-08](paths/API-08_forgot-password.yaml) | R-08 | POST | `/api/auth/password/forgot` | 不要 | ACC-08, ACC-09, 6.2 | SCR-06 |
| [API-09](paths/API-09_check-reset-token.yaml) | R-09 | POST | `/api/auth/password/reset/check` | 不要 | ACC-08, ACC-09 | SCR-07 |
| [API-10](paths/API-10_reset-password.yaml) | R-10 | POST | `/api/auth/password/reset` | 不要 | ACC-08, 6.2 | SCR-07 |
| [API-11](paths/API-11_global-timeline.yaml) | R-11 | GET | `/api/timelines/global` | 任意 | TML-01, TML-02, TML-04, TML-05, TML-06, LIK-03 | SCR-01 |
| [API-12](paths/API-12_following-timeline.yaml) | R-12 | GET | `/api/timelines/following` | 要 | TML-01, TML-03, TML-05, TML-06, LIK-03 | SCR-01 |
| [API-13](paths/API-13_create-post.yaml) | R-13 | POST | `/api/posts` | 要 | PST-01〜PST-05, PST-08, TML-06, 6.2, 6.3 | SCR-08 |
| [API-14](paths/API-14_get-post.yaml) | R-14 | GET | `/api/posts/{post_id}` | 任意 | PST-07, LIK-03, TML-06 | SCR-09 |
| [API-15](paths/API-15_delete-post.yaml) | R-15 | DELETE | `/api/posts/{post_id}` | 要 | PST-06 | SCR-09 |
| [API-16](paths/API-16_like-post.yaml) | R-16 | PUT | `/api/posts/{post_id}/like` | 要 | LIK-01, LIK-03 | SCR-01, SCR-09 |
| [API-17](paths/API-17_unlike-post.yaml) | R-17 | DELETE | `/api/posts/{post_id}/like` | 要 | LIK-02, LIK-03 | SCR-01, SCR-09 |
| [API-18](paths/API-18_get-profile.yaml) | R-18 | GET | `/api/users/{username}` | 任意 | PRF-01, PRF-02, FOL-01 | SCR-10, SCR-11, SCR-12 |
| [API-19](paths/API-19_list-user-posts.yaml) | R-19 | GET | `/api/users/{username}/posts` | 任意 | PRF-02, TML-05, TML-06, LIK-03 | SCR-10 |
| [API-20](paths/API-20_follow-user.yaml) | R-20 | PUT | `/api/users/{username}/follow` | 要 | FOL-01, FOL-03 | SCR-10, SCR-11 |
| [API-21](paths/API-21_unfollow-user.yaml) | R-21 | DELETE | `/api/users/{username}/follow` | 要 | FOL-02 | SCR-10, SCR-11 |
| [API-22](paths/API-22_list-following.yaml) | R-22 | GET | `/api/users/{username}/following` | 任意 | PRF-05, TML-05 | SCR-11 |
| [API-23](paths/API-23_list-followers.yaml) | R-23 | GET | `/api/users/{username}/followers` | 任意 | PRF-05, TML-05 | SCR-11 |
| [API-24](paths/API-24_update-profile.yaml) | R-24 | PUT | `/api/me/profile` | 要 | PRF-03, PRF-04, PST-02〜PST-05, PST-08 | SCR-12 |

- **仮の案からの変更は 1 点**: `/api/timeline/...` を `/api/timelines/...` に変えた（リソース名は複数形。設計判断 D-2）。R-02 の `GET /api/config`、R-01 の `/api/me` は単数のまま採用した（設定と本人は 1 つしかないため）。
- ログイン状態によって内容が変わる「任意」の API は、Cookie が無効なら未ログインとして扱う（1.3）。

### 共通のオブジェクト

応答に出るオブジェクトの定義は OpenAPI のスキーマに移した（すべての API で同じ形にする）。各項目の「出どころ」（db.md のテーブル・カラム。DB 設計に無い項目は返さない）は、各プロパティの description に書いてある。

| オブジェクト | 用途 | 定義 |
| --- | --- | --- |
| UserBrief | 投稿の `author`、API-01 の `user`、API-06 の `user` | [components/schemas/UserBrief.yaml](components/schemas/UserBrief.yaml) |
| Post | 投稿 | [components/schemas/Post.yaml](components/schemas/Post.yaml) |
| UserSummary | フォロー／フォロワー一覧の項目 | [components/schemas/UserSummary.yaml](components/schemas/UserSummary.yaml) |
| Profile | プロフィール（UserSummary に `post_count`・`following_count`・`follower_count` を加える） | [components/schemas/Profile.yaml](components/schemas/Profile.yaml) |

- 「このユーザーが閲覧者をフォローしているか」（逆向きの関係）は、画面で使わず要件にもないので、MVP では返さない（設計判断 D-24）。
- 仮登録のまま（`email_verified_at` が `NULL`）のユーザーは、どの API でも「存在しない」として扱う（404 `USER_NOT_FOUND`、一覧にも出さない）。

## 3. 各エンドポイントの詳細

各 API の詳細（概要、リクエスト、レスポンス、エラー、備考）は OpenAPI の Operation に書いてある。リンク先のファイルを参照する。

- [API-01 現在のユーザーの取得](paths/API-01_get-me.yaml)
- [API-02 設定値の取得](paths/API-02_get-config.yaml)
- [API-03 仮登録](paths/API-03_signup.yaml)
- [API-04 メールアドレスの確認](paths/API-04_verify-email.yaml)
- [API-05 確認メールの再送](paths/API-05_resend-verification.yaml)
- [API-06 ログイン](paths/API-06_login.yaml)
- [API-07 ログアウト](paths/API-07_logout.yaml)
- [API-08 パスワード再設定メールの申請](paths/API-08_forgot-password.yaml)
- [API-09 再設定リンクの確認](paths/API-09_check-reset-token.yaml)
- [API-10 パスワードの再設定](paths/API-10_reset-password.yaml)
- [API-11 全体タイムライン](paths/API-11_global-timeline.yaml)
- [API-12 フォロー中タイムライン](paths/API-12_following-timeline.yaml)
- [API-13 投稿の作成](paths/API-13_create-post.yaml)
- [API-14 投稿の取得](paths/API-14_get-post.yaml)
- [API-15 投稿の削除](paths/API-15_delete-post.yaml)
- [API-16 いいねを付ける](paths/API-16_like-post.yaml)
- [API-17 いいねを取り消す](paths/API-17_unlike-post.yaml)
- [API-18 プロフィールの取得](paths/API-18_get-profile.yaml)
- [API-19 ユーザーの投稿一覧](paths/API-19_list-user-posts.yaml)
- [API-20 フォローする](paths/API-20_follow-user.yaml)
- [API-21 フォローを解除する](paths/API-21_unfollow-user.yaml)
- [API-22 フォロー一覧](paths/API-22_list-following.yaml)
- [API-23 フォロワー一覧](paths/API-23_list-followers.yaml)
- [API-24 プロフィールの更新](paths/API-24_update-profile.yaml)

## 4. 画面操作との対応の確認

screens/README.md の画面×操作（付録 A.3）を、すべて API に対応づけた。

| 画面 | 操作 | API ID |
| --- | --- | --- |
| 全画面 | 起動時のユーザー判定、CSRF トークンの取得 | API-01 |
| 全画面 | ログアウト | API-07 |
| SCR-01 | 全体タブの読み込み・追加読み込み | API-11 |
| SCR-01 | フォロー中タブの読み込み・追加読み込み | API-12 |
| SCR-01 / SCR-09 | いいね・取り消し | API-16 / API-17 |
| SCR-02 | 登録 | API-03 |
| SCR-03 | 確認メールの再送 | API-05 |
| SCR-04 | リンクの確認 | API-04 |
| SCR-04 / SCR-05 | 確認メールの再送 | API-05 |
| SCR-05 | ログイン | API-06 |
| SCR-06 | 再設定メールの申請 | API-08 |
| SCR-07 | リンクの確認 | API-09 |
| SCR-07 | パスワードの再設定 | API-10 |
| SCR-08 / SCR-12 | 設定値の取得 | API-02 |
| SCR-08 | 投稿 | API-13 |
| SCR-09 | 投稿の取得 | API-14 |
| SCR-09 | 削除 | API-15 |
| SCR-10 | プロフィールの取得 | API-18 |
| SCR-10 | 投稿一覧の読み込み・追加読み込み | API-19 |
| SCR-10 / SCR-11 | フォロー・解除 | API-20 / API-21 |
| SCR-11 | フォロー一覧 | API-22 |
| SCR-11 | フォロワー一覧 | API-23 |
| SCR-12 | 現在の値の取得 | API-18、API-01 |
| SCR-12 | 保存 | API-24 |

- API を呼ばない操作（画面遷移、ブラウザ内の白黒変換とプレビュー、タブの切替、確認ダイアログ）は表に載せていない。
- 要件定義書の機能要件（ACC-01〜09、PRF-01〜05、PST-01〜08、LIK-01〜03、FOL-01〜03、TML-01〜06）は、2 章の「関連要件 ID」の列ですべてどれかの API に対応している。

## 5. 設計判断

| # | 決めたこと | 理由・検討した別案 |
| --- | --- | --- |
| D-1 | API ID の番号を screens の R 番号と同じにする | 画面設計の R 番号を API ID に置き換えるときの手間と取り違えを避けるため。別案: 機能の領域ごとに採番し直す。意味の区切りは付くが、対応表が要る |
| D-2 | リソース名の規則: `timeline` を `timelines` に直す。`/api/me`・`/api/config`・`/api/auth/*` は例外として残す | 規則は複数形・小文字・kebab-case。`me`（本人）と `config`（設定）は 1 つしかないリソースで、`/api/me` は広く使われる慣習。認証の操作は `sessions` や `users` のリソースにするより、`/api/auth/login` のような動作名のほうが画面設計の仮案と一致し、Echo のルートも単純になる。別案: `POST /api/sessions`（ログイン）、`DELETE /api/sessions/current`（ログアウト）、`POST /api/users`（登録）。REST としては整うが、再設定やメール確認の操作が自然なリソースにならず、結局動作名が混ざる。パスワード再設定の確認・実行は動作名の `auth/password/reset/check`・`reset` とした |
| D-3 | CSRF は「Origin の検査 + セッションに結び付いた HMAC トークン（ヘッダー `X-CSRF-Token`）」の二段にする | セッション Cookie を `HttpOnly` のまま保ち、トークンを DB に保存せずに済む。ログイン不要の変更系 API は Origin の検査のみにして、トークン取得の手間を増やさない。別案 1: ダブルサブミットクッキー。トークン用の Cookie を JS から読む必要があり、サブドメインからの Cookie 上書きにも弱い。別案 2: トークンを DB（`sessions`）に保存する。列と書き込みが増える。別案 3: SameSite=Lax だけに頼る。古いブラウザや、`GET` で状態を変えた場合に弱く、要件 6.2 の「CSRF 対策を行う」を満たしたと言いにくい |
| D-4 | トークンはセッションごとに固定（リクエストごとに更新しない） | 画面側の複雑さ（並行リクエストでのトークンの食い違い）を避ける。HMAC で導出するだけなので、漏れてもセッション Cookie が無ければ使えない |
| D-5 | Cookie 名を `nocolor_session` とし、`__Host-` を付けない | 1.3 に記載。HTTP の開発環境で保存できなくなるため |
| D-6 | カーソルは AES-GCM で暗号化した不透明な文字列にする | 内部連番のユーザー ID が含まれる（フォロー一覧）ため、平文の Base64 では「内部 ID を API に出さない」方針（db.md D-2）に反する。一覧の種類を中に持たせて別の一覧への流用を拒否できる。コストは鍵（`APP_SECRET`）の管理だけ。別案: 投稿の一覧だけは ULID の文字列そのもの（db.md 4.1 の例）。簡素だが形式が 2 種類になり、カーソルの仕様が内部に漏れる。別案: HMAC の署名のみ。改ざんは防げるが、中身が見える |
| D-7 | `limit` を指定できる（1〜50、既定 20）。範囲外は丸めず 400 | 画面は既定の 20 件を使う。上限があれば負荷を抑えられる。丸めると呼び出し側の誤りに気づけない。別案: `limit` を設けず 20 件固定。単純だが、将来の画面（グリッドで多く取るなど）で API を変える必要が出る |
| D-8 | `TOKEN_EXPIRED` を設けず `TOKEN_INVALID` にまとめる。`ALREADY_VERIFIED` は残す | 期限切れの行は定期ジョブで削除され、削除前後で「期限切れ」と「存在しない」を区別できない。画面でも区別して見せない（SCR-04）。`ALREADY_VERIFIED` は、メールを 2 回開いた利用者に「再送してください」と誤案内しないために有用。そのためには、確認済みの後もトークンを残す必要がある。確認が完了してもトークンは削除せず、期限切れのジョブに任せる（トークンはハッシュで、確認の更新は条件付きなので安全性は変わらない）。db.md 3.2 は、この前提で記述されている。別案: 確認完了時にトークンをすべて削除する。その場合は 2 回目以降のアクセスが `TOKEN_INVALID` になり、SCR-04 の「すでに確認済み」の状態を出せない |
| D-9 | メール再送・パスワード再設定の申請は 202 とし、アドレスの検索以降をすべて非同期にする | ACC-09 を、文言だけでなく処理時間でも守るため。同期の経路が存在の有無で同一になる。メール送信（桁違いに遅い）だけを切り離しても、DB アクセスの差が残る。欠点: ジョブの失敗は利用者に伝わらない（再送で対処）。ジョブはプロセス内の容量付きキューで処理し（1.11）、満杯なら 503 を返す。プロセスの異常終了でジョブが失われうるが、MVP では許容する。別案: 同期で処理し、遅延を揃えるスリープを入れる。確実性が低く、負荷が増える |
| D-10 | ログインの 403 `EMAIL_NOT_VERIFIED` は、パスワードの照合が成功したときだけ返す | ACC-04 の案内を出しつつ、第三者に確認状態を調べさせない（screens D-10）。ユーザーが存在しない場合もダミーの bcrypt を実行して処理時間を揃える |
| D-11 | 画像の拒否は、形式が違えば 415 `UNSUPPORTED_IMAGE_TYPE`、壊れていれば 400 `INVALID_IMAGE` に分ける | 画面の文言が異なる（screens 3.5.2 の「種類」と「デコード」）ため。ファイルサイズの超過は 413。画面設計の `IMAGE_TOO_LARGE_RESOLUTION` は、他のコードと語順をそろえて `IMAGE_RESOLUTION_TOO_LARGE` に改名した |
| D-12 | レート制限の既定値は 1.8 の表のとおりとし、すべての試行（成功を含む）を数える。メール系は 1 時間 3 回（アドレス）／10 回（IP） | db.md Q-6 の例を採用した。成功を数えないと、成功するまで試せる攻撃者を止められない。アカウント単位の制限は、他人に枠を使い切られる（ロックアウトに近い）欠点があるが、ブルートフォースの防止を優先する。ウィンドウが過ぎれば解除される。別案: 失敗だけを数え、成功で戻す。正当な利用者の負担は減るが、`rate_limits` の更新が複雑になる。`token_ip`・`post_user`・`profile_user` は screens が 429 を想定した API に、DB の `rate_limits.scope`（自由な文字列）で追加した |
| D-13 | `PUT /api/me/profile` は表示名・自己紹介を常に送る完全置換にする | 「送られていない」と「空にする」の区別が要らない（部分更新の `PATCH` だと `bio` の空文字と省略の扱いが必要）。冪等。アイコンだけが `icon_action` で 3 値になる。画面設計 SCR-12 の「変更がない項目は送らない」のうち、テキスト項目は常に送る |
| D-14 | 投稿の二重送信を防ぐ `Idempotency-Key` は設けない | 通信断で結果が不明な場合の対処は、画面設計 SCR-08 が「自分のプロフィールで確認する」案内で補う。キーを DB に保存する仕組みが増える。別案は 6 章の Q-5 |
| D-15 | パスワードの 72 バイト制限は、実装が事前ハッシュ等で吸収する | 要件は 32 文字までで、マルチバイト文字だと 72 バイトを超えうる。拒否すると「32 文字以内」の規則と食い違い、黙って切り詰めると別のパスワードで入れてしまう。別案: 72 バイト超を `TOO_LONG` で拒否する。仕様は単純だが、日本語のパスフレーズで利用者を混乱させる |
| D-16 | 存在しない／形式不正の ID・ユーザー名は、404 で統一する。仮登録のユーザーも「存在しない」とする | 画面設計 D-25 のとおり。仮登録中の存在をプロフィールの取得で調べさせないため（4.1 の ACC-05 により、仮登録の名前を他人が知る理由がない） |
| D-17 | 自分自身の `is_following` は常に `false` | 画面は自分と比較してボタンの表示を決める（screens C-04）。API は閲覧者との関係だけを返す |
| D-18 | いいね・フォローは冪等な PUT / DELETE にし、操作後の状態と件数を返す | 連打や再送でエラーにならない。画面は応答で表示を確定できる（screens 3.2.3、3.2.4）。別案: `POST /likes`（2 回目は 409）。画面の再試行が複雑になる。`/like` は閲覧者本人のいいね 1 件を表す単数のサブリソースとして扱う |
| D-19 | 投稿の削除（API-15）は冪等にしない（2 回目は 404） | 削除済みの検知で画面が動作を分ける（screens SCR-09）。いいね・フォローと異なり、結果を「状態」で表すより、存在の有無が意味を持つため |
| D-20 | パスワード再設定の確認（API-09）は読み取りだが `POST` にし、トークンをボディで送る | URL に入れるとアクセスログ・リファラに残る。メール確認（API-04）と形をそろえる |
| D-21 | メール確認・パスワード再設定の成功時に自動ログインしない | トークン付き URL でセッションを作らない（screens 3.4.3）。再設定の成功時は全セッションを破棄するため、むしろログイン状態の維持と矛盾する |
| D-22 | 画像の URL は相対パス、寸法は表示向きに揃えた px の整数 | ADR 001 の同一オリジン。画面は `aspect-ratio` の確保に使う（screens 3.2.2） |
| D-23 | 応答の `no-store` | 閲覧者ごとの状態やトークンを含み、共有キャッシュや戻る操作で別人の状態が見えないようにするため。画像（`/images`）は別で長期キャッシュ（ADR 001） |
| D-24 | Profile に「相手が自分をフォローしているか」（`follows_me`）を含めない | 画面で使わず、要件にもない。必要になった時点で追加する（項目の追加は後方互換）。 |
| D-25 | 新規登録（API-03）でメールアドレスの重複を告知しない（`EMAIL_TAKEN` を廃止）。登録済みのアドレスには通常の成功と同じ 201 を返し、メールで知らせる | 登録画面でアドレスの登録の有無を調べられるのを防ぐ（ACC-09 と同じ方針）。ユーザー名は公開情報なので従来どおり 409 `USERNAME_TAKEN`。ユーザー名が空いていてアドレスが登録済み（本登録済み、または 7 日以内の有効な仮登録）なら、ユーザーを作らずに 201 を返す。処理時間を揃えるため bcrypt は分岐の前に必ず行う。重複の判定手順は db.md 3.1 を正とし、1062 はキー名ではなく `GetUserByUsername` で振り分ける。メール重複の処理（本登録済みには案内メール、有効な仮登録には旧トークン削除・新トークン発行・確認メール再送）は、すべてワーカーで行う。**既知の問題（受け入れ済み）**: ユーザー名はメールアドレスの重複時に確保されないので、(1) 未使用のユーザー名 X と調べたいアドレスで登録、(2) 同じ X と自分のアドレスで登録、を順に行い、(2) が 201 なら (1) のアドレスは登録済み、409 なら (1) は新規登録だった、と判定できる。受け入れた理由: ACC-09 が直接求めるのは再設定画面（API-08）のみであること、MVP で塞ぐ複雑さ（ユーザー名の仮確保など）が見合わないこと。緩和策: `signup_mail_ip` など IP 単位のレート制限で、大量の調査だけを抑える（1 件ずつの調査は防げない）。実装は今の設計のままとする（ユーザーの決定）。どちらも `signup_mail_*` で数える。欠点: 利用者が間違えて登録済みのアドレスを入れると、画面は成功を示すがメールは案内になる（メール本文で補う）。応答に入力の `email` を含めるのは、入力した本人に返すだけなので問題ない。別案: 重複を 409 で告知する（利用者には親切だが、アドレスの登録の有無が分かる）。同期でも同じ処理時間にするスリープを入れる案（確実性が低い） |
| D-26 | セッションの有効期限は、ログインからの固定 30 日にする（スライディングしない） | 盗まれた Cookie が使い続けられる期間に上限を持たせる。期限の延長のための書き込みが要らない。欠点: 毎日使っていても 30 日後に再ログインが要る。別案: アクセスごとに延長する（利便性は上がるが、使われ続けるセッションは失効しない） |
| D-27 | アイコンはサーバーで長辺 256px に縮小して保存し、元画像は保存しない | アイコンはタイムラインの全投稿に並ぶので、最大 10MB の原寸のまま配ると表示が重くなる。縦横比を保つ補間だけで色は変換しないので、要件 2.2 に抵触しない。形式はサムネイルと同じ（要件 8 章・未決）。別案: 原寸で保存する（実装は単純だが帯域を無駄にする）。切り抜き（正方形）にする案は、画面の見え方の判断なので採らない（CSS で切り抜く） |
| D-28 | 画像の解像度に、要件 8 章の未決とは別の固定の安全弁を置く（各辺 65535px 以下、総画素数 1 億以下。ヘッダーで判定し、超えたら復号せず 400） | ファイルサイズが 10MB 以下でも、圧縮率の高い PNG などは数億画素に展開でき、メモリを使い切られる（DoS）。65535px は DB の `SMALLINT UNSIGNED` の範囲とも一致する。要件 8 章で最大解像度が決まれば、より厳しい側が効く。別案: 上限が決まるまで無制限にする（DoS の危険がある） |
| D-29 | `Origin` は `APP_BASE_URL` との完全一致で検査し、`Host` とは比べない | リバースプロキシ・TLS 終端でホスト、ポート、スキームが変わりうるため。スキームも比べる（http と https の混在を許さない）。1.4 参照。欠点: 開発・本番ごとに `APP_BASE_URL` を正しく設定する必要がある（メール内のリンクで元から必要） |
| D-30 | 作成から 7 日を超えた仮登録は、削除ジョブの実行前でも「存在しない」と扱う（参照時の条件で判定する） | API-06 は `INVALID_CREDENTIALS`、API-05 は再送しない、API-18〜23 は 404、API-03 は再登録を許す。db.md 5 章の方針と同じ。削除ジョブの実行間隔に挙動が左右されない |
| D-31 | API 設計書を OpenAPI 3.1.0 で書く。1 API 1 ファイルとし、ルートの `paths` でメソッドごとに `$ref` する。コード生成は bundle したファイルで行う | OpenAPI で表せる仕様（項目・型・入力規則・応答・エラー）を機械可読にして、Markdown との食い違いを防ぎ、コード生成・lint に使えるようにするため。同じパスの複数メソッドは 1 つの Path Item に書く必要があるので、Operation 単位で分割して `$ref` する。表せない共通規則・運用・設計判断・未決事項はこの README に残す。別案: 1 つの巨大な YAML にする。レビューの差分が見づらくなる |

## 6. 未決事項

要件定義書 8 章の未決事項は確定していない。API に影響するものは Q-1〜Q-4 に推奨案を書く。

| # | 内容 | API への影響 | 推奨案 |
| --- | --- | --- | --- |
| Q-1 | **変換後の画像形式**（要件 8 章。screens U-01） | 保存する画像の形式とキーの拡張子、`Content-Type`。API の項目は変わらない（`image_url` の拡張子だけが変わる）。**依存への影響**: WebP で保存するなら WebP エンコーダー（Go 標準・`x/image` には無い。libwebp の cgo 等）が要る（API-13 の備考） | 画面設計 U-01 の案（JPEG 入力は JPEG、PNG・WebP 入力は PNG）。追加の依存が要らず、メタデータの除去も単純になる。最終判断は要件側 |
| Q-2 | **画像の最大解像度**（要件 8 章。screens U-02） | API-02 の `max_image_side`・`max_image_pixels`（決まるまで `null`）、`IMAGE_RESOLUTION_TOO_LARGE` の発生条件。**固定の安全弁（各辺 65535px・1 億画素。D-28）は、この未決とは別に常に働く** | 画面設計 U-02 の案（長辺 8192px・約 5000 万画素超を拒否）を、安全弁より厳しい上限として採用する |
| Q-3 | **サムネイルのサイズと形式**（要件 8 章）。アイコンの縮小後の形式もこれに従う（D-27） | API の項目は変わらない（寸法を `thumbnail_width`・`thumbnail_height` で返すため）。**参考**: 推奨が食い違っている。screens U-03 は長辺 1080px、db.md Q-1 は長辺 600px 程度の WebP。WebP のサムネイルには WebP エンコーダー（cgo 等）が要る | 要件側で決定。API の例では 600px を使ったが、確定ではない。cgo を避けるなら JPEG／PNG のサムネイル |
| Q-4 | **白黒判定の許容値の既定値**（要件 8 章） | API には影響しない（`NOT_GRAYSCALE` の判定のみ。環境変数 `GRAYSCALE_TOLERANCE`） | JPEG の実画像で検証して決める |
| Q-5 | 投稿の二重送信を防ぐ `Idempotency-Key` の要否 | API-13 のヘッダー追加 | MVP では設けない（D-14）。実運用で二重投稿が問題になったら追加する |
| Q-6 | 試行制限の具体値（1.8） | 429 になる条件 | 1.8 の値で確定してよいか。メールは 1 時間 3 回だと、届かずに再送を繰り返す利用者が止まりうる。画面側は 60 秒の待機（screens SCR-03）があるので、ほとんど到達しない。新規登録の重複メール（D-25）も同じ枠で数える |
| Q-7 | **コード生成器（oapi-codegen の版）の選定と、3.1 の書き方で型が素直に生成されるかの確認**。対象は `type: [T, "null"]`、API-01 の `user` の `oneOf [UserBrief, null]`、Schema の `examples` | 生成される型。API の項目・仕様は変わらない | 実装の最初に、bundle 後のファイルで oapi-codegen v2 の最新版を実行して確認する。`oneOf` が union 型になるなど素直でなければ、null を許す `$ref` の書き方を規約で決め直す |

確定済みの旧 Q-5（確認トークンを残す）は D-8、旧 Q-8（メール重複の告知）は D-25、旧 Q-9（7 日超の仮登録の扱い）は D-30、旧 Q-2 のうち安全弁の部分は D-28 に移した。
