# テトリス

ブラウザで動くテトリスゲームです。1人プレイとオンライン対戦（2人）に対応しています。

## 機能

### 1人プレイ
- 難易度選択（EASY / NORMAL / HARD）でスタートレベルを変更可能
- スコア・レベル・消去ライン数をリアルタイム表示
- ハイスコア上位5件をタイトル画面に表示（localStorage保存）
- HOLD機能・NEXT表示・ゴーストピース（落下先プレビュー）
- ゲームオーバーアニメーション演出

### オンライン対戦
- ルームIDを共有して2人でリアルタイム対戦
- Firebase Realtime Database でゲーム状態を同期
- 先取3本制（ラウンド制）・再戦機能
- 攻撃ライン（ガベージライン）送受信
  - 2ライン消し → 相手に1ライン
  - 3ライン消し → 相手に2ライン
  - テトリス（4ライン）→ 相手に4ライン
- 相手のフィールドをリアルタイム表示
- 切断時の自動クリーンアップ（onDisconnect）
- 同時入室の競合をトランザクションで防止

### その他
- BGM・効果音（タイトル / ゲーム中 / ゲームオーバー）
- キー設定のカスタマイズ（設定はlocalStorageに保存）
- 一時停止機能

## 操作方法

| キー | 操作 |
|------|------|
| `← →` | 左右移動 |
| `↑` | 回転 |
| `↓` | 低速落下 |
| `Space` | 即落下（ハードドロップ） |
| `C` | ホールド |
| `P` | 一時停止 |

※ キー設定画面（⚙ キー設定）から変更可能

## オンライン対戦の始め方

1. タイトル画面でニックネームとルームIDを入力し「対戦を始める」を押す
2. 相手に同じルームIDを共有する
3. 両者が待機画面で「準備完了」を押すとカウントダウン開始
4. キャンセルを押すとタイトルに戻る（URLパラメータも自動でリセット）

## ファイル構成

```
tetris/
├── tetris.html          # メインファイル（ゲームロジック含む）
├── tetris.css           # スタイル
├── firebase-config.js   # Firebase設定（.gitignore済み）
├── MULTIPLAYER_SPEC.md  # オンライン対戦の実装仕様書
├── music/               # BGM・効果音ファイル
└── README.md            # このファイル
```

## 技術スタック

- HTML / CSS / JavaScript（バニラ、フレームワークなし）
- [Firebase Realtime Database](https://firebase.google.com/products/realtime-database)（オンライン対戦の状態同期）

## セットアップ

1. リポジトリをクローン
2. `firebase-config.js` を作成し、自分のFirebaseプロジェクトの設定を記述

```js
// firebase-config.js
export const firebaseConfig = {
  apiKey: "...",
  authDomain: "...",
  databaseURL: "...",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "..."
};
```

3. `tetris.html` をブラウザで開く（またはローカルサーバー経由で開く）
