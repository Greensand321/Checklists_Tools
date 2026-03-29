# FlowBoard — Sync & Android Deployment Report

**Version:** 1.0  
**Date:** March 2026  
**Scope:** Real-time cross-device sync via Firebase, Android PWA packaging, and Board Key access model

---

## 1. Overview

FlowBoard currently runs as a self-contained single HTML file using `localStorage` for persistence. This report describes the architecture and step-by-step implementation plan to extend it with:

- Real-time cross-device sync via Firebase Realtime Database
- A Board Key access model (no user accounts required)
- Offline support with automatic reconnect sync
- Android PWA installation (home screen, fullscreen, no Play Store required)

The goal is to preserve the single-file nature of the desktop app while enabling it to optionally connect to a cloud backend when configured.

---

## 2. Architecture Decision: Why Firebase Realtime Database

Several backend options were considered:

| Option | Pros | Cons |
|---|---|---|
| Firebase Realtime Database | Free tier, real-time push, works from a static HTML file via CDN SDK, offline support built-in | Requires Google account to set up project |
| Firestore | More powerful querying | Slightly more complex for a flat JSON board structure |
| Supabase | Open source, Postgres | WebSocket sync requires more manual setup |
| Custom server (Node/Express) | Full control | Requires hosting, ongoing maintenance |

**Firebase Realtime Database is the correct choice** for this use case because:
1. The board state is a single JSON object — exactly what RTDB is optimised for
2. The JavaScript SDK loads from Google CDN, requiring no build step or package manager
3. It has native offline persistence that queues writes and flushes on reconnect
4. The free Spark plan allows 1GB stored data and 10GB/month transfer — more than sufficient for a personal Kanban board
5. Security Rules allow per-key isolation without user authentication

---

## 3. Board Key Access Model

### Concept
A **Board Key** is a random alphanumeric string (e.g. `xk7f-29ab-m3qr`) that maps to an isolated path in the Firebase database:

```
/boards/{boardKey}/state   ← full board JSON
/boards/{boardKey}/meta    ← lastUpdated timestamp, device info
```

### Properties
- **No login required** — the key IS the credential
- **Each key is its own isolated board** — different keys cannot see each other's data
- **Shareable** — give someone your key and they see and edit the same board in real-time
- **Private by default** — keys are generated with enough entropy (~20 chars) that brute-forcing is not feasible
- **Multiple keys per device** — a user can store several keys locally and switch between boards

### Key Generation
Keys should be generated client-side using `crypto.getRandomValues()`:

```javascript
function generateKey() {
  var arr = new Uint8Array(12);
  crypto.getRandomValues(arr);
  var hex = Array.from(arr).map(function(b) {
    return b.toString(16).padStart(2, '0');
  }).join('');
  // Format as xxxx-xxxx-xxxx-xxxx-xxxx for readability
  return hex.match(/.{4}/g).join('-');
}
```

### Key Storage
Keys are stored in `localStorage` under `fb4_keys` as an array of objects:

```json
[
  { "key": "xk7f-29ab-m3qr-11cd-9fa2", "label": "Work Board", "lastUsed": 1712000000000 },
  { "key": "ab12-cd34-ef56-gh78-ij90", "label": "Personal", "lastUsed": 1711900000000 }
]
```

The active key is stored under `fb4_active_key`.

---

## 4. Firebase Setup (One-Time, ~5 Minutes)

### Step 1 — Create a Firebase Project
1. Go to [https://console.firebase.google.com](https://console.firebase.google.com)
2. Click **Add project**
3. Name it (e.g. `flowboard-sync`)
4. Disable Google Analytics (not needed)
5. Click **Create project**

### Step 2 — Enable Realtime Database
1. In the left sidebar, click **Build → Realtime Database**
2. Click **Create Database**
3. Choose a region (pick closest to your location)
4. Start in **locked mode** (you will add rules in Step 4)

### Step 3 — Get Config Values
1. Click the gear icon → **Project settings**
2. Scroll to **Your apps** → click the `</>` Web icon
3. Register the app (name it anything)
4. Copy the `firebaseConfig` object — it contains 7 values:

```javascript
var firebaseConfig = {
  apiKey: "AIza...",
  authDomain: "your-project.firebaseapp.com",
  databaseURL: "https://your-project-default-rtdb.firebaseio.com",
  projectId: "your-project",
  storageBucket: "your-project.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abc123"
};
```

> **Note:** The `apiKey` in Firebase is NOT a secret — it is a project identifier. Access control is enforced entirely by Security Rules (Step 4), not by keeping the API key hidden. It is safe to embed in the HTML file.

### Step 4 — Security Rules
In the Realtime Database console, click the **Rules** tab and set:

```json
{
  "rules": {
    "boards": {
      "$boardKey": {
        ".read": true,
        ".write": true
      }
    }
  }
}
```

This means:
- Anyone who knows a board key can read/write that specific board
- No other paths in the database are accessible
- No authentication is required (the key itself is the access control mechanism)

**Optional stricter rules** — if you want to prevent a board from exceeding a size limit:

```json
{
  "rules": {
    "boards": {
      "$boardKey": {
        ".read": true,
        ".write": "newData.val() == null || newData.toString().length < 524288"
      }
    }
  }
}
```

This caps each board at 512KB — well above what a normal board would ever reach.

---

## 5. HTML File Changes

### 5.1 Add Firebase SDK (in `<head>`)

```html
<!-- Firebase App (required) -->
<script src="https://www.gstatic.com/firebasejs/10.12.0/firebase-app-compat.js"></script>
<!-- Firebase Realtime Database -->
<script src="https://www.gstatic.com/firebasejs/10.12.0/firebase-database-compat.js"></script>
```

> Always pin to a specific version (e.g. `10.12.0`) rather than `latest` to avoid unexpected breaking changes. Check the current stable version at [https://firebase.google.com/docs/web/setup](https://firebase.google.com/docs/web/setup).

### 5.2 Config Block (user fills this in once)

```javascript
// ── FIREBASE CONFIG ── Fill in your project values below
var FIREBASE_CONFIG = {
  apiKey: "",
  authDomain: "",
  databaseURL: "",
  projectId: "",
  storageBucket: "",
  messagingSenderId: "",
  appId: ""
};
var FIREBASE_ENABLED = FIREBASE_CONFIG.databaseURL !== "";
```

### 5.3 Initialisation

```javascript
var db = null;
var _syncRef = null;
var _syncListener = null;
var _localWriting = false;

if (FIREBASE_ENABLED) {
  firebase.initializeApp(FIREBASE_CONFIG);
  db = firebase.database();
  firebase.database().goOnline();
}
```

### 5.4 Connecting to a Board Key

```javascript
function connectToKey(key) {
  // Detach any existing listener
  if (_syncRef && _syncListener) {
    _syncRef.off('value', _syncListener);
  }

  if (!FIREBASE_ENABLED || !key) return;

  _syncRef = db.ref('boards/' + key + '/state');

  _syncListener = _syncRef.on('value', function(snapshot) {
    // Ignore echoes of our own writes
    if (_localWriting) return;

    var remote = snapshot.val();
    if (remote) {
      S = remote;
      // Migrate any missing fields
      S.cols.forEach(function(c) {
        c.cards.forEach(function(k) {
          if (k.desc === undefined) k.desc = '';
          if (k.due === undefined) k.due = '';
        });
      });
      render();
      updateSyncStatus('synced');
    }
  });
}
```

### 5.5 Modified `save()` Function

```javascript
function save() {
  // Always write to localStorage as fallback
  try {
    var data = JSON.stringify(S);
    localStorage.setItem('fb4', data);
    var used = new Blob([data]).size + new Blob([localStorage.getItem('fb4_baks') || '']).size;
    if (used > 4 * 1024 * 1024 && !save._warned) {
      save._warned = true;
      notify('⚠️ Storage nearly full — export a backup soon!');
    } else if (used <= 4 * 1024 * 1024) {
      save._warned = false; }
  } catch(e) {
    notify('❌ Local save failed!');
  }

  // Write to Firebase if connected
  if (FIREBASE_ENABLED && _syncRef && activeKey) {
    _localWriting = true;
    _syncRef.set(S).then(function() {
      _localWriting = false;
      updateSyncStatus('synced');
    }).catch(function(err) {
      _localWriting = false;
      updateSyncStatus('error');
      console.error('Firebase write failed:', err);
    });
  }
}
```

### 5.6 Sync Status Indicator

Add a small indicator to the topbar that shows connection state:

```javascript
// States: 'offline' | 'syncing' | 'synced' | 'error' | 'local'
function updateSyncStatus(state) {
  var el = document.getElementById('sync-status');
  if (!el) return;
  var map = {
    local:   { text: '💾 Local',   color: 'var(--txt-dd)' },
    syncing: { text: '🔄 Syncing', color: '#f59e0b' },
    synced:  { text: '☁ Synced',  color: '#10b981' },
    error:   { text: '⚠ Error',   color: '#e11d48' },
    offline: { text: '📴 Offline', color: 'var(--txt-d)' }
  };
  var s = map[state] || map.local;
  el.textContent = s.text;
  el.style.color = s.color;
}
```

Monitor connection state:

```javascript
if (FIREBASE_ENABLED) {
  db.ref('.info/connected').on('value', function(snap) {
    updateSyncStatus(snap.val() ? 'synced' : 'offline');
  });
}
```

---

## 6. Offline Support

Firebase RTDB has built-in offline persistence. Enable it once at startup:

```javascript
if (FIREBASE_ENABLED) {
  firebase.database().goOffline(); // start paused
  firebase.database().goOnline();  // then reconnect — this initialises the cache
}
```

For full disk-backed offline persistence (survives page refresh while offline):

```javascript
firebase.database().setPersistenceEnabled(true); // Must be called before any other DB calls
```

> **Important:** `setPersistenceEnabled` must be called before any `db.ref()` calls. Place it immediately after `firebase.initializeApp()`.

When offline, writes are queued in memory (or on disk if persistence is enabled) and automatically flushed when the connection restores. No additional code is needed.

---

## 7. Key Management UI

A simple key management panel should be added to the app, accessible from the topbar. It should support:

- **Creating a new key** (generates locally, immediately connects)
- **Entering an existing key** (to join a shared board)
- **Labelling keys** (e.g. "Work", "Personal")
- **Switching between keys** (disconnects current listener, connects new one)
- **Copying a key to clipboard** (for sharing)
- **Disconnecting** (fall back to local-only mode)

The panel follows the same modal pattern already used throughout FlowBoard.

---

## 8. Android PWA

### What Is a PWA?
A Progressive Web App is a website that Android Chrome can install to the home screen. Once installed it:
- Launches fullscreen with no browser chrome
- Has its own icon and splash screen
- Caches assets for offline launch
- Behaves like a native app

### Required Files

You need two additional files alongside the HTML file when hosting:

**`manifest.json`**
```json
{
  "name": "FlowBoard",
  "short_name": "FlowBoard",
  "description": "Personal Kanban board with cloud sync",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#060d18",
  "theme_color": "#060d18",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

**`sw.js`** (Service Worker — enables offline launch)
```javascript
var CACHE = 'flowboard-v1';
var ASSETS = ['/', '/index.html', '/manifest.json'];

self.addEventListener('install', function(e) {
  e.waitUntil(caches.open(CACHE).then(function(c) { return c.addAll(ASSETS); }));
});

self.addEventListener('fetch', function(e) {
  e.respondWith(
    caches.match(e.request).then(function(r) { return r || fetch(e.request); })
  );
});
```

**In the HTML `<head>`:**
```html
<link rel="manifest" href="/manifest.json">
<meta name="theme-color" content="#060d18">
<meta name="mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-capable" content="yes">
```

**Register the service worker (end of `<body>`):**
```javascript
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js');
}
```

### Hosting Options

The HTML file must be served over HTTPS for PWA installation to work. Free options:

| Service | Notes |
|---|---|
| **GitHub Pages** | Free, HTTPS automatic, push to deploy. Ideal for this use case. |
| **Netlify** | Free tier, drag-and-drop deploy, HTTPS automatic |
| **Cloudflare Pages** | Free, very fast CDN, HTTPS automatic |

For GitHub Pages: create a repo, push `index.html`, `manifest.json`, `sw.js`, and two icon PNGs, enable Pages in repo settings — done.

### Installing on Android
1. Open the hosted URL in Chrome on Android
2. Chrome shows an **"Add to Home screen"** banner automatically after a few visits, or
3. Tap the three-dot menu → **Add to Home screen**
4. The app installs with an icon, opens fullscreen on next launch

---

## 9. Data Flow Summary

```
[Desktop Browser]          [Android PWA]
      |                          |
   save()                     save()
      |                          |
localStorage              localStorage
      |                          |
   Firebase ←—— real-time ——→ Firebase
   RTDB write              RTDB listener
```

Both clients write to Firebase on every change. Both listen for remote changes. The `_localWriting` flag prevents a device from re-applying its own echoed write, avoiding render flicker.

---

## 10. Implementation Order

1. Set up Firebase project and Security Rules (5 min, one-time)
2. Add Firebase SDK `<script>` tags to HTML
3. Add `FIREBASE_CONFIG` block and `FIREBASE_ENABLED` flag
4. Implement `connectToKey()` and modify `save()`
5. Add sync status indicator to topbar
6. Add Key Management modal
7. Enable offline persistence
8. Host on GitHub Pages (or equivalent)
9. Add `manifest.json`, `sw.js`, icons
10. Register service worker in HTML
11. Test PWA install on Android Chrome

---

## 11. Security Considerations

- The Firebase API key is **not a secret** — it is a public project identifier. Access is controlled by Security Rules, not the key.
- Board Keys should be treated like passwords — do not share publicly or commit to a public repo.
- Consider adding Firebase App Check in future to prevent automated abuse of the database.
- The current Security Rules allow any read/write to any board key path. For production hardening, add rate limiting via Cloud Functions or tighter Rules if needed.
- All data is transmitted over HTTPS (Firebase enforces this).

---

## 12. Free Tier Limits (Firebase Spark Plan)

| Resource | Limit | Expected Usage |
|---|---|---|
| Simultaneous connections | 100 | Well within for personal use |
| Storage | 1 GB | A board is ~10–50KB; thousands of boards fit |
| Download | 10 GB/month | Each sync is a few KB |
| Upload | 1 GB/month | Same |

For a personal or small-team tool, the free tier is effectively unlimited.

---

*End of report.*
