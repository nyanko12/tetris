# 対戦テトリス ハードドロップ・設置エフェクト 仕様書

## 概要

ハードドロップ時の**縦の風エフェクト（WindEffect）**と、ミノ設置時の**上昇するキラキラパーティクル（SparkleParticle）**の実装仕様を定義する。メインのテトリス処理とは独立したエフェクト管理レイヤーとして実装する。

---

## 1. エフェクト管理の全体像

```
[テトリス処理]
    │
    ├─ ハードドロップ ──→ [WindEffect]       縦の風・一瞬で消える
    │
    └─ ミノ設置       ──→ [SparkleParticle]  キラキラ・上昇しながら消える
```

エフェクトはメイン処理とは**別の配列**で管理し、毎フレームの描画ループ内で独立して更新・描画する。

```javascript
let windEffects    = [];  // WindEffect のアクティブ一覧
let sparkleEffects = [];  // SparkleParticle のアクティブ一覧

// 毎フレーム呼び出す
function updateEffects(ctx) {
  windEffects    = windEffects.filter(e => e.alpha > 0);
  sparkleEffects = sparkleEffects.filter(e => e.life  > 0);

  windEffects.forEach(e    => { e.update(); e.draw(ctx); });
  sparkleEffects.forEach(e => { e.update(); e.draw(ctx); });
}
```

---

## 2. WindEffect（縦の風）

### 2-1. 概要

ハードドロップ実行時に、ミノが瞬間移動した**軌跡**を白い細い線で描画する。一瞬だけ表示してスピード感を演出する。

### 2-2. データ構造

```javascript
// 1本の風の線につき1オブジェクト
{
  x:      Number,  // 線のX座標（ミノ幅内のランダム位置）
  yStart: Number,  // 線の上端Y座標（移動前のY）
  yEnd:   Number,  // 線の下端Y座標（yStart + ランダムな長さ）
  alpha:  Number,  // 現在の不透明度（1.0 → 0）
  width:  Number,  // 線の太さ（1〜2px）
}
```

### 2-3. 生成ロジック

```javascript
function createWindLines(oldY, newY, minoX, minoWidth) {
  const count = 4; // 生成する線の本数（3〜5本）

  for (let i = 0; i < count; i++) {
    const x      = minoX + Math.random() * minoWidth;
    const length = BLOCK_SIZE * (1.5 + Math.random()); // blockSize × 1.5〜2.5

    windEffects.push({
      x,
      yStart: oldY  * BLOCK_SIZE,
      yEnd:   (oldY * BLOCK_SIZE) + length,
      alpha:  1.0,
      width:  Math.random() < 0.5 ? 1 : 2, // 1px か 2px
    });
  }
}
```

### 2-4. 更新・描画

```javascript
// WindEffect の update
function updateWindEffect(effect) {
  effect.alpha -= 0.2; // 1フレームごとに減少 → 5フレームで消える
}

// WindEffect の draw
function drawWindEffect(ctx, effect) {
  ctx.globalAlpha = Math.max(effect.alpha, 0);
  ctx.strokeStyle = '#FFFFFF';
  ctx.lineWidth   = effect.width;

  ctx.beginPath();
  ctx.moveTo(effect.x, effect.yStart);
  ctx.lineTo(effect.x, effect.yEnd);
  ctx.stroke();

  ctx.globalAlpha = 1.0;
}
```

### 2-5. 調整パラメータ

| パラメータ      | 推奨値             | 説明                           |
|:-------------:|:-----------------:|:------------------------------|
| 生成本数        | 3〜5本            | 多すぎると画面が騒がしくなる       |
| 線の太さ        | 1〜2px            | 細いほどスピード感が増す           |
| 線の長さ        | `blockSize × 1.5〜2.5` | ランダムにすると自然に見える  |
| 不透明度の減衰   | `-0.2 / frame`    | 5フレームで完全に消える           |
| 生存フレーム数   | 5〜8フレーム       | 一瞬だけ見せるのがコツ            |

---

## 3. SparkleParticle（上昇するキラキラ）

### 3-1. 概要

ミノ設置時に、各ブロックの**接地面付近の4隅**からミノの色に合わせたパーティクルを放出する。上昇しながらゆらゆらと揺れ、消えていく。

### 3-2. データ構造

```javascript
// 1粒のパーティクルにつき1オブジェクト
{
  x:       Number,  // 現在のX座標
  y:       Number,  // 現在のY座標
  vx:      Number,  // X方向の速度
  vy:      Number,  // Y方向の速度（負の値 = 上昇）
  life:    Number,  // 残存量（1.0 → 0）
  size:    Number,  // 粒のサイズ（px）
  color:   String,  // 描画色（HSL文字列）
  elapsed: Number,  // 経過フレーム数（ゆらぎ計算に使用）
}
```

### 3-3. 色の決定ロジック

設置したミノの種類に応じて色を選択し、**輝度を上げてパステル化**する。

```javascript
// ミノ種別 → 基本色のHSL値
const MINO_COLOR_HSL = {
  I: [180, 100, 50],  // 水色
  O: [60,  100, 50],  // 黄色
  T: [300, 100, 50],  // 紫
  S: [120, 100, 40],  // 緑
  Z: [0,   100, 50],  // 赤
  J: [240, 100, 50],  // 青
  L: [30,  100, 50],  // オレンジ
};

function getSparkleColor(minoType, life) {
  const [h, s, l] = MINO_COLOR_HSL[minoType];

  // 消え際ほど白く（明度を上げて発光が収まる表現）
  const brightness = l + (100 - l) * (1 - life);

  // 彩度は life に比例して下げる（消えるにつれてパステルへ）
  const saturation = s * life;

  return `hsla(${h}, ${saturation}%, ${brightness}%, ${life})`;
}
```

### 3-4. 生成ロジック

各ブロックの**接地面の4隅**からパーティクルを放出する。

```javascript
function createSparkles(blocks, minoType) {
  // blocks: ミノを構成する各セルの座標配列 [{col, row}, ...]
  blocks.forEach(({ col, row }) => {
    const baseX = col * BLOCK_SIZE;
    const baseY = row * BLOCK_SIZE;

    // 4隅からそれぞれ放出
    const corners = [
      { x: baseX,              y: baseY + BLOCK_SIZE },
      { x: baseX + BLOCK_SIZE, y: baseY + BLOCK_SIZE },
      { x: baseX,              y: baseY },
      { x: baseX + BLOCK_SIZE, y: baseY },
    ];

    corners.forEach(({ x, y }) => {
      sparkleEffects.push({
        x, y,
        vx:      (Math.random() - 0.5) * 1.5,
        vy:      -(1.5 + Math.random() * 1.5),  // -1.5 〜 -3.0
        life:    1.0,
        size:    2 + Math.random() * 2,           // 2〜4px
        color:   getSparkleColor(minoType, 1.0),
        elapsed: 0,
      });
    });
  });
}
```

### 3-5. 更新・描画

```javascript
// SparkleParticle の update
function updateSparkle(effect, minoType) {
  effect.elapsed += 1;

  // 上昇 + ゆらぎ
  effect.vy *= 0.98;                                     // 浮力で緩やかに減速
  effect.y  += effect.vy;
  effect.x  += effect.vx + Math.sin(effect.elapsed * 10) * 0.5; // ゆらぎ

  // 上昇につれてサイズを縮小
  effect.size *= 0.97;

  // life を減算（色・透明度はここから都度計算）
  effect.life -= 0.04;  // 約25フレームで消える

  // 色を更新（消え際ほど白く）
  effect.color = getSparkleColor(minoType, effect.life);
}

// SparkleParticle の draw
function drawSparkle(ctx, effect) {
  ctx.fillStyle   = effect.color;
  ctx.globalAlpha = Math.max(effect.life, 0);

  ctx.beginPath();
  ctx.arc(effect.x, effect.y, Math.max(effect.size, 0), 0, Math.PI * 2);
  ctx.fill();

  ctx.globalAlpha = 1.0;
}
```

### 3-6. 調整パラメータ

| パラメータ      | 推奨値            | 説明                                    |
|:-------------:|:----------------:|:---------------------------------------|
| 初速 `vy`      | `-1.5 〜 -3.0`   | 負の値で上昇。大きいほど勢いよく飛ぶ        |
| 浮力            | `vy × 0.98`      | 毎フレーム減速。空気に溶ける質感になる      |
| ゆらぎ幅        | `Math.sin × 0.5` | 係数を大きくするほどゆらゆらが強くなる      |
| サイズ減衰      | `size × 0.97`    | 上昇中に少しずつ小さくなる                |
| 生存時間        | `life -= 0.04`   | 約25フレームで消える                      |
| 初期サイズ      | `2〜4px`          | 小さいほど上品なキラキラになる             |
| 放出数（per隅） | 1粒              | 4隅 × ブロック数 = 合計粒数               |

---

## 4. 演出の統合フロー

ハードドロップ実行時の処理順序を以下に示す。

```
1. 移動前のY座標を記録
      │  oldY = currentPieceY
      ▼
2. ハードドロップ先を計算・瞬間移動
      │  currentPieceY = hardDropDestY
      ▼
3. WindEffect を生成
      │  createWindLines(oldY, currentPieceY, minoX, minoWidth)
      ▼
4. 盤面にミノを固定 & ライン消去判定
      │  lockPiece() → checkLineClear()
      ▼
5. SparkleParticle を生成
      │  createSparkles(lockedBlocks, minoType)
      │  ※ 各ブロックの接地面の4隅から放出
      ▼
6. 次のミノをスポーン
```

---

## 5. 色の調整指針

| 段階         | 色の状態                           | 実装方法                            |
|:-----------:|:---------------------------------:|:-----------------------------------|
| 生成直後      | ミノの基本色（鮮やか）              | `hsla(H, 100%, 60%, 1.0)`          |
| 上昇中（中盤）| 彩度が下がりパステルカラーへ         | `saturation = s * life`            |
| 消え際        | 白に近づいてから透明に消える         | `brightness = l + (100-l) * (1-life)` |

```
発生 ──────────────────────────────── 消滅
鮮やかなミノ色 → パステル化 → 白く発光 → 透明
```

---

## 6. 定数・設定値一覧

| 定数名                  | 値      | 説明                               |
|:-----------------------|:-------:|:----------------------------------|
| `WIND_LINE_COUNT`      | 4       | 風の線の生成本数                    |
| `WIND_ALPHA_DECAY`     | 0.2     | 風の不透明度の1フレームあたりの減衰量 |
| `WIND_LENGTH_MIN`      | 1.5     | 風の線の最小長さ倍率（× blockSize）  |
| `WIND_LENGTH_MAX`      | 2.5     | 風の線の最大長さ倍率（× blockSize）  |
| `SPARKLE_VY_MIN`       | -1.5    | キラキラの初速（最小）               |
| `SPARKLE_VY_MAX`       | -3.0    | キラキラの初速（最大）               |
| `SPARKLE_BUOYANCY`     | 0.98    | 浮力（毎フレームの速度維持率）        |
| `SPARKLE_WOBBLE`       | 0.5     | ゆらぎの強さ（Math.sin の係数）      |
| `SPARKLE_LIFE_DECAY`   | 0.04    | キラキラの life の1フレームあたりの減衰 |
| `SPARKLE_SIZE_DECAY`   | 0.97    | キラキラのサイズの1フレームあたりの維持率 |
| `SPARKLE_SIZE_MIN`     | 2       | キラキラの最小初期サイズ（px）        |
| `SPARKLE_SIZE_MAX`     | 4       | キラキラの最大初期サイズ（px）        |

---

*作成日：2026-04-18*
