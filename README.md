# このプロジェクトについて
白黒の画像のみを投稿可能なSNSサービスとする。
公開リポジトリのため、Dockerのみでの動作を目的とする。

## 使用技術
### フロントエンド
- React
- typescript
- tailwindcss
- vite

### バックエンド
- Go
- Echo
- MySQL
- MinIO（投稿画像の保存先）
- sqlc（DBアクセス）
- golang-migrate（マイグレーション）
- セッション + Cookie（認証）

### 配信
- nginx（リバースプロキシ。構成は [ADR 001](docs/adr/001_delivery-architecture.md) を参照）

## 開発ルール
ブランチ運用とコミットメッセージの規則は [.claude/rules/git.md](.claude/rules/git.md) を参照。
