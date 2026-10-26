# コントリビューションガイド

コミットメッセージは [Angular Commit Message Format](https://github.com/angular/angular/blob/main/contributing-docs/commit-message-guidelines.md)（Google の Angular チームが定めた形式）に、説明文の書き方は [Google Engineering Practices: Writing good CL descriptions](https://google.github.io/eng-practices/review/developer/cl-descriptions.html) にもとづく。

## ブランチ運用

| ブランチ | 役割 | 派生元 | マージ先 |
| --- | --- | --- | --- |
| `main` | リリース済みの安定版 | - | - |
| `develop` | 開発の統合先 | `main` | `main`（リリース時） |
| 作業ブランチ | 機能追加・修正など | `develop` | `develop` |
| `hotfix/*` | リリース版の緊急修正 | `main` | `main` と `develop` |

- `main` と `develop` へ直接 push しない。必ず Pull Request を経由する。
- 作業ブランチは小さく短命に保つ。1つのブランチで扱う変更は1つにする。
- 作業ブランチから `develop` へは **Squash and merge** でマージし、1変更 = 1コミットにする。
- `develop` から `main` へは **Create a merge commit** でマージする。

### ブランチ名

```
<type>/<短い説明>
```

- `<type>` にはコミットの type（後述）と同じものを使う。
- 説明は英小文字の kebab-case で書く。Issue があれば番号を付ける。

```
feat/post-upload
fix/123-login-error
docs/commit-rules
hotfix/image-validation
```

## コミットメッセージ

### 形式

```
<type>(<scope>): <summary>

<body>

<footer>
```

- ヘッダー（1行目）は必須。本文とフッターは任意だが、自明な変更以外は本文を書く。
- ヘッダー・本文・フッターの間は空行で区切る。
- どの行も 100 文字以内にする。

### type

| type | 用途 |
| --- | --- |
| `feat` | 新機能 |
| `fix` | バグ修正 |
| `docs` | ドキュメントのみの変更 |
| `style` | 動作に影響しない変更（フォーマット、空白など） |
| `refactor` | 機能追加でもバグ修正でもないコード改善 |
| `perf` | パフォーマンス改善 |
| `test` | テストの追加・修正 |
| `build` | ビルド・依存関係・Docker の変更 |
| `ci` | CI 設定の変更 |
| `chore` | 上記以外の雑務 |
| `revert` | 過去のコミットの取り消し |

### scope

変更した領域を書く。複数にまたがる場合や全体に関わる場合は省略してよい。

| scope | 対象 |
| --- | --- |
| `frontend` | フロントエンド（React） |
| `backend` | バックエンド（Go / Echo） |
| `db` | スキーマ・マイグレーション・sqlc のクエリ |
| `docker` | Dockerfile / docker compose |

### summary（1行目の要約）

Google の CL 説明の指針に従い、1行目だけで「何をしたか」がわかるように書く。

- 英語の命令形・現在形で書く（`add`。`added` や `adds` は使わない）。
- 先頭は小文字にし、末尾にピリオドを付けない。
- `fix bug` や `update code` のような、内容がわからない要約は書かない。

### body（本文）

- 「何を」「なぜ」変更したかを書く。「どのように」はコードを見ればわかるので最小限でよい。
- 背景、検討した別案、制約、既知の問題があれば書く。
- 英語の命令形・現在形で書く。

### footer（フッター）

- 互換性のない変更は `BREAKING CHANGE: <説明>` を書く。
- 関連する Issue は `Closes #123` や `Refs #123` で示す。
- `revert` の場合は本文に `This reverts commit <hash>.` と取り消す理由を書く。

### 例

```
feat(backend): reject non-grayscale images on upload

Posts must contain only black-and-white images. Check every pixel
and return 400 when R, G and B differ beyond a small tolerance,
since JPEG compression slightly shifts gray pixels.

Closes #12
```

```
fix(frontend): keep form input after a failed upload
```

```
build(docker): add MinIO service to docker compose
```

```
refactor(db)!: rename posts.image_path to posts.object_key

BREAKING CHANGE: run the 0003 migration before starting the new backend.
```
