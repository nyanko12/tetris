# テトリス オンライン対戦 実装仕様書（改訂版）

> 初版から以下を修正・追加：Race Condition対策、攻撃加算競合対策、勝利カウント設計、切断処理、待機画面、gameState管理、カウントダウン同期

---

## ファイル構成

```
tetris/
├── tetris.html          # メインファイル
├── tetris.css           # スタイル
├── firebase-config.js   # Firebase設定（.gitignore済み）
└── MULTIPLAYER_SPEC.md  # この仕様書
```

---

## 1. 画面構成と遷移

```
[タイトル画面]
  ↓ ルームIDを入力して「対戦を始める」
[URLリロード: ?room=ID]
  ↓ Firebase初期化完了後に自動で部屋に参加
[待機画面]  ← NEW（タイトル画面とは別の専用画面）
  ↓ 両者が「準備完了」を押す
[カウントダウン演出（3→2→1→GO）]
  ↓
[ゲーム画面（対戦中）]
  ↓ どちらかがゲームオーバー
[ラウンドリザルト画面]
  ↓ maxWinsに達した場合はマッチ終了 / 未達なら「再戦」で待機画面へ
[マッチ終了画面]
```

### 画面ID一覧

| ID | 説明 |
|----|------|
| `title-screen` | タイトル・シングルプレイ入口 |
| `waiting-screen` | 対戦待機・準備完了ボタン（NEW） |
| `game-screen` | 対戦中のゲーム画面 |
| `result-overlay` | ラウンド/マッチ終了結果 |

---

## 2. データベース構造

```
rooms/
  {ROOM_ID}/
    config/
      maxWins:    3          # 何本先取か（player1が入室時に書き込む）
      gameState:  "waiting"  # waiting | ready | countdown | playing | result

    player1/
      name:       "Player1"  # 待機画面で入力したニックネーム
      ready:      false      # 準備完了ボタンを押したか
      board:      [[...]]    # 20×10の盤面配列（ブロック着地ごとに送信）
      score:      0          # 現在のスコア
      attackQueue: []        # 攻撃ラインのキュー（競合防止のため配列）
      wins:       0          # 通算勝利数（自分で自分のwinsを+1する）
      isGameOver: false      # このラウンドのゲームオーバーフラグ

    player2/
      name:       "Player2"
      ready:      false
      board:      [[...]]
      score:      0
      attackQueue: []
      wins:       0
      isGameOver: false
```

### gameState 状態遷移

```
waiting   : 誰かが入室した直後。相手を待っている状態
ready     : 両者が「準備完了」を押した直後（カウントダウン開始トリガー）
countdown : カウントダウン演出中（この状態を検知した側が演出を開始）
playing   : ゲーム進行中
result    : ラウンド終了（どちらかのisGameOverがtrue）
```

---

## 3. 入室処理（Race Condition対策）

### 問題
`onValue` で「player1が空なら登録」すると、同時入室時に両者がplayer1になる。

### 解決策：`runTransaction()` で原子的に読み書き

```js
import { runTransaction } from "firebase/database";

function joinRoom(nickname) {
  const roomRef = _ref(_db, `rooms/${ROOM_ID}`);

  runTransaction(roomRef, (current) => {
    if (current === null) {
      // 部屋が存在しない → player1として作成
      return {
        config: { maxWins: 3, gameState: 'waiting' },
        player1: makePlayerData(nickname),
      };
    }
    if (!current.player1 || !current.player1.ready === undefined) {
      // player1の枠が空 → player1として登録
      current.player1 = makePlayerData(nickname);
      return current;
    }
    if (!current.player2 || current.player2.ready === undefined) {
      // player2の枠が空 → player2として登録
      current.player2 = makePlayerData(nickname);
      return current;
    }
    // 満員 → transactionを中断（undefinedを返すとキャンセル）
    return undefined;
  }).then(result => {
    if (!result.committed) {
      showWaitingStatus('ルームが満員です');
      return;
    }
    // 自分の役割を確定してから待機画面へ
    determineMyRole();
    showPage('waiting');
  });
}

function makePlayerData(nickname) {
  return {
    name: nickname || 'Player',
    ready: false,
    board: null,
    score: 0,
    attackQueue: [],
    wins: 0,
    isGameOver: false,
  };
}
```

### 役割の確定

```js
let myRole = null; // 'player1' | 'player2'

function determineMyRole() {
  // transactionの書き込み結果から自分の役割を特定する
  // （入室直後に一度だけ読み取り）
  _onValue(_ref(_db, `rooms/${ROOM_ID}`), snap => {
    const data = snap.val();
    // 自分のnameが入っている方が自分の役割
    if (data.player1 && data.player1.name === myNickname) myRole = 'player1';
    else if (data.player2 && data.player2.name === myNickname) myRole = 'player2';
  }, { onlyOnce: true });
}
```

> **注意**: ニックネームで判定する代わりに、`sessionStorage` に保存した UUID を使う方が確実。

---

## 4. 切断処理（onDisconnect）

### 問題
タブを閉じるとルームデータが残り、相手が無限に待ち続ける。

### 解決策：`onDisconnect()` で自動クリーンアップ

```js
import { onDisconnect } from "firebase/database";

function registerDisconnectHandler() {
  const myRef = _ref(_db, `rooms/${ROOM_ID}/${myRole}`);

  // 切断時に自分のデータを削除
  onDisconnect(myRef).remove();

  // gameState も waiting に戻す（相手に切断を知らせる）
  onDisconnect(_ref(_db, `rooms/${ROOM_ID}/config/gameState`)).set('waiting');
}
```

### 相手の切断を検知する

```js
function watchOpponentDisconnect() {
  const opRole = myRole === 'player1' ? 'player2' : 'player1';
  _onValue(_ref(_db, `rooms/${ROOM_ID}/${opRole}`), snap => {
    if (snap.val() === null && gameState === 'playing') {
      // 対戦中に相手データが消えた → 切断とみなす
      showResultOverlay('相手が切断しました', 'あなたの勝ち！');
      cleanupRoom();
    }
  });
}
```

---

## 5. 待機画面

### HTML

```html
<div id="waiting-screen" class="page hidden">
  <div id="waiting-inner">
    <h2>対戦待機中</h2>
    <p id="waiting-room-id">ルームID: <strong id="waiting-room-label"></strong></p>
    <p id="waiting-hint">このIDを相手に共有してください</p>

    <div id="waiting-players">
      <div class="player-slot" id="slot-p1">
        <span class="slot-name" id="slot-p1-name">待機中...</span>
        <span class="slot-ready" id="slot-p1-ready">○</span>
      </div>
      <div class="player-slot" id="slot-p2">
        <span class="slot-name" id="slot-p2-name">待機中...</span>
        <span class="slot-ready" id="slot-p2-ready">○</span>
      </div>
    </div>

    <div id="waiting-wins">
      <!-- 再戦時の累計勝利数表示 -->
      <span id="wins-p1">0</span> 勝 - <span id="wins-p2">0</span> 勝
    </div>

    <button id="ready-btn" disabled>準備完了</button>
    <p id="waiting-status"></p>
    <button id="waiting-cancel-btn">キャンセル</button>
  </div>
</div>
```

### 待機画面のロジック

```js
function initWaitingScreen() {
  document.getElementById('waiting-room-label').textContent = ROOM_ID;

  // 相手の入室を監視してスロットに表示
  _onValue(_ref(_db, `rooms/${ROOM_ID}`), snap => {
    const data = snap.val() || {};
    updatePlayerSlot('p1', data.player1);
    updatePlayerSlot('p2', data.player2);

    // 両プレイヤーが揃ったら準備完了ボタンを有効化
    if (data.player1 && data.player2) {
      document.getElementById('ready-btn').disabled = false;
    }

    // 勝利数の更新
    document.getElementById('wins-p1').textContent = data.player1?.wins ?? 0;
    document.getElementById('wins-p2').textContent = data.player2?.wins ?? 0;
  });
}

// 準備完了ボタン
document.getElementById('ready-btn').addEventListener('click', () => {
  document.getElementById('ready-btn').disabled = true;
  _set(_ref(_db, `rooms/${ROOM_ID}/${myRole}/ready`), true);
  document.getElementById('waiting-status').textContent = '相手の準備を待っています...';
});

// キャンセルボタン
document.getElementById('waiting-cancel-btn').addEventListener('click', () => {
  cleanupRoom();
  showPage('title');
});
```

---

## 6. カウントダウン同期

### 問題
各自で `setTimeout` するとネットワーク遅延でタイミングがずれる。

### 解決策：`gameState` の変更をトリガーにする

```js
// player1が両者のreadyを検知したら gameState を countdown に変更
function watchBothReady() {
  _onValue(_ref(_db, `rooms/${ROOM_ID}`), snap => {
    const data = snap.val();
    if (
      data?.player1?.ready &&
      data?.player2?.ready &&
      data?.config?.gameState === 'waiting' &&
      myRole === 'player1'  // player1だけが書き込む（二重書き込み防止）
    ) {
      _set(_ref(_db, `rooms/${ROOM_ID}/config/gameState`), 'countdown');
    }
  });
}

// 全員が gameState の変化を監視してカウントダウンを開始
function watchGameState() {
  _onValue(_ref(_db, `rooms/${ROOM_ID}/config/gameState`), snap => {
    const state = snap.val();
    if (state === 'countdown') {
      startCountdownAnimation(() => {
        _set(_ref(_db, `rooms/${ROOM_ID}/config/gameState`), 'playing');
        showPage('game');
        startGame();
      });
    }
    if (state === 'result') {
      showResultOverlay();
    }
  });
}

// カウントダウン演出（3→2→1→GO）
function startCountdownAnimation(callback) {
  showPage('countdown');
  let count = 3;
  const tick = () => {
    document.getElementById('countdown-number').textContent = count > 0 ? count : 'GO!';
    if (count-- >= 0) setTimeout(tick, 1000);
    else callback();
  };
  tick();
}
```

> **ポイント**: `gameState = 'countdown'` の書き込みは `player1` だけが行い、
> 両者は `onValue` でその変化を受け取ることで同時にカウントダウンを開始する。

---

## 7. 攻撃ラインの送受信（競合防止）

### 問題
`attack` を単一数値にすると、両者が同時に書き込んだとき片方が上書きされる。

### 解決策：攻撃をキュー（配列）で管理

#### 攻撃の送信側（自分がラインを消したとき）

```js
import { push } from "firebase/database";

function sendAttack(clearedLines) {
  const attackLines = [0, 0, 1, 2, 4][clearedLines] ?? 4;
  if (attackLines === 0) return;

  const opRole = myRole === 'player1' ? 'player2' : 'player1';
  // push() でキューに追記（上書きにならない）
  push(_ref(_db, `rooms/${ROOM_ID}/${opRole}/attackQueue`), attackLines);
}
```

#### 攻撃の受信側（自分のキューを監視）

```js
function watchAttackQueue() {
  _onValue(_ref(_db, `rooms/${ROOM_ID}/${myRole}/attackQueue`), snap => {
    const queue = snap.val();
    if (!queue || gameOver) return;

    // キューにある全攻撃を合算して適用
    const total = Object.values(queue).reduce((sum, n) => sum + n, 0);
    addGarbageLines(total);

    // 処理済みとしてキューをクリア
    _set(_ref(_db, `rooms/${ROOM_ID}/${myRole}/attackQueue`), null);
  });
}
```

---

## 8. ゲーム中の同期

### 8-1. 盤面の送信（ブロック着地ごと）

```js
// lockPiece() 末尾に追加
function lockPiece() {
  // 既存処理...
  if (IS_ONLINE && myRole) {
    _set(_ref(_db, `rooms/${ROOM_ID}/${myRole}/board`), board);
    _set(_ref(_db, `rooms/${ROOM_ID}/${myRole}/score`), score);
  }
}
```

### 8-2. 相手盤面の受信と描画

```js
function listenOpponentBoard() {
  const opRole = myRole === 'player1' ? 'player2' : 'player1';
  _onValue(_ref(_db, `rooms/${ROOM_ID}/${opRole}/board`), snap => {
    if (snap.val()) drawOpponentBoard(snap.val());
  });
  _onValue(_ref(_db, `rooms/${ROOM_ID}/${opRole}/score`), snap => {
    document.getElementById('opponent-score-display').textContent = snap.val() ?? 0;
  });
}
```

---

## 9. ラウンド決着と勝利カウント

### 設計方針
- **自分がゲームオーバーになったら** `isGameOver = true` を自分で書く
- **相手の `isGameOver` が `true` になったら** 自分の `wins` を `+1` する
- → 「自分の `wins` は自分だけが触る」ルールで競合を防ぐ

```js
// ゲームオーバー時（自分の処理）
function handleGameOver() {
  gameOver = true;
  if (IS_ONLINE && myRole) {
    _set(_ref(_db, `rooms/${ROOM_ID}/${myRole}/isGameOver`), true);
  }
}

// 相手のゲームオーバーを監視
function watchOpponentGameOver() {
  const opRole = myRole === 'player1' ? 'player2' : 'player1';
  _onValue(_ref(_db, `rooms/${ROOM_ID}/${opRole}/isGameOver`), snap => {
    if (snap.val() === true && !gameOver) {
      // 相手が先に死んだ → 自分の勝ち
      _set(_ref(_db, `rooms/${ROOM_ID}/${myRole}/wins`),
        (currentWins + 1));  // currentWinsは入室時にローカルに保持

      // ラウンド終了をgameStateで通知
      _set(_ref(_db, `rooms/${ROOM_ID}/config/gameState`), 'result');
    }
  });
}
```

### マッチ終了判定（リザルト画面で行う）

```js
function showResultOverlay() {
  _onValue(_ref(_db, `rooms/${ROOM_ID}`), snap => {
    const data = snap.val();
    const p1Wins = data.player1?.wins ?? 0;
    const p2Wins = data.player2?.wins ?? 0;
    const maxWins = data.config?.maxWins ?? 3;

    if (p1Wins >= maxWins || p2Wins >= maxWins) {
      // マッチ終了
      const winner = p1Wins >= maxWins ? data.player1?.name : data.player2?.name;
      showMatchEnd(winner);
    } else {
      // ラウンド終了 → 再戦ボタンを表示
      showRoundEnd(p1Wins, p2Wins);
    }
  }, { onlyOnce: true });
}
```

---

## 10. 再戦処理

### 再戦時にリセットするもの・しないもの

| フィールド | 再戦時 | 説明 |
|-----------|--------|------|
| `wins` | **保持** | 通算勝利数なので継続 |
| `board` | リセット（null） | 盤面を空にする |
| `score` | リセット（0） | スコアをリセット |
| `isGameOver` | リセット（false） | 生存フラグを戻す |
| `ready` | リセット（false） | 準備完了を再度押させる |
| `attackQueue` | リセット（null） | 残留攻撃を消去 |
| `config.gameState` | `'waiting'` に戻す | 待機画面に遷移させる |

```js
// 再戦ボタン押下
function rematch() {
  const resetData = {
    board: null,
    score: 0,
    isGameOver: false,
    ready: false,
    attackQueue: null,
  };
  _update(_ref(_db, `rooms/${ROOM_ID}/${myRole}`), resetData);

  // 両者がリセットしたら gameState を waiting に戻す
  // （player1が担当）
  if (myRole === 'player1') {
    _set(_ref(_db, `rooms/${ROOM_ID}/config/gameState`), 'waiting');
  }

  showPage('waiting');
  initWaitingScreen();
}
```

---

## 11. 邪魔ブロック（ガベージライン）

```js
function addGarbageLines(count) {
  // 上から count 行分を押し出す（ブロックが天井を超えると即ゲームオーバー）
  board.splice(0, count);
  for (let i = 0; i < count; i++) {
    const hole = Math.floor(Math.random() * COLS);
    const row  = new Array(COLS).fill(8); // 8 = ガベージ色
    row[hole]  = 0;  // ランダムな1マスだけ穴を開ける
    board.push(row);
  }
  draw();
}
```

### COLORS配列（`tetris.html` 既存配列に追記）

```js
const COLORS = [
  null,
  '#00f0f0', // I
  '#f0f000', // O
  '#a000f0', // T
  '#00f000', // S
  '#f00000', // Z
  '#0000f0', // J
  '#f0a000', // L
  '#888888', // 8: ガベージ ← 追加
];
```

---

## 12. 相手フィールドの描画

### HTML（`#game-container` 内に追加）

```html
<div id="opponent-col" class="hidden">
  <div class="panel-box">
    <h3 id="opponent-label">OPPONENT</h3>
    <div class="value" style="font-size:16px;" id="opponent-score-display">0</div>
  </div>
  <canvas id="opponent-board" width="150" height="300"></canvas>
</div>
```

### 描画関数

```js
const opCanvas = document.getElementById('opponent-board');
const opCtx    = opCanvas.getContext('2d');
const OP_CELL  = 15; // 自分の半分サイズ（CELL=30の半分）

function drawOpponentBoard(opBoard) {
  opCtx.clearRect(0, 0, opCanvas.width, opCanvas.height);
  opBoard.forEach((row, r) => {
    row.forEach((val, c) => {
      if (!val) return;
      opCtx.fillStyle = COLORS[val] || '#888';
      opCtx.fillRect(c * OP_CELL + 1, r * OP_CELL + 1, OP_CELL - 2, OP_CELL - 2);
    });
  });
}
```

---

## 13. 攻撃ライン換算表

| 消去ライン数 | 相手への攻撃 |
|------------|------------|
| 1          | 0（攻撃なし） |
| 2          | 1ライン     |
| 3          | 2ライン     |
| 4（テトリス） | 4ライン    |

---

## 14. 実装順序（推奨）

1. `waiting-screen` のHTML・CSSを追加
2. `showPage()` に `'waiting'` を追加
3. Firebase の `runTransaction` で入室処理を実装
4. `onDisconnect()` の登録
5. 待機画面の表示ロジック（`initWaitingScreen()`）
6. `gameState` の監視と `watchBothReady()` / `watchGameState()` の実装
7. カウントダウン同期（`startCountdownAnimation()`）
8. `sendAttack()` / `watchAttackQueue()` の実装（攻撃キュー方式）
9. `lockPiece()` に盤面送信を追加
10. `clearLines()` に攻撃送信（`sendAttack()`）を追加
11. `handleGameOver()` にゲームオーバー送信を追加
12. `watchOpponentGameOver()` で勝利判定
13. リザルト画面と再戦処理（`rematch()`）
14. 相手フィールドの描画（`drawOpponentBoard()`）

---

## 15. セキュリティルール（Firebase）

```json
{
  "rules": {
    "rooms": {
      "$roomId": {
        ".read": true,
        ".write": true,
        "player1": {
          "wins": {
            // wins を書けるのは自分のロールのみ（認証なしの場合は暫定的にtrue）
            ".write": true
          }
        },
        "player2": {
          "wins": {
            ".write": true
          }
        }
      }
    }
  }
}
```

> **注意**: 認証（Firebase Auth）を導入していない場合はルールによる完全な保護は困難。
> 本番公開時は匿名認証を追加し、自分のロールのデータだけ書けるようにすることを推奨。

---

## 16. 既知の制約と注意事項

| 項目 | 内容 |
|------|------|
| 同時入室 | `runTransaction` で対策済み。ただし三者以上の入室は「満員」として弾く |
| 切断 | `onDisconnect` で対策済み。再接続機能はなし（切断 = 敗北扱い） |
| 攻撃競合 | キュー（配列）方式で対策済み |
| 勝利カウント競合 | 「自分のwinsは自分だけが+1」ルールで対策済み |
| データ残留 | 対戦終了またはキャンセル時に `remove()` でルームを削除すること |
| ニックネーム重複 | 同じニックネームだとロール判定が誤る可能性あり（UUIDを使うと確実） |
