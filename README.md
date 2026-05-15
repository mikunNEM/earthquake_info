# Symbol Earthquake Viewer

> Symbolブロックチェーンに記録された気象庁の地震情報 ＋ P2P地震速報のリアルタイム通知を組み合わせたWebアプリケーション

![Header Demo](https://img.shields.io/badge/Symbol-Blockchain-7c3aed)
![P2P](https://img.shields.io/badge/P2P地震速報-API-0d9488)
![Leaflet](https://img.shields.io/badge/Leaflet-1.9.4-green)
![License](https://img.shields.io/badge/license-MIT-blue)

## 概要

**Symbol Earthquake Viewer**は、2つのデータソースを組み合わせた地震情報Webアプリです。

| データソース | 役割 | 速度 |
|---|---|---|
| **P2P地震速報** | リアルタイム速報（通知・地図ポップアップ） | 数秒 |
| **Symbolブロックチェーン** | 履歴リスト表示（ブロックチェーン記録） | 30秒〜数分 |

## 主な機能

### 1. P2P地震速報（リアルタイム）
- `wss://api.p2pquake.net/v2/ws` にWebSocket接続
- 新着地震を検知すると：
  - 地図がその場所に自動移動してポップアップを表示
  - 画面右上に通知カードを表示（×ボタンで手動閉じ）
- 左パネル上部に接続ステータスを表示

### 2. Symbolブロックチェーン（履歴）
- ページ読み込み時に承認済みトランザクション最新50件を取得して履歴リストに表示
- WebSocketで未承認→承認のリアルタイム更新
- Symbol Explorerへのリンク付き

### 3. 地図表示
- Leaflet.js ＋ 国土地理院タイルによる地図
- 震度・津波情報に応じたカラーマーカー
- クリックで詳細ポップアップ表示

### 4. 震度別カラーリング

| 震度 | 色 |
|---|---|
| 1 | 水色 |
| 2 | 青 |
| 3 | 緑 |
| 4 | 黄色 |
| 5弱 | オレンジ |
| 5強 | 濃いオレンジ |
| 6弱 | 赤 |
| 6強 | 濃い赤 |
| 7 | 暗赤 |
| 津波注意報以上 | 津波レベル優先色 |

## 技術スタック

| 技術 | 用途 |
|------|------|
| **Leaflet.js 1.9.4** | 地図表示ライブラリ |
| **国土地理院タイル** | 日本地図ベースマップ |
| **P2P地震速報 API** | リアルタイム地震速報 |
| **Symbol Blockchain** | 地震データ履歴ソース |
| **WebSocket** | リアルタイム通信（Symbol・P2P両方） |
| **Fetch API** | HTTPリクエスト |
| **HTML5/CSS3/ES6+** | フロントエンド基盤 |

## アーキテクチャ

```
P2P地震速報
  wss://api.p2pquake.net/v2/ws
  └─ 新着検知 → 地図ポップアップ ＋ 通知カード

Symbol Blockchain Network
  ├─ REST API（起動時）→ 履歴リスト表示
  └─ WebSocket（常時）→ 未承認→承認のリアルタイム更新

                  ↓
┌──────────────────────────────────────────────────┐
│           Symbol Earthquake Viewer               │
│                                                  │
│  ┌────────────┐         ┌────────────────────┐  │
│  │  左パネル   │         │    右パネル         │  │
│  │ P2P接続状態 │         │  (Leaflet地図)      │  │
│  │ Symbolノード│         │  マーカー＋ポップアップ│  │
│  │ (地震リスト) │         │                    │  │
│  └────────────┘         └────────────────────┘  │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  通知エリア（右上）                        │   │
│  │  新着地震を検知時に通知カード表示           │   │
│  │  ×ボタンで手動クローズ                    │   │
│  └──────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

## 使い方

```bash
# ローカルサーバー起動
python -m http.server 8000
# http://localhost:8000 にアクセス
```

1. ページ読み込み → Symbolブロックチェーンから最新50件の地震履歴を表示
2. P2P速報WebSocketに自動接続（左パネルに「接続中」表示）
3. 新着地震を検知すると地図が自動移動 ＋ 通知カード表示
4. 通知カードをクリック → 地図が該当地震の位置に移動
5. ×ボタンで通知を閉じる

## カスタマイズ

### Symbol署名者公開鍵の変更
```javascript
const ALLOWED_PUBKEY = "あなたの公開鍵";
```

### フォールバックノードの変更
```javascript
const FALLBACK_NODE = "https://your-node.com:3001";
```

### Symbol地震データアドレスの変更
```javascript
const address = "あなたのSymbolアドレス";
```

## データフォーマット

### Symbol（ブロックチェーンに記録されたJSON）
```json
{
  "earthquake": {
    "time": "2024-01-15T10:30:00+09:00",
    "hypocenter": {
      "name": "東京都23区",
      "latitude": 35.6895,
      "longitude": 139.6917,
      "depth": 30,
      "magnitude": 4.5
    },
    "maxScale": 40,
    "domesticTsunami": "None"
  }
}
```

### P2P地震速報（code: 551）
```json
{
  "code": 551,
  "earthquake": {
    "time": "2024/01/15 10:30:00",
    "hypocenter": {
      "name": "東京都23区",
      "latitude": 35.6895,
      "longitude": 139.6917,
      "depth": 30,
      "magnitude": 4.5
    },
    "maxScale": 40,
    "domesticTsunami": "None"
  }
}
```

### 震度コード対応表
| コード | 震度 |
|--------|------|
| 10 | 1 |
| 20 | 2 |
| 30 | 3 |
| 40 | 4 |
| 45 | 5弱 |
| 50 | 5強 |
| 55 | 6弱 |
| 60 | 6強 |
| 70 | 7 |

## 参考リンク

- [P2P地震速報 API](https://www.p2pquake.net/)
- [Symbol Documentation](https://docs.symbol.dev/)
- [Leaflet.js](https://leafletjs.com/)
- [国土地理院タイル](https://maps.gsi.go.jp/development/ichiran.html)
- [Symbol Explorer](https://symbol.fyi/)

---

**注意**: このアプリケーションは公式の気象情報システムではありません。実際の防災・避難行動には気象庁の公式情報を参照してください。

## ライセンス

MIT License
