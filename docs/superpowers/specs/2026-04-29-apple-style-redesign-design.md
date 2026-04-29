# Sub-project B: Apple-Style Redesign + Loading Page Design

**Date:** 2026-04-29
**Status:** Approved

## Goal

Redesign `index.html` with an iOS Light visual style and add a loading screen. All game logic remains unchanged — only CSS, layout HTML, and a new loading overlay are modified.

## Design Decisions (User Approved)

| Decision | Choice |
|---|---|
| Overall style | C — iOS Light (淺灰底 `#f2f2f7`, 白色圓角卡片) |
| Loading page | B — 進度條載入 (Emoji + 雙行大標 + 進度條) |
| Primary color | A — iOS 系統藍 `#007AFF` |

---

## Visual Design Tokens

```css
/* Colors */
--bg:           #f2f2f7;   /* iOS grouped background */
--card:         #ffffff;   /* Card surface */
--label-1:      #1c1c1e;   /* Primary text */
--label-2:      #3c3c43;   /* Secondary text */
--label-3:      #8e8e93;   /* Tertiary / caption */
--separator:    #c6c6c8;   /* Dividers */
--fill:         #e5e5ea;   /* Input fills, progress track */
--blue:         #007aff;   /* Primary action */

/* Game mode icon background tints */
--tint-blue:    #e8f3ff;
--tint-orange:  #fff4e6;
--tint-green:   #e8faf0;
--tint-purple:  #f0eeff;
--tint-yellow:  #fffbe6;
--tint-red:     #fff0f0;

/* Typography */
--font: -apple-system, BlinkMacSystemFont, 'SF Pro Text', 'Helvetica Neue', sans-serif;

/* Shape */
--radius-sm:  10px;
--radius-md:  14px;
--radius-lg:  16px;
--radius-xl:  20px;

/* Shadows */
--shadow-card: 0 1px 0 rgba(0,0,0,.05), 0 2px 8px rgba(0,0,0,.06);
--shadow-sm:   0 1px 0 rgba(0,0,0,.04);
```

---

## Loading Screen

Shown immediately on page load, hidden once `init()` completes.

**Structure:**
```html
<div id="loadingScreen">
  <span class="loading-emoji">📚</span>
  <div class="loading-title">國小英文<br><span>單字遊戲</span></div>
  <div class="progress-wrap"><div id="loadingBar" class="progress-bar"></div></div>
  <div class="loading-sub">載入中…</div>
</div>
```

**Behavior:**
- Shown by default (`display: flex`)
- JS animates `#loadingBar` width from 0% → 100% over 800ms
- After 800ms, fade out with CSS transition, then `display: none`
- Main app (`#mainApp`) becomes visible after loading screen hides

**CSS:**
```css
#loadingScreen {
  position: fixed; inset: 0; z-index: 9999;
  background: var(--bg);
  display: flex; flex-direction: column;
  align-items: center; justify-content: center;
  text-align: center; padding: 40px 32px;
  transition: opacity .3s ease;
}
#loadingScreen.hide { opacity: 0; pointer-events: none; }
.loading-emoji { font-size: 60px; margin-bottom: 20px; }
.loading-title {
  font-size: 30px; font-weight: 700;
  color: var(--label-1); line-height: 1.2;
  letter-spacing: -.5px; margin-bottom: 32px;
}
.loading-title span { color: var(--blue); }
.progress-wrap {
  width: 200px; height: 4px;
  background: var(--fill); border-radius: 99px; margin-bottom: 12px;
}
.progress-bar {
  height: 100%; background: var(--blue);
  border-radius: 99px; width: 0%;
  transition: width .8s ease;
}
.loading-sub { font-size: 13px; color: var(--label-3); }
```

---

## App Header

Sticky top bar with title and user avatar:
```html
<div class="app-header">
  <div class="app-header-row">
    <div class="app-title">國小英文單字</div>
    <div id="headerAvatar" class="header-avatar">👤</div>
  </div>
</div>
```

```css
.app-header {
  background: rgba(242,242,247,.92);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  padding: 14px 16px 10px;
  border-bottom: .5px solid var(--separator);
  position: sticky; top: 0; z-index: 100;
}
.app-title { font-size: 17px; font-weight: 600; color: var(--label-1); }
.header-avatar {
  width: 32px; height: 32px; border-radius: 50%;
  background: var(--blue); color: #fff;
  display: flex; align-items: center; justify-content: center;
  font-size: 16px; cursor: pointer;
}
```

---

## Stats Card

White card replacing current colorful stats bar:
```css
.stats-card {
  margin: 12px 16px;
  background: var(--card);
  border-radius: var(--radius-lg);
  padding: 14px 16px;
  box-shadow: var(--shadow-card);
  display: flex; justify-content: space-between;
}
.stat-val { font-size: 20px; font-weight: 700; color: var(--label-1); }
.stat-lab { font-size: 10px; color: var(--label-3); margin-top: 2px; }
```

---

## Level Selector → Segmented Control

Replace level buttons with iOS segmented control:
```css
.seg-control {
  display: flex; background: var(--fill);
  border-radius: var(--radius-sm);
  margin: 0 16px 12px; padding: 2px;
}
.seg-btn {
  flex: 1; text-align: center;
  font-size: 13px; font-weight: 600; color: var(--label-3);
  padding: 7px 4px; border-radius: 8px; cursor: pointer;
}
.seg-btn.active {
  background: var(--card); color: var(--label-1);
  box-shadow: 0 1px 4px rgba(0,0,0,.12);
}
```

---

## Game Mode List

Replace grid cards with iOS list rows:
```css
.mode-list { padding: 0 16px; display: flex; flex-direction: column; gap: 1px; }
.mode-row {
  background: var(--card);
  padding: 13px 14px;
  display: flex; align-items: center; gap: 12px;
  box-shadow: var(--shadow-sm);
}
.mode-row:first-child { border-radius: var(--radius-md) var(--radius-md) 0 0; }
.mode-row:last-child  { border-radius: 0 0 var(--radius-md) var(--radius-md); }
.mode-row:only-child  { border-radius: var(--radius-md); }
.mode-icon {
  width: 36px; height: 36px; border-radius: 10px;
  display: flex; align-items: center; justify-content: center; font-size: 20px;
}
.mode-name { font-size: 15px; font-weight: 600; color: var(--label-1); }
.mode-desc { font-size: 12px; color: var(--label-3); margin-top: 1px; }
.mode-chevron { color: var(--separator); font-size: 16px; margin-left: auto; }
```

Mode icon tint classes: `quiz→blue`, `listen→orange`, `spell→green`, `reverse→purple`, `scramble→yellow`, `speed→red`

---

## Category Grid

Replace colored category buttons with white cards:
```css
.cat-grid {
  display: grid; grid-template-columns: 1fr 1fr;
  gap: 8px; padding: 0 16px 24px;
}
.cat-card {
  background: var(--card); border-radius: var(--radius-md);
  padding: 14px 12px; box-shadow: var(--shadow-sm);
  cursor: pointer;
}
.cat-card.active { border: 1.5px solid var(--blue); }
.cat-dot {
  width: 8px; height: 8px; border-radius: 50%;
  background: var(--blue); margin-bottom: 8px;
}
.cat-name { font-size: 14px; font-weight: 600; color: var(--label-1); }
.cat-count { font-size: 11px; color: var(--label-3); margin-top: 2px; }
```

---

## Section Labels

iOS-style uppercase labels above each section:
```css
.section-label {
  font-size: 12px; font-weight: 600;
  color: var(--label-3); text-transform: uppercase;
  letter-spacing: .05em; padding: 0 16px; margin: 16px 0 8px;
}
```

---

## Global Changes

- `body` background → `var(--bg)` (`#f2f2f7`)
- All fonts → `var(--font)` (system font stack)
- Remove all colorful gradient backgrounds from layout elements
- Remove existing `.game-modes`, `.cat-grid`, `.stats-bar`, `.level-bar` styles entirely
- Game play area (`.game-area`, choice buttons, progress bar) → keep functional, update colors to use `var(--blue)` and `var(--card)`
- TTS status badge → update to match new style (small pill, subtle)
- Remove the `h1` gradient text header (replaced by `.app-header`)

---

## What Does NOT Change

- All game logic JS (quiz, reverse, listen, spell, scramble, speed)
- Word database
- Profile system logic
- Audio / TTS code
- localStorage persistence
- Result screen logic (CSS updated, logic intact)
