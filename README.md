# Map Style Switcher

## 概要

このアプリケーションは、異なる地図プロバイダーの地図スタイルを切り替えて表示できる Web アプリケーションです。Amazon Location Service、OpenStreetMap、MapTiler の3つの地図スタイルに対応しています。

## 機能

- 3種類の地図スタイルの切り替え
  - Amazon Location Service
  - OpenStreetMap
  - MapTiler White
- Amazon Location Service 使用時の国境線表示/非表示切り替え
- スケールコントロール表示
- 地図の中心座標、ズーム、回転、傾きの保持（スタイル切り替え時）

## 必要条件

- Node.js
- Nuxt.js
- AWS アカウント
- MapTiler アカウント

## 環境変数

以下の環境変数を `.env` ファイルに設定する必要があります：

```
VITE_MAPTILER_KEY=your_maptiler_api_key
```

## AWS設定

`amplify_outputs.json` に以下の設定が必要です：

```json
{
  "geo": {
    "aws_region": "your_region",
    "maps": {
      "default": "your_map_name"
    }
  },
  "auth": {
    "identity_pool_id": "your_identity_pool_id"
  }
}
```

## インストール

```bash
# 依存パッケージのインストール
npm install

# 開発サーバーの起動
npm run dev
```

## 使用している主なライブラリ

- @aws-amplify/geo
- @aws/amazon-location-utilities-auth-helper
- maplibre-gl

## ライセンス

各地図プロバイダーの利用規約に従ってください：

- Amazon Location Service
- OpenStreetMap
- MapTiler

## 注意事項

- Amazon Location Service を使用するには、適切な AWS 認証情報が必要です
- MapTiler を使用するには、有効な API キーが必要です
- 地図スタイルの切り替え時に、一時的に地図が再読み込みされます

## 貢献

バグ報告や機能改善の提案は、Issue または Pull Request でお願いします。
