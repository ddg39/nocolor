# CLAUDE.md

## 開発ルール

- ブランチ運用・コミットメッセージ: [.claude/rules/git.md](.claude/rules/git.md)（`.claude/rules/` 配下は自動で読み込まれる）

## 設計チーム

画面設計・API 設計・DB 設計は `/design [all | screens | api | db]` で行う。メインセッションが監督となり、次のエージェントを指揮する。

| 役割 | エージェント | モデル | 成果物 |
| --- | --- | --- | --- |
| 画面設計 | `screen-designer` | sonnet | `docs/design/screens/`（1 画面 1 ファイル） |
| API 設計 | `api-designer` | sonnet | `docs/design/api/`（OpenAPI 3.1.0、1 API 1 ファイル） |
| DB 設計 | `db-designer` | sonnet | `docs/design/db.md` |
| レビュー | `design-reviewer` | opus | 指摘と判定 |

手順は [.claude/skills/design/SKILL.md](.claude/skills/design/SKILL.md)、各エージェントの指示は `.claude/agents/` を参照。
