# AGENTS.md — Focused Reddit (FocusRed)

## Project Overview

FocusRed is a minimalist Progressive Web App (PWA) that serves as a focused Reddit reader. It intentionally limits browsing to user-chosen subreddits with no search, no explore, and no infinite scroll — designed to reduce doom-scrolling.

**Live site:** https://focus.rwgs.net  
**Stack:** Vanilla JavaScript, zero dependencies, no build step.

---

## Repository Structure

```
focused-reddit/
├── index.html      # Entire application (~1755 lines: HTML + CSS + JS)
├── sw.js           # Service Worker (caching strategy)
├── manifest.json   # PWA manifest (name: "FocusRed")
├── CNAME           # Custom domain: focus.rwgs.net
├── README.md       # GitHub Pages deployment guide
└── icons/          # PWA icons (72px – 512px PNGs)
```

**Single-file architecture:** All application logic lives in `index.html`. There is no src/ directory, no bundler, no transpiler, no package.json.

---

## Development Workflow

### Running locally

No build step is required. Serve the files with any static HTTP server:

```bash
# Python
python3 -m http.server 8080

# Node (npx)
npx serve .

# Or open index.html directly in browser (some PWA features may be limited)
```

### Deployment

Push to the `main` branch. GitHub Pages serves the repo root directly. The service worker caches the app shell on first load.

### No tests, no linting

This project has no test suite, no CI, and no linting configuration. Correctness is validated manually in-browser.

---

## Architecture

### Application structure inside `index.html`

| Lines       | Contents                                              |
|-------------|-------------------------------------------------------|
| 1–14        | HTML head, meta tags, PWA / Apple mobile config      |
| 15–905      | All CSS (embedded `<style>` block)                   |
| 906–910     | Service Worker registration                          |
| 911–926     | Global constants and mutable state variables         |
| 927–961     | Utility functions (`timeAgo`, `kFmt`, `decode`, `esc`, `sanitizeSubName`) |
| 963–1035    | Persistence layer (localStorage + URL hash)          |
| 1037–1049   | `renderPills()` — subreddit filter nav               |
| 1051–1109   | `renderPosts()`, `renderError()`                     |
| 1111–1228   | Data fetching (`fetchRedditJson`, `fetchPosts`, `fetchPostsChunked`) |
| 1229–1755   | Sort/filter controls, detail panel, comments, bookmarks, edit modal, event listeners, `init()` |

### Data flow

1. `init()` calls `loadSubs()` (URL hash → localStorage → defaults)
2. `fetchPosts()` builds a multi-subreddit URL (`/r/sub1+sub2+sub3/hot.json`) and calls `fetchRedditJson()`
3. Response is stored in `posts[]` and rendered with `renderPosts()`
4. Clicking a post calls `openDetail()` which fetches comments and renders the detail panel
5. State changes (filter, sort, subreddit edits) re-call `fetchPosts()` or `renderPosts()`

---

## Global State

All state is module-level variables inside the `<script>` block:

```js
let subs = [];                    // Ordered list of subreddit names
let selectedSubs = new Set();     // Active filter (empty = show all)
let currentSort = 'hot';          // 'hot' | 'top' | 'new' | 'rising'
let currentTime = 'day';          // 'hour'|'day'|'week'|'month'|'year'|'all' (top only)
let showingBookmarks = false;     // True when in bookmarks view
let bookmarks = [];               // Array of saved post objects
let posts = [];                   // Current post list from API
let afterToken = null;            // Reddit pagination cursor
let loading = false;              // Fetch in-progress guard
```

**Constants:**
```js
const STORAGE_KEY = 'focused_reddit_subs';   // localStorage key for subreddit list
const BOOKMARKS_KEY = 'focusred_bookmarks';  // localStorage key for bookmarks
const POSTS_PER_PAGE = 15;
const DEFAULT_SUBS = ['claudeai', 'selfhosted'];
```

---

## Persistence

Two-layer persistence; URL hash takes priority:

- **URL hash** (primary): `#sub1+sub2!selected1+selected2`  
  - Before `!`: full subreddit list  
  - After `!`: currently selected filter subs  
  - Enables shareable URLs
- **localStorage** (backup/sync): stores `subs` array as JSON under `STORAGE_KEY`
- **Bookmarks**: stored as full post data objects under `BOOKMARKS_KEY`

Key functions: `buildHash()`, `parseHash()`, `loadSubs()`, `saveSubs()`, `updateHash()`

---

## Reddit API

**Endpoints used:**

```
/r/{sub1}+{sub2}/hot.json?limit=15&raw_json=1
/r/{sub}/{sort}.json?limit=15&raw_json=1&t={time}   (top sort)
/r/{sub}/{id}/comments/{post_id}.json               (comments)
```

**Base URL fallback chain** (`REDDIT_BASES`):
1. `https://www.reddit.com` (primary)
2. `https://old.reddit.com`
3. `https://api.reddit.com`

**Error handling in `fetchRedditJson()`:**
- 8-second `AbortController` timeout per request
- 404 → user-friendly "subreddit not found" message
- 403 → "subreddit is private" message
- 429 → 2-second delay + one retry on same base
- 5xx → try next base URL
- Network/timeout → try next base URL

**Fallback fetch strategy:** If combined multi-subreddit URL fails, `fetchPostsChunked()` fetches in groups of 5, then individually per sub if those also fail.

---

## Service Worker (`sw.js`)

**Cache name:** `focusred-v1.1` — increment this when deploying breaking changes to force cache invalidation.

**Caching strategy:**
- Reddit API requests (`*.reddit.com`): **always network** (never cached)
- Everything else (app shell, fonts): **cache-first**, then network + cache on miss

To update the service worker cache, change `CACHE_NAME` in `sw.js`.

---

## CSS / Design System

All styles are in the `<style>` block (lines 15–905). Design tokens are CSS custom properties on `:root`:

```css
--bg: #0d1117           /* Page background (GitHub dark) */
--surface: #161b22      /* Card/modal background */
--surface-hover: #1c2230
--border: #21262d
--border-accent: #30363d
--text: #e6edf3         /* Primary text */
--text-dim: #8b949e     /* Secondary text */
--text-faint: #484f58   /* Placeholder / disabled text */
--accent: #58a6ff       /* Links, active states (GitHub blue) */
--accent-dim: #1f6feb33 /* Accent background tint */
--upvote: #f78166       /* Upvote score (coral) */
--comment: #7ee787      /* Comment count (green) */
--pill-bg: #21262d      /* Filter pill background */
--danger: #f85149       /* Delete / error (red) */
--radius: 12px          /* Standard border-radius */
--font: 'DM Sans', sans-serif
--mono: 'JetBrains Mono', monospace
```

**Responsive breakpoints:**
- `max-width: 480px` — mobile (1 column, smaller fonts)
- `min-width: 768px` — tablet (2-column grid)
- `min-width: 1100px` — desktop (3-column grid)

**Class naming:** kebab-case (`.post-card`, `.pill-nav`, `.detail-panel`, `.modal-close`).

---

## JavaScript Conventions

**Naming:**
- Functions: camelCase (`renderPosts`, `fetchRedditJson`, `openDetail`)
- CSS classes: kebab-case
- Constants: SCREAMING_SNAKE_CASE (`STORAGE_KEY`, `POSTS_PER_PAGE`)

**DOM updates:** Direct `innerHTML` assignment with template literals. No virtual DOM, no templating library.

**Event handling:** `onclick` attributes in generated HTML strings (not `addEventListener`). Global functions are called from inline handlers.

**XSS prevention — critical pattern:**
- `esc(str)` — escapes user/API data for safe HTML insertion (uses `textContent` internally)
- `decode(str)` — decodes HTML entities from Reddit API (uses `textarea.innerHTML`)
- **Never** insert raw API data into `innerHTML` without wrapping in `esc()`
- URL safety: only `http://` and `https://` URLs are rendered as links in markdown

**Subreddit validation:** Always pass user-entered subreddit names through `sanitizeSubName()` before storing or using. Accepts 2–21 chars, alphanumeric + underscores only, strips leading `r/` or `/r/`.

---

## Key Functions Reference

| Function | Purpose |
|---|---|
| `init()` | App entry point: loads subs, adjusts layout, fetches posts |
| `loadSubs()` | Loads subreddit list from URL hash or localStorage |
| `saveSubs()` | Writes subreddits to localStorage and URL hash |
| `renderPills()` | Renders subreddit filter buttons in header |
| `renderPosts()` | Renders post card grid (or bookmarks view) |
| `fetchPosts(append)` | Fetches posts; `append=true` for "load more" |
| `fetchPostsChunked()` | Fallback fetcher for large/broken multi-sub requests |
| `fetchRedditJson(path)` | Core fetch with fallback bases, timeout, rate-limit handling |
| `openDetail(sub, id, ev)` | Opens the comment/detail side panel |
| `renderComments(tree, depth)` | Recursive comment tree renderer (max depth 5) |
| `formatRedditText(text)` | Inline markdown parser (bold, italic, code, links, etc.) |
| `toggleBookmarkPost(id, ev)` | Saves/removes a post to localStorage bookmarks |
| `openEdit()` / `closeEdit()` | Shows/hides the "Edit subs" modal |
| `addSub()` / `removeSub(name)` | Add/remove subreddits from the list |
| `selectSub(name)` | Toggle subreddit filter (`__all__` clears selection) |
| `changeSort(sort)` | Changes sort mode and re-fetches |
| `sanitizeSubName(name)` | Validates and normalises subreddit name strings |
| `esc(str)` | HTML-escapes a string for safe innerHTML insertion |
| `decode(str)` | Decodes HTML entities from API responses |
| `timeAgo(utc)` | Formats Unix timestamp as "Xm/h/d/mo ago" |
| `kFmt(n)` | Formats number as "1.2k" or "3.4M" |

---

## Making Changes

### Adding a feature
All code goes in `index.html`. Add CSS in the `<style>` block, HTML in the `<body>`, and JavaScript in the `<script>` block. Follow the existing organisation (CSS → HTML → JS utilities → state → rendering → network → UI handlers).

### Changing the visual design
Edit CSS custom properties in `:root` for global changes. Individual components use the `--variable` tokens; avoid hardcoded colors.

### Changing the cache version
Update `CACHE_NAME` in `sw.js` when making changes that should invalidate cached assets:
```js
const CACHE_NAME = 'focusred-v1.2';  // bump minor version
```

### Adding a new subreddit sort mode
1. Add a button in the sort controls HTML section
2. Call `changeSort('newmode')` from its `onclick`
3. The value is passed directly into the Reddit API path — no other changes needed

### Modifying comment rendering
`renderComments()` is a recursive function. The depth limit (currently 5) prevents excessive nesting. Comment collapse/expand is handled by toggling `hidden` class via `collapseComment(el)`.

---

## Security Notes

- **Never** use `innerHTML = userInput` or `innerHTML = apiData` without `esc()`.
- The `formatRedditText()` markdown parser enforces URL allowlist (`http`/`https` only).
- `sanitizeSubName()` must be called on all user-submitted subreddit names before use in API paths or storage.
- No authentication tokens, API keys, or secrets — the app uses Reddit's public JSON API exclusively.

---

## PWA Notes

- **Installable** as standalone app on Android (Chrome) and iOS (Safari Add to Home Screen)
- **Offline:** App shell (HTML, fonts) is cached; post content requires network
- **Manifest:** `manifest.json` — app name is "FocusRed", theme `#0d1117`
- **Icons:** PNGs from 72×72 to 512×512 in `icons/`

---

## Git Branch

Production branch: `main` (auto-deployed to GitHub Pages)
