# Sub-project A: TTS Voice Improvement + Remove Memory Mode Design

**Date:** 2026-04-29
**Status:** Approved

## Goal

Two independent functional changes to `index.html`:
1. Remove the 記憶翻牌 (Memory Card) game mode entirely
2. Improve TTS female voice selection using a priority-based strategy that works across iOS, Windows, and Android

## Scope

Single file: `index.html` (~1940 lines). No new files. No UI redesign. No backend.

---

## Change 1: Remove Memory Card Mode

### What to Remove

**`gameModes` array (line ~628):**
```javascript
// DELETE this entry:
{ id: 'memory', icon: '🃏', name: '記憶翻牌', desc: '配對英文和中文' },
```

**CSS** — delete all rules for:
- `.memory-grid`
- `.memory-card`
- `.memory-card-inner`
- `.memory-front`
- `.memory-back`
- `.memory-card.flipped .memory-card-inner`
- `.memory-card.matched`

**JavaScript functions** — delete entirely:
- `startMemory()` (~lines 1558–1600)
- `renderMemory()` (~lines 1601–1640)
- `flipCard(idx)` (~lines 1641–1653)

**State object** — remove keys:
- `memoryCards`
- `flippedCards`
- `matchedPairs`

### Result

6 game modes remain: quiz, reverse, listen, spell, scramble, speed.
No dead code, no broken references.

---

## Change 2: TTS Voice Improvement

### Problem

Current voice selection:
```javascript
const enVoice = voices.find(v => v.lang.startsWith('en') && v.name.includes('Female'))
  || voices.find(v => v.lang.startsWith('en'));
```

`includes('Female')` matches nothing on any real platform — voice names like "Samantha", "Zira", "Google US English" don't contain "Female". Result: always falls through to "any English voice", which may be male or robotic.

### Solution

Add a known-female-voice priority list and an Enhanced/Premium check:

```javascript
// Near line 651 (with other TTS constants):
const FEMALE_VOICE_NAMES = [
  'Samantha', 'Zira', 'Eva', 'Karen',
  'Moira', 'Ava', 'Tessa', 'Victoria',
  'Allison', 'Susan', 'Fiona', 'Google US English'
];
```

Replace voice selection inside `speakSpeechSynthesis()`:
```javascript
const enVoices = voices.filter(v => v.lang.startsWith('en'));
const voice =
  enVoices.find(v => /enhanced|premium/i.test(v.name)) ||
  enVoices.find(v => FEMALE_VOICE_NAMES.some(n => v.name.includes(n))) ||
  enVoices[0];
if (voice) utterance.voice = voice;
```

### Priority Order

| Priority | Match | Platform example |
|---|---|---|
| 1st | Name contains "Enhanced" or "Premium" | iOS "Samantha (Enhanced)" |
| 2nd | Name matches known female list | Windows "Microsoft Zira", macOS "Samantha" |
| 3rd | Any English voice | Fallback |

### Scope

Only affects `speakSpeechSynthesis()`. Google Translate TTS path (non-iOS) is unchanged.

---

## What Does NOT Change

- All other game modes (quiz, reverse, listen, spell, scramble, speed)
- Google Translate TTS logic
- iOS audio unlock flow
- Profile system, stats, leaderboard
- Visual design / layout
