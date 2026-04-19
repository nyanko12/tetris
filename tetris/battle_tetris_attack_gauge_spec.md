# 対戦テトリス 攻撃ゲージ（左サイドバー）仕様書

## 概要

相手からの保留中攻撃ライン数を、盤面の**左隣に配置した縦型ゲージ**で可視化する仕様を定義する。
上部パネル方式から左サイドバー方式へ移行することで、「盤面の積み上がり高さ」と「攻撃の蓄積量」を横並びで直感的に把握できるようにする。

---

## 1. レイアウト・配置

### 1-1. ゲージの位置と寸法

盤面（10×20マス）の**すぐ左隣**に細長い長方形のゲージを配置する。

```
┌────┬──────────────────────┬──────────┐
│    │                      │          │
│ 攻 │                      │  HOLD    │
│ 撃 │   メイン盤面          │          │
│ ゲ │   (10 × 20)          ├──────────┤
│ ー │                      │  NEXT    │
│ ジ │                      │          │
│    │                      │          │
└────┴──────────────────────┴──────────┘
```

| 項目           | 値                          |
|:-------------|:---------------------------|
| 幅            | `blockSize × 0.6`（約15px） |
| 高さ           | 盤面と同じ高さ（`blockSize × 20`）|
| 配置           | 盤面の左端に密着              |
| 最大容量        | 20ライン（盤面の高さと同一）    |

### 1-2. 視線設計の意図

| 方向 | 役割           | 理由                             |
|:---:|:-------------:|:--------------------------------|
| 右側 | 次のミノ（NEXT）確認 | 未来の操作を準備する場所         |
| 左側 | 攻撃ゲージ確認   | 迫りくる危機を「高さ」として把握する場所 |

---

## 2. ゲージの内部状態

### 2-1. 危険度による色の変化

| 状態          | 色         | 条件                                      |
|:-----------:|:---------:|:-----------------------------------------|
| 待機中（安全）  | 黄色 `#FFD700` | 攻撃が届いたばかり。まだ猶予タイマーが残っている |
| 確定直前（危険）| 赤色 `#FF3333` | タイマー残り1秒未満、またはミノ設置で即反映される状態 |
| 点滅（警告）   | 赤色点滅    | 盤面への反映が0.5秒以内に確定する寸前        |

```javascript
function getGaugeColor(attackTimer) {
  if (attackTimer > 1.0) return '#FFD700'; // 黄色：待機中
  return '#FF3333';                         // 赤色：確定直前
}
```

### 2-2. セグメント区切り

5ライン区切りで太い仕切り線を表示し、攻撃量を一目で把握しやすくする。

```javascript
function drawGaugeSegments(ctx, gaugeX, gaugeY, gaugeW, gaugeH) {
  const segmentLines = [5, 10, 15]; // 5ラインごとの区切り位置

  ctx.strokeStyle = 'rgba(0, 0, 0, 0.4)';

  segmentLines.forEach(line => {
    const ratio = line / MAX_GAUGE_LINES;           // ゲージ全体に対する割合
    const y     = gaugeY + gaugeH * (1 - ratio);   // 下から積み上がる

    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(gaugeX, y);
    ctx.lineTo(gaugeX + gaugeW, y);
    ctx.stroke();
  });
}
```

---

## 3. データ構造とロジック

### 3-1. ゲージ管理オブジェクト

```javascript
const attackGauge = {
  totalLines:    0,       // 現在の保留中ライン数
  displayHeight: 0,       // 現在の描画高さ（アニメーション用・px）
  targetHeight:  0,       // 目標の描画高さ（px）
  color:         '#FFD700',
  isBlinking:    false,
  blinkTimer:    0,
};
```

### 3-2. ① データの蓄積（スタック）

Firebaseから攻撃データを受信したら `totalLines` に加算し、ゲージの目標高さを更新する。

```javascript
// Firebase から pendingAttack の変化を検知
onValue(pendingRef, (snapshot) => {
  const pendingAttack = snapshot.val() || [];
  const total = pendingAttack.reduce((sum, a) => sum + a.lines, 0);

  attackGauge.totalLines  = total;
  attackGauge.targetHeight = calcGaugeHeight(total);

  // 新規攻撃受信 → 色を黄色にリセット
  attackGauge.color      = '#FFD700';
  attackGauge.isBlinking = false;
});

function calcGaugeHeight(lines) {
  const ratio = Math.min(lines / MAX_GAUGE_LINES, 1.0);
  return GAUGE_TOTAL_HEIGHT * ratio;
}
```

### 3-3. ② 反映タイマーとの連動

毎フレーム `attackTimer` の残量に応じてゲージの色と点滅状態を更新する。

```javascript
function updateGaugeState(attackTimer) {
  // 色の切り替え
  attackGauge.color = getGaugeColor(attackTimer);

  // 点滅判定（残り 0.5 秒未満）
  attackGauge.isBlinking = attackTimer < 0.5 && attackGauge.totalLines > 0;

  // タイマーリセット時（新規攻撃・ライン消去）→ 黄色に戻す
  // ※ 呼び出し元で attackTimer = ATTACK_TIMER_MAX にセットした直後に呼ぶ
}

// 毎フレーム：displayHeight を targetHeight に向けてスムーズに追従させる
function updateGaugeDisplay() {
  const speed = 0.15; // 追従速度（0〜1）
  attackGauge.displayHeight +=
    (attackGauge.targetHeight - attackGauge.displayHeight) * speed;
}
```

### 3-4. ③ 相殺（消滅）演出

プレイヤーがラインを消した際、ゲージの上端から火花（SparkleParticle）を散らしながらゲージを縮小させる。

```javascript
function onGaugeReduced(prevLines, newLines) {
  const reducedLines = prevLines - newLines;
  if (reducedLines <= 0) return;

  // ゲージ上端の座標を計算
  const gaugeTopY = GAUGE_BASE_Y - attackGauge.displayHeight;
  const gaugeX    = GAUGE_X + GAUGE_W / 2;

  // 削られた量に比例して火花を生成
  const sparkCount = reducedLines * 3;
  spawnGaugeSparks(gaugeX, gaugeTopY, sparkCount);

  // ゲージの目標高さを更新
  attackGauge.targetHeight = calcGaugeHeight(newLines);

  // 攻撃を防いだ → 色を黄色にリセット
  attackGauge.color      = '#FFD700';
  attackGauge.isBlinking = false;
}

function spawnGaugeSparks(x, y, count) {
  for (let i = 0; i < count; i++) {
    sparkleEffects.push({
      x, y,
      vx:      (Math.random() - 0.5) * 3,
      vy:      -(1.0 + Math.random() * 2.0),
      life:    1.0,
      size:    2 + Math.random() * 2,
      color:   '#FFD700',
      elapsed: 0,
    });
  }
}
```

---

## 4. 描画処理

### 4-1. ゲージ本体の描画

```javascript
function drawAttackGauge(ctx) {
  const { displayHeight, color, isBlinking, blinkTimer } = attackGauge;

  // 点滅処理
  if (isBlinking) {
    attackGauge.blinkTimer += 1;
    if (attackGauge.blinkTimer % 10 < 5) return; // 5フレームごとに非表示
  }

  // ゲージの背景（空の部分）
  ctx.fillStyle = 'rgba(255, 255, 255, 0.08)';
  ctx.fillRect(GAUGE_X, GAUGE_Y, GAUGE_W, GAUGE_TOTAL_HEIGHT);

  // ゲージの塗り（下から積み上がる）
  const fillY = GAUGE_Y + GAUGE_TOTAL_HEIGHT - displayHeight;
  ctx.fillStyle = color;
  ctx.fillRect(GAUGE_X, fillY, GAUGE_W, displayHeight);

  // グロー（赤色時のみ）
  if (color === '#FF3333') {
    ctx.shadowBlur  = 12;
    ctx.shadowColor = '#FF3333';
    ctx.fillRect(GAUGE_X, fillY, GAUGE_W, displayHeight);
    ctx.shadowBlur  = 0;
  }

  // セグメント区切り線
  drawGaugeSegments(ctx, GAUGE_X, GAUGE_Y, GAUGE_W, GAUGE_TOTAL_HEIGHT);

  // ゲージ上部に合計ライン数を表示
  if (attackGauge.totalLines > 0) {
    ctx.fillStyle  = '#FFFFFF';
    ctx.font       = `bold 11px Arial`;
    ctx.textAlign  = 'center';
    ctx.fillText(attackGauge.totalLines, GAUGE_X + GAUGE_W / 2, fillY - 4);
  }
}
```

---

## 5. 猶予タイマーとの連動フロー全体図

```
[Firebase: pendingAttack 増加を検知]
      │
      ▼
totalLines を更新
targetHeight を再計算
color = '#FFD700'（黄色にリセット）
isBlinking = false
      │
      ▼
[毎フレーム]
  updateGaugeDisplay()     // displayHeight を targetHeight に追従
  updateGaugeState()       // タイマー残量に応じて色・点滅を更新
  drawAttackGauge()        // ゲージを描画
      │
      ├─ ライン消去イベント
      │     │
      │     ▼
      │  onGaugeReduced()  // ゲージ縮小 + 火花生成
      │  color = '#FFD700' // 黄色にリセット
      │
      └─ attackTimer = 0 / ミノ固定
            │
            ▼
         盤面にお邪魔ブロックを反映
         totalLines = 0
         targetHeight = 0
```

---

## 6. 旧方式（上部パネル）との比較

| 項目         | 旧：上部パネル              | 新：左サイドバー                   |
|:-----------:|:-------------------------:|:---------------------------------|
| 視認性        | 画面が縦に伸び見づらい        | 盤面と横並びで直感的に確認できる    |
| 緊張感        | 数字のみで記号的              | 物理的な「高さ」として迫ってくる感覚 |
| 演出         | 文字の点滅程度               | 伸縮・色変化・火花で手応えを表現    |
| 盤面との対応  | 高さの比較が困難              | 「あと何段でゲームオーバー」が一目瞭然 |

---

## 7. 定数・設定値一覧

| 定数名                | 値                  | 説明                              |
|:--------------------|:-------------------:|:----------------------------------|
| `MAX_GAUGE_LINES`   | 20                  | ゲージの最大容量（ライン数）         |
| `GAUGE_W`           | `blockSize × 0.6`   | ゲージの幅（px）                   |
| `GAUGE_TOTAL_HEIGHT`| `blockSize × 20`    | ゲージの高さ（盤面と同じ）           |
| `GAUGE_X`           | 盤面左端 − `GAUGE_W` | ゲージのX座標                     |
| `GAUGE_Y`           | 盤面の上端Y座標       | ゲージのY座標                     |
| `GAUGE_DISPLAY_SPEED`| 0.15               | displayHeight の追従速度（0〜1）   |
| `COLOR_SAFE`        | `#FFD700`           | 待機中（黄色）                     |
| `COLOR_DANGER`      | `#FF3333`           | 確定直前（赤色）                   |
| `BLINK_THRESHOLD`   | 0.5                 | 点滅を開始する attackTimer の閾値（秒）|
| `BLINK_INTERVAL`    | 10                  | 点滅の周期（フレーム数）            |
| `SEGMENT_INTERVAL`  | 5                   | セグメント区切りのライン間隔         |
| `SPARK_COUNT_PER_LINE`| 3                 | 相殺時の火花数（削られた1ラインあたり）|

---

*作成日：2026-04-19*
