# Cloudflare Pages Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
> **Note:** This plan is pure configuration — no code changes. Tasks 2–3 require browser/dashboard interaction by the human operator.

**Goal:** Connect the GitHub repo to Cloudflare Pages for automatic deployment on every push to `main`, serving the app at `https://vocabulary-games.pages.dev`.

**Architecture:** Cloudflare Pages pulls directly from `RexLai-TW/vocabulary-games` on GitHub. No build step — `index.html` at the repo root is served as-is. Every push to `main` triggers a new production deploy automatically.

**Tech Stack:** Cloudflare Pages (static hosting), GitHub (source), Git CLI.

---

## File Map

No files are created or modified in the repo. This plan is purely operational.

---

### Task 1: Push Latest Commits to GitHub

**Files:** None

- [ ] **Step 1: Verify local state**

```bash
cd /d/Dev_workspace/vocabulary-games
git status
git log --oneline origin/main..HEAD
```

Expected output: one unpushed commit —
`94519e0 docs: add Cloudflare Pages deployment design spec`

- [ ] **Step 2: Push to origin**

```bash
git push origin main
```

Expected:
```
To https://github.com/RexLai-TW/vocabulary-games.git
   5734b68..94519e0  main -> main
```

- [ ] **Step 3: Confirm on GitHub**

Open `https://github.com/RexLai-TW/vocabulary-games/commits/main` and verify the latest commit is `docs: add Cloudflare Pages deployment design spec`.

---

### Task 2: Create Cloudflare Pages Project (Dashboard)

**Files:** None — dashboard configuration only.

- [ ] **Step 1: Open Cloudflare Pages**

Go to `https://dash.cloudflare.com` → log in → click **Workers & Pages** in the left sidebar → click **Pages** tab → click **Create a project**.

- [ ] **Step 2: Connect to Git**

Click **Connect to Git** → click **Connect GitHub** → authorize Cloudflare to access your GitHub account → select repository `RexLai-TW/vocabulary-games` → click **Begin setup**.

- [ ] **Step 3: Configure build settings**

Fill in the form exactly as follows:

| Field | Value |
|---|---|
| Project name | `vocabulary-games` |
| Production branch | `main` |
| Framework preset | `None` |
| Build command | *(leave empty)* |
| Build output directory | *(leave empty)* |
| Root directory | *(leave empty)* |

Click **Save and Deploy**.

- [ ] **Step 4: Wait for first deploy**

Cloudflare will show a build log. Wait until status changes to **Success** (usually 30–60 seconds).

Expected: green checkmark, message like "Your site has been deployed."

- [ ] **Step 5: Note the assigned URL**

Cloudflare assigns a permanent URL. It will be:
`https://vocabulary-games.pages.dev`

(If `vocabulary-games` is already taken by another Cloudflare account, Cloudflare appends a suffix like `vocabulary-games-abc.pages.dev` — use whatever URL is shown.)

---

### Task 3: Verify Live Deployment

**Files:** None

- [ ] **Step 1: Open the live URL**

Open `https://vocabulary-games.pages.dev` in a browser (desktop or mobile).

Expected: the game loads — you see the profile selection screen and the game mode grid.

- [ ] **Step 2: Test audio (iOS)**

Open the URL on an iOS device (Safari or Chrome).

1. Tap anywhere to unlock audio
2. Start a game → select **聽音選詞** mode
3. Tap the speaker button

Expected: word is pronounced via Web Speech API.

- [ ] **Step 3: Test audio (desktop)**

Open the URL on a desktop browser (Chrome or Firefox).

1. Start a game → select any mode
2. Answer correctly

Expected: ascending chime plays; answer wrong → buzzer plays.

- [ ] **Step 4: Confirm auto-deploy is wired**

Make a trivial change to verify the pipeline works end-to-end:

```bash
cd /d/Dev_workspace/vocabulary-games
# Add a comment to index.html — pick line 1 (<!DOCTYPE html>) and add nothing, just touch the file:
git commit --allow-empty -m "chore: verify Cloudflare Pages auto-deploy"
git push origin main
```

Then go to `https://dash.cloudflare.com` → **Pages** → `vocabulary-games` → **Deployments** tab.

Expected: a new deployment appears and reaches **Success** within 60 seconds.

---

## Self-Review

**Spec coverage:**
- [x] GitHub integration → Task 2, Step 2
- [x] `main` branch → Task 2, Step 3
- [x] No build command → Task 2, Step 3 (empty fields)
- [x] `vocabulary-games.pages.dev` → Task 2, Step 5
- [x] Auto-deploy verification → Task 3, Step 4
- [x] No code changes → confirmed — zero file edits in any task

**Placeholder scan:** None found.

**Type consistency:** N/A — no code.
