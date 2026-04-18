# 対戦テトリス お邪魔ブロック演出・タイマー管理 仕様書

## 概要

本ドキュメントは、お邪魔ブロックの**データ設計**・**予告演出**・**猶予タイマー管理**・**盤面への反映処理**・**レベル別視覚効果**の実装仕様を定義する。

---

## 1. データの持ち方（Firebase同期設計）

### 1-1. pendingAttack（保留中の攻撃リスト）

単なる数値ではなく、**オブジェクトの配列**として管理する。

```json
pendingAttack: [
  { "lines": 4, "type": "tetris",  "timestamp": 1700000001 },
  { "lines": 2, "type": "normal",  "timestamp": 1700000003 },
  { "lines": 6, "type": "tspin",   "timestamp": 1700000007 }
]
```

| フィールド  | 型      | 説明                                          |
|:----------:|:-------:|:---------------------------------------------|
| `lines`    | Number  | 攻撃のライン数                                 |
| `type`     | String  | 攻撃の種別（`normal` / `tetris` / `tspin` など） |
| `timestamp`| Number  | 攻撃が追加されたUnixタイムスタンプ（ms）        |

### 1-2. targetTimer（反映までの残り時間）

各プレイヤーが「いつ攻撃が盤面に反映されるか」を管理するカウントダウン値。

```
/rooms/{roomId}/players/{playerId}/targetTimer: Number（秒）
```

> **補足：** `targetTimer` はクライアント側のゲームループで毎フレーム減算し、Firebaseには主にリセット時（攻撃受信・相殺時）に書き込む。

---

## 2. 画面上部への「ぶら下がり」予告演出

### 2-1. 描画エリアの分離

メインの盤面（10×20）の**上部に仮想エリア（高さ5マス分）**を設け、そこに予告ブロックを天井からぶら下げて描画する。

```
┌──────────────────────┐
│  予告エリア（5マス）   │ ← pendingAttack の合計量を視覚化
├──────────────────────┤
│                      │
│   メイン盤面（20マス）│
│                      │
└──────────────────────┘
```

### 2-2. アイコン換算ロジック

`pendingAttack` の合計ライン数を、以下の**優先順位**でアイコンに変換して描画する。

| 優先度 | ライン数     | アイコン                        | 換算単位 |
|:-----:|:-----------:|:------------------------------:|:-------:|
| 1     | 10ライン以上 | 巨大な岩ブロック / ドクロアイコン | 1個 = 10ライン |
| 2     | 4〜9ライン  | 横に長いテトリスブロックの塊      | 1個 = 4ライン |
| 3     | 1〜3ライン  | 小さな単体ブロックの粒            | 1個 = 1ライン |

**変換例（合計13ライン）:**

```
[ドクロ×1] + [テトリス塊×0] + [単体×3]
= 10 + 0 + 3 = 13ライン
```

### 2-3. 物理的な揺れ演出

`pendingAttack` の合計ライン数が多いほど、アイコンを大きく左右に揺らす。

```javascript
// 揺れ幅はpending合計量に比例させる
const totalPending = pendingAttack.reduce((sum, a) => sum + a.lines, 0);
const speed  = 0.003 + totalPending * 0.0005;  // ライン数が多いほど速く
const amplitude = 2 + totalPending * 0.5;       // ライン数が多いほど大きく

const offsetX = Math.sin(Date.now() * speed) * amplitude;
drawIcon(x + offsetX, y);
```

---

## 3. 猶予時間の初期化（タイマー管理）

### 3-1. 攻撃受信時のリセット処理

Firebaseの `pendingAttack` が更新（追加）されたことを `onValue` で検知した瞬間、クライアント側の `attackTimer` を**最大値（3.0秒）**に上書きする。

```javascript
onValue(pendingRef, (snapshot) => {
  const newPending = snapshot.val() || [];
  if (newPending.length > localPending.length) {
    // 新規攻撃を検知 → タイマーをリセット
    attackTimer = ATTACK_TIMER_MAX;  // 3.0秒
    localPending = newPending;
    updateWarningIcons();
  }
});
```

> **意図：** 反映直前にさらに攻撃が来た場合でも、プレイヤーは再び「相殺」のための猶予を得られる。

### 3-2. 相殺によるタイマー維持

ラインを消して `pendingAttack` を減らしている間も、タイマーを最大値に戻し続ける。消し続けている限り、お邪魔ブロックは盤面に反映されない。

```javascript
function onLineClear(clearedLines) {
  // 相殺処理
  let remaining = calculateAttackPower(clearedLines);
  let newPending = [...localPending];

  while (remaining > 0 && newPending.length > 0) {
    const diff = Math.min(remaining, newPending[0].lines);
    newPending[0].lines -= diff;
    remaining -= diff;
    if (newPending[0].lines <= 0) newPending.shift();
  }

  localPending = newPending;

  // タイマーをリセット（相殺中は攻撃が来ない）
  attackTimer = ATTACK_TIMER_MAX;

  // Firebase に書き込み
  set(pendingRef, localPending);
  updateWarningIcons();
}
```

### 3-3. タイマー状態遷移

```
[攻撃受信]
    │
    ▼
attackTimer = 3.0s ──→ 毎フレーム減算
    │
    ├─ ライン消去 ──→ attackTimer = 3.0s にリセット（繰り返し）
    │
    └─ タイムアップ / ミノ固定 ──→ 盤面への反映処理へ
```

---

## 4. 攻撃の実体化（盤面への反映）

### 4-1. 反映トリガー

以下のいずれかの条件で確定する。

| トリガー           | 説明                                  |
|:-----------------:|:-------------------------------------|
| タイムアップ        | `attackTimer` が `0` になった瞬間      |
| ミノ固定           | ラインを消さずにミノを設置した瞬間       |

### 4-2. 反映処理の手順

```
1. pendingAttack の合計ライン数を集計
2. 現在の盤面データを合計ライン数分だけ上方向にシフト
3. 盤面の一番下の行に「穴あきお邪魔ブロック行」を挿入
   └─ 1回の攻撃内は穴の列を統一（詳細は攻撃システム仕様書 セクション4を参照）
4. pendingAttack をクリア（空配列にリセット）
5. 画面上部の予告ブロックを消去または縮小
6. Firebase に更新後の board と pendingAttack を書き込む
```

```javascript
function applyGarbageToBoard(board, pendingAttack) {
  const totalLines = pendingAttack.reduce((sum, a) => sum + a.lines, 0);
  const holeColumn  = Math.floor(Math.random() * BOARD_WIDTH);

  // 盤面を上にシフト
  const newBoard = board.slice(totalLines);

  // お邪魔ブロック行を下から挿入
  for (let i = 0; i < totalLines; i++) {
    const garbageRow = Array(BOARD_WIDTH).fill(GARBAGE_BLOCK);
    garbageRow[holeColumn] = EMPTY;
    newBoard.push(garbageRow);
  }

  return newBoard;
}
```

---

## 5. レベルアップによる視覚効果

`pendingAttack` の合計ライン数に応じて、予告ブロックの見た目を4段階で変化させる。

| レベル     | 合計ライン数の目安 | ブロック色 | 揺れ       | 追加エフェクト             |
|:---------:|:---------------:|:---------:|:---------:|:------------------------:|
| Lv.1 安全  | 1〜3ライン      | 薄いグレー | なし（静止）| —                        |
| Lv.2 警告  | 4〜7ライン      | 黄色       | ゆっくり揺れ | —                       |
| Lv.3 危険  | 8〜11ライン     | 赤色       | 激しく震える | 点滅アニメーション         |
| Lv.MAX 致命| 12ライン以上    | 赤色＋炎   | 最大振幅   | ドクロ・炎合成、画面赤フラッシュ |

```javascript
function getWarningLevel(totalLines) {
  if (totalLines >= 12) return 'MAX';
  if (totalLines >= 8)  return 3;
  if (totalLines >= 4)  return 2;
  return 1;
}
```

---

## 6. 処理フロー全体図

```
[Firebase: pendingAttack 増加を検知]
    │
    ▼
[attackTimer = 3.0s にリセット]
[上部予告エリアにアイコンを描画]
    │
    ▼
[毎フレーム: attackTimer を減算 & アイコンを揺らす]
    │
    ├─ ライン消去イベント
    │       │
    │       ▼
    │  [相殺処理: pendingAttack を減算]
    │  [attackTimer = 3.0s にリセット]
    │  [予告アイコンを縮小・更新]
    │  [Firebase に pendingAttack を書き込み]
    │
    └─ attackTimer = 0 / ミノ固定
            │
            ▼
       [applyGarbageToBoard() 実行]
       [pendingAttack をクリア]
       [予告エリアを消去]
       [Firebase に board & pendingAttack を書き込み]
            │
            ▼
       [次のミノをスポーン]
```

---

## 7. 定数・設定値一覧

| 定数名              | 値    | 説明                              |
|:-------------------|:-----:|:---------------------------------|
| `ATTACK_TIMER_MAX` | 3.0   | 猶予タイマーの初期値（秒）           |
| `PREVIEW_AREA_HEIGHT` | 5  | 予告エリアの高さ（マス数）           |
| `ICON_UNIT_LARGE`  | 10    | 巨大アイコン1個あたりのライン換算数   |
| `ICON_UNIT_MEDIUM` | 4     | 中型アイコン1個あたりのライン換算数   |
| `ICON_UNIT_SMALL`  | 1     | 小型アイコン1個あたりのライン換算数   |
| `WARN_LEVEL_2`     | 4     | Lv.2（警告）の閾値ライン数          |
| `WARN_LEVEL_3`     | 8     | Lv.3（危険）の閾値ライン数          |
| `WARN_LEVEL_MAX`   | 12    | Lv.MAX（致命的）の閾値ライン数       |
| `BOARD_WIDTH`      | 10    | 盤面の横幅（マス数）                |
| `GARBAGE_BLOCK`    | 8     | お邪魔ブロックのセル値              |
| `EMPTY`            | 0     | 空セルの値                         |

---

*作成日：2026-04-18*
