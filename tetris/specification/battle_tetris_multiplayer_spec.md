# 対戦テトリス マルチプレイヤー対応（最大4人）仕様書

## 概要

2人固定の対戦モードを**2〜4人の可変対応**に拡張するための実装仕様を定義する。
Firebaseのリアルタイムデータベースを用いてルーム管理・入室制御・ゲーム進行同期を行う。

---

## 1. Firebaseデータ構造

### 1-1. ルームデータ

```
/rooms/{roomId}/
  maxPlayers:   Number    // 2〜4（ルーム作成時に設定）
  playerCount:  Number    // 現在の入室数
  status:       String    // "waiting" | "playing" | "finished"
  createdAt:    Number    // 作成時のUnixタイムスタンプ
  players/
    playerA/
      uid:         String   // Firebase Auth の UID
      displayName: String   // 表示名
      active:      Boolean  // 入室中かどうか
      ready:       Boolean  // Ready 状態かどうか
      isGameOver:  Boolean  // 敗北済みかどうか
      board:       Array    // 盤面データ（20×10）
      pendingAttack: Array  // 保留中の攻撃キュー
      score:       Number
      ren:         Number
    playerB/  // 同上
    playerC/  // 同上（3〜4人時のみ使用）
    playerD/  // 同上（4人時のみ使用）
```

### 1-2. スロット名とプレイヤー番号の対応

| スロット名  | プレイヤー番号 | 使用条件      |
|:---------:|:-----------:|:-----------:|
| playerA   | Player 1    | 常時使用      |
| playerB   | Player 2    | 常時使用      |
| playerC   | Player 3    | maxPlayers≥3 |
| playerD   | Player 4    | maxPlayers=4 |

---

## 2. ルーム作成

### 2-1. 作成フロー

```
[ルーム作成画面]
      │
      ▼
maxPlayers を選択（2 / 3 / 4）
      │
      ▼
createRoom(maxPlayers) を呼び出し
      │
      ▼
Firebase に以下を書き込む
  status       = "waiting"
  maxPlayers   = 選択値
  playerCount  = 1
  players/playerA = { uid, displayName, active: true, ready: false, isGameOver: false, ... }
      │
      ▼
作成者を playerA として入室
roomId を発行して待機画面へ遷移
```

### 2-2. 実装例

```javascript
async function createRoom(maxPlayers) {
  const roomId  = generateRoomId();   // ランダムなID生成
  const roomRef = ref(db, `/rooms/${roomId}`);

  await set(roomRef, {
    maxPlayers,
    playerCount: 1,
    status:      'waiting',
    createdAt:   Date.now(),
    players: {
      playerA: {
        uid:          auth.currentUser.uid,
        displayName:  auth.currentUser.displayName,
        active:       true,
        ready:        false,
        isGameOver:   false,
        board:        createEmptyBoard(),
        pendingAttack:[],
        score:        0,
        ren:          0,
      }
    }
  });

  return roomId;
}
```

---

## 3. 入室処理

### 3-1. 入室フロー

```
[入室リクエスト]
      │
      ▼
roomId で /rooms/{roomId} を取得
      │
      ├─ status !== "waiting" ──→ エラー「ゲームはすでに開始しています」
      ├─ playerCount >= maxPlayers ──→ エラー「満員です」
      │
      ▼
空いているスロットを検索
（playerA → playerB → playerC → playerD の順）
      │
      ▼
該当スロットに自分のデータを書き込む
playerCount を +1 する
      │
      ▼
待機画面へ遷移（全員の Ready 待ち）
```

### 3-2. 実装例（トランザクション処理）

```javascript
async function joinRoom(roomId) {
  const roomRef = ref(db, `/rooms/${roomId}`);

  await runTransaction(roomRef, (room) => {
    if (!room) return; // ルームが存在しない

    if (room.status !== 'waiting')            return; // 開始済み
    if (room.playerCount >= room.maxPlayers)  return; // 満員

    // 空きスロットを探す
    const slots = ['playerA', 'playerB', 'playerC', 'playerD'];
    const emptySlot = slots.find(s => !room.players?.[s]?.active);
    if (!emptySlot) return;

    // スロットに入室
    room.players = room.players || {};
    room.players[emptySlot] = {
      uid:           auth.currentUser.uid,
      displayName:   auth.currentUser.displayName,
      active:        true,
      ready:         false,
      isGameOver:    false,
      board:         createEmptyBoard(),
      pendingAttack: [],
      score:         0,
      ren:           0,
    };
    room.playerCount += 1;

    return room;
  });
}
```

### 3-3. 自分のスロット名の特定

入室後、`uid` をキーにして自分のスロット名を取得する。

```javascript
function getMySlot(players, myUid) {
  return Object.keys(players).find(
    slot => players[slot]?.uid === myUid
  );
}
// 例: "playerB" が返ってくる
```

---

## 4. 待機画面（ロビー）

### 4-1. Ready 管理

```
[待機画面]
  全スロットをリアルタイム表示
  ├─ active: true  → プレイヤー名を表示
  └─ active: false → 「空き待ち」を表示

  [Ready ボタンを押す]
      │
      ▼
  自分のスロットの ready = true を Firebase に書き込む
      │
      ▼
  全員の ready === true かつ playerCount === maxPlayers であれば
      │
      ▼
  status = "playing" に更新
  → 全クライアントがゲーム画面へ遷移
```

### 4-2. ゲーム開始判定

```javascript
function checkAllReady(players, maxPlayers) {
  const activePlayers = Object.values(players).filter(p => p?.active);
  if (activePlayers.length < maxPlayers) return false;
  return activePlayers.every(p => p.ready === true);
}

// onValue で players の変化を監視して自動チェック
onValue(playersRef, (snapshot) => {
  const players = snapshot.val() || {};
  if (checkAllReady(players, maxPlayers)) {
    set(ref(db, `/rooms/${roomId}/status`), 'playing');
  }
});
```

---

## 5. ゲーム中の処理

### 5-1. 画面レイアウト（人数別）

プレイヤー数に応じてレイアウトを切り替える。

```
[2人]
┌──────────────┬──────────────┐
│   Player 1   │   Player 2   │
└──────────────┴──────────────┘

[3人]
┌──────────┬──────────┬──────────┐
│ Player 1 │ Player 2 │ Player 3 │
└──────────┴──────────┴──────────┘

[4人]
┌──────────┬──────────┬──────────┬──────────┐
│ Player 1 │ Player 2 │ Player 3 │ Player 4 │
└──────────┴──────────┴──────────┴──────────┘
```

自分の操作画面はスロット順に対応するパネルで行う。
他プレイヤーの盤面は**縮小表示**（自分の盤面より小さくしても可）で並べる。

### 5-2. 攻撃ターゲットの決定

```javascript
// 攻撃先：自分以外の生存プレイヤーからランダムに1人を選ぶ
function selectAttackTarget(players, mySlot) {
  const targets = Object.keys(players).filter(slot =>
    slot !== mySlot &&
    players[slot]?.active &&
    !players[slot]?.isGameOver
  );
  if (targets.length === 0) return null;
  return targets[Math.floor(Math.random() * targets.length)];
}
```

> **補足：** 将来的にターゲット手動選択（特定の相手を狙う）を実装する場合は、UIにターゲット選択ボタンを追加し、`selectedTarget` 変数で管理する。

### 5-3. 生存者カウントと勝敗判定

```javascript
// 生存者数を取得
function getAlivePlayers(players) {
  return Object.values(players).filter(
    p => p?.active && !p?.isGameOver
  );
}

// 敗北処理
async function onGameOver(roomId, mySlot) {
  await update(ref(db, `/rooms/${roomId}/players/${mySlot}`), {
    isGameOver: true,
  });
}

// 勝利判定（onValue で監視）
onValue(playersRef, (snapshot) => {
  const players  = snapshot.val() || {};
  const alive    = getAlivePlayers(players);
  const myPlayer = players[mySlot];

  // 生存者が1人 かつ 自分が生き残っている
  if (alive.length === 1 && !myPlayer?.isGameOver) {
    triggerGameEnd('win');
    set(ref(db, `/rooms/${roomId}/status`), 'finished');
  }
});
```

---

## 6. 退室・切断処理

### 6-1. 正常退室

```javascript
async function leaveRoom(roomId, mySlot) {
  const updates = {};
  updates[`players/${mySlot}/active`]    = false;
  updates[`players/${mySlot}/isGameOver`] = true;
  updates['playerCount'] = increment(-1);
  await update(ref(db, `/rooms/${roomId}`), updates);
}
```

### 6-2. 異常切断（onDisconnect）

入室時に `onDisconnect` を設定し、ブラウザを閉じた場合にも自動で退室処理が走るようにする。

```javascript
function setDisconnectHandler(roomId, mySlot) {
  const slotRef = ref(db, `/rooms/${roomId}/players/${mySlot}`);
  onDisconnect(slotRef).update({
    active:      false,
    isGameOver:  true,
  });

  const countRef = ref(db, `/rooms/${roomId}/playerCount`);
  onDisconnect(countRef).set(increment(-1));
}
```

### 6-3. 切断プレイヤーの扱い

| 状況              | 処理                                      |
|:----------------:|:-----------------------------------------|
| waiting 中に切断  | スロットを空き状態に戻し、別プレイヤーが入室可能にする |
| playing 中に切断  | `isGameOver: true` として敗北扱いにする     |
| 残り1人になった    | 残ったプレイヤーを自動的に勝者として扱う      |

---

## 7. 実装フロー全体図

```
[ルーム作成]
  createRoom(maxPlayers)
  → Firebase にルームデータ書き込み
  → 作成者を playerA として入室
        │
        ▼
[待機画面（ロビー）]
  onValue で players を監視
  → 入室者のリアルタイム表示
  → 自分が Ready → ready = true を書き込み
  → 全員 ready → status = "playing"
        │
        ▼
[ゲーム画面]
  getMySlot() で自分のスロットを特定
  人数に応じたレイアウトで盤面を並べる
        │
  [ゲーム中ループ]
  ├─ 攻撃送信: selectAttackTarget() → 相手の pendingAttack に加算
  ├─ 受信:     自分の pendingAttack を監視 → ゲージ表示・相殺
  └─ 敗北:     onGameOver() → isGameOver = true
        │
        ▼
[勝敗判定]
  alive === 1 かつ 自分が生存
  → triggerGameEnd('win')
  → status = "finished"
        │
        ▼
[リザルト画面]
  最終順位・スコアを表示
  → ルームに戻る / ホームに戻る
```

---

## 8. 定数・設定値一覧

| 定数名                 | 値                               | 説明                            |
|:---------------------|:--------------------------------:|:-------------------------------|
| `MIN_PLAYERS`        | 2                                | 最小プレイヤー数                  |
| `MAX_PLAYERS`        | 4                                | 最大プレイヤー数                  |
| `PLAYER_SLOTS`       | `['playerA','playerB','playerC','playerD']` | スロット名の配列    |
| `STATUS_WAITING`     | `'waiting'`                      | 募集中状態                       |
| `STATUS_PLAYING`     | `'playing'`                      | 対戦中状態                       |
| `STATUS_FINISHED`    | `'finished'`                     | 終了状態                         |

---

*作成日：2026-04-19*
