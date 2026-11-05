# syntax=docker/dockerfile:1
# ビルドコンテキストはリポジトリ直下（docker-compose.yml の build.context: .）

FROM node:24-alpine AS base
WORKDIR /app
# 依存関係だけを先に入れ、ソースの変更時は npm ci をキャッシュで省く
COPY src/frontend/package.json src/frontend/package-lock.json ./
RUN npm ci

# 開発用: ソースはボリュームでマウントし、Vite 開発サーバーを起動する
FROM base AS dev
EXPOSE 5173
CMD ["npm", "run", "dev"]

# 本番用: イメージ内でビルドする
FROM base AS build
COPY src/frontend/ ./
RUN npm run build

# 本番配信: 入口の nginx がビルド済みの静的ファイルを直接配信する
FROM nginx:1.29-alpine AS prod
COPY docker/nginx/prod.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
