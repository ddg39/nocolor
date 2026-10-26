# API-02 設定値の取得

[API 設計](README.md) / API-02 設定値の取得

> 節番号（1.x）・共通のオブジェクト・設計判断（D-xx）・未決事項（Q-x）は [API 設計の README](README.md) を指す。

`GET /api/config`（R-02）

**概要**: 画面が検査に使う設定値を返す（画像の上限など）。環境変数で変わる値を画面に持たせないために設ける。

**リクエスト**: なし。

**レスポンス**: 200

```json
{
  "max_image_bytes": 10485760,
  "allowed_image_types": ["image/jpeg", "image/png", "image/webp"],
  "max_image_side": null,
  "max_image_pixels": null
}
```

| キー | 型 | 説明 |
| --- | --- | --- |
| `max_image_bytes` | integer | 画像ファイルの最大バイト数（PST-02。`MAX_IMAGE_BYTES`） |
| `allowed_image_types` | string[] | 許可する形式（PST-02）。アイコンにも同じ規則を使う（screens U-05） |
| `max_image_side` | integer \| null | 画像の長辺の最大 px（要件 8 章の未決の上限）。`null` は未決。`null` でも、サーバーは固定の安全弁（各辺 65535px・総画素数 1 億まで。API-13 の処理順 (4)）で 400 `IMAGE_RESOLUTION_TOO_LARGE` を返す |
| `max_image_pixels` | integer \| null | 画像の総画素数の上限（同上） |

**エラー**: E1 のみ。

**備考**: DB は使わない。副作用なし。文字数の上限（キャプション 500 など）は要件で固定なので含めない。
