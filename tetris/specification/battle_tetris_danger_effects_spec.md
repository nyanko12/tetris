# 対戦テトリス 危険度演出ロードマップ 仕様書

## 概要

キャラクター演出の代わりに**「光」と「揺れ」**で緊張感をコントロールする、危険度トリガーに連動した段階別演出の実装仕様を定義する。

---

## 1. 危険度レベルの定義

| レベル       | トリガー（積み上げ高さ） | 状態        |
|:-----------:|:-------------------:|:----------:|
| 通常         | 盤面の中央より低い     | 安全        |
| LEVEL 1 注意 | 盤面の中央を超えた     | 注意        |
| LEVEL 2 警告 | 残り4マス以下        | 警告        |
| LEVEL 3 限界 | 残り2マス以下        | 限界        |

```javascript
function getDangerLevel(board) {
  const highestRow = board.findIndex(row => row.some(cell => cell !== EMPTY));
  const remainingRows = highestRow; // 上から何行空いているか

  if (remainingRows <= 2)  return 3;
  if (remainingRows <= 4)  return 2;
  if (highestRow < BOARD_HEIGHT / 2) return 1;
  return 0;
}
```

---

## 2. 段階別演出仕様

### LEVEL 1 ── 注意

**トリガー：** ブロックの積み上げ高さが盤面の中央（10マス）を超えた瞬間

#### 2-1. ヴィネット（画面隅の赤み）

画面の四隅に薄い赤色のグラデーションオーバーレイを表示する。

```css
/* ヴィネット用オーバーレイ */
.vignette {
  position: absolute;
  inset: 0;
  pointer-events: none;
  background: radial-gradient(
    ellipse at center,
    transparent 60%,
    rgba(200, 0, 0, 0.25) 100%
  );
  opacity: 0;
  transition: opacity 0.5s ease;
}

.vignette.level-1 { opacity: 1; }
```

#### 2-2. BGMテンポ変化

BGMの再生速度を通常の **1.2倍** に変更する。

```javascript
// Web Audio API を使用する場合
audioSource.playbackRate.value = 1.2;
```

---

### LEVEL 2 ── 警告

**トリガー：** 残り空き行が4マス以下になった瞬間

#### 2-3. ヴィネットのパルス（点滅）

ヴィネットの透明度をゆっくりと周期的に変化させる。

```javascript
// ゲームループ内で毎フレーム更新
function updateVignettePulse(timestamp) {
  const period    = 1500; // 点滅の周期（ms）
  const minAlpha  = 0.3;
  const maxAlpha  = 0.65;
  const t = (Math.sin((timestamp / period) * Math.PI * 2) + 1) / 2;
  vignetteElement.style.opacity = minAlpha + t * (maxAlpha - minAlpha);
}
```

```css
.vignette.level-2 {
  background: radial-gradient(
    ellipse at center,
    transparent 50%,
    rgba(200, 0, 0, 0.65) 100%
  );
}
```

#### 2-4. 左右の警告帯

盤面の左端・右端に縦長の警告帯を表示する。

```css
.warning-bar {
  position: absolute;
  top: 0;
  width: 6px;
  height: 100%;
  background: linear-gradient(
    to bottom,
    transparent,
    rgba(255, 60, 60, 0.8),
    transparent
  );
  opacity: 0;
  transition: opacity 0.3s ease;
}

.warning-bar.left  { left: 0; }
.warning-bar.right { right: 0; }
.warning-bar.level-2 { opacity: 1; }
```

---

### LEVEL 3 ── 限界

**トリガー：** 残り空き行が2マス以下になった瞬間

#### 2-5. 画面全体の激しい赤点滅

ヴィネットの点滅周期を短くし、最大不透明度を上げることで激しい赤点滅を演出する。

```javascript
function updateCriticalFlash(timestamp) {
  const period   = 400; // 高速点滅（ms）
  const minAlpha = 0.5;
  const maxAlpha = 0.9;
  const t = (Math.sin((timestamp / period) * Math.PI * 2) + 1) / 2;
  vignetteElement.style.opacity = minAlpha + t * (maxAlpha - minAlpha);
}
```

#### 2-6. 画面シェイク（揺れ）

ゲームキャンバス全体を小刻みに揺らす。一定間隔でランダムに発生させる。

```javascript
let shakeTimer = 0;

function updateScreenShake(deltaTime) {
  shakeTimer -= deltaTime;

  if (shakeTimer <= 0) {
    const intensity = 4; // 揺れの最大ピクセル数
    const dx = (Math.random() - 0.5) * intensity * 2;
    const dy = (Math.random() - 0.5) * intensity * 2;
    gameCanvas.style.transform = `translate(${dx}px, ${dy}px)`;
    shakeTimer = 80 + Math.random() * 120; // 次のシェイクまで 80〜200ms
  }
}

// LEVEL 3 解除時は必ずリセット
function resetScreenShake() {
  gameCanvas.style.transform = 'translate(0, 0)';
}
```

#### 2-7. ノイズエフェクト

Canvas上に半透明のランダムドットを散らし、ノイズ走査線のような演出を加える。

```javascript
function drawNoise(ctx, width, height) {
  const imageData = ctx.createImageData(width, height);
  const data = imageData.data;

  for (let i = 0; i < data.length; i += 4) {
    if (Math.random() < 0.04) { // ノイズ密度（4%）
      const brightness = Math.random() * 255;
      data[i]     = brightness; // R
      data[i + 1] = brightness; // G
      data[i + 2] = brightness; // B
      data[i + 3] = 60;         // A（薄く重ねる）
    }
  }
  ctx.putImageData(imageData, 0, 0);
}
```

---

## 3. 演出の組み合わせ一覧

| レベル       | ヴィネット   | BGMテンポ  | 警告帯  | 点滅       | 画面シェイク | ノイズ |
|:-----------:|:----------:|:---------:|:------:|:---------:|:-----------:|:-----:|
| 通常         | なし        | ×1.0      | なし    | なし       | なし         | なし   |
| LEVEL 1 注意 | 薄い・静止  | ×1.2      | なし    | なし       | なし         | なし   |
| LEVEL 2 警告 | 濃い・低速パルス | ×1.2 | あり（赤） | ゆっくり  | なし         | なし   |
| LEVEL 3 限界 | 最大・高速点滅 | ×1.5（推奨）| あり（赤）| 激しく  | あり         | あり   |

---

## 4. レベル変化時の演出切り替え処理

```javascript
let currentLevel = 0;

function updateDangerEffects(board, timestamp, deltaTime) {
  const level = getDangerLevel(board);

  // レベルが変わった瞬間だけ切り替え処理を実行
  if (level !== currentLevel) {
    onDangerLevelChange(currentLevel, level);
    currentLevel = level;
  }

  // 毎フレームの更新処理
  if (level === 2) updateVignettePulse(timestamp);
  if (level === 3) {
    updateCriticalFlash(timestamp);
    updateScreenShake(deltaTime);
  }
}

function onDangerLevelChange(prevLevel, nextLevel) {
  // 演出クラスの切り替え
  vignetteElement.className   = `vignette level-${nextLevel}`;
  warningBarLeft.className    = `warning-bar left ${nextLevel >= 2 ? 'level-2' : ''}`;
  warningBarRight.className   = `warning-bar right ${nextLevel >= 2 ? 'level-2' : ''}`;

  // BGMテンポの変更
  const tempoMap = { 0: 1.0, 1: 1.2, 2: 1.2, 3: 1.5 };
  audioSource.playbackRate.value = tempoMap[nextLevel];

  // シェイク解除
  if (nextLevel < 3) resetScreenShake();
}
```

---

## 5. 定数・設定値一覧

| 定数名                   | 値    | 説明                            |
|:------------------------|:-----:|:-------------------------------|
| `BOARD_HEIGHT`          | 20    | 盤面の縦幅（マス数）              |
| `LEVEL1_THRESHOLD`      | 10    | LEVEL1 のトリガー行（中央）        |
| `LEVEL2_THRESHOLD`      | 4     | LEVEL2 のトリガー残りマス数        |
| `LEVEL3_THRESHOLD`      | 2     | LEVEL3 のトリガー残りマス数        |
| `PULSE_PERIOD_L2`       | 1500  | LEVEL2 の点滅周期（ms）           |
| `PULSE_PERIOD_L3`       | 400   | LEVEL3 の点滅周期（ms）           |
| `SHAKE_INTENSITY`       | 4     | シェイクの最大ピクセル数           |
| `SHAKE_INTERVAL_MIN`    | 80    | シェイク間隔の最小値（ms）         |
| `SHAKE_INTERVAL_MAX`    | 200   | シェイク間隔の最大値（ms）         |
| `NOISE_DENSITY`         | 0.04  | ノイズドットの発生確率（0〜1）      |
| `BGM_TEMPO_NORMAL`      | 1.0   | 通常時の再生速度倍率              |
| `BGM_TEMPO_LEVEL1`      | 1.2   | LEVEL1 以上の再生速度倍率         |
| `BGM_TEMPO_LEVEL3`      | 1.5   | LEVEL3 の再生速度倍率（推奨）      |

---

*作成日：2026-04-18*
