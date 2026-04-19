# 対戦テトリス ゲーム終了演出 仕様書

## 概要

ゲーム終了時（敗北・勝利）に表示する演出の実装仕様を定義する。
敗北時は**「絶望と崩壊」**、勝利時は**「栄光と昇華」**をテーマに、光・揺れ・パーティクルで感情的なフィードバックを演出する。

---

## 1. トリガー条件

| 結果   | トリガー条件                                      | 呼び出す処理                  |
|:-----:|:------------------------------------------------:|:---------------------------:|
| 敗北   | `isGameOver === true`（ブロックが最上部に到達）     | `triggerGameEnd('lose')`    |
| 勝利   | `alivePlayers === 1`（自分以外が全員ゲームオーバー） | `triggerGameEnd('win')`     |

---

## 2. 判定・分岐レイヤー

```javascript
function triggerGameEnd(result) {
  if (result === 'lose') {
    startDeconstructionEffect();              // 盤面崩壊
    showEndText('GAME OVER', {
      shake:  'shiver',   // ガタガタと震える
      easing: 'weak',     // 弱いバネ的動き
    });
  } else {
    startAscensionEffect();                   // 宝石昇天
    showEndText('1st WINNER', {
      shake:  'impact',   // ズンッと着地
      easing: 'strong',   // 強力な Back Ease-out
    });
  }
}
```

---

## 3. 敗北演出：絶望と崩壊

### 3-1. 背景・盤面処理

#### モノクロ化

CSS `filter` を使い、画面全体の彩度をゼロにしてセピア～深い青に変える。

```javascript
function applyGameOverFilter() {
  const canvas = document.getElementById('gameCanvas');
  // トランジションで徐々に適用
  canvas.style.transition = 'filter 0.8s ease';
  canvas.style.filter = 'grayscale(100%) sepia(30%) hue-rotate(200deg)';
}
```

#### 盤面崩壊（DeconstructionEffect）

固定されていた全ブロックに下方向の重力を与え、バラバラに崩れ落ちる物理シミュレーションを開始する。

```javascript
// 崩壊ブロックのデータ構造
{
  x:      Number,  // 現在のX座標（px）
  y:      Number,  // 現在のY座標（px）
  vx:     Number,  // X方向の速度（ランダムな横ブレ）
  vy:     Number,  // Y方向の速度（下方向に加速）
  angle:  Number,  // 回転角度（rad）
  vAngle: Number,  // 回転速度
  color:  String,  // 元のブロック色
  alpha:  Number,  // 不透明度（1.0 → 0）
}

function startDeconstructionEffect(board) {
  board.forEach((row, rowIdx) => {
    row.forEach((cell, colIdx) => {
      if (cell === EMPTY) return;

      debrisParticles.push({
        x:      colIdx * BLOCK_SIZE,
        y:      rowIdx * BLOCK_SIZE,
        vx:     (Math.random() - 0.5) * 3,
        vy:     Math.random() * 2 + 1,   // 初速は小さく、重力で加速
        angle:  0,
        vAngle: (Math.random() - 0.5) * 0.2,
        color:  getCellColor(cell),
        alpha:  1.0,
      });
    });
  });
}

function updateDebrisParticle(p) {
  p.vy    += GRAVITY;       // 重力加速（GRAVITY = 0.4 推奨）
  p.x     += p.vx;
  p.y     += p.vy;
  p.angle += p.vAngle;
  p.alpha -= 0.012;         // 約80フレームで消える
}
```

### 3-2. GAME OVER 文字の挙動

#### 登場アニメーション

画面上部から不規則な速度でふらふらと降りてくる。停止位置付近でバネのように減衰する。

```javascript
// GAME OVER テキストの状態
let gameOverText = {
  y:        -100,           // 開始Y（画面外上部）
  targetY:  CANVAS_H / 2,  // 停止目標Y（画面中央）
  vy:       0,
  x:        CANVAS_W / 2,
  shiver:   0,              // ガタガタ強度
};

function updateGameOverTextEntry() {
  // バネ物理（減衰振動）
  const spring  = 0.08;  // バネ係数
  const damping = 0.75;  // 減衰係数
  const dy = gameOverText.targetY - gameOverText.y;

  gameOverText.vy = gameOverText.vy * damping + dy * spring;
  gameOverText.y += gameOverText.vy;

  // 十分に収束したらガタガタ状態へ移行
  if (Math.abs(gameOverText.vy) < 0.5 && Math.abs(dy) < 1) {
    gameOverText.shiver = 1.2; // ガタガタ強度を開始
  }
}
```

#### ガタガタ震え（shiver）

文字が止まった後も、毎フレームランダムに座標を微細にずらす。強度は時間とともに最小値まで減衰させるが、ゼロにはしない。

```javascript
function applyShiver(textObj) {
  const intensity = textObj.shiver;

  // 毎フレームランダムにオフセット
  const offsetX = (Math.random() - 0.5) * intensity;
  const offsetY = (Math.random() - 0.5) * intensity;

  // 強度を徐々に最小値まで減衰（完全には止まらない）
  textObj.shiver = Math.max(intensity * 0.995, 0.3);

  return { offsetX, offsetY };
}
```

#### 色収差（RGB ズレ）・グリッチ

一定間隔でランダムにトリガーし、文字の R / G / B チャンネルをそれぞれずらして描画する。

```javascript
function drawWithChromaticAberration(ctx, text, x, y, fontSize) {
  const glitchActive = Math.random() < 0.08; // 8% の確率で発動
  const shift = glitchActive ? 4 + Math.random() * 4 : 1;

  ctx.font = `bold ${fontSize}px Arial`;

  // R チャンネル（左にずらす）
  ctx.fillStyle = `rgba(255, 0, 0, 0.6)`;
  ctx.fillText(text, x - shift, y);

  // G チャンネル（中央）
  ctx.fillStyle = `rgba(0, 255, 0, 0.6)`;
  ctx.fillText(text, x, y);

  // B チャンネル（右にずらす）
  ctx.fillStyle = `rgba(0, 0, 255, 0.6)`;
  ctx.fillText(text, x + shift, y);

  // 白の本体テキスト
  ctx.fillStyle = '#FFFFFF';
  ctx.fillText(text, x, y);
}
```

---

## 4. 勝利演出：栄光と昇華

### 4-1. 背景・盤面処理

#### ホワイトアウト

勝利確定の瞬間に画面を一瞬白く光らせ、その後徐々に元の画面に戻す。

```javascript
let whiteoutAlpha = 0;

function triggerWhiteout() {
  whiteoutAlpha = 1.0;
}

function updateWhiteout(ctx) {
  if (whiteoutAlpha <= 0) return;

  ctx.fillStyle   = '#FFFFFF';
  ctx.globalAlpha = whiteoutAlpha;
  ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);
  ctx.globalAlpha = 1.0;

  whiteoutAlpha -= 0.025; // 約40フレームで消える
}
```

#### 宝石化・昇天（AscensionEffect）

盤面のブロックを金色・クリスタル色に差し替え、上空へ上昇するキラキラパーティクルとして昇天させる。

```javascript
const ASCENSION_COLORS = ['#FFD700', '#FFF8DC', '#E0E0FF', '#AAFFFF'];

function startAscensionEffect(board) {
  board.forEach((row, rowIdx) => {
    row.forEach((cell, colIdx) => {
      if (cell === EMPTY) return;

      const color = ASCENSION_COLORS[Math.floor(Math.random() * ASCENSION_COLORS.length)];

      ascensionParticles.push({
        x:    colIdx * BLOCK_SIZE + BLOCK_SIZE / 2,
        y:    rowIdx * BLOCK_SIZE,
        vx:   (Math.random() - 0.5) * 1.5,
        vy:   -(2.0 + Math.random() * 2.0),  // 上昇
        life: 1.0,
        size: BLOCK_SIZE * 0.6,
        color,
        elapsed: 0,
      });
    });
  });
}
```

### 4-2. 1st WINNER 文字の挙動

#### 登場アニメーション（Z軸ズームイン）

画面の奥から手前に向かって巨大サイズで迫り、Back Ease-out で「ズンッ」と着地する。

```javascript
let winnerText = {
  scale:   3.0,       // 開始スケール（大きい）
  targetScale: 1.0,   // 着地スケール
  alpha:   0,
  landed:  false,
};

// Back Ease-out：一度 target を通り過ぎて戻る
function easeOutBack(t) {
  const c1 = 1.70158;
  const c3 = c1 + 1;
  return 1 + c3 * Math.pow(t - 1, 3) + c1 * Math.pow(t - 1, 2);
}

function updateWinnerTextEntry(progress) {
  // progress: 0.0 → 1.0（外部でインクリメント）
  const t = easeOutBack(Math.min(progress, 1.0));
  winnerText.scale = 3.0 - (3.0 - 1.0) * t;
  winnerText.alpha = Math.min(progress * 3, 1.0);

  // 着地判定（scale が 1.0 に到達した瞬間）
  if (!winnerText.landed && winnerText.scale <= 1.05) {
    winnerText.landed = true;
    isCameraShake    = true;  // カメラシェイクを発動
    spawnConfetti();          // 紙吹雪を生成
  }
}
```

#### 後光（放射状の光）

文字着地後、後ろで放射状の光を回転させる。

```javascript
function drawHaloEffect(ctx, cx, cy, rotation) {
  const rayCount = 12;
  const innerR   = 80;
  const outerR   = 200;

  ctx.save();
  ctx.translate(cx, cy);
  ctx.rotate(rotation);

  for (let i = 0; i < rayCount; i++) {
    const angle = (Math.PI * 2 / rayCount) * i;
    const grad  = ctx.createLinearGradient(
      Math.cos(angle) * innerR, Math.sin(angle) * innerR,
      Math.cos(angle) * outerR, Math.sin(angle) * outerR,
    );
    grad.addColorStop(0, 'rgba(255, 215, 0, 0.4)');
    grad.addColorStop(1, 'rgba(255, 215, 0, 0)');

    ctx.beginPath();
    ctx.moveTo(Math.cos(angle) * innerR, Math.sin(angle) * innerR);
    ctx.lineTo(Math.cos(angle) * outerR, Math.sin(angle) * outerR);
    ctx.strokeStyle = grad;
    ctx.lineWidth   = 8;
    ctx.stroke();
  }

  ctx.restore();
}
```

### 4-3. 紙吹雪（ConfettiParticle）

着地と同時に左右から紙吹雪を生成する。

```javascript
const CONFETTI_COLORS = ['#FF4444', '#FFD700', '#44FF44', '#4488FF', '#FF44FF', '#FFFFFF'];

function spawnConfetti(count = 80) {
  for (let i = 0; i < count; i++) {
    const fromLeft = Math.random() < 0.5;
    confettiParticles.push({
      x:      fromLeft ? 0 : CANVAS_W,
      y:      Math.random() * CANVAS_H * 0.5,
      vx:     fromLeft ? (2 + Math.random() * 4) : -(2 + Math.random() * 4),
      vy:     Math.random() * 2 - 1,
      gravity: 0.1,
      color:  CONFETTI_COLORS[Math.floor(Math.random() * CONFETTI_COLORS.length)],
      width:  4 + Math.random() * 4,
      height: 8 + Math.random() * 6,
      angle:  Math.random() * Math.PI * 2,
      vAngle: (Math.random() - 0.5) * 0.2,
      alpha:  1.0,
    });
  }
}

function updateConfetti(p) {
  p.vy    += p.gravity;
  p.x     += p.vx;
  p.y     += p.vy;
  p.angle += p.vAngle;
  if (p.y > CANVAS_H * 0.8) p.alpha -= 0.02;
}
```

---

## 5. 演出の統合フロー

### 敗北フロー

```
[isGameOver === true]
      │
      ▼
applyGameOverFilter()        // モノクロ化
startDeconstructionEffect()  // 盤面崩壊パーティクル生成
showEndText('GAME OVER')
      │
      ▼
[毎フレーム]
  updateDebrisParticles()    // 崩壊ブロックを落下させる
  updateGameOverTextEntry()  // バネ動作で降下
  applyShiver()              // 停止後にガタガタ
  drawWithChromaticAberration() // 色収差・グリッチ
```

### 勝利フロー

```
[alivePlayers === 1]
      │
      ▼
triggerWhiteout()           // ホワイトアウト
startAscensionEffect()      // 宝石昇天パーティクル生成
showEndText('1st WINNER')
      │
      ▼
[毎フレーム]
  updateWhiteout()           // ホワイトアウトを徐々に戻す
  updateAscensionParticles() // 宝石パーティクルを上昇させる
  updateWinnerTextEntry()    // Back Ease-out でズームイン
      │
      └─ landed === true になった瞬間
            isCameraShake = true  // カメラシェイク
            spawnConfetti()       // 紙吹雪生成
            drawHaloEffect()      // 後光を回転
```

---

## 6. 定数・設定値一覧

| 定数名                    | 値      | 説明                                  |
|:-------------------------|:-------:|:-------------------------------------|
| `GRAVITY`                | 0.4     | 崩壊ブロックの重力加速度               |
| `DEBRIS_ALPHA_DECAY`     | 0.012   | 崩壊ブロックの不透明度減衰量            |
| `SPRING_K`               | 0.08    | GAME OVER 文字のバネ係数              |
| `SPRING_DAMPING`         | 0.75    | バネの減衰係数                         |
| `SHIVER_MIN`             | 0.3     | ガタガタ強度の最小値（ゼロにしない）     |
| `SHIVER_DECAY`           | 0.995   | ガタガタ強度の減衰率                   |
| `GLITCH_PROBABILITY`     | 0.08    | 色収差が発動する確率（フレームあたり）   |
| `WHITEOUT_DECAY`         | 0.025   | ホワイトアウトの不透明度減衰量          |
| `WINNER_SCALE_START`     | 3.0     | 1st WINNER 文字の初期スケール          |
| `HALO_RAY_COUNT`         | 12      | 後光の光線数                           |
| `HALO_INNER_R`           | 80      | 後光の内径（px）                       |
| `HALO_OUTER_R`           | 200     | 後光の外径（px）                       |
| `CONFETTI_COUNT`         | 80      | 紙吹雪の生成数                         |
| `CONFETTI_GRAVITY`       | 0.1     | 紙吹雪の重力加速度                     |

---

*作成日：2026-04-18*
