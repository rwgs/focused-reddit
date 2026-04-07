# Focused Reddit — PWA

A locked-down Reddit reader. Only your chosen subreddits. No search, no explore, no doom scrolling.

## Deploy to GitHub Pages (free hosting, 5 minutes)

### 1. Create a GitHub repo
- Go to https://github.com/new
- Name it `focused-reddit` (or whatever you like)
- Set it to **Public**
- Click **Create repository**

### 2. Upload the files
- On the repo page, click **"uploading an existing file"**
- Drag the entire contents of this zip into the upload area:
  - `index.html`
  - `sw.js`
  - `manifest.json`
  - `icons/` folder (with all the .png files)
- Click **Commit changes**

### 3. Enable GitHub Pages
- Go to **Settings** → **Pages**
- Under "Source", select **Deploy from a branch**
- Branch: `main`, folder: `/ (root)`
- Click **Save**
- Wait ~60 seconds, then your app is live at:
  `https://YOUR-USERNAME.github.io/focused-reddit/`

### 4. Install on Android
- Open the URL above in **Chrome** on your phone
- Tap the **three-dot menu** → **"Add to Home screen"** (or Chrome may show an install banner automatically)
- It now appears as a standalone app with its own icon — no browser chrome, no address bar

### 5. Hide the real Reddit app
- Move the real Reddit app into a buried folder or uninstall it
- Your home screen now has "Focused" — which only shows your chosen subs

## Editing your subreddits
Tap **Edit subs** in the app header. Your list is saved in the browser's localStorage and persists across sessions.

## How it works
- Fetches from Reddit's public JSON API (no account or API key needed)
- Service worker caches the app shell for fast loads
- Pull down to refresh on mobile
- "Load more" button instead of infinite scroll (intentional friction)
