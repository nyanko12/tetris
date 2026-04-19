# 対戦テトリス ターゲットシステム・攻撃昇華 仕様書

## 概要

攻撃先の選択モード（ターゲットモード）・消し方による単体/全体攻撃の切り替え（攻撃の昇華）・ターゲットの視覚的可視化の実装仕様を定義する。

---

## 1. ターゲットモードの設計

### 1-1. 選択肢一覧

| モード名      | 識別子        | 動作                                              |
|:-----------:|:-----------:|:------------------------------------------------|
| ランダム      | `random`    | 攻撃のたびに生存者からランダムに1人を選ぶ              |
| トドメ        | `ko`        | 盤面が最も高く積み上がっている（死に近い）人を狙う      |
| カウンター    | `counter`   | 自分を最後に攻撃してきた人に反撃する。該当なしの場合はランダム |

### 1-2. Firebase への保存

```
/rooms/{roomId}/players/{slot}/
  targetMode: String   // "random" | "ko" | "counter"
```

### 1-3. UI 配置

待機画面（ロビー）の自分のスロット付近に切り替えボタンを設置する。
ゲーム中も操作画面の盤面外（HOLDパネル下など）に常時表示して切り替え可能にする。

```
┌─────────────┐
│  HOLD       │
├─────────────┤
│ TARGET      │
│ [Random]    │  ← クリックで切り替え
│ [K.O.  ]    │
│ [Counter]   │
└─────────────┘
```

---

## 2. ターゲット決定ロジック

### 2-1. getTargetSlot()

ラインを消した瞬間に呼び出し、現在のターゲットモードに基づいて攻撃先スロットを決定する。

```javascript
function getTargetSlot(mySlot, targetMode, allPlayers, lastAttackerSlot) {
  // 生存している敵のみ抽出
  const enemies = Object.entries(allPlayers).filter(
    ([slot, p]) => slot !== mySlot && p?.active && !p?.isGameOver
  );
  if (enemies.length === 0) return null;

  switch (targetMode) {
    case 'ko':
      // 盤面が最も高く積み上がっている（fieldHeight が最大）の敵を選ぶ
      return enemies.sort(([, a], [, b]) =>
        b.fieldHeight - a.fieldHeight
      )[0][0];

    case 'counter':
      // 最後に自分を攻撃してきたスロットが生存していればそこを狙う
      const counterTarget = enemies.find(([slot]) => slot === lastAttackerSlot);
      if (counterTarget) return counterTarget[0];
      // いなければランダム
      return enemies[Math.floor(Math.random() * enemies.length)][0];

    case 'random':
    default:
      return enemies[Math.floor(Math.random() * enemies.length)][0];
  }
}
```

### 2-2. lastAttackerSlot の管理

相手から攻撃を受けた際に `lastAttackerSlot` を更新する。攻撃データに送信元スロットを含める。

```javascript
// 攻撃データの構造（pendingAttack の各要素）
{
  lines:       Number,  // 攻撃ライン数
  type:        String,  // "normal" | "tetris" | "tspin" | "ren"
  fromSlot:    String,  // 送信元スロット（例: "playerA"）
  timestamp:   Number,
}

// 受信時に lastAttackerSlot を更新
onValue(pendingRef, (snapshot) => {
  const pending = snapshot.val() || [];
  if (pending.length > 0) {
    const latest = pending[pending.length - 1];
    lastAttackerSlot = latest.fromSlot;
  }
});
```

### 2-3. fieldHeight の計算と同期

盤面の積み上がり高さを毎ターン計算し、Firebaseに書き込む。

```javascript
// 盤面の最高行（上から何行目にブロックがあるか）を計算
function calcFieldHeight(board) {
  const firstFilledRow = board.findIndex(row => row.some(cell => cell !== EMPTY));
  if (firstFilledRow === -1) return 0;
  return BOARD_HEIGHT - firstFilledRow; // 下から何段積まれているか
}

// ミノ設置後に Firebase に書き込む
update(ref(db, `/rooms/${roomId}/players/${mySlot}`), {
  fieldHeight: calcFieldHeight(board),
});
```

---

## 3. 攻撃の昇華システム

### 3-1. 単体攻撃 vs 全体攻撃の判定

| 消し方               | 攻撃タイプ | ターゲット       |
|:------------------:|:--------:|:--------------:|
| 1〜3ライン消し        | 単体攻撃   | ターゲットモードで決定した1人 |
| 通常 T-Spin          | 単体攻撃   | 同上              |
| 4ライン消し（テトリス） | 全体攻撃   | 生存者全員        |
| REN 7以上            | 全体攻撃   | 生存者全員        |

```javascript
function isAllTargetAttack(clearType, ren) {
  if (clearType === 'tetris') return true;
  if (ren >= ALL_ATTACK_REN_THRESHOLD) return true; // 閾値: 7
  return false;
}
```

### 3-2. 全体攻撃時の火力補正

1人を狙う時と全員を狙う時では、1人当たりの攻撃力を弱体化してゲームバランスを保つ。

```javascript
const ALL_ATTACK_MULTIPLIER = 0.7; // 全体攻撃時の1人当たり火力倍率

function calcDamageForTarget(baseDamage, isAllTarget) {
  if (!isAllTarget) return baseDamage;
  return Math.max(1, Math.floor(baseDamage * ALL_ATTACK_MULTIPLIER));
}
```

| 消し方   | 基礎火力 | 単体時  | 全体時（×0.7） |
|:-------:|:-------:|:------:|:------------:|
| テトリス  | 4       | 4      | 2〜3          |
| 高REN    | 3+      | 3+     | 2+           |

---

## 4. 攻撃送信の統合フロー

### 4-1. 攻撃処理の全体フロー

```javascript
async function processAttack(clearType, ren, board) {
  // 1. 基礎火力を計算
  const baseDamage = calcBaseDamage(clearType, ren);
  if (baseDamage <= 0) return;

  // 2. 相殺優先：自分の pendingAttack を先に削る
  const { remaining, newPending } = cancelWithPending(baseDamage, localPending);
  localPending = newPending;
  await set(ref(db, `/rooms/${roomId}/players/${mySlot}/pendingAttack`), localPending);

  // 相殺で全部消えたら終了
  if (remaining <= 0) return;

  // 3. 単体 or 全体 の判定
  const allTarget = isAllTargetAttack(clearType, ren);
  const damage    = calcDamageForTarget(remaining, allTarget);

  if (allTarget) {
    // 4a. 全体攻撃：生存者全員に送信
    await sendAttackToAll(damage);
  } else {
    // 4b. 単体攻撃：ターゲットモードで1人を決定
    const allPlayers = (await get(playersRef)).val();
    const targetSlot = getTargetSlot(mySlot, targetMode, allPlayers, lastAttackerSlot);
    if (targetSlot) await sendAttackToOne(targetSlot, damage);
  }
}
```

### 4-2. sendAttackToOne / sendAttackToAll

```javascript
async function sendAttackToOne(targetSlot, damage) {
  const targetPendingRef = ref(db,
    `/rooms/${roomId}/players/${targetSlot}/pendingAttack`
  );
  const snapshot = await get(targetPendingRef);
  const current  = snapshot.val() || [];

  current.push({
    lines:     damage,
    type:      currentClearType,
    fromSlot:  mySlot,
    timestamp: Date.now(),
  });
  await set(targetPendingRef, current);
}

async function sendAttackToAll(damage) {
  const allPlayers = (await get(playersRef)).val();
  const promises   = Object.entries(allPlayers)
    .filter(([slot, p]) => slot !== mySlot && p?.active && !p?.isGameOver)
    .map(([slot]) => sendAttackToOne(slot, damage));
  await Promise.all(promises);
}
```

---

## 5. 視覚演出

### 5-1. ターゲットライン

自分の盤面から現在狙っている相手の盤面へ向かって、細い光の線を常時表示する。

```javascript
function drawTargetLine(ctx, myBoardRect, targetBoardRect, isAllTarget) {
  const startX = myBoardRect.x + myBoardRect.w / 2;
  const startY = myBoardRect.y + myBoardRect.h / 2;

  const targets = isAllTarget
    ? getAllEnemyRects()      // 全体攻撃：全員に向かって線を引く
    : [getTargetRect()];     // 単体攻撃：1人だけ

  targets.forEach(rect => {
    const endX = rect.x + rect.w / 2;
    const endY = rect.y + rect.h / 2;

    const gradient = ctx.createLinearGradient(startX, startY, endX, endY);
    gradient.addColorStop(0, isAllTarget ? 'rgba(255,200,0,0.8)' : 'rgba(255,80,80,0.6)');
    gradient.addColorStop(1, 'rgba(255,255,255,0)');

    ctx.beginPath();
    ctx.moveTo(startX, startY);
    ctx.lineTo(endX, endY);
    ctx.strokeStyle = gradient;
    ctx.lineWidth   = isAllTarget ? 2 : 1;
    ctx.stroke();
  });
}
```

### 5-2. ターゲット枠

狙われているプレイヤーの盤面を赤い枠で囲む。攻撃を受けるたびに一時的に明滅させる。

```javascript
function drawTargetFrame(ctx, boardRect, isTargeted) {
  if (!isTargeted) return;

  const pulse = (Math.sin(Date.now() * 0.005) + 1) / 2; // 0〜1 で点滅
  ctx.strokeStyle = `rgba(255, 50, 50, ${0.5 + pulse * 0.5})`;
  ctx.lineWidth   = 3;
  ctx.shadowBlur  = 12;
  ctx.shadowColor = '#FF3333';
  ctx.strokeRect(boardRect.x - 3, boardRect.y - 3, boardRect.w + 6, boardRect.h + 6);
  ctx.shadowBlur  = 0;
}
```

### 5-3. 全体攻撃の予兆演出

テトリス・高RENが決まる直前（ライン消去エフェクト開始時）に発動する。

```javascript
function triggerAllAttackPreview(ctx, myBoardRect) {
  // 自分の盤面を強く発光させる
  ctx.shadowBlur  = 30;
  ctx.shadowColor = '#FFD700';
  ctx.strokeStyle = '#FFD700';
  ctx.lineWidth   = 4;
  ctx.strokeRect(myBoardRect.x, myBoardRect.y, myBoardRect.w, myBoardRect.h);
  ctx.shadowBlur  = 0;

  // 全プレイヤーに向かって光の線を伸ばす（拡散アニメーション）
  drawTargetLine(ctx, myBoardRect, null, true);
}
```

---

## 6. 処理フロー全体図

```
[ライン消去]
      │
      ▼
calcBaseDamage()          // 基礎火力を計算
      │
      ▼
cancelWithPending()       // 相殺優先で自分の pendingAttack を削る
      │
      ├─ remaining = 0 → 終了（攻撃は出ない）
      │
      ▼
isAllTargetAttack()       // 全体攻撃かどうかを判定
      │
      ├─ true（テトリス / 高REN）
      │     │
      │     ▼
      │   triggerAllAttackPreview()  // 予兆演出
      │   sendAttackToAll(damage × 0.7)
      │
      └─ false（通常）
            │
            ▼
          getTargetSlot()     // ターゲットモードで1人を決定
          drawTargetLine()    // ターゲットラインを更新
          sendAttackToOne(targetSlot, damage)
```

---

## 7. 定数・設定値一覧

| 定数名                     | 値       | 説明                                 |
|:--------------------------|:--------:|:------------------------------------|
| `TARGET_MODE_RANDOM`      | `'random'`  | ランダムモードの識別子              |
| `TARGET_MODE_KO`          | `'ko'`      | トドメモードの識別子                |
| `TARGET_MODE_COUNTER`     | `'counter'` | カウンターモードの識別子            |
| `ALL_ATTACK_REN_THRESHOLD`| 7           | 全体攻撃に切り替わるRENの閾値       |
| `ALL_ATTACK_MULTIPLIER`   | 0.7         | 全体攻撃時の1人当たり火力倍率       |
| `TARGET_LINE_WIDTH_SINGLE`| 1           | 単体ターゲットラインの太さ（px）    |
| `TARGET_LINE_WIDTH_ALL`   | 2           | 全体攻撃ラインの太さ（px）          |
| `TARGET_FRAME_GLOW`       | 12          | ターゲット枠のグロー半径（px）      |
| `TARGET_FRAME_WIDTH`      | 3           | ターゲット枠の線の太さ（px）        |

---

*作成日：2026-04-19*
