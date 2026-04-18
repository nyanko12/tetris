# 対戦テトリス ライン消去エフェクト 仕様書

## 概要

ラインが揃った際に表示する**フラッシュ→凝縮→拡散**の3フェーズで構成されるライン消去エフェクトの実装仕様を定義する。キャラクター演出の代わりに、光と動きで「消えた気持ちよさ」を演出する。

---

## 1. アニメーションのフェーズ構成

```
[フラッシュ]          [凝縮]              [拡散]
消える行が白く光る → 高さが中央に縮まる → 細い線が左右に消えていく

progress: 0.0 ──────── 0.5 ───────────────── 1.0
              Phase A              Phase B
```

| フェーズ | progress の範囲 | 内容                             |
|:-------:|:--------------:|:--------------------------------|
| フラッシュ | 0.0            | 行全体を白くフラッシュ             |
| 凝縮（Phase A） | 0.0〜0.5 | 行の高さを中央に向けてギュッと縮める |
| 拡散（Phase B） | 0.5〜1.0 | 極細の線を中央から左右に伸ばして消す |

---

## 2. エフェクトオブジェクトの設計

ラインが揃った行ごとに、以下の構造のエフェクトオブジェクトを生成して管理する。

```javascript
// 消える行ごとに1つ生成する
{
  y:        15,           // 対象の行インデックス（0始まり）
  progress: 0.0,          // アニメーションの進捗（0.0〜1.0）
  status:   'shrinking',  // 'shrinking'（凝縮中）→ 'spreading'（拡散中）
  color:    '#FFFFFF',    // エフェクトの描画色
}
```

### 2-1. エフェクトの配列管理

```javascript
let clearEffects = []; // アクティブなエフェクト一覧

function createClearEffects(clearedRows) {
  clearEffects = clearedRows.map(y => ({
    y,
    progress: 0.0,
    status: 'shrinking',
    color: '#FFFFFF',
  }));
}
```

---

## 3. 描画ステップの詳細

### 3-1. Phase A：凝縮（縦に縮める）

`progress` が `0.0〜0.5` の間、行の描画高さを中央に向けて縮小する。

- ブロックの個別形状は維持せず、**行全体を白い長方形1つ**として描画する。
- 上下の端が行の中央ピクセルに向かって収束するように高さを計算する。

```javascript
function drawShrinkingEffect(ctx, effect, blockSize) {
  const t = effect.progress / 0.5;          // 0.0〜1.0 に正規化
  const ease = easeOutCubic(t);

  const centerY  = effect.y * blockSize + blockSize / 2;
  const height   = blockSize * (1 - ease); // 縮むにつれて高さが0に近づく
  const drawY    = centerY - height / 2;

  ctx.fillStyle   = effect.color;
  ctx.globalAlpha = 1.0;
  ctx.fillRect(0, drawY, BOARD_WIDTH * blockSize, height);
}
```

### 3-2. Phase B：拡散（横に伸ばして消す）

`progress` が `0.5〜1.0` の間、高さを極細（1〜2px）に固定して左右に広げる。

- 描画の起点は**盤面の中央X座標**とする。
- 外側に伸びるにつれて不透明度を `1.0 → 0` に下げ、光が溶けるように消えるよう演出する。

```javascript
function drawSpreadingEffect(ctx, effect, blockSize) {
  const t = (effect.progress - 0.5) / 0.5; // 0.0〜1.0 に正規化
  const ease = easeOutExpo(t);              // 最後に弾けるよう Ease-out

  const centerX  = (BOARD_WIDTH * blockSize) / 2;
  const centerY  = effect.y * blockSize + blockSize / 2;
  const lineH    = 2;                             // 線の高さ（px）
  const halfW    = centerX * ease;                // 中央から外側への幅

  ctx.globalAlpha = 1.0 - ease;                   // 外側に行くほど透明になる
  ctx.fillStyle   = effect.color;

  // 左右それぞれ描画
  ctx.fillRect(centerX - halfW, centerY - lineH / 2, halfW, lineH); // 左
  ctx.fillRect(centerX,         centerY - lineH / 2, halfW, lineH); // 右

  ctx.globalAlpha = 1.0; // リセット
}
```

---

## 4. イージング関数

一定速度ではなく「溜め→弾け」の緩急をつけることで、キレのあるアニメーションを実現する。

```javascript
// Phase A（凝縮）に使用：最後に向かって加速する
function easeOutCubic(t) {
  return 1 - Math.pow(1 - t, 3);
}

// Phase B（拡散）に使用：最初は遅く、最後に一気に弾ける
function easeOutExpo(t) {
  return t === 1 ? 1 : 1 - Math.pow(2, -10 * t);
}
```

---

## 5. 残像・火花エフェクト（スパイス）

### 5-1. 残像（トレイル）

Phase B の拡散時に、太さと速度が微妙に異なる線を2〜3本重ねて描画する。

```javascript
const trails = [
  { widthMult: 1.0, speedMult: 1.0, alpha: 0.9 },
  { widthMult: 0.6, speedMult: 1.2, alpha: 0.5 },
  { widthMult: 0.3, speedMult: 1.5, alpha: 0.25 },
];

trails.forEach(trail => {
  const halfW = centerX * ease * trail.speedMult;
  ctx.globalAlpha = trail.alpha * (1.0 - ease);
  ctx.fillRect(centerX - halfW, centerY - 1, halfW, 2 * trail.widthMult);
  ctx.fillRect(centerX,         centerY - 1, halfW, 2 * trail.widthMult);
});
```

### 5-2. 火花（パーティクル）

`progress` が `1.0` に達した瞬間（線が伸び切った瞬間）に、小さな光の粒を左右の端点から放出する。

```javascript
// パーティクルオブジェクトの構造
{
  x: Number,      // 発生X座標（左右の端点）
  y: Number,      // 発生Y座標
  vx: Number,     // X方向の速度
  vy: Number,     // Y方向の速度
  life: Number,   // 残存時間（ms）
  maxLife: Number,
}

function spawnSparkParticles(x, y, count = 5) {
  for (let i = 0; i < count; i++) {
    particles.push({
      x, y,
      vx: (Math.random() - 0.5) * 4,
      vy: (Math.random() - 0.5) * 4,
      life: 300,
      maxLife: 300,
    });
  }
}
```

---

## 6. 盤面データの更新タイミング

エフェクト中は上のブロックが落下しないよう、処理の実行順序を厳密に管理する。

```
1. ラインが揃った！
      │
      ▼
2. 盤面配列から該当行を削除（または空行に置換）
      │
      ▼
3. createClearEffects() でエフェクトオブジェクトを生成
      │
      ▼
4. isClearAnimating = true にセット
   → dropLines()（上ブロックの落下処理）を一時停止
      │
      ▼
5. 描画ループでエフェクトアニメーションを再生
      │
      ▼
6. 全エフェクトの progress が 1.0 に達したら
   isClearAnimating = false に戻す
      │
      ▼
7. dropLines() を実行 → 上のブロックをガクンと落とす
```

```javascript
// ゲームループ内での制御フロー
function update(deltaTime) {
  if (isClearAnimating) {
    updateClearEffects(deltaTime); // エフェクトの進捗を更新
    return;                        // 落下処理はスキップ
  }
  dropLines(); // 通常の落下処理
}

function updateClearEffects(deltaTime) {
  const ANIM_DURATION = 400; // エフェクト全体の再生時間（ms）

  clearEffects.forEach(effect => {
    effect.progress += deltaTime / ANIM_DURATION;
    if (effect.progress >= 0.5) effect.status = 'spreading';
    if (effect.progress >= 1.0 && effect.status === 'spreading') {
      // 伸び切った瞬間に火花を発生
      const endX = BOARD_WIDTH * BLOCK_SIZE;
      spawnSparkParticles(0, effect.y * BLOCK_SIZE);
      spawnSparkParticles(endX, effect.y * BLOCK_SIZE);
    }
    effect.progress = Math.min(effect.progress, 1.0);
  });

  // 全エフェクトが完了したら一時停止を解除
  if (clearEffects.every(e => e.progress >= 1.0)) {
    clearEffects = [];
    isClearAnimating = false;
  }
}
```

---

## 7. 描画ループへの組み込み

```javascript
function draw(ctx, timestamp) {
  // 1. 通常の盤面描画
  drawBoard(ctx);

  // 2. エフェクトの描画（盤面の上にオーバーレイ）
  clearEffects.forEach(effect => {
    if (effect.status === 'shrinking') {
      drawShrinkingEffect(ctx, effect, BLOCK_SIZE);
    } else {
      drawSpreadingEffect(ctx, effect, BLOCK_SIZE);
    }
  });

  // 3. パーティクルの描画
  drawParticles(ctx);
}
```

---

## 8. 定数・設定値一覧

| 定数名              | 値    | 説明                               |
|:-------------------|:-----:|:----------------------------------|
| `ANIM_DURATION`    | 400   | エフェクト全体の再生時間（ms）        |
| `SPREAD_LINE_H`    | 2     | 拡散フェーズの線の高さ（px）          |
| `TRAIL_COUNT`      | 3     | 残像の本数                          |
| `SPARK_COUNT`      | 5     | 火花パーティクルの個数（片側）         |
| `SPARK_LIFE`       | 300   | 火花の残存時間（ms）                 |
| `SPARK_SPEED`      | 4     | 火花の最大初速（px/frame）           |
| `PHASE_A_END`      | 0.5   | Phase A が終わる progress の値      |
| `EFFECT_COLOR`     | `#FFFFFF` | エフェクトの描画色               |

---

*作成日：2026-04-18*
