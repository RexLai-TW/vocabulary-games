# GitHub Sync + iOS Audio Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Initialize git version control, connect to GitHub, and fix audio playback (TTS + sound effects) on iOS Chrome and iOS Safari.

**Architecture:** Single-file app (`index.html`, 1957 lines). Git setup is pure shell work. iOS audio fix patches three bugs in the existing TTS/AudioContext code: undeclared `ttsAudio` variable, wrong iOS detection guard (`isIOSafari` misses Chrome), and async AudioContext resume racing with sound playback.

**Tech Stack:** Vanilla HTML/CSS/JS, Web Speech Synthesis API, Web Audio API, Google Translate TTS, Git/GitHub CLI.

---

## Scope Note

Two independent subsystems. Git setup (Tasks 1–2) requires no code edits. iOS audio fix (Tasks 3–5) requires edits to `index.html`. Either half can be done independently.

---

## File Map

| File | Change |
|---|---|
| `index.html` | Fix lines 650, 663, 667–674, 766–796, 822–863, 865–893 |
| `.gitignore` | Create new |

---

### Task 1: Initialize Git and Make First Commit

**Files:**
- Create: `.gitignore`

- [ ] **Step 1: Initialize git repo**

```bash
cd /d/Dev_workspace/vocabulary-games
git init
git branch -M main
```

Expected: `Initialized empty Git repository in .../vocabulary-games/.git/`

- [ ] **Step 2: Create `.gitignore`**

Create file `/d/Dev_workspace/vocabulary-games/.gitignore` with this content:

```
# OS
.DS_Store
Thumbs.db

# Editor
.vscode/
.idea/
*.swp

# Logs
*.log
```

- [ ] **Step 3: Stage and commit**

```bash
cd /d/Dev_workspace/vocabulary-games
git add index.html .gitignore
git commit -m "feat: initial commit — 國小英文單字遊戲 v2.1"
```

Expected: `[main (root-commit) xxxxxxx] feat: initial commit...`

---

### Task 2: Connect to GitHub and Push

**Prerequisites:** You need a GitHub account. This task requires running commands yourself in the terminal.

- [ ] **Step 1: Create repo on GitHub**

Option A — GitHub CLI (if installed):
```bash
gh repo create vocabulary-games --public --source=. --remote=origin --push
```

Option B — Manual:
1. Go to https://github.com/new
2. Name: `vocabulary-games`
3. Keep it empty (no README, no .gitignore)
4. Click **Create repository**
5. Run:

```bash
cd /d/Dev_workspace/vocabulary-games
git remote add origin https://github.com/YOUR_USERNAME/vocabulary-games.git
git push -u origin main
```

- [ ] **Step 2: Verify**

```bash
git remote -v
git log --oneline
```

Expected: remote `origin` listed, one commit on `main`.

---

### Task 3: Fix Undeclared `ttsAudio` Variable

**Files:**
- Modify: `index.html` at line 650

**Root cause:** Line 663 reads `if (ttsAudio)` but `ttsAudio` is never declared anywhere in the file. This causes a `ReferenceError` in all browsers, silently killing the `speakWord()` function on every call.

- [ ] **Step 1: Read the current variable block**

Open `index.html` and confirm line 650 reads:
```javascript
let ttsMethod = null; // cached: 'google' | 'speechsynthesis' | null
```

- [ ] **Step 2: Add the missing declaration**

In `index.html`, replace:
```javascript
let ttsMethod = null; // cached: 'google' | 'speechsynthesis' | null
```
with:
```javascript
let ttsMethod = null; // cached: 'google' | 'speechsynthesis' | null
let ttsAudio = null;  // Audio element for Google TTS playback
```

- [ ] **Step 3: Verify no other references are missing**

Search for all `ttsAudio` occurrences — should be exactly 2 now:
- `let ttsAudio = null;` (declaration)
- `if (ttsAudio) { ttsAudio.pause(); ttsAudio.currentTime = 0; }` (usage at line ~664)

- [ ] **Step 4: Commit**

```bash
cd /d/Dev_workspace/vocabulary-games
git add index.html
git commit -m "fix: declare missing ttsAudio variable (caused ReferenceError in speakWord)"
```

---

### Task 4: Fix iOS TTS — Chrome Not Detected, Async Context Lost

**Files:**
- Modify: `index.html` lines 653–674

**Root causes:**

1. **iOS Chrome missed:** `isIOSafari` is `true` only when `!/CriOS/` (not Chrome). iOS Chrome falls into the `else` branch, which calls `speakGoogle()` first — an async operation. When that fails and `.catch()` calls `speakSpeechSynthesis()`, iOS blocks it because it's no longer within a synchronous user-gesture callback.

2. **Google TTS unreliable on iOS:** `fetch()` with `mode: 'cors'` fails (Google doesn't serve CORS headers). The `Audio()` fallback also fails on iOS when created from an async `.catch()` context.

3. **Background Google TTS on iOS Safari:** Even for iOS Safari (line 669), firing `speakGoogle()` in the background causes errors and noise.

**Fix:** For **all** iOS devices, call `speakSpeechSynthesis()` synchronously and skip Google TTS entirely.

- [ ] **Step 1: Read current `speakWord` function**

Lines 655–675 of `index.html` should look like:

```javascript
function speakWord(word, lang = 'en-US') {
  if (!word || word.trim() === '') return;

  const cleanWord = word.replace(/[^a-zA-Z\s'-]/g, '').trim();
  if (!cleanWord) return;

  // Cancel any ongoing speech (both Web Audio and Speech Synthesis)
  if ('speechSynthesis' in window) window.speechSynthesis.cancel();
  if (ttsAudio) { ttsAudio.pause(); ttsAudio.currentTime = 0; }

  // iOS Safari: use SpeechSynthesis first (most reliable), Google TTS as enhancement
  // Other browsers: use Google TTS first (better quality), SpeechSynthesis as fallback
  if (isIOSafari && 'speechSynthesis' in window) {
    speakSpeechSynthesis(cleanWord, lang);
    // Also try Google TTS in background for better quality
    speakGoogle(cleanWord, lang).catch(() => {});
  } else {
    // Try Google Translate TTS first, fallback to SpeechSynthesis
    speakGoogle(cleanWord, lang).catch(() => speakSpeechSynthesis(cleanWord, lang));
  }
}
```

- [ ] **Step 2: Replace the iOS branch**

Replace the block from `// iOS Safari: use SpeechSynthesis...` through the closing `}` of `speakWord` with:

```javascript
  // iOS (Safari + Chrome): MUST call SpeechSynthesis synchronously within user gesture.
  // Google Translate TTS fails on iOS — fetch() blocked by CORS, Audio() blocked outside gesture.
  if (isIOS && 'speechSynthesis' in window) {
    speakSpeechSynthesis(cleanWord, lang);
    return;
  }

  // Non-iOS: Try Google Translate TTS first (better quality), SpeechSynthesis as fallback
  speakGoogle(cleanWord, lang).catch(() => speakSpeechSynthesis(cleanWord, lang));
}
```

The full `speakWord` function after the edit:

```javascript
function speakWord(word, lang = 'en-US') {
  if (!word || word.trim() === '') return;

  const cleanWord = word.replace(/[^a-zA-Z\s'-]/g, '').trim();
  if (!cleanWord) return;

  // Cancel any ongoing speech
  if ('speechSynthesis' in window) window.speechSynthesis.cancel();
  if (ttsAudio) { ttsAudio.pause(); ttsAudio.currentTime = 0; }

  // iOS (Safari + Chrome): MUST call SpeechSynthesis synchronously within user gesture.
  // Google Translate TTS fails on iOS — fetch() blocked by CORS, Audio() blocked outside gesture.
  if (isIOS && 'speechSynthesis' in window) {
    speakSpeechSynthesis(cleanWord, lang);
    return;
  }

  // Non-iOS: Try Google Translate TTS first (better quality), SpeechSynthesis as fallback
  speakGoogle(cleanWord, lang).catch(() => speakSpeechSynthesis(cleanWord, lang));
}
```

- [ ] **Step 3: Test in browser**

Open `index.html` on iOS Safari or iOS Chrome. Tap any word in Listen mode — you should hear the word pronounced. Check the TTS status badge shows `🟡 TTS: Web Speech API（備援）`.

- [ ] **Step 4: Commit**

```bash
cd /d/Dev_workspace/vocabulary-games
git add index.html
git commit -m "fix: use SpeechSynthesis for all iOS devices — Google TTS blocked by CORS and async context"
```

---

### Task 5: Fix iOS Sound Effects — AudioContext Suspend Race

**Files:**
- Modify: `index.html` lines 766–797 (`initAudio`), 822–863 (`playCorrectSound`), 865–893 (`playWrongSound`)

**Root causes:**

1. **`initAudio` beep fires before resume completes:** Current code calls `ctx.resume()` (async), then immediately creates oscillators — but the context is still suspended so oscillators are silently dropped.

2. **`playCorrectSound` / `playWrongSound` don't await resume:** `getAudioCtx()` calls `ctx.resume()` but doesn't wait for it. On iOS, if the context is still `suspended` when `osc.start()` runs, the sound is dropped silently.

- [ ] **Step 1: Read current `initAudio`**

Lines 766–797 should look like:

```javascript
function initAudio() {
  if (audioInitialized) return;
  try {
    const ctx = new (window.AudioContext || window.webkitAudioContext)();
    // Resume on iOS — must be called from user gesture
    if (ctx.state === 'suspended') {
      ctx.resume().then(() => { audioCtx = ctx; audioInitialized = true; });
    } else {
      audioCtx = ctx;
      audioInitialized = true;
    }

    // iOS: play a short beep to confirm audio is working
    if (isIOS) {
      const osc = ctx.createOscillator();
      ...
      // Hide overlay after beep
      const overlay = document.getElementById('iosAudioOverlay');
      if (overlay) { overlay.style.display = 'none'; }
    }
  } catch(e) {}
}
```

- [ ] **Step 2: Replace `initAudio` with resume-first version**

Replace the entire `initAudio` function body:

```javascript
function initAudio() {
  if (audioInitialized) return;
  try {
    const ctx = new (window.AudioContext || window.webkitAudioContext)();
    const readyPromise = ctx.state === 'suspended' ? ctx.resume() : Promise.resolve();
    readyPromise.then(() => {
      audioCtx = ctx;
      audioInitialized = true;

      // iOS: play silent buffer to fully unlock AudioContext, then hide overlay
      if (isIOS) {
        const buffer = ctx.createBuffer(1, 1, 22050);
        const source = ctx.createBufferSource();
        source.buffer = buffer;
        source.connect(ctx.destination);
        source.start(0);
        const overlay = document.getElementById('iosAudioOverlay');
        if (overlay) overlay.style.display = 'none';
      }
    });
  } catch(e) {}
}
```

- [ ] **Step 3: Read current `playCorrectSound`**

Lines 822–863 — confirms the function uses `getAudioCtx()` and oscillators with no await.

- [ ] **Step 4: Replace `playCorrectSound` with resume-aware version**

Replace the entire `playCorrectSound` function:

```javascript
function playCorrectSound() {
  try {
    const ctx = getAudioCtx();
    const play = () => {
      const osc1 = ctx.createOscillator();
      const gain1 = ctx.createGain();
      osc1.type = 'sine';
      osc1.frequency.setValueAtTime(523.25, ctx.currentTime); // C5
      gain1.gain.setValueAtTime(0.3, ctx.currentTime);
      gain1.gain.exponentialRampToValueAtTime(0.01, ctx.currentTime + 0.3);
      osc1.connect(gain1).connect(ctx.destination);
      osc1.start(ctx.currentTime);
      osc1.stop(ctx.currentTime + 0.3);

      const osc2 = ctx.createOscillator();
      const gain2 = ctx.createGain();
      osc2.type = 'sine';
      osc2.frequency.setValueAtTime(659.25, ctx.currentTime + 0.1); // E5
      gain2.gain.setValueAtTime(0, ctx.currentTime);
      gain2.gain.linearRampToValueAtTime(0.3, ctx.currentTime + 0.1);
      gain2.gain.exponentialRampToValueAtTime(0.01, ctx.currentTime + 0.5);
      osc2.connect(gain2).connect(ctx.destination);
      osc2.start(ctx.currentTime + 0.1);
      osc2.stop(ctx.currentTime + 0.5);

      const osc3 = ctx.createOscillator();
      const gain3 = ctx.createGain();
      osc3.type = 'sine';
      osc3.frequency.setValueAtTime(783.99, ctx.currentTime + 0.2); // G5
      gain3.gain.setValueAtTime(0, ctx.currentTime);
      gain3.gain.linearRampToValueAtTime(0.25, ctx.currentTime + 0.2);
      gain3.gain.exponentialRampToValueAtTime(0.01, ctx.currentTime + 0.7);
      osc3.connect(gain3).connect(ctx.destination);
      osc3.start(ctx.currentTime + 0.2);
      osc3.stop(ctx.currentTime + 0.7);
    };
    if (ctx.state === 'suspended') { ctx.resume().then(play); } else { play(); }
  } catch(e) {}
}
```

- [ ] **Step 5: Replace `playWrongSound` with resume-aware version**

Replace the entire `playWrongSound` function:

```javascript
function playWrongSound() {
  try {
    const ctx = getAudioCtx();
    const play = () => {
      const osc1 = ctx.createOscillator();
      const gain1 = ctx.createGain();
      osc1.type = 'square';
      osc1.frequency.setValueAtTime(200, ctx.currentTime);
      gain1.gain.setValueAtTime(0.2, ctx.currentTime);
      gain1.gain.exponentialRampToValueAtTime(0.01, ctx.currentTime + 0.35);
      osc1.connect(gain1).connect(ctx.destination);
      osc1.start(ctx.currentTime);
      osc1.stop(ctx.currentTime + 0.35);

      const osc2 = ctx.createOscillator();
      const gain2 = ctx.createGain();
      osc2.type = 'square';
      osc2.frequency.setValueAtTime(150, ctx.currentTime + 0.15);
      gain2.gain.setValueAtTime(0, ctx.currentTime);
      gain2.gain.linearRampToValueAtTime(0.2, ctx.currentTime + 0.15);
      gain2.gain.exponentialRampToValueAtTime(0.01, ctx.currentTime + 0.6);
      osc2.connect(gain2).connect(ctx.destination);
      osc2.start(ctx.currentTime + 0.15);
      osc2.stop(ctx.currentTime + 0.6);
    };
    if (ctx.state === 'suspended') { ctx.resume().then(play); } else { play(); }
  } catch(e) {}
}
```

- [ ] **Step 6: Test in browser**

On iOS Safari or iOS Chrome:
1. Open `index.html`
2. Tap the screen once (to trigger AudioContext unlock)
3. Start a quiz game
4. Answer correctly → should hear ascending 3-tone chime
5. Answer wrong → should hear descending 2-tone buzz

- [ ] **Step 7: Commit**

```bash
cd /d/Dev_workspace/vocabulary-games
git add index.html
git commit -m "fix: await AudioContext.resume() before playing sounds on iOS"
```

---

### Task 6: Push All Commits to GitHub

- [ ] **Step 1: Push**

```bash
cd /d/Dev_workspace/vocabulary-games
git push origin main
```

- [ ] **Step 2: Verify on GitHub**

Open `https://github.com/YOUR_USERNAME/vocabulary-games` — should show 4 commits:
1. `feat: initial commit`
2. `fix: declare missing ttsAudio variable`
3. `fix: use SpeechSynthesis for all iOS devices`
4. `fix: await AudioContext.resume() before playing sounds on iOS`

---

## Self-Review Checklist

**Spec coverage:**
- [x] GitHub sync — Task 1 (git init + commit) + Task 2 (remote + push)
- [x] Version control — Task 1 establishes git history; Task 6 keeps remote in sync
- [x] iOS Chrome cannot produce sound — Task 4 fixes `speakWord()` to use SpeechSynthesis for `CriOS`
- [x] iOS Safari cannot produce sound — Task 4 removes unreliable Google TTS background call; Task 5 fixes AudioContext resume race

**Placeholder scan:** None found — all steps contain exact commands or complete code blocks.

**Type consistency:** `audioCtx`, `audioInitialized`, `isIOS`, `ttsAudio` — all consistent across tasks.
