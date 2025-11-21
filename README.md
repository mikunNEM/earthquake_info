# Symbol Earthquake Viewer

> Symbolブロックチェーンに記録された気象庁の地震情報をリアルタイムで可視化するWebアプリケーション

![Header Demo](https://img.shields.io/badge/Symbol-Blockchain-7c3aed)
![Leaflet](https://img.shields.io/badge/Leaflet-1.9.4-green)
![License](https://img.shields.io/badge/license-MIT-blue)

## 📋 目次

- [概要](#概要)
- [主な機能](#主な機能)
- [技術スタック](#技術スタック)
- [アーキテクチャ](#アーキテクチャ)
- [コード解説](#コード解説)
  - [1. 署名者フィルタリング](#1-署名者フィルタリング)
  - [2. 通知システム](#2-通知システム)
  - [3. 地図表示とマーカー](#3-地図表示とマーカー)
  - [4. ノード選択](#4-ノード選択)
  - [5. データ取得と解析](#5-データ取得と解析)
  - [6. WebSocketリアルタイム更新](#6-websocketリアルタイム更新)
- [使い方](#使い方)
- [カスタマイズ](#カスタマイズ)

---

## 概要

**Symbol Earthquake Viewer**は、Symbolブロックチェーン上に記録された気象庁の地震情報をリアルタイムで取得・表示するWebアプリケーションです。

### 特徴
- 🌍 **地図ビュー**: 地震の震源地を視覚的に表示
- 🔔 **リアルタイム通知**: 新しい地震情報を自動検知
- 📊 **震度別カラーリング**: 震度や津波情報に応じた色分け
- 🔗 **ブロックチェーン連携**: Symbol Explorer へのリンク
- 📱 **レスポンシブデザイン**: モバイル対応

---

## 主な機能

### 1. **リアルタイム地震情報表示**
- WebSocketを使用して、Symbolブロックチェーンから地震情報を自動取得
- 未承認トランザクション(pending)と承認済みトランザクション(confirmed)を区別して表示

### 2. **地図マッピング**
- Leaflet.jsと国土地理院タイルを使用した地図表示
- 震源地にカラーマーカーを配置
- クリックで詳細情報をポップアップ表示

### 3. **震度・津波情報の色分け**
- 震度1〜7、津波情報に応じた視覚的な色分け
- 津波警報レベルは特に目立つ色で表示

### 4. **通知システム**
- 新しい地震検知時に画面右上に通知カード表示
- 自動的にフェードアウト

### 5. **ノード自動選択**
- NodeWatchからランダムに高ブロック高のノードを選択
- フォールバック機能付き

---

## 技術スタック

| 技術 | 用途 |
|------|------|
| **Leaflet.js 1.9.4** | 地図表示ライブラリ |
| **国土地理院タイル** | 日本地図ベースマップ |
| **Symbol Blockchain** | 地震データソース |
| **WebSocket** | リアルタイム通信 |
| **Fetch API** | HTTPリクエスト |
| **HTML5/CSS3/ES6+** | フロントエンド基盤 |

---

## アーキテクチャ

```
┌─────────────────────────────────────────────────┐
│         Symbol Blockchain Network               │
│  (地震情報を記録したトランザクション)              │
└────────────┬────────────────────────────────────┘
             │
             ├─ REST API (過去データ取得)
             │   /transactions/confirmed
             │
             └─ WebSocket (リアルタイム更新)
                 ├─ unconfirmedAdded
                 └─ confirmedAdded
                           │
                           ▼
┌──────────────────────────────────────────────────┐
│           Symbol Earthquake Viewer               │
│                                                  │
│  ┌────────────┐         ┌────────────────────┐  │
│  │  左パネル   │         │    右パネル         │  │
│  │ (地震リスト) │         │  (Leaflet地図)      │  │
│  └────────────┘         └────────────────────┘  │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │        通知エリア (右上)                   │   │
│  │    新しい地震を検知時にポップアップ         │   │
│  └──────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

---

## コード解説

### 1. 署名者フィルタリング

```javascript
const ALLOWED_PUBKEY = "B1A216D31CF6A1F10F393064DD1A447F02AE327FC27359DDC32B07B56021326E";
```

**目的**: 特定の公開鍵で署名されたトランザクションのみを処理  
**理由**: 信頼できる気象データソースからの情報のみを表示するため

#### 処理フロー
```javascript
if (tx.signerPublicKey !== ALLOWED_PUBKEY) return; // フィルタリング
```

---

### 2. 通知システム

#### 通知表示関数
```javascript
function showNotify(hash, j, color) {
  const area = document.getElementById("notifyArea");
  const card = document.createElement("div");
  card.className = "notifyCard";
  card.style.borderLeftColor = color; // 震度に応じた色

  // 通知カードのHTML生成
  card.innerHTML = `<b>新しい地震を検知しました</b><br>...`;

  // クリック時に地図を移動
  card.onclick = () => {
    map.setView([hypo.latitude, hypo.longitude], 7, { animate:true });
  };

  area.prepend(card); // 最新通知を上に追加
  notifyMap[hash] = card; // 管理用マップに格納
}
```

**ポイント**:
- `prepend()`: 新しい通知を上に表示
- `notifyMap`: トランザクションハッシュをキーに通知カードを管理
- クリックで該当地震の位置に地図を移動

#### 通知削除関数
```javascript
function removeNotify(hash) {
  const card = notifyMap[hash];
  if (!card) return;
  card.classList.add("hide"); // フェードアウトアニメーション
  setTimeout(() => card.remove(), 400); // 400ms後にDOM削除
  delete notifyMap[hash];
}
```

---

### 3. 地図表示とマーカー

#### 地図初期化
```javascript
let map = L.map('map').setView([36.5, 137.5], 5); // 日本中心
L.tileLayer("https://cyberjapandata.gsi.go.jp/xyz/std/{z}/{x}/{y}.png", {
  attribution: "地理院タイル",
  maxZoom: 18,
  minZoom: 3
}).addTo(map);
```

**使用タイル**: 国土地理院の標準地図タイル  
**初期位置**: 北緯36.5度、東経137.5度(日本中央部)

#### 震度別カラーリング
```javascript
function getColorByShindo(s, pending, tsunamiType) {
  const tsunamiColor = getTsunamiColor(tsunamiType);
  if (tsunamiColor) return tsunamiColor; // 津波情報優先

  if (pending) return "#ff66cc"; // 未承認はピンク
  if (s<1.5) return "#60a5fa";   // 震度1: 水色
  if (s<2.5) return "#3b82f6";   // 震度2: 青
  if (s<3.5) return "#22c55e";   // 震度3: 緑
  if (s<4.5) return "#eab308";   // 震度4: 黄色
  if (s<5.0) return "#f97316";   // 震度5弱: オレンジ
  if (s<5.8) return "#ea580c";   // 震度5強: 濃いオレンジ
  if (s<6.3) return "#ef4444";   // 震度6弱: 赤
  if (s<6.8) return "#b91c1c";   // 震度6強: 濃い赤
  return "#7f1d1d";              // 震度7: 暗赤
}
```

**優先順位**: 津波情報 > pending状態 > 震度

#### マーカー作成
```javascript
const marker = L.circleMarker(
  [hypo.latitude, hypo.longitude],
  {
    radius: isPending ? 12 : 9,     // pending時は大きく
    fillColor: color,
    color: "#fff",                  // 白い縁取り
    fillOpacity: 0.95
  }
).addTo(map);
```

#### ポップアップの枠線カラー設定
```javascript
marker.on("popupopen", (ev) => {
  const container = ev.popup._container;
  const wrapper = container.querySelector(".leaflet-popup-content-wrapper");
  const tip = container.querySelector(".leaflet-popup-tip");

  if (wrapper) wrapper.style.border = `3px solid ${color}`;
  if (tip) tip.style.border = `3px solid ${color}`;
});
```

**ポイント**: ポップアップ表示時に動的に枠線色を設定

---

### 4. ノード選択

#### ベストノード自動選択
```javascript
async function selectBestNode() {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), 1500); // 1.5秒タイムアウト

  try {
    const res = await fetch(NODEWATCH_URL, { signal: controller.signal });
    clearTimeout(timeoutId);

    const nodes = await res.json();
    nodes.sort((a,b) => b.height - a.height); // ブロック高でソート

    let ep = new URL(nodes[0].endpoint);
    ep.protocol = "https:"; // HTTPSに変換
    SELECTED_NODE = ep.origin;

  } catch {
    SELECTED_NODE = FALLBACK_NODE; // フォールバック
  }
}
```

**戦略**:
1. NodeWatchから最新300件のノードをランダム取得
2. ブロック高でソートし最も高いノードを選択
3. タイムアウト(1.5秒)またはエラー時はフォールバックノードを使用

---

### 5. データ取得と解析

#### 地震データ取得
```javascript
async function loadEarthquake() {
  const address = "NADMA4NNPH2E2XMFGJNTKFYJARRH5VTKXAPUJNQ"; // 地震情報アドレス
  const url = `${SELECTED_NODE}/transactions/confirmed?address=${address}&pageSize=50&order=desc`;

  const res = await fetch(url);
  const data = await res.json();
  const txs = (data.data || []);

  txs.forEach(item => {
    const tx = item.transaction;

    // 署名者フィルタ
    if (tx.signerPublicKey !== ALLOWED_PUBKEY) return;

    // メッセージをHEX→UTF-8変換
    const msg = hexToUtf8(extractMsg(tx));

    // JSON解析
    const j = parseEarthquakeJSON(msg);
    if (!j || !j.earthquake) return;

    // UIに追加
    const added = addEarthquakeToUI(j, false, item.meta.hash, false);
    if (added) confirmedMap[item.meta.hash] = j;
  });
}
```

**処理ステップ**:
1. 承認済みトランザクション最新50件を取得
2. 署名者フィルタリング
3. HEXメッセージをUTF-8にデコード
4. JSON解析
5. UIに追加してconfirmedMapに保存

#### HEX→UTF-8変換
```javascript
function hexToUtf8(hex) {
  if (!hex) return "";
  if (hex.length % 2) hex = "0" + hex; // 奇数長の場合は先頭に0追加

  return new TextDecoder().decode(
    new Uint8Array(hex.match(/.{1,2}/g).map(b => parseInt(b, 16)))
  );
}
```

**仕組み**:
1. HEX文字列を2文字ずつ分割
2. 16進数→10進数変換
3. Uint8Arrayに変換
4. TextDecoderでUTF-8デコード

#### JSON解析(エスケープ処理)
```javascript
function parseEarthquakeJSON(msg) {
  if (!msg) return null;
  let raw = msg.trim();

  // ダブルクォートで囲まれている場合は除去
  if (raw.startsWith('"') && raw.endsWith('"')) raw = raw.slice(1, -1);

  // エスケープ処理
  raw = raw.replace(/\\"/g, '"')            // \" → "
           .replace(/[\u0000-\u001F]+/g, ""); // 制御文字削除

  try {
    return JSON.parse(raw);
  } catch {
    return null;
  }
}
```

---

### 6. WebSocketリアルタイム更新

#### WebSocket接続
```javascript
function startWebSocket() {
  const wsUrl = SELECTED_NODE.replace("https", "wss") + "/ws";
  ws = new WebSocket(wsUrl);

  ws.onmessage = (e) => {
    const data = JSON.parse(e.data);

    // UID取得時
    if (data.uid !== undefined) {
      uid = data.uid;
      subscribeChannels(); // チャンネル登録
      return;
    }

    // データ受信時
    if (funcs[data.topic]) funcs[data.topic].forEach(f => f(data.data));
  };

  ws.onclose = () => {
    uid = "";
    funcs = {};
    setTimeout(startWebSocket, 1500); // 1.5秒後に再接続
  };
}
```

**フロー**:
1. WebSocket接続
2. UIDを受信
3. チャンネルを登録
4. データ受信時にコールバック実行
5. 切断時は自動再接続

#### チャンネル登録
```javascript
function subscribeChannels() {
  const address = "NADMA4NNPH2E2XMFGJNTKFYJARRH5VTKXAPUJNQ";

  // 1. 未承認トランザクション監視
  const ch1 = ListenerChannelName.unconfirmedAdded + "/" + address;
  addCallback(ch1, (txObj) => {
    // 署名者チェック
    if (txObj.transaction.signerPublicKey !== ALLOWED_PUBKEY) return;

    // データ解析
    const msg = hexToUtf8(extractMsg(txObj.transaction));
    const j = parseEarthquakeJSON(msg);
    if (!j || !j.earthquake) return;

    // 通知表示
    const color = getColorByShindo(shindo, true, tsunamiType);
    showNotify(txObj.meta.hash, j, color);

    // UIに追加(pending状態)
    const ui = addEarthquakeToUI(j, true, txObj.meta.hash, true);
    pendingMap[txObj.meta.hash] = ui;
  });
  ws.send(JSON.stringify({uid, subscribe: ch1}));

  // 2. 承認済みトランザクション監視
  const ch2 = ListenerChannelName.confirmedAdded + "/" + address;
  addCallback(ch2, (txObj) => {
    if (txObj.transaction.signerPublicKey !== ALLOWED_PUBKEY) return;

    const msg = hexToUtf8(extractMsg(txObj.transaction));
    const j = parseEarthquakeJSON(msg);
    if (!j || !j.earthquake) return;

    // confirmedMapに保存
    confirmedMap[txObj.meta.hash] = j;

    // pending→confirmed昇格
    promotePendingToConfirmed(txObj.meta.hash, j);
  });
  ws.send(JSON.stringify({uid, subscribe: ch2}));
}
```

**監視チャンネル**:
- `unconfirmedAdded`: 未承認トランザクション(新規地震)
- `confirmedAdded`: 承認済みトランザクション(確定)

#### Pending→Confirmed昇格
```javascript
function promotePendingToConfirmed(hash, j) {
  const item = pendingMap[hash];
  if (!item) return;
  const { marker, card } = item;

  // マーカー更新
  const color = getColorByShindo(shindo, false, tsunamiType);
  marker.setStyle({ fillColor: color, radius: 9 }); // サイズを小さく

  // カード更新
  card.style.borderLeftColor = color;
  card.querySelector(".badgeSymbol").outerHTML =
    `<a class="badgeSymbol" href="https://symbol.fyi/transactions/${hash}" target="_blank">Symbol Explorer</a>`;

  // pendingMapから削除
  delete pendingMap[hash];

  // 通知削除
  removeNotify(hash);
}
```

**処理内容**:
1. マーカーの色・サイズを更新
2. カードバッジを「新着」→「Symbol Explorer」リンクに変更
3. pendingMapから削除
4. 通知を非表示

---

## 使い方

### 1. セットアップ
このHTMLファイルをWebサーバーに配置、またはローカルで開く

```bash
# ローカルサーバー起動例
python -m http.server 8000
# http://localhost:8000 にアクセス
```

### 2. 動作確認
- ページ読み込み時に過去50件の地震情報が表示される
- WebSocketでリアルタイム更新が開始される
- 新しい地震発生時に通知が表示される

### 3. 操作方法
- **地震カードクリック**: 地図上の該当位置に移動
- **マーカークリック**: 詳細情報をポップアップ表示
- **通知カードクリック**: 地図上の該当位置に移動

---

## カスタマイズ

### 署名者公開鍵の変更
```javascript
const ALLOWED_PUBKEY = "あなたの公開鍵";
```

### フォールバックノードの変更
```javascript
const FALLBACK_NODE = "https://your-node.com:3001";
```

### 地震データアドレスの変更
```javascript
const address = "あなたのSymbolアドレス";
```

### 取得件数の変更
```javascript
const url = `${SELECTED_NODE}/transactions/confirmed?address=${address}&pageSize=100&order=desc`;
// pageSize を変更
```

### 色のカスタマイズ
`getColorByShindo()` 関数内の色コードを変更

---

## データフォーマット

### 期待されるJSON構造
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

### 津波情報
- `None`: 津波の心配なし
- `NonEffective`: 若干の海面変動の可能性
- `Watch`: 津波注意報
- `Warning`: 津波警報
- `MajorWarning`: 大津波警報

---

## ライセンス

MIT License

---

## 参考リンク

- [Symbol Documentation](https://docs.symbol.dev/)
- [Leaflet.js](https://leafletjs.com/)
- [国土地理院タイル](https://maps.gsi.go.jp/development/ichiran.html)
- [Symbol Explorer](https://symbol.fyi/)

---

**注意**: このアプリケーションはSymbolブロックチェーン上のデータを可視化するものであり、公式の気象情報システムではありません。実際の防災・避難行動には気象庁の公式情報を参照してください。
