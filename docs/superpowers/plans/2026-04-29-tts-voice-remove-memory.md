# Sub-project A: TTS Voice + Remove Memory Mode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove 記憶翻牌 game mode entirely and improve Web Speech API female voice selection across iOS, Windows, and Android.

**Architecture:** Single file `index.html` (~1940 lines). Two independent changes: (1) delete all memory-mode code from CSS, JS, and state; (2) add a prioritized female-voice lookup that tries Enhanced → known-female-name list → any English voice.

**Tech Stack:** Vanilla JS, Web Speech Synthesis API, Git.

---

## File Map

| File | Change |
|---|---|
| `index.html` lines 226–276 | Delete memory CSS block |
| `index.html` line 628 | Delete memory entry from gameModes array |
| `index.html` lines 959–962 | Delete memory state keys |
| `index.html` line 1814 | Fix showResult total calculation |
| `index.html` lines 1559–1652 | Delete startMemory / renderMemory / flipCard |
| `index.html` line 651 | Add FEMALE_VOICE_NAMES constant |
| `index.html` lines 730–735 | Replace voice selection block |

---

### Task 1: Remove Memory Mode

**Files:**
- Modify: `index.html` lines 226–276, 628, 959–962, 1814, 1559–1652

- [ ] **Step 1: Remove memory entry from `gameModes` array**

In `index.html`, find and delete this exact line (line 628):
```javascript
  { id: 'memory', icon: '🃏', name: '記憶翻牌', desc: '配對英文和中文' },
```

After deletion the array should have 6 entries: quiz, reverse, listen, spell, scramble, speed.

- [ ] **Step 2: Remove memory CSS block**

Find and delete the entire Memory Game CSS section (lines 226–276). The block to remove starts with `/* Memory Game */` and ends with `.memory-card.matched .memory-back { ... }`:

```css
  /* Memory Game */
  .memory-grid {
    display: grid;
    gap: 8px;
    max-width: 500px;
    width: 100%;
  }

  .memory-card {
    aspect-ratio: 1;
    border-radius: 12px;
    cursor: pointer;
    perspective: 600px;
    position: relative;
  }

  .memory-card-inner {
    width: 100%; height: 100%;
    position: relative;
    transition: transform .5s;
    transform-style: preserve-3d;
  }

  .memory-card.flipped .memory-card-inner,
  .memory-card.matched .memory-card-inner { transform: rotateY(180deg); }

  .memory-front, .memory-back {
    position: absolute; inset: 0;
    border-radius: 12px;
    display: flex; align-items: center; justify-content: center;
    backface-visibility: hidden;
    font-weight: 700;
  }

  .memory-front {
    background: linear-gradient(135deg, var(--purple), var(--blue));
    color: #fff;
    font-size: 1.6rem;
  }

  .memory-back {
    background: #fff;
    border: 3px solid #eee;
    transform: rotateY(180deg);
    font-size: .85rem;
    color: #333;
    padding: 4px;
    word-break: break-word;
  }

  .memory-card.matched .memory-back { border-color: var(--green); background: #ecfdf5; }
```

After deletion, the line immediately after the removed block should be `/* Scramble Game */`.

- [ ] **Step 3: Remove memory state keys**

Find and delete the three memory state lines (lines 959–962):
```javascript
  // Memory game state
  memoryCards: [],
  flippedCards: [],
  matchedPairs: 0,
```

After deletion, the line before `// Spell game state` should be `totalQuestions: 10,`.

- [ ] **Step 4: Fix `showResult` total calculation**

Find line 1814 (in `showResult` function):
```javascript
  const total = state.gameMode === 'memory' ? state.memoryCards.length / 2 : state.currentWords.length;
```

Replace with:
```javascript
  const total = state.currentWords.length;
```

- [ ] **Step 5: Remove memory JS functions**

Find and delete the entire block from `function startMemory()` through the closing `}` of `flipCard()` — lines 1559–1652. The block to delete:

```javascript
function startMemory() {
  const pairCount = Math.min(6, state.currentWords.length);
  const words = shuffle([...state.currentWords]).slice(0, pairCount);

  state.memoryCards = [];
  state.flippedCards = [];
  state.matchedPairs = 0;
  state.currentIndex = 0;
  state.correctCount = 0;

  words.forEach(([en, zh], i) => {
    state.memoryCards.push({ id: i*2, pairId: i, text: en, type: 'en' });
    state.memoryCards.push({ id: i*2+1, pairId: i, text: zh, type: 'zh' });
  });

  state.memoryCards = shuffle(state.memoryCards);
  renderMemory();
}

function renderMemory() {
  const cols = state.memoryCards.length > 12 ? 4 : 3;
  const gc = document.getElementById('gameContent');
  gc.innerHTML = `
    <h2>🃏 記憶翻牌</h2>
    <div class="game-subtitle">配對英文和中文 — 已找到 ${state.matchedPairs}/${state.memoryCards.length/2}</div>
    <div class="memory-grid" style="grid-template-columns:repeat(${cols},1fr);max-width:${cols*100}px;">
      ${state.memoryCards.map((card, i) => `
        <div class="memory-card${card.flipped?' flipped':''}${card.matched?' matched':''}" data-idx="${i}" onclick="flipCard(${i})">
          <div class="memory-card-inner">
            <div class="memory-front">?</div>
            <div class="memory-back">${card.text}</div>
          </div>
        </div>`).join('')}
    </div>`;
}

function flipCard(idx) {
  const card = state.memoryCards[idx];
  if (card.flipped || card.matched || state.flippedCards.length >= 2) return;

  card.flipped = true;
  state.flippedCards.push(idx);
  renderMemory();

  if (state.flippedCards.length === 2) {
    const [a, b] = state.flippedCards;
    const ca = state.memoryCards[a], cb = state.memoryCards[b];

    if (ca.pairId === cb.pairId && ca.type !== cb.type) {
      // Match!
      playCorrectSound();
      setTimeout(() => {
        ca.matched = true; cb.matched = true;
        state.matchedPairs++;
        state.correctCount++;
        state.streak++;
        markLearned(ca.text);
        state.flippedCards = [];
        renderMemory();
        updateStreak();

        if (state.matchedPairs >= state.memoryCards.length / 2) {
          setTimeout(() => showResult(), 500);
        }
      }, 500);
    } else {
      // No match — play wrong sound and show hint briefly
      playWrongSound();
      const wordPair = state.currentWords[ca.pairId];
      const [en, zh] = wordPair || ['', ''];
      setTimeout(() => {
        ca.flipped = false; cb.flipped = false;
        state.streak = 0;
        state.flippedCards = [];

        // Show hint briefly before flipping back
        const gc = document.getElementById('gameContent');
        const hintDiv = document.createElement('div');
        hintDiv.className = 'wrong-hint';
        hintDiv.innerHTML = `
          <div class="wrong-hint-label">💡 配對答案是：</div>
          <div class="wrong-hint-answer">${en}</div>
          <div class="wrong-hint-meaning">${zh}</div>`;
        gc.querySelector('.memory-grid').after(hintDiv);

        renderMemory();
        updateStreak();

        // Remove hint after delay
        setTimeout(() => { if (hintDiv.parentNode) hintDiv.remove(); }, 1500);
      }, 800);
    }
  }
}
```

After deletion the next line should be `// ==================== SCRAMBLE MODE ====================`.

- [ ] **Step 6: Verify no memory references remain**

```bash
grep -n "memory\|Memory\|flipCard\|memoryCard\|matchedPair\|flippedCard" /d/Dev_workspace/vocabulary-games/index.html
```

Expected: zero matches. If any remain, delete them.

- [ ] **Step 7: Open in browser and verify**

Open `index.html` in a browser. Confirm:
- Game mode grid shows 6 modes (no 🃏 記憶翻牌)
- All other modes start without errors
- Browser console shows no `ReferenceError`

- [ ] **Step 8: Commit**

```bash
cd /d/Dev_workspace/vocabulary-games
git add index.html
git commit -m "feat: remove 記憶翻牌 game mode"
```

---

### Task 2: Improve TTS Female Voice Selection

**Files:**
- Modify: `index.html` lines 651–652 (add constant), lines 730–735 (replace voice selection)

- [ ] **Step 1: Add `FEMALE_VOICE_NAMES` constant**

In `index.html`, find these two lines (lines 650–651):
```javascript
let ttsMethod = null; // cached: 'google' | 'speechsynthesis' | null
let ttsAudio = null;  // Audio element for Google TTS playback
```

Replace with:
```javascript
let ttsMethod = null; // cached: 'google' | 'speechsynthesis' | null
let ttsAudio = null;  // Audio element for Google TTS playback

// Known natural English female voices across platforms
// Priority: Enhanced/Premium (iOS 16+) → named females → any English
const FEMALE_VOICE_NAMES = [
  'Samantha', 'Zira', 'Eva', 'Karen',
  'Moira', 'Ava', 'Tessa', 'Victoria',
  'Allison', 'Susan', 'Fiona', 'Google US English'
];
```

- [ ] **Step 2: Replace voice selection block in `speakSpeechSynthesis`**

Find this block (lines 730–735):
```javascript
  // Prefer English female voice on iOS
  if (voices.length > 0) {
    const enVoice = voices.find(v => v.lang.startsWith('en') && v.name.includes('Female'))
      || voices.find(v => v.lang.startsWith('en'));
    if (enVoice) utterance.voice = enVoice;
  }
```

Replace with:
```javascript
  // Select best available English female voice:
  // 1. Enhanced/Premium (iOS 16+ — most natural)
  // 2. Known female voice names (cross-platform list)
  // 3. Any English voice (last resort)
  const enVoices = voices.filter(v => v.lang.startsWith('en'));
  const voice =
    enVoices.find(v => /enhanced|premium/i.test(v.name)) ||
    enVoices.find(v => FEMALE_VOICE_NAMES.some(n => v.name.includes(n))) ||
    enVoices[0];
  if (voice) utterance.voice = voice;
```

- [ ] **Step 3: Verify `FEMALE_VOICE_NAMES` is referenced**

```bash
grep -n "FEMALE_VOICE_NAMES" /d/Dev_workspace/vocabulary-games/index.html
```

Expected: exactly 2 matches — the `const` declaration and the `.some(n => v.name.includes(n))` usage.

- [ ] **Step 4: Test voice selection in browser console**

Open `index.html` in Chrome or Safari. Open DevTools console and run:

```javascript
const voices = speechSynthesis.getVoices();
const enVoices = voices.filter(v => v.lang.startsWith('en'));
console.log('Enhanced:', enVoices.find(v => /enhanced|premium/i.test(v.name))?.name);
console.log('Female list:', enVoices.find(v => ['Samantha','Zira','Eva','Karen','Moira','Ava','Tessa','Victoria','Allison','Susan','Fiona','Google US English'].some(n => v.name.includes(n)))?.name);
console.log('Fallback:', enVoices[0]?.name);
```

Expected: at least one of Enhanced or Female list shows a named voice (not `undefined`).

- [ ] **Step 5: Test TTS in Listen mode**

Start a game → select 聽音選詞 → tap the speaker button. Confirm:
- Voice sounds female / natural
- TTS status badge shows the correct method

- [ ] **Step 6: Commit**

```bash
cd /d/Dev_workspace/vocabulary-games
git add index.html
git commit -m "feat: improve TTS voice selection — Enhanced/Premium → female name list → any English"
```

---

### Task 3: Push to GitHub

- [ ] **Step 1: Push both commits**

```bash
cd /d/Dev_workspace/vocabulary-games
git push origin main
```

Expected:
```
To https://github.com/RexLai-TW/vocabulary-games.git
   xxxxxxx..xxxxxxx  main -> main
```

- [ ] **Step 2: Verify on Cloudflare Pages**

If Cloudflare Pages is already connected: open `https://dash.cloudflare.com` → Pages → `vocabulary-games` → Deployments. A new deploy should appear and complete within 60 seconds.

Then open `https://vocabulary-games.pages.dev` and confirm 6 game modes are shown (no 記憶翻牌).

---

## Self-Review

**Spec coverage:**
- [x] Remove memory from gameModes array → Task 1, Step 1
- [x] Remove memory CSS → Task 1, Step 2
- [x] Remove memory state keys → Task 1, Step 3
- [x] Fix showResult (no dead memory branch) → Task 1, Step 4
- [x] Remove JS functions → Task 1, Step 5
- [x] FEMALE_VOICE_NAMES constant → Task 2, Step 1
- [x] Voice priority: Enhanced → name list → any English → Task 2, Step 2
- [x] Google Translate TTS path unchanged → confirmed: only `speakSpeechSynthesis` is modified

**Placeholder scan:** None. All code blocks are complete.

**Type consistency:** `FEMALE_VOICE_NAMES` declared in Task 2 Step 1, used in Task 2 Step 2. Consistent.
