# F1 2026 PWA — iPhone App

A Progressive Web App that installs on your iPhone home screen
and sends push notifications when race results drop.

---

## 🚀 Deploy to Vercel (2 minutes, free)

1. Go to **vercel.com** and sign in with GitHub
2. Click **"Add New Project"** → **"Browse"**
3. Drag and drop this entire `f1_pwa` folder
4. Click **Deploy** — done! You get a URL like `f1-2026.vercel.app`

---

## 📱 Install on iPhone

1. Open your Vercel URL in **Safari** (must be Safari, not Chrome)
2. Tap the **Share button** (box with arrow at bottom)
3. Scroll down and tap **"Add to Home Screen"**
4. Tap **"Add"** — the F1 icon appears on your home screen

---

## 🔔 Enable Notifications

1. Open the app from your home screen
2. Tap the **🔔 bell icon** in the top right
3. Tap **Allow** when Safari asks for permission
4. You'll get a notification whenever new race results appear

> **Note:** iPhone notifications require iOS 16.4+ and the app must be
> installed to your home screen (not opened via Safari directly).

---

## How it works

- Hits the **Jolpica F1 API** (free, no key needed) on every sync
- Polls automatically every **10 minutes** when notifications are on
- Stores "seen" round markers in localStorage so you only get one
  notification per race result — no spam
- Service worker caches the app shell so it works offline

---

## Files

| File | Purpose |
|---|---|
| `index.html` | Entire app (HTML + CSS + JS) |
| `sw.js` | Service worker (caching + background sync + notifications) |
| `manifest.json` | PWA metadata (name, icon, theme color) |
| `icon-192.png` | App icon (home screen) |
| `icon-512.png` | App icon (splash screen) |
| `vercel.json` | Hosting headers (required for service worker) |
