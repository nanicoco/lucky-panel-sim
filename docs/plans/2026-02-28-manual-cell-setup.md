# Manual Cell Setup Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** グリッドサイズ選択後、空グリッドを表示してユーザーがクリック順にセルへ絵文字ペアをセットし、全マス完了後に入れ替えプレイへ移行する。

**Architecture:** 単一ファイル `index.html` のスクリプト部分を修正する。`phase`（setup/play）と事前生成した `sequence` 配列を中心に状態管理し、`handleCardClick` でフェーズ分岐する。自動配置関数群は削除する。

**Tech Stack:** Vanilla HTML/CSS/JavaScript（ビルドツールなし）

---

### Task 1: state 変数の追加と `generateSequence()` 実装

**Files:**
- Modify: `index.html:223-226`（既存 let 宣言の直下）
- Modify: `index.html:257-308`（`createArrangement` 群の代替として）

**Step 1: state 変数を追加する**

`index.html` のスクリプト内、既存の `let selectedIndex = null;` の直下に以下を追加:

```javascript
let phase = 'setup';    // 'setup' | 'play'
let sequence = [];      // クリック順に割り当てる絵文字の配列
let nextIdx = 0;        // sequence の次に割り当てる位置
```

**Step 2: `generateSequence()` 関数を追加する**

`createArrangement` 関数の直前に以下を追加:

```javascript
// セットアップ用シーケンス生成
// [emoji0, emoji0, emoji1, emoji1, ..., ⭐️, 😈]
function generateSequence(rows, cols) {
    const totalSlots = rows * cols;
    const pairCount = Math.floor((totalSlots - 2) / 2);
    const seq = [];
    for (let i = 0; i < pairCount; i++) {
        seq.push(EMOJI_POOL[i]);
        seq.push(EMOJI_POOL[i]);
    }
    seq.push('⭐️');
    seq.push('😈');
    return seq;
}
```

**Step 3: ブラウザで動作確認（コンソール）**

`index.html` をブラウザで開き、DevTools コンソールで実行:
```javascript
generateSequence(3, 4)
// Expected: ['🟥','🟥','🟧','🟧','🟨','🟨','🟩','🟩','🟦','🟦','⭐️','😈']（12要素）

generateSequence(4, 6)
// Expected: 24要素、末尾が ['⭐️','😈']
```

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add phase state and generateSequence()"
```

---

### Task 2: `initGame()` をセットアップフェーズ方式に書き換え

**Files:**
- Modify: `index.html:242-251`（`initGame` 関数全体）

**Step 1: `initGame()` を以下で置き換える**

```javascript
function initGame() {
    sequence = generateSequence(currentRows, currentCols);
    cards = Array(currentRows * currentCols).fill(null);
    nextIdx = 0;
    phase = 'setup';
    selectedIndex = null;
    renderGrid(currentRows, currentCols);
}
```

**Step 2: ブラウザで確認**

リロード後、グリッドが全マス空白で表示されること。コンソールで:
```javascript
cards   // Expected: [null, null, null, ...] (12個)
phase   // Expected: 'setup'
nextIdx // Expected: 0
```

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: rewrite initGame() for setup phase"
```

---

### Task 3: 不要な自動配置関数を削除

**Files:**
- Modify: `index.html:253-308`（`createArrangement` 関連4関数）

**Step 1: 以下の関数を丸ごと削除する**

- `createArrangement(rows, cols, actualTypes)`（5行）
- `createHorizontalArrangement(rows, cols, actualTypes)`（10行）
- `createVerticalArrangement(rows, cols, actualTypes)`（12行）
- `createHorizontalRowArrangement(rows, cols, actualTypes)`（12行）

**Step 2: ブラウザでエラーがないことを確認**

DevTools コンソールでエラーが出ないこと。リロード後もグリッドが正常表示されること。

**Step 3: Commit**

```bash
git add index.html
git commit -m "refactor: remove unused auto-arrangement functions"
```

---

### Task 4: HTML ヘッダーの `<p>` に ID を付与

**Files:**
- Modify: `index.html:206-208`（`game-header` 内の `<p>` タグ）

**Step 1: `<p>` タグを以下で置き換える**

変更前:
```html
<p>カードをクリックして入れ替えます</p>
```

変更後:
```html
<p id="game-hint">次: </p>
```

**Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add id to game-hint paragraph"
```

---

### Task 5: `renderGrid()` でフェーズに応じたヒント表示

**Files:**
- Modify: `index.html:311-330`（`renderGrid` 関数）

**Step 1: `renderGrid()` の先頭（`board.innerHTML = '';` の直後）に以下を追加**

```javascript
// ヒントテキストをフェーズに応じて更新
const hint = document.getElementById('game-hint');
if (hint) {
    if (phase === 'setup' && nextIdx < sequence.length) {
        hint.textContent = `次: ${sequence[nextIdx]}`;
    } else {
        hint.textContent = 'カードをクリックして入れ替えます';
    }
}
```

**Step 2: ブラウザで確認**

リロード後、ヘッダーに `次: 🟥` と表示されること。

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: dynamic hint text based on phase in renderGrid()"
```

---

### Task 6: `handleCardClick()` にフェーズ分岐を追加

**Files:**
- Modify: `index.html:333-345`（`handleCardClick` 関数全体）

**Step 1: `handleCardClick()` を以下で置き換える**

```javascript
function handleCardClick(index, rows, cols) {
    if (phase === 'setup') {
        // セットアップ: 未セットのマスのみ受け付ける
        if (cards[index] !== null) return;
        cards[index] = sequence[nextIdx++];
        if (nextIdx >= sequence.length) {
            phase = 'play';
        }
        renderGrid(rows, cols);
        return;
    }

    // プレイフェーズ: 既存の入れ替えロジック
    if (selectedIndex === null) {
        selectedIndex = index;
    } else if (selectedIndex === index) {
        selectedIndex = null;
    } else {
        const temp = cards[selectedIndex];
        cards[selectedIndex] = cards[index];
        cards[index] = temp;
        selectedIndex = null;
    }
    renderGrid(rows, cols);
}
```

**Step 2: ブラウザで E2E テスト**

1. ページをリロード → ヘッダーに `次: 🟥`、全マス空白
2. マスを1つクリック → `🟥` が表示される、ヘッダーが `次: 🟥`（まだ同じ絵文字）に変化
3. 別のマスをクリック → `🟥` が表示される、ヘッダーが `次: 🟧` に変化
4. 既にセット済みのマスをクリック → 何も起きない
5. 全マスをクリックし埋める → ヘッダーが `カードをクリックして入れ替えます` に変わる
6. マスをクリックして選択（赤ハイライト）→ 別のマスをクリックして入れ替え確認
7. 左メニューのプリセットを切り替え → 空グリッドにリセット、ヘッダーが `次: 🟥` に戻る
8. 最後の2マス: 1つ目 → `⭐️`、2つ目 → `😈` が表示されること

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: phase-branching in handleCardClick() for setup/play modes"
```

---

### Task 7: 最終確認と仕上げ

**Step 1: 全プリセットで動作確認**

- 3×4（12マス）: pairCount=5、sequence長=12
- 4×4（16マス）: pairCount=7、sequence長=16
- 4×5（20マス）: pairCount=9、sequence長=20
- 4×6（24マス）: pairCount=11、sequence長=24

各プリセットで:
- セットアップ → 全マス埋め → プレイフェーズ移行 → 入れ替え操作

**Step 2: 最終 Commit**

```bash
git add index.html
git commit -m "feat: complete manual cell setup with phase-based game flow"
```
