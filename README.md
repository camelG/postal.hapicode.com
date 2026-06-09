# postal.hapicode.com

日本の郵便番号から住所を検索できる、静的 JSON API です。認証不要で、ブラウザやサーバーからそのまま利用できます。

**ベース URL:** `https://postal.hapicode.com`

## 特徴

- 認証・API キー不要
- CORS 対応（ブラウザから直接 `fetch` 可能）
- 日本郵便の [全国地方公共団体コード（町域）](https://www.post.japanpost.jp/zipcode/dl/kogaki-zip.html) を元にしたデータ
- GitHub Pages 上の静的ファイル配信（サーバーレス）

## エンドポイント

| 用途 | メソッド | パス |
|------|----------|------|
| API メタ情報 | GET | `/api/index.json` |
| 郵便番号検索 | GET | `/api/v1/{上3桁}/{下4桁}.json` |
| 都道府県一覧 | GET | `/api/v1/regions/index.json` |
| 都道府県別 市区町村・町域 | GET | `/api/v1/regions/{都道府県名}.json` |

## 郵便番号検索

7 桁の郵便番号を、**上 3 桁（ディレクトリ）** と **下 4 桁（ファイル名）** に分けてリクエストします。

| 郵便番号 | リクエスト URL |
|----------|----------------|
| `0010023` | `https://postal.hapicode.com/api/v1/001/0023.json` |
| `5590005` | `https://postal.hapicode.com/api/v1/559/0005.json` |
| `0600000` | `https://postal.hapicode.com/api/v1/060/0000.json` |

### レスポンス形式

常に次の 4 フィールドを持つフラットオブジェクトです。

```json
{
  "prefecture": "大阪府",
  "city": "大阪市住之江区",
  "town": "西住之江",
  "address": "大阪府大阪市住之江区西住之江"
}
```

| フィールド | 説明 |
|------------|------|
| `prefecture` | 都道府県名 |
| `city` | 市区町村名 |
| `town` | 町域名（正規化済み。該当なしの場合は空文字） |
| `address` | `prefecture` + `city` + `town` を連結した文字列 |

### 複数町域がある郵便番号

1 つの郵便番号に複数の町域が紐づく場合でも、レスポンスは 1 件のみです。日本郵便 CSV の**先頭行**を採用します。

例: `5152514`（一志町小山）は `一志町小山` を返します（`（小山台地）` などの括弧注記は除去）。

### 存在しない郵便番号

該当ファイルがない場合は HTTP `404` を返します。

## 住所の正規化

出力前に町域・市区町村名から括弧注記と日本郵便のプレースホルダを削除します。詳細は `/api/index.json` の `normalization` を参照してください。

| 変換前 | 変換後 |
|--------|--------|
| `大手町（次のビルを除く）` | `大手町` |
| `北一条西（１～１９丁目）` | `北一条西` |
| `伊敷町（その他）` | `伊敷町` |
| `以下に掲載がない場合` | ``（空文字） |

## API メタ情報

`GET /api/index.json` でバージョンや統計を確認できます。

```json
{
  "version": 6,
  "updatedAt": "2026-06-07T15:57:39.863Z",
  "source": "https://www.post.japanpost.jp/service/search/zipcode/download/kogaki/zip/ken_all.zip",
  "recordCount": 124825,
  "zipcodeCount": 120680,
  "normalization": { "town": "remove_parentheses" },
  "chunkBy": "prefix3+suffix4",
  "lookupPath": "/api/v1/{prefix3}/{suffix4}.json",
  "stats": { "prefix3Dirs": 948, "shardFiles": 120680 },
  "apis": {
    "zipLookup": "/api/v1/{prefix3}/{suffix4}.json",
    "regionsIndex": "/api/v1/regions/index.json",
    "region": "/api/v1/regions/{prefecture}.json"
  }
}
```

## 都道府県 API

### 都道府県一覧

```json
{
  "updatedAt": "2026-06-07T15:57:39.863Z",
  "prefectures": [
    {
      "name": "北海道",
      "path": "/api/v1/regions/%E5%8C%97%E6%B5%B7%E9%81%93.json",
      "cityCount": 188
    }
  ]
}
```

### 都道府県別データ

都道府県名を URL エンコードして指定します（例: `大阪府` → `%E5%A4%A7%E9%98%AA%E5%BA%9C`）。

```json
{
  "prefecture": "大阪府",
  "cities": {
    "大阪市住之江区": ["西住之江", "柴谷", "住之江"]
  }
}
```

町域名も郵便番号検索と同様に正規化されています。

## 利用例

### JavaScript

```javascript
const API_BASE = "https://postal.hapicode.com";

export async function lookupZipcode(zipInput) {
  const zip = zipInput.replace(/\D/g, "");
  if (zip.length !== 7) return null;

  const res = await fetch(
    `${API_BASE}/api/v1/${zip.slice(0, 3)}/${zip.slice(3)}.json`,
  );
  if (!res.ok) return null;

  return res.json();
}

// 使用例
const address = await lookupZipcode("559-0005");
// { prefecture: "大阪府", city: "大阪市住之江区", town: "西住之江", address: "..." }
```

### curl

```bash
curl -s https://postal.hapicode.com/api/v1/559/0005.json
curl -s https://postal.hapicode.com/api/index.json
```

## データについて

- **出典:** [日本郵便 全国地方公共団体コード（町域）データ](https://www.post.japanpost.jp/zipcode/dl/kogaki-zip.html)（`ken_all.zip`）
- **更新日時:** `/api/index.json` の `updatedAt` を参照
- **件数:** 約 12 万件の郵便番号（`zipcodeCount`）
- **生成:** `postal-backend.hapicode.com` により毎日自動更新

## 利用上の注意

- 本 API は無償で提供していますが、**可用性や SLA は保証しません**。本番サービスではフォールバックやキャッシュの導入を推奨します。
- 住所データの正確性は日本郵便の公開データに依存します。利用・再配布については [日本郵便の利用規約](https://www.post.japanpost.jp/zipcode/dl/readme.html) に従ってください。
- 過度なリクエストは控えてください。静的ファイル配信のため、CDN キャッシュが効きます。

## ライセンス

住所データの利用条件は [日本郵便の利用規約](https://www.post.japanpost.jp/zipcode/dl/readme.html) に従ってください。

本リポジトリの API 配信形式については、データ提供者の規約を遵守した上で自由に利用できます。
