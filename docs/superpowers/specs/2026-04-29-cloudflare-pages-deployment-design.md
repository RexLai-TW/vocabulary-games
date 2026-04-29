# Cloudflare Pages Deployment Design

**Date:** 2026-04-29
**Status:** Approved

## Goal

Deploy 國小英文單字遊戲 to Cloudflare Pages via GitHub integration, enabling automatic deploys on every push to `main`.

## Architecture

```
Developer
   │ git push origin main
   ▼
GitHub (RexLai-TW/vocabulary-games)
   │ webhook trigger
   ▼
Cloudflare Pages Build Runner
   │ no build step — copies root directly
   ▼
CDN Edge (global)
   │
   ▼
https://vocabulary-games.pages.dev
```

## Configuration

| Setting | Value |
|---|---|
| GitHub repo | `RexLai-TW/vocabulary-games` |
| Production branch | `main` |
| Build command | *(empty)* |
| Build output directory | `/` |
| Root directory | *(empty)* |
| Node.js version | N/A — static site |

## Deployment Behavior

- **Production deploy:** every push to `main` → `vocabulary-games.pages.dev`
- **Preview deploy:** every push to any other branch / every PR → unique preview URL (`<hash>.vocabulary-games.pages.dev`)
- **Deploy time:** ~30–60 seconds after push

## Domain

- Production URL: `https://vocabulary-games.pages.dev`
- Custom domain: not needed (using free Cloudflare subdomain)

## Setup Steps (Dashboard — no code changes)

1. Log in to Cloudflare Dashboard → **Pages** → **Create a project**
2. Choose **Connect to Git** → authorize GitHub → select `RexLai-TW/vocabulary-games`
3. Set production branch to `main`
4. Leave Build command and Build output directory **empty**
5. Click **Save and Deploy**
6. First deploy runs automatically — visit `vocabulary-games.pages.dev` to verify

## What Does NOT Change

- No files added or modified in the repo
- No `wrangler.toml`, no `_headers`, no `_redirects` needed
- Existing git workflow unchanged: `git add → git commit → git push`

## Out of Scope

- Custom domain (not requested)
- Environment variables (not needed for static site)
- Access restrictions / authentication
