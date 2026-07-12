<div align="center">
  <img src="momo.png" alt="Chat Logo" width="80" height="80" />
  <h1>Momo Chat</h1>
  <p><strong>A private two-person real-time chat app with a 5-channel notification system and ESP32 hardware integration.</strong></p>

  ![Node.js](https://img.shields.io/badge/Node.js-18%2B-339933?style=flat-square&logo=node.js&logoColor=white)
  ![Firebase](https://img.shields.io/badge/Firebase-Realtime%20DB-FFCA28?style=flat-square&logo=firebase&logoColor=black)
  ![WebSocket](https://img.shields.io/badge/WebSocket-ws%208.x-010101?style=flat-square)
  ![ESP32](https://img.shields.io/badge/ESP32-Arduino-E7352C?style=flat-square&logo=arduino)
  ![Render](https://img.shields.io/badge/Hosted%20on-Render-46E3B7?style=flat-square)
</div>

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Project Structure](#3-project-structure)
4. [File-by-File Breakdown](#4-file-by-file-breakdown)
5. [Firebase Setup](#5-firebase-setup)
6. [Environment Variables](#6-environment-variables)
7. [Frontend Deep-Dive](#7-frontend-deep-dive)
8. [Notification System](#8-notification-system)
9. [ESP32 Hardware Client](#9-esp32-hardware-client)
10. [Running Locally](#10-running-locally)
11. [Deployment on Render](#11-deployment-on-render)
12. [Testing](#12-testing)
13. [Security](#13-security)
14. [Troubleshooting / FAQ](#14-troubleshooting--faq)

---

## 1. Overview

**chat-main** is a private, invite-only chat application built for exactly two users — **Sayem** and **Shajeda**. It is not a generic chat platform. Every design decision is intentional: PIN-based auth, panic logout, presence monitoring, and a notification pipeline that spans five different channels simultaneously.

**What makes it different from a normal chat app:**

- No signup flow. Identity is determined entirely by which 5-digit PIN you enter.
- When either user comes **online**, a cascade of notifications fires instantly across Discord, Telegram, Pushover, Android (FCM), and physical ESP32 hardware — all in parallel.
- The browser tab self-destructs (wipes DOM, redirects to `about:blank`) the moment it loses focus, for privacy.
- An ESP32 microcontroller with an OLED display and buzzer sits on a desk and plays a musical tone + shows a card when the other person comes online.
- The entire frontend is a single HTML file (~2,700 lines). No build step. No bundler.

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Firebase (obsidian-8234e)                         │
│                                                                       │
│  /messages        — all chat messages                                 │
│  /presence        — online/offline state per user                    │
│  /typing          — live typing indicator                             │
│  /readReceipts    — last-read message ID per user                    │
│  /drafts          — unsent draft text per user                       │
│  /settings        — shared font size, per-user PINs                 │
│  /userMappings    — Firebase UID → username mapping                  │
└──────────────┬──────────────────────────────────────────────────────┘
               │
               │  Admin SDK (write/watch)    Client SDK (read/write)
               │
┌──────────────▼──────────────────────────────────────────────────────┐
│               server.js  (Node.js · port 3000)                       │
│                                                                       │
│  ┌─────────────────────┐   ┌──────────────────────────────────────┐ │
│  │  HTTP static server  │   │  Notification dispatcher              │ │
│  │  GET  /             │   │                                       │ │
│  │  POST /api/login-   │   │  → Discord webhook                   │ │
│  │       visit         │   │  → Telegram Bot API                  │ │
│  └─────────────────────┘   │  → Pushover API                      │ │
│                             │  → FCM (momofy-32be8 project)        │ │
│  ┌─────────────────────┐   │  → WebSocket push (Android)          │ │
│  │  WebSocket Server    │   │  → WebSocket push (ESP32)            │ │
│  │  /       Android     │   └──────────────────────────────────────┘ │
│  │  /esp32  Hardware    │                                             │
│  └─────────────────────┘                                             │
└──────┬────────────────────────────────────┬──────────────────────────┘
       │                                    │
┌──────▼──────┐                   ┌─────────▼──────────────────┐
│  index.html  │                   │  ESP32 Hardware             │
│  (Browser)   │                   │  OLED SSD1306 + Buzzer      │
│  Full SPA    │                   │  wss://moo.qzz.io/esp32     │
└──────┬───────┘                   └────────────────────────────┘
       │
┌──────▼────────────────────────┐
│  desktop-notifier.js           │
│  (Separate process, any PC)    │
│  Native OS desktop popups      │
└───────────────────────────────┘
```

### Two Firebase Projects

| Project | ID | Purpose |
|---|---|---|
| Obsidian | `obsidian-8234e` | Realtime Database + Firebase Auth |
| Momofy | `momofy-32be8` | FCM push notifications to Android |

They are separate named Admin SDK instances to avoid collision:
```js
admin.initializeApp({ ... });                          // default — database
admin.initializeApp({ ... }, 'fcm-app');               // named — FCM only
```

### Notification Flow (Presence Change)

```
User opens tab
    → index.html writes presence/Sayem = { state: "online" }
        → Firebase triggers server.js watcher
            → sendDiscord()    (parallel)
            → sendTelegram()   (parallel)
            → sendPushover()   (parallel)
            → sendFCM()        (parallel)
            → pushToAndroid()  (parallel, via WebSocket)
            → pushToESP32()    (parallel, via WebSocket)
```

This means **FCM fires even when you open `index.html` as a standalone file** — the browser writes to Firebase, and the always-on hosted server picks that up and sends the push.

---

## 3. Project Structure

```
chat-main/
│
├── server.js                          # Main backend — HTTP + WebSocket + Firebase watcher
├── server_help.js                     # Reference/tutorial file (not used in production)
├── desktop-notifier.js                # Standalone PC presence monitor (native OS popups)
├── test-esp32-connection.js           # Dev utility — simulates ESP32 WebSocket client
│
├── index.html                         # Entire frontend SPA (~2,700 lines, no build step)
├── momo.png                           # App logo / favicon
│
├── esp32-notification-client.ino      # Arduino sketch for ESP32 hardware client
│
├── stickers.json                      # 6 sticker packs, ~400+ ImgBB-hosted .webp URLs
├── emojis.json                        # ~1,500 emojis in 12 categories
│
├── obsidian-firebase.json             # Service account — Realtime DB (obsidian-8234e)
├── android-firebase.json              # Service account — FCM (momofy-32be8)
├── momofy-32be8-firebase-adminsdk-... # Duplicate/backup of android FCM service account
├── obsidian-8234e-firebase-adminsdk-. # Duplicate/backup of database service account
│
├── .env                               # All secrets (loaded by dotenv in server.js)
├── package.json                       # Project manifest and dependencies
├── package-lock.json                  # Locked dependency tree
│
└── node_modules/                      # Installed packages (not committed)
```

> The two `*-adminsdk-*.json` files in the root are the same credentials as `obsidian-firebase.json` and `android-firebase.json` — kept as named backups. Only the short-named versions are `require()`d by the server.

---

## 4. File-by-File Breakdown

### `server.js` — The Backend Brain

The only file that runs on the server. It does five jobs simultaneously:

**1. Static file server**
Serves `index.html` and all assets (PNG, JSON, JS) from the project root. No Express — raw `http.createServer` with a manual MIME table.

**2. Login-visit endpoint**
```
POST /api/login-visit
Body: { "device": "Samsung Galaxy S23" }
```
Fires every time a browser loads `index.html`. Dispatches to all 5 notification channels + pushes to ESP32. This is how you know the moment someone opens the app, before they even enter their PIN.

**3. WebSocket server (two endpoints)**

| Path | Clients | Purpose |
|---|---|---|
| `/` (default) | Android app | Real-time presence pushes |
| `/esp32` | ESP32 hardware | Presence + login-visit alerts |

Uses the `ws` library attached directly to the HTTP server — one port (3000) serves both HTTP and WebSocket.

**4. Firebase presence watcher**
```js
db.ref('presence').on('value', (snapshot) => { ... });
```
Watches the entire `presence/` tree. On any state change, it compares against `knownStates{}` (in-memory map) to avoid re-firing on identical states. Only triggers notifications on actual transitions (`offline → online`).

**5. Heartbeat for ESP32**
```js
setInterval(() => {
  for (const ws of esp32Clients) {
    ws.send(JSON.stringify({ type: 'PING' }));
  }
}, 30000); // every 30 seconds
```
Keeps the ESP32 WebSocket connection alive through NAT/firewall timeouts.

---

### `index.html` — The Entire Frontend

A single HTML file that contains all CSS, all JavaScript, and all markup inline. No React, no Vue, no bundler. Loads Firebase SDK as ES modules directly from Google's CDN:

```html
<script type="module">
  import { initializeApp } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-app.js";
  import { getDatabase, ref, push, onValue, set, get,
           onDisconnect, serverTimestamp } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-database.js";
  import { getAuth, signInAnonymously } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-auth.js";
</script>
```

**Why a single file?**
- Can be opened directly in a browser (`file://`) for testing without a server
- Trivially deployable — drop it anywhere that serves static files
- Zero build toolchain to maintain

The file is split into logical sections internally:
1. CSS variables + global styles (~600 lines)
2. HTML markup — auth screen, main app shell, settings overlay (~400 lines)
3. JavaScript — auth, messaging, presence, UI (~1,700 lines)

---

### `desktop-notifier.js` — Standalone PC Monitor

Run this on any Windows/Mac/Linux machine to get **native OS desktop popups** when users come online or go offline. Completely independent of `server.js`.

```bash
node desktop-notifier.js
```

Uses `node-notifier` for cross-platform desktop popups. Connects directly to Firebase via Admin SDK — no HTTP, no WebSocket.

**Behavior on first run:**
- Loads all current presence states silently (no spam for users already online)
- Only fires desktop notifications for *changes* after startup
- Exception: if a user is *already online* when the script starts, it still shows one notification so you know they're active

---

### `esp32-notification-client.ino` — Hardware Sketch

Arduino sketch for an ESP32 board. Connects to the server's `/esp32` WebSocket endpoint over SSL (`wss://`). Handles two incoming event types pushed from the server:

| Event Type | Trigger | Hardware Response |
|---|---|---|
| `PRESENCE_UPDATE` | User comes online | OLED card + C5→E5→G5→E5 tone + 5× LED blink |
| `LOGIN_VISIT` | Someone opens the login page | OLED alert + A5→B5→C6→B5→A5 urgent tone + 7× rapid blink |
| `PING` | Server heartbeat (every 30s) | Sends back `PONG`, no display change |

---

### `server_help.js` — Reference Document

Not executed anywhere. It's a heavily-commented tutorial demonstrating the Discord webhook + WebSocket pattern used in `server.js`. Also includes copy-paste client-side connection code for:
- Browser JavaScript
- Android (Java, using `org.java-websocket`)
- iOS (Swift, using Starscream)

Useful if you want to replicate this notification architecture in another project.

---

### `test-esp32-connection.js` — Hardware Simulator

Simulates a physical ESP32 connecting to the server's `/esp32` WebSocket endpoint. Useful for testing server-side ESP32 handling without having hardware on hand.

```bash
node test-esp32-connection.js
```

Connects to `ws://localhost:3000/esp32`, sends `ESP32_HELLO`, and pretty-prints any `PRESENCE_UPDATE` frames it receives in a box format:

```
╔════════════════════════════════════════╗
║       PRESENCE UPDATE                  ║
╠════════════════════════════════════════╣
║ User:   Sayem                          ║
║ State:  online                         ║
║ Device: Samsung Galaxy S23             ║
║ Time:   10:42:15 AM                    ║
╚════════════════════════════════════════╝
```

---

### `stickers.json`

Flat JSON object. Top-level keys are pack names (e.g., `milk-mocha`, `peach-goma`, `brown-cony`). Each pack is an array of ImgBB-hosted `.webp` URLs. The browser fetches this once on first sticker-picker open and caches it in memory for the session.

### `emojis.json`

Array of category objects. Each has a `name` and `emojis` array. 12 categories including smileys, gestures, people, animals, food, activities, travel, objects, symbols, and nature. Lazy-loaded on first emoji picker open.

---

## 5. Firebase Setup

This project uses **two separate Firebase projects**. They must both exist and be configured correctly before the server will work.

---

### Project 1 — `obsidian-8234e` (Database + Auth)

This is the core project. All chat data lives here.

**Step 1: Create the project**
1. Go to [console.firebase.google.com](https://console.firebase.google.com)
2. Click **Add Project** → name it anything (e.g. `obsidian`)
3. Disable Google Analytics (not needed)

**Step 2: Enable Realtime Database**
1. In the left sidebar → **Build → Realtime Database**
2. Click **Create Database**
3. Choose a region (e.g. `us-central1`)
4. Start in **locked mode** (you'll set rules next)
5. Note the database URL: `https://your-project-default-rtdb.firebaseio.com`

**Step 3: Set database rules**
In the Realtime Database console → **Rules** tab, paste:
```json
{
  "rules": {
    ".read": "auth != null",
    ".write": "auth != null"
  }
}
```
This is the content of `rules.json`. It means only authenticated users (even anonymous) can read or write.

**Step 4: Enable Anonymous Authentication**
1. In the sidebar → **Build → Authentication**
2. Click **Get Started**
3. Under **Sign-in providers**, enable **Anonymous**

Without this, `signInAnonymously()` in the frontend will throw `auth/operation-not-allowed`.

**Step 5: Generate a service account**
1. In the sidebar → **Project Settings** (gear icon) → **Service Accounts**
2. Click **Generate new private key**
3. Save the downloaded JSON as `obsidian-firebase.json` in the project root
4. Also paste the entire JSON (minified) into your `.env` as `FIREBASE_SERVICE_ACCOUNT`

**Step 6: Seed initial data**
The app reads PINs from Firebase on login. You must set them manually before first use. In the Realtime Database console, create this structure:

```
settings/
  Sayem/
    pin: "12345"
  Shajeda/
    pin: "67890"
  shared/
    fontSize: 15
```

> Choose any 5-digit PINs. They can be changed later from the Settings menu inside the app.

---

### Project 2 — `momofy-32be8` (FCM / Android Push)

This project exists solely to send push notifications to the Android app. The Realtime Database is not used here.

**Step 1: Create the project**
1. Go to [console.firebase.google.com](https://console.firebase.google.com)
2. Click **Add Project** → name it anything (e.g. `momofy`)

**Step 2: Register your Android app**
1. In the project overview → click the Android icon
2. Enter your Android app's package name
3. Download `google-services.json` and add it to your Android project

**Step 3: Enable FCM**
Firebase Cloud Messaging is enabled by default. No extra steps needed.

**Step 4: Subscribe Android app to the `all_users` topic**
In your Android app's `MainActivity` (or `Application` class), add:
```java
FirebaseMessaging.getInstance().subscribeToTopic("all_users")
    .addOnCompleteListener(task -> {
        Log.d("FCM", "Subscribed to all_users topic");
    });
```
The server sends to topic `all_users` by default:
```js
sendFCM('User Online', `${device} is online`, 'all_users');
```

**Step 5: Generate service account**
1. **Project Settings → Service Accounts → Generate new private key**
2. Save as `android-firebase.json` in the project root
3. Also paste into `.env` as `ANDROID_FIREBASE_SERVICE_ACCOUNT`

---

### Firebase Data Structure Reference

```
/
├── messages/
│   └── {pushId}/
│       ├── sender: "Sayem"
│       ├── text: "hey"
│       ├── timestamp: 1720000000000   (serverTimestamp)
│       ├── imageUrl: "https://..."    (optional, ImgBB URL)
│       ├── sticker: "https://..."     (optional)
│       ├── edited: true               (optional)
│       ├── deleted: true              (optional)
│       ├── replyTo: {                 (optional)
│       │   sender: "Shajeda",
│       │   text: "original message"
│       │ }
│       └── reactions/
│           └── {emoji}/
│               └── {username}: true
│
├── presence/
│   └── {username}/
│       ├── state: "online" | "offline"
│       ├── lastSeen: 1720000000000
│       └── device: "Samsung Galaxy S23"
│
├── typing/
│   └── {username}: true | false
│
├── readReceipts/
│   └── {username}: "{pushId}"         (last read message ID)
│
├── drafts/
│   └── {username}: "unsent text..."
│
├── settings/
│   ├── shared/
│   │   └── fontSize: 15
│   ├── Sayem/
│   │   └── pin: "12345"
│   └── Shajeda/
│       └── pin: "67890"
│
└── userMappings/
    └── {firebaseUid}/
        ├── username: "Sayem"
        └── lastLogin: 1720000000000
```

---

## 6. Environment Variables

All secrets live in `.env` at the project root. The server loads them via `dotenv` at startup. On Render (production), these are set as environment variables in the dashboard — the local `.env` file is never uploaded.

```env
# ── Server ───────────────────────────────────────────────────────────
PORT=3000

# ── Firebase: Realtime Database (obsidian-8234e) ─────────────────────
# Paste the entire service account JSON as a single minified string
FIREBASE_SERVICE_ACCOUNT='{"type":"service_account","project_id":"obsidian-8234e",...}'

# ── Firebase: FCM / Android Push (momofy-32be8) ──────────────────────
# Paste the entire service account JSON as a single minified string
ANDROID_FIREBASE_SERVICE_ACCOUNT='{"type":"service_account","project_id":"momofy-32be8",...}'

# ── Discord ───────────────────────────────────────────────────────────
# From: Discord channel → Edit → Integrations → Webhooks → New Webhook
DISCORD_WEBHOOK=https://discord.com/api/webhooks/YOUR_ID/YOUR_TOKEN

# ── Telegram ──────────────────────────────────────────────────────────
# Bot token from @BotFather. Chat ID from @userinfobot or the getUpdates API.
TELEGRAM_BOT_TOKEN=1234567890:AABBCCDDEEFFaabbccddeeff
TELEGRAM_CHAT_ID=123456789

# ── Pushover ──────────────────────────────────────────────────────────
# App token from pushover.net/apps/build. User key from pushover.net dashboard.
PUSHOVER_APP_TOKEN=azGDORePK8gMaC0QOYAMyEEuzJnyUi
PUSHOVER_USER_KEY=uQiRzpo4DXghDmr9QzzfQu
```

### How credentials are loaded (fallback chain)

```js
// server.js — same pattern for both Firebase accounts
if (process.env.FIREBASE_SERVICE_ACCOUNT) {
  serviceAccount = JSON.parse(process.env.FIREBASE_SERVICE_ACCOUNT);  // production
} else {
  serviceAccount = require('./obsidian-firebase.json');                 // local dev
}
```

This means:
- **Local development**: put the JSON files in the project root, no `.env` needed for Firebase
- **Production (Render)**: set the env vars in the dashboard, never upload the JSON files

### Minifying JSON for env vars

To paste a service account JSON into an env var without line breaks:

**PowerShell:**
```powershell
(Get-Content obsidian-firebase.json -Raw) -replace '\s+', ' ' | Set-Clipboard
```

**Node.js one-liner:**
```bash
node -e "console.log(JSON.stringify(require('./obsidian-firebase.json')))"
```

Then paste the output as the value in your `.env` or Render dashboard (wrap in single quotes).

---

## 7. Frontend Deep-Dive

Everything in this section lives inside `index.html`. There is no separate JS file — all logic is inline in a `<script type="module">` block at the bottom of the file.

---

### 7.1 Authentication

The app uses a **PIN-based identity system** on top of Firebase Anonymous Auth. There is no email, no password, no account creation.

**Flow:**

```
User sees 5-dot PIN keypad
    ↓
User enters 5-digit PIN (tap or physical keyboard)
    ↓
signInAnonymously(auth)          ← creates a throwaway Firebase Auth session
    ↓
get(ref(db, 'settings/Sayem/pin'))   ← fetch both PINs from Firebase
get(ref(db, 'settings/Shajeda/pin'))
    ↓
Compare entered PIN against both
    ↓  match found
currentUser = 'Sayem' | 'Shajeda'
    ↓
set(ref(db, `userMappings/${uid}`), { username, lastLogin })
    ↓
initMainApp()                    ← load the full chat UI
```

**Why Anonymous Auth at all?**
Firebase Realtime Database rules require `auth != null`. Anonymous Auth satisfies this without any real user account. The UID is mapped to a username in `/userMappings/` for audit purposes.

**Wrong PIN behavior:**
- Auth session is signed out immediately
- PIN dots play a shake animation (`shakeDots()`)
- Error message shows for 2 seconds
- Keyboard and keypad are re-enabled

**Physical keyboard support:**
The PIN screen listens for `keydown` events. Digits 0–9 fill the buffer; `Backspace` deletes. Auto-submits as soon as the buffer reaches 5 characters.

---

### 7.2 Panic Logout

The app wipes itself the moment the browser tab loses focus. This is the core privacy feature.

**Triggers:**
```js
document.addEventListener('visibilitychange', () => {
  if (document.hidden) triggerPanicLogout();
});
window.addEventListener('blur', triggerPanicLogout);
window.addEventListener('pagehide', triggerPanicLogout);
```

**What `triggerPanicLogout()` does:**
1. Signs out from Firebase Auth (`auth.signOut()`)
2. Clears `localStorage` and `sessionStorage`
3. Calls `window.location.reload()` — returns to the PIN screen

**Guards that prevent false triggers:**

| Guard | Purpose |
|---|---|
| `DEV_MODE = false` | Set to `true` during development to disable panic logout entirely |
| `isSelectingFile = true` | Set before opening file picker (image upload) — prevents logout when the OS file dialog steals focus |
| Call button sets `isSelectingFile = true` | Prevents logout when navigating to the video call page |

**Manual kill switch:**
The lightning bolt button (`.btn-flash`) in the header also triggers panic logout and additionally opens `https://translate.google.com` in a new tab as a decoy.

---

### 7.3 Presence System

The presence system tracks online/offline state in real time and shows a live "last seen" countdown.

**Going online:**
```js
// Fires when Firebase reports the client is connected
onValue(connectedRef, (snap) => {
  if (snap.val() === true) {
    // Register what to write when disconnected (handled server-side by Firebase)
    onDisconnect(myPresenceRef).set({
      state: 'offline',
      lastSeen: serverTimestamp(),
      device: ''
    });
    // Write online state immediately
    updateMyPresence(); // writes { state: 'online', lastSeen, device }
  }
});
```

**`onDisconnect`** is a Firebase feature: it pre-registers a write that Firebase executes automatically if the client disconnects unexpectedly (tab closed, network drop, app killed). This guarantees the `offline` state is always written even if JavaScript never runs again.

**Device detection:**
```js
async function getDeviceModel() {
  // Uses navigator.userAgentData.getHighEntropyValues(['model'])
  // Falls back to navigator.platform if unavailable
}
```

**Live countdown:**
The header shows `last seen X·Y·Z` (hours·minutes·seconds) for the other user when they're offline. A `setInterval` updates this every 1 second using the `lastSeen` timestamp stored in Firebase.

**Presence display in header:**
- Green dot + "online" when partner is connected
- Grey dot + countdown timer when offline
- Device model shown next to status

---

### 7.4 Messaging

**Sending a message:**
```js
push(messagesRef, {
  sender: currentUser,
  text: chatInput.value.trim(),
  timestamp: serverTimestamp()
});
```
`push()` generates a unique, time-ordered key (a Firebase "push ID"). Messages are never manually sorted — the push ID order is chronological.

**Receiving messages:**
`onValue(messagesRef, ...)` fires on every change to the entire messages tree. The handler diffs the incoming snapshot against `allMessages` (in-memory array) and only renders new/changed nodes to avoid full redraws.

**Column-reverse scroll trick:**
```css
.messages {
  display: flex;
  flex-direction: column-reverse;
}
```
The messages container is flipped upside-down with CSS. New messages are prepended to the DOM (at the visual bottom). This means the browser's natural scroll anchor keeps the view pinned to the latest message with zero JavaScript scroll manipulation.

**FLIP animation:**
When a new bubble appears, it uses the FLIP technique (First/Last/Invert/Play) for a smooth push-up animation:
1. **First**: Record current positions of all visible bubbles
2. **Last**: Insert new bubble (DOM reflows)
3. **Invert**: Apply CSS transforms to move bubbles back to their old positions
4. **Play**: Remove transforms with a CSS transition — bubbles animate to their new positions

This is O(n) but runs entirely in the compositor thread (no layout thrashing).

---

### 7.5 Read Receipts

Sent messages show a small status indicator that transitions from grey (sent) to white (seen).

**How it works:**
```js
// IntersectionObserver watches each received message bubble
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      // This message is visible on screen
      set(ref(db, 'readReceipts/' + currentUser), messageId);
      observer.unobserve(entry.target);
    }
  });
}, { threshold: 0.5 });
```

The observer fires when a received bubble is at least 50% visible. It writes the message ID to `/readReceipts/{currentUser}`. The other user's client watches that path and updates the `data-status` attribute on matching sent bubbles:

```css
.bubble[data-status="seen"] .message-text { color: var(--text); }
.bubble[data-status="sent"] .message-text { color: var(--text-secondary); }
```

No polling. No timers. Pure event-driven.

---

### 7.6 Typing Indicator

```js
// On every keypress in the input
set(ref(db, 'typing/' + currentUser), true);

// Clear after 2 seconds of no typing
clearTimeout(typingTimeout);
typingTimeout = setTimeout(() => {
  set(ref(db, 'typing/' + currentUser), false);
}, 2000);
```

The partner's client watches `/typing/{otherUser}`. When `true`, a pulsing animated border appears on the top edge of the input area — subtle and non-intrusive.

---

### 7.7 Image Sharing

Images are uploaded to **ImgBB** (not Firebase Storage). The API key is hardcoded in the frontend:
```js
const IMGBB_API_KEY = "c3b5dc7293956c433f6b2cd1cdd121f6";
```

**Upload flow:**
1. User selects a file (or pastes from clipboard)
2. A skeleton placeholder with the correct aspect ratio is inserted into the message list immediately (zero perceived latency)
3. File is uploaded to ImgBB in the background via `fetch()`
4. On success, the returned URL is pushed to Firebase as `{ imageUrl: "https://..." }`
5. Skeleton is replaced by the actual image

**Clipboard paste support:**
```js
document.addEventListener('paste', (e) => {
  const file = e.clipboardData.files[0];
  if (file && file.type.startsWith('image/')) {
    uploadImage(file);
  }
});
```

---

### 7.8 Sticker Picker

6 sticker packs with ~400+ stickers total. Pack names: `milk-mocha`, `peach-goma`, `brown-cony`, `little-bean`, `muzzi`, and one additional pack.

**Performance design:**
- `stickers.json` is fetched once, on first open, and kept in memory
- Each tab's sticker DOM is built once and cached in a `Map` keyed by pack name
- Switching tabs swaps a single CSS class — no re-render
- Individual sticker images use `IntersectionObserver` for lazy loading — only stickers scrolled into view have their `src` set

```js
const stickerObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      entry.target.src = entry.target.dataset.src;
      stickerObserver.unobserve(entry.target);
    }
  });
});
```

---

### 7.9 Reactions

Any message can receive emoji reactions. Tap and hold (long-press) a message bubble to open the reaction picker (7 emojis).

**Data structure:**
```
/messages/{id}/reactions/{emoji}/{username}: true
```

Example:
```
/messages/-abc123/reactions/❤️/Sayem: true
/messages/-abc123/reactions/😂/Shajeda: true
```

**Rendering:** Reactions are grouped by emoji below the bubble. Each shows a count. If the current user reacted with that emoji, it highlights. Tapping an existing reaction of yours removes it (toggle).

---

### 7.10 Message Actions (Long-Press Menu)

Long-pressing a bubble opens a context menu with:

| Action | Available to |
|---|---|
| Reply | Both users |
| React | Both users |
| Copy text | Both users |
| Edit | Sender only |
| Delete | Sender only |

**Edit:** Updates the Firebase node in place, adds `edited: true`. The bubble shows a small "edited" label.

**Delete:** Sets `deleted: true` on the node and clears `text` and `imageUrl`. The bubble shows "This message was deleted" in muted style. The node is never removed — this preserves reply-chain context.

**Reply:** Sets a `currentReplyTo` local state. The input area shows a preview strip. The sent message includes a `replyTo: { sender, text }` object. Replied-to content renders as a quoted block above the bubble.

---

### 7.11 Settings

Accessed via the dropdown menu in the header (the `activeBtn`).

| Setting | Storage | Notes |
|---|---|---|
| Font size | `/settings/shared/fontSize` | Synced in real-time to both users via `onValue` |
| Update PIN | `/settings/{currentUser}/pin` | Validates current PIN, checks no collision with other user's PIN |
| Clear chat | `set(messagesRef, null)` | Wipes entire `/messages/` node. Irreversible. |
| Export chat | Local JSON blob download | Saves `[{sender, text, timestamp}]` as a `.json` file |

Font size changes are **shared** — if Sayem changes the font size, Shajeda's screen updates immediately too (they share `/settings/shared`). This is intentional.

---

### 7.12 URL Linkification

Message text is scanned with a regex before rendering. URLs are wrapped in `<a>` tags:

```js
const urlRegex = /(https?:\/\/[^\s]+)|(www\.[^\s]+)|([a-zA-Z0-9][a-zA-Z0-9-]*\.[a-zA-Z]{2,}[^\s]*)/g;
```

- Adds `https://` prefix if the URL has no protocol
- Opens in `_blank` with `rel="noopener noreferrer"`
- Long-press on a link still opens the message menu (link's `contextmenu` is suppressed; message's long-press handler takes over)

---

### 7.13 Video Call

The call button in the header navigates to:
```
https://webrtc-video-call-krb6.onrender.com/
```
This is a separate WebRTC app hosted independently. The button sets `isSelectingFile = true` before navigating to prevent panic logout from firing as the tab transitions.

---

## 8. Notification System

Every significant event fires to **all 5 channels simultaneously** via non-blocking async calls. A failure in one channel never blocks or affects the others.

```
Event occurs
    ├─→ sendDiscord()    — fire and forget (https.request)
    ├─→ sendTelegram()   — fire and forget (https.request)
    ├─→ sendPushover()   — fire and forget (https.request)
    ├─→ sendFCM()        — async/await (firebase-admin messaging)
    ├─→ pushToAndroid()  — synchronous loop over WebSocket Set
    └─→ pushToESP32()    — synchronous loop over WebSocket Set
```

**Events that trigger notifications:**

| Event | Channels fired |
|---|---|
| Server starts | Discord, FCM, Pushover, Telegram |
| Login page visited (`/api/login-visit`) | Discord, Pushover, Telegram, FCM, ESP32 |
| User comes online (presence: offline → online) | Discord, Pushover, Telegram, FCM, Android WS, ESP32 WS |

Going **offline** is logged to the server console but does **not** fire external notifications — only the online transition does.

---

### 8.1 Discord

Uses a Discord **Incoming Webhook** — no bot token, no OAuth, no library.

**How to create a webhook:**
1. Open Discord → right-click any text channel → **Edit Channel**
2. **Integrations → Webhooks → New Webhook**
3. Give it a name and icon, then **Copy Webhook URL**
4. Paste into `.env` as `DISCORD_WEBHOOK`

**What the server sends:**
```
Login page visited:
🚨 **Login Page Visited**
Device: Samsung Galaxy S23
Time: 10:42:15 AM

User online:
Samsung Galaxy S23 is online | 10:42:15 AM

Server start:
Server started and monitoring presence
```

**Implementation note:** Uses `https.request` (built-in Node.js) instead of `fetch`. This avoids any extra dependency and works on all Node versions.

```js
function sendDiscord(message) {
  const body = JSON.stringify({ content: message });
  // POST to webhook URL with Content-Type: application/json
  // Expects 204 No Content on success
}
```

---

### 8.2 Telegram

Uses the **Telegram Bot API** — send a message to a chat via HTTP POST.

**How to set up:**

1. **Create a bot:** Message `@BotFather` on Telegram → `/newbot` → follow prompts → copy the bot token
2. **Get your Chat ID:**
   - Start a conversation with your bot (send `/start`)
   - Visit: `https://api.telegram.org/bot{YOUR_TOKEN}/getUpdates`
   - Find `"chat":{"id":XXXXXXXXX}` in the response — that number is your chat ID
3. Set `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` in `.env`

**What the server sends (HTML parse mode):**
```
Login page visited:
🚨 <b>Login Page Visited</b>
Device: Samsung Galaxy S23
Time: 10:42:15 AM

User online:
🟢 <b>User Online</b>
<b>Sayem</b> is online on Samsung Galaxy S23 at 10:42:15 AM

Server start:
🤖 <b>Server Started</b>
Server started and monitoring presence.
```

Messages use Telegram's `parse_mode: 'HTML'` — `<b>` tags render as bold.

---

### 8.3 Pushover

[Pushover](https://pushover.net) is a paid notification service ($5 one-time per platform) that delivers instant push notifications to iOS and Android with a native app.

**How to set up:**
1. Create an account at [pushover.net](https://pushover.net)
2. Note your **User Key** from the dashboard
3. Go to [pushover.net/apps/build](https://pushover.net/apps/build) → create an application → note the **API Token**
4. Install the Pushover app on your phone
5. Set `PUSHOVER_APP_TOKEN` and `PUSHOVER_USER_KEY` in `.env`

**What the server sends:**

| Event | Title | Message |
|---|---|---|
| Server start | `⬢ SERVER` | `ACTIVE  ·  10:42:15 AM` |
| Login page visited | `⬢ LOGIN` | `Samsung Galaxy S23  ·  10:42:15 AM` |
| User online | `◉ ONLINE` | `Samsung Galaxy S23  ·  10:42:15 AM` |

Pushover messages are intentionally terse — title + one line. They appear as a system notification with a distinct sound.

**Implementation:** Posts to `https://api.pushover.net/1/messages.json` using `application/x-www-form-urlencoded` (not JSON).

---

### 8.4 FCM (Firebase Cloud Messaging)

Sends push notifications to the Android app even when it's in the background or completely closed.

**How it works in this project:**

The server uses the **Admin SDK** (`fcm-app` named instance) to send to the `all_users` topic. Any Android device that subscribed to that topic receives the push.

```js
async function sendFCM(title, body, topic = 'all_users') {
  const message = {
    data: { title, body },        // data payload (not notification payload)
    android: { priority: 'high' },
    topic: topic
  };
  await fcmAdmin.messaging().send(message);
}
```

**Why `data` payload instead of `notification` payload?**
A `notification` payload is handled by the Android system tray automatically. A `data` payload is delivered to the app's `FirebaseMessagingService` for full custom handling — the Android app controls how and whether to show it.

**Android app requirements:**
```java
// In Application or MainActivity:
FirebaseMessaging.getInstance().subscribeToTopic("all_users");

// In your FirebaseMessagingService:
@Override
public void onMessageReceived(RemoteMessage remoteMessage) {
    Map<String, String> data = remoteMessage.getData();
    String title = data.get("title");
    String body = data.get("body");
    // Show your custom notification here
}
```

**Why a second Firebase project?**
The Admin SDK only allows one default app instance. Since the server already initialized the default app for the Realtime Database (`obsidian-8234e`), FCM needs a **named** app instance pointing to `momofy-32be8`:
```js
fcmAdmin = admin.initializeApp({
  credential: admin.credential.cert(fcmServiceAccount)
}, 'fcm-app');  // ← named instance, not default
```

**Important:** FCM fires even when the server is running on Render and you open `index.html` as a local file. The browser writes to Firebase → Render's server sees the presence change → Render fires FCM. The local machine is irrelevant.

---

### 8.5 WebSocket Push (Android)

Real-time push to a connected Android app over a persistent WebSocket connection. Faster than FCM for foreground notifications since there's no FCM delivery latency.

**Connection:**
```
ws://your-server:3000/          (local)
wss://your-render-url/          (production)
```

The Android app connects to the root path `/`. The server adds the socket to the `androidClients` Set.

**Messages pushed to Android:**

```json
// User comes online
{
  "type": "PRESENCE_UPDATE",
  "user": "Sayem",
  "state": "online",
  "device": "Samsung Galaxy S23",
  "time": "10:42:15 AM"
}
```

**Push helper:**
```js
function pushToAndroid(data) {
  const payload = JSON.stringify(data);
  for (const ws of androidClients) {
    if (ws.readyState === 1) {   // 1 = OPEN
      ws.send(payload);
    }
  }
}
```

`readyState === 1` checks that the socket is actually open before sending — stale/closed sockets are silently skipped (they'll be cleaned up on their `close` event).

---

### 8.6 WebSocket Push (ESP32)

Same mechanism as Android push, but on a separate `/esp32` path with different handling.

**Connection:**
```
wss://moo.qzz.io/esp32          (primary — Cloudflare tunnel)
wss://your-render-url/esp32     (fallback)
```

**Messages pushed to ESP32:**

```json
// User comes online
{
  "type": "PRESENCE_UPDATE",
  "user": "Sayem",
  "state": "online",
  "device": "Samsung Galaxy S23",
  "time": "10:42:15 AM"
}

// Login page visited
{
  "type": "LOGIN_VISIT",
  "device": "Samsung Galaxy S23",
  "time": "10:42:15 AM"
}
```

**Heartbeat:** The server sends a `PING` every 30 seconds to every connected ESP32 client. The ESP32 sketch responds with a `PONG`. If the ESP32 doesn't receive a PING for 45 seconds, it assumes the connection is dead and reconnects.

**Cloudflare Tunnel (`moo.qzz.io`):**
The ESP32 connects to `moo.qzz.io` which is a Cloudflare tunnel pointing to the server. This gives:
- Free HTTPS/WSS without a certificate on the server
- A stable public hostname even if the server's IP changes
- DDoS protection from Cloudflare

To set up your own tunnel: install `cloudflared`, run `cloudflared tunnel create`, map it to `localhost:3000`.

---

### 8.7 Desktop Notifier (Bonus Channel)

Not part of `server.js` — a separate process you run on any PC.

```bash
node desktop-notifier.js
```

Uses `node-notifier` which wraps:
- **Windows**: Windows Toast Notifications
- **macOS**: macOS Notification Center
- **Linux**: `notify-send`

No configuration needed beyond having `obsidian-firebase.json` in the same directory. Connects directly to Firebase — no dependency on the server being up.

---

## 9. ESP32 Hardware Client

The `esp32-notification-client.ino` sketch turns a cheap ESP32 board into a physical notification device that sits on your desk. When the other person comes online or opens the app, the OLED lights up with their name and a musical tone plays.

---

### 9.1 Required Hardware

| Component | Spec | Notes |
|---|---|---|
| ESP32 board | Any variant | ESP32-DevKitC, WROOM-32, S2, S3 all work |
| OLED Display | SSD1306, 128×64, I2C | 4-pin module (VCC, GND, SCL, SDA) |
| Buzzer | Active or passive, 3-pin | Must be a **passive** buzzer for tones (active buzzer only beeps) |
| Breadboard + jumper wires | — | For prototyping |

> A passive buzzer is essential. An active buzzer only produces a single fixed tone regardless of frequency. The sketch uses PWM to generate musical notes.

---

### 9.2 Wiring Diagram

```
ESP32 Pin       Component
─────────────────────────────────────────
3.3V    ──────→ OLED VCC
GND     ──────→ OLED GND
GPIO 22 ──────→ OLED SCL  (I2C clock)
GPIO 21 ──────→ OLED SDA  (I2C data)

3.3V    ──────→ Buzzer VCC  (or use 5V for louder tone)
GND     ──────→ Buzzer GND
GPIO 25 ──────→ Buzzer Signal
```

If your OLED module has a different I2C address than `0x3C`, scan for it:
```cpp
// Run a quick I2C scanner sketch to find your OLED address
// Common addresses: 0x3C (most 0.96" modules) or 0x3D
#define SCREEN_ADDRESS 0x3C
```

Pin assignments can be changed at the top of the sketch:
```cpp
#define BUZZER_PIN 25
#define I2C_SDA    21
#define I2C_SCL    22
#define LED_PIN    2   // built-in LED
```

---

### 9.3 Arduino IDE Setup

**Step 1: Install ESP32 board support**
1. Open Arduino IDE → **File → Preferences**
2. In "Additional Board Manager URLs" add:
   ```
   https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
   ```
3. **Tools → Board → Boards Manager** → search `esp32` → install **esp32 by Espressif Systems**

**Step 2: Install required libraries**
Go to **Sketch → Include Library → Manage Libraries** and install:

| Library | Author | Version |
|---|---|---|
| WebSocketsClient_Generic | Khoi Hoang | latest |
| ArduinoJson | Benoit Blanchon | 7.x |
| Adafruit SSD1306 | Adafruit | latest |
| Adafruit GFX Library | Adafruit | latest |

> `WiFi.h` and `Wire.h` are built into the ESP32 Arduino core — no separate install needed.

**Step 3: Select your board**
- **Tools → Board → ESP32 Arduino → ESP32 Dev Module** (or match your specific board)
- **Tools → Port** → select the COM port your ESP32 is on

---

### 9.4 Sketch Configuration

Before uploading, edit the top of `esp32-notification-client.ino`:

```cpp
// ── WiFi ────────────────────────────────────────────────────────────
const char* ssid     = "YourWiFiName";
const char* password = "YourWiFiPassword";

// ── WebSocket server ────────────────────────────────────────────────
// Primary: your Cloudflare tunnel hostname (no https://, no trailing slash)
const char* ws_host_primary  = "moo.qzz.io";
const uint16_t ws_port_primary = 443;

// Fallback: your Render app hostname
const char* ws_host_fallback  = "your-app.onrender.com";
const uint16_t ws_port_fallback = 443;

const char* ws_path = "/esp32";
```

Both connections use **SSL** (`beginSSL`) — the sketch connects over `wss://` automatically.

---

### 9.5 How the Sketch Works

**Startup sequence:**
1. Initialize LED, buzzer (PWM on channel 0), I2C, OLED
2. Play a startup tone (1000 Hz → 1500 Hz)
3. Display "Connecting WiFi..." on OLED
4. Connect to WiFi (`WiFi.begin`)
5. Connect WebSocket to primary server via SSL
6. Register `webSocketEvent` callback

**Main loop:**
```cpp
void loop() {
  webSocket.loop();          // process incoming WS frames
  checkWiFi();               // reconnect WiFi if dropped
  checkHeartbeat();          // reconnect WS if no PING received in 45s
}
```

**WebSocket event handler:**

```cpp
void webSocketEvent(WStype_t type, uint8_t* payload, size_t length) {
  switch(type) {
    case WStype_CONNECTED:
      // Send ESP32_HELLO handshake to server
      // Display "Connected" on OLED
      break;

    case WStype_TEXT:
      // Parse JSON
      // Dispatch to handlePresenceUpdate() or handleLoginVisit()
      // Handle PING → send PONG
      break;

    case WStype_DISCONNECTED:
      // Display "Disconnected" on OLED
      // webSocket.setReconnectInterval(5000) handles auto-reconnect
      break;
  }
}
```

---

### 9.6 OLED Display Cards

**Presence Update card** (user comes online):
```
┌────────────────────────────┐
│ ◉ ONLINE                   │  ← inverted header bar
├────────────────────────────┤
│                            │
│  Sayem                     │  ← large text (size 2)
│                            │
│  Galaxy S23                │  ← device, small text
│  10:42:15 AM               │  ← time, small text
└────────────────────────────┘
```

**Login Visit card** (someone opens the app):
```
┌────────────────────────────┐
│ ⚠ VISITOR                  │  ← inverted header bar
├────────────────────────────┤
│                            │
│  LOGIN PAGE                │  ← large text
│                            │
│  Galaxy S23                │
│  10:42:15 AM               │
└────────────────────────────┘
```

**Status card** (connecting, WiFi lost, etc.):
```
┌────────────────────────────┐
│ Chat Monitor               │
│ ────────────────────────── │
│ Connecting...              │
│                            │
│ ▓▓▓▓▓░░░░░░░░░░░░░░░░░░░░ │  ← animated progress bar
└────────────────────────────┘
```

The display also shows a signal strength indicator (4 bars) in the top-right corner when connected.

---

### 9.7 Buzzer Tones

The sketch uses ESP32's `ledcAttach` / `ledcWriteTone` API (v3.x) to generate PWM tones.

**Presence Update tone** (pleasant ascending chord):
```
C5 (523 Hz) → E5 (659 Hz) → G5 (784 Hz) → E5 (659 Hz)
Each note: 150ms on, 50ms gap
```

**Login Visit tone** (urgent, attention-grabbing):
```
A5 (880 Hz) → B5 (988 Hz) → C6 (1047 Hz) → B5 (988 Hz) → A5 (880 Hz)
Each note: 100ms on, 30ms gap
```

**Startup tone:**
```
1000 Hz (100ms) → 1500 Hz (100ms)
```

```cpp
void playBuzzer(int frequency, int duration) {
  ledcWriteTone(BUZZER_PIN, frequency);
  delay(duration);
  ledcWrite(BUZZER_PIN, 0);  // silence
}
```

> If using an older ESP32 Arduino core (< v3.0), replace `ledcAttach` with `ledcSetup` + `ledcAttachPin`.

---

### 9.8 LED Blink Patterns

| Event | Pattern |
|---|---|
| Presence Update | 5× slow blink (200ms on / 200ms off) |
| Login Visit | 7× rapid blink (100ms on / 100ms off) |

Uses `LED_PIN 2` — the built-in blue LED on most ESP32-DevKitC boards.

---

### 9.9 Server Failover

The sketch tries the primary server first. If it disconnects and fails to reconnect after several attempts, it switches to the fallback:

```cpp
void switchServer() {
  usePrimaryServer = !usePrimaryServer;
  current_host = usePrimaryServer ? ws_host_primary : ws_host_fallback;
  current_port = usePrimaryServer ? ws_port_primary : ws_port_fallback;

  webSocket.disconnect();
  webSocket.beginSSL(current_host, current_port, ws_path);
}
```

The heartbeat watchdog triggers this — if no `PING` is received from the server for 45 seconds, `switchServer()` is called.

---

### 9.10 Serial Monitor Output

After uploading, open **Tools → Serial Monitor** at **115200 baud** to see connection status:

```
╔════════════════════════════════════════╗
║  ESP32 Online Notification Client     ║
║  with OLED Display & Buzzer            ║
╚════════════════════════════════════════╝

✓ OLED Display initialized
✓ WiFi connected: 192.168.1.105
✓ WebSocket client initialized
  Connecting to: wss://moo.qzz.io:443/esp32

[WS] Connected to server
→ Sent: ESP32_HELLO
← Received: WELCOME

← PRESENCE_UPDATE: Sayem is online on Galaxy S23
← PING received, sending PONG
```

---

## 10. Running Locally

### 10.1 Prerequisites

- **Node.js** 18 or higher — [nodejs.org](https://nodejs.org)
- **npm** (comes with Node.js)
- Both Firebase service account JSON files in the project root
- A `.env` file populated with all required secrets

### 10.2 Install Dependencies

```bash
cd chat-main
npm install
```

This installs the four production dependencies:

| Package | Version | Purpose |
|---|---|---|
| `dotenv` | ^17.x | Load `.env` file into `process.env` |
| `firebase-admin` | ^13.x | Realtime DB watcher + FCM sender |
| `node-notifier` | ^10.x | Desktop OS popups (desktop-notifier.js only) |
| `ws` | ^8.x | WebSocket server |

### 10.3 Start the Server

```bash
npm start
# or
node server.js
```

Expected output:
```
[SYSTEM] Server & WebSocket running on port 3000
[INFO]   Testing Discord webhook...
[OK]     Discord notification sent
[INFO]   Testing FCM notification...
[OK]     FCM notification sent: projects/momofy-32be8/messages/...
[INFO]   Testing Pushover notification...
[OK]     Pushover notification sent
[INFO]   Testing Telegram notification...
[OK]     Telegram notification sent
[SYSTEM] Watching Firebase presence...
```

Then open `http://localhost:3000` in your browser.

### 10.4 Open Without the Server (Standalone Mode)

`index.html` can be opened directly as a local file:

```
File → Open File → index.html
```

or drag it into a browser tab. The chat will fully work because Firebase Client SDK runs entirely in the browser. The only things that won't work:

- The `/api/login-visit` POST (fetch will fail silently — `.catch(() => {})`)
- Loading `stickers.json` and `emojis.json` (relative paths won't resolve over `file://`)

> This is why the `fetch('/api/login-visit', ...).catch(() => {})` has an empty catch — it's intentionally silent for standalone use.

For full feature parity including stickers and emoji, run the server.

### 10.5 Run Desktop Notifier (Optional)

In a separate terminal:

```bash
node desktop-notifier.js
```

This watches Firebase `presence/` independently and fires native OS desktop popups. Keep it running in the background. It does not need `server.js` to be running.

### 10.6 Test ESP32 WebSocket (Without Hardware)

With `server.js` running, in a separate terminal:

```bash
node test-esp32-connection.js
```

This simulates a physical ESP32 connecting to `ws://localhost:3000/esp32`. You'll see it receive `WELCOME`, then any `PRESENCE_UPDATE` or `LOGIN_VISIT` frames when you trigger them (e.g. open the app in another tab).

### 10.7 Logger Color Reference

The server uses a colored logger for easy reading:

| Color | Level | Meaning |
|---|---|---|
| Cyan `[INFO]` | Info | General status messages |
| Green `[OK]` | Success | Notification sent, operation succeeded |
| Yellow `[WARN]` | Warning | Non-fatal issues (e.g. FCM not available) |
| Red `[ERROR]` | Error | Failed operations |
| Magenta `[SYSTEM]` | System | Server lifecycle events |
| Blue `[EVENT]` | Event | Firebase presence changes, WS connections |

---

## 11. Deployment on Render

Render is a cloud platform that runs Node.js apps for free (with limitations). The server is designed to deploy here with zero config changes.

### 11.1 First-Time Setup

**Step 1: Push to GitHub**
```bash
git init
git add server.js index.html package.json package-lock.json \
        stickers.json emojis.json momo.png desktop-notifier.js \
        test-esp32-connection.js server_help.js
git commit -m "initial commit"
git remote add origin https://github.com/YOUR_USERNAME/chat-main.git
git push -u origin main
```

> Do **not** commit `.env`, `obsidian-firebase.json`, `android-firebase.json`, or any `*-adminsdk-*.json` files. Add them to `.gitignore`.

**.gitignore:**
```
.env
obsidian-firebase.json
android-firebase.json
*-adminsdk-*.json
node_modules/
```

**Step 2: Create a Render Web Service**
1. Go to [render.com](https://render.com) → **New → Web Service**
2. Connect your GitHub repo
3. Set:
   - **Name:** `chat-main` (or anything)
   - **Runtime:** Node
   - **Build Command:** `npm install`
   - **Start Command:** `node server.js`
   - **Instance Type:** Free

**Step 3: Add Environment Variables**
In the Render dashboard → your service → **Environment** tab, add each variable:

| Key | Value |
|---|---|
| `FIREBASE_SERVICE_ACCOUNT` | Minified JSON of `obsidian-firebase.json` |
| `ANDROID_FIREBASE_SERVICE_ACCOUNT` | Minified JSON of `android-firebase.json` |
| `DISCORD_WEBHOOK` | Your Discord webhook URL |
| `TELEGRAM_BOT_TOKEN` | Your Telegram bot token |
| `TELEGRAM_CHAT_ID` | Your Telegram chat ID |
| `PUSHOVER_APP_TOKEN` | Your Pushover app token |
| `PUSHOVER_USER_KEY` | Your Pushover user key |

Do not set `PORT` — Render injects it automatically.

**Step 4: Deploy**
Click **Deploy**. Render will run `npm install` then `node server.js`. Watch the logs — you should see the same startup output as local.

### 11.2 Free Tier Behavior

Render's free tier **spins down** the service after 15 minutes of inactivity (no HTTP requests). The next request takes ~30 seconds to cold-start.

**Impact on this project:**
- The Firebase `presence/` watcher **stops** while the service is spun down
- A user coming online while the server is sleeping will **not** trigger notifications until the server wakes up
- The server wakes up when someone loads `index.html` (that's an HTTP request to `/`)

**Solution options:**
1. **Render paid tier** ($7/month) — always-on, no spin-down
2. **UptimeRobot** — free service that pings your Render URL every 5 minutes to keep it awake
   - Sign up at [uptimerobot.com](https://uptimerobot.com)
   - Add HTTP monitor → URL: `https://your-app.onrender.com`
   - Interval: every 5 minutes

### 11.3 Updating the Deployment

Any push to the connected GitHub branch triggers an automatic redeploy:

```bash
git add .
git commit -m "your change"
git push
```

Render will rebuild and restart automatically. Downtime is typically under 30 seconds.

### 11.4 Custom Domain

1. Render dashboard → your service → **Settings → Custom Domains**
2. Add your domain and follow the DNS instructions
3. Render provisions a free TLS certificate automatically via Let's Encrypt
4. Update the ESP32 sketch's `ws_host_fallback` to your custom domain

---

## 12. Testing

### 12.1 Test Notification Channels Individually

**Discord** — trigger a login-visit manually:
```bash
curl -X POST http://localhost:3000/api/login-visit \
  -H "Content-Type: application/json" \
  -d '{"device":"Test Device"}'
```

**Telegram / Pushover / FCM** — same endpoint above fires all channels simultaneously.

**WebSocket (Android path):**
```js
// In browser console or Node.js
const ws = new WebSocket('ws://localhost:3000');
ws.onmessage = (e) => console.log(JSON.parse(e.data));
// Then go online in another tab — you'll see PRESENCE_UPDATE
```

**WebSocket (ESP32 path):**
```bash
node test-esp32-connection.js
```

### 12.2 Test Firebase Presence Manually

In the Firebase Realtime Database console, manually set:
```
presence/Sayem/state = "online"
```
The server should immediately log `[EVENT] ONLINE | Sayem` and fire all notification channels.

Set it back to `"offline"` — server logs it but fires no external notifications.

### 12.3 Verify FCM Topic Subscription

To check if your Android device is subscribed to `all_users`, add a log in your `FirebaseMessagingService`:

```java
FirebaseMessaging.getInstance().subscribeToTopic("all_users")
    .addOnSuccessListener(aVoid -> Log.d("FCM", "Subscribed"))
    .addOnFailureListener(e -> Log.e("FCM", "Failed: " + e.getMessage()));
```

### 12.4 Test ESP32 Without Hardware

Run `node test-esp32-connection.js` while the server is running. Then in a separate browser tab, open `http://localhost:3000` and log in — the test script will receive a `PRESENCE_UPDATE` frame within 1–2 seconds of login.

---

## 13. Security

### 13.1 What Is and Isn't Protected

| Surface | Protection | Notes |
|---|---|---|
| Chat messages | Firebase Auth required (`auth != null`) | Anonymous auth satisfies this |
| PIN verification | Server-side comparison via Firebase read | PINs stored in Firebase, not hardcoded |
| Service account keys | `.env` + env vars on Render | Never committed to git |
| Discord webhook | `.env` only | Hardcoded fallback in `server.js` — change before sharing code |
| Telegram / Pushover tokens | `.env` only | Same — hardcoded fallbacks exist |
| ImgBB API key | Hardcoded in `index.html` | Client-side — visible in browser DevTools |
| Firebase client config | Hardcoded in `index.html` | Normal for Firebase web apps — rules are the real protection |

---

### 13.2 Credentials in the Repository

**The `.env` file contains live private keys.** It is not in `.gitignore` by default in this project — you must add it before pushing to any public repository.

Immediately create a `.gitignore`:
```
.env
obsidian-firebase.json
android-firebase.json
*-adminsdk-*.json
node_modules/
```

The hardcoded fallback credentials in `server.js` (Discord webhook, Telegram token, Pushover keys) should be rotated if the repository is ever made public:
- Discord: delete the webhook and create a new one
- Telegram: use `@BotFather` → `/revoke` to regenerate the bot token
- Pushover: regenerate the API token from the app dashboard

---

### 13.3 Firebase Security Rules

The current rules allow any authenticated user to read and write the entire database:
```json
{
  "rules": {
    ".read": "auth != null",
    ".write": "auth != null"
  }
}
```

This is acceptable for a private two-person app, but be aware: anyone who obtains the Firebase client config (visible in `index.html`) can sign in anonymously and read all messages. The client config is not secret — Firebase's security model relies entirely on the rules, not on keeping the config private.

For stricter rules, you could lock down by username stored in `/userMappings/`:
```json
{
  "rules": {
    "messages": {
      ".read": "auth != null",
      ".write": "auth != null"
    },
    "presence": {
      "$user": {
        ".read": "auth != null",
        ".write": "auth != null"
      }
    }
  }
}
```

---

### 13.4 PIN Security

- PINs are stored as **plaintext** in Firebase (`settings/Sayem/pin`)
- They are transmitted over HTTPS (Firebase uses TLS) but are not hashed
- The PIN is 5 digits — 100,000 possible combinations
- There is no brute-force lockout mechanism

For a private two-person app this is acceptable. If you want to harden it, hash the PIN client-side with `crypto.subtle` before storing and comparing.

---

### 13.5 Panic Logout Limitations

The panic logout fires on `blur` and `visibilitychange`. However:
- It does **not** fire if the device is locked without closing the tab (screen lock keeps the tab technically "visible" on some OS/browser combinations)
- On iOS Safari, `pagehide` is more reliable than `blur`
- `DEV_MODE = false` must remain `false` in production — setting it to `true` disables all panic logout

The `isSelectingFile` guard prevents false-positive logouts during file picker usage, but it is a simple boolean — a rapid focus-blur cycle could theoretically bypass it.

---

### 13.6 WebSocket Authentication

The WebSocket endpoints (`/` and `/esp32`) have **no authentication**. Anyone who knows your server URL can connect to `/esp32` and receive presence notifications. For a private deployment this is low risk, but worth knowing.

To add basic auth, check a query parameter on connect:
```js
wss.on('connection', (ws, req) => {
  const url = new URL(req.url, 'http://localhost');
  const token = url.searchParams.get('token');
  if (token !== process.env.WS_SECRET) {
    ws.close(4001, 'Unauthorized');
    return;
  }
  // proceed
});
```

---

## 14. Troubleshooting / FAQ

---

**Q: Android is receiving FCM notifications even though I'm running `index.html` as a local file, not through the server.**

A: Expected behavior. The browser loads the Firebase Client SDK directly from Google's CDN and writes to Firebase regardless of how `index.html` was opened. The always-on server hosted on Render watches Firebase and fires FCM. The local file and the remote server are completely independent. See [Architecture — Notification Flow](#2-architecture).

---

**Q: `auth/operation-not-allowed` error on the PIN screen.**

A: Anonymous Authentication is not enabled in your Firebase project. Go to Firebase Console → **Authentication → Sign-in providers → Anonymous → Enable**.

---

**Q: Stickers and emojis don't load when opening `index.html` directly (file://).**

A: `fetch('stickers.json')` uses a relative path that only resolves when served over HTTP. Run `node server.js` and open `http://localhost:3000` instead.

---

**Q: `FCM Firebase Admin initialization failed` in server logs.**

A: The `ANDROID_FIREBASE_SERVICE_ACCOUNT` env var is missing or malformed, and `android-firebase.json` is not present in the project root. The server continues running but FCM pushes are skipped. Check that the JSON is valid and properly minified (no unescaped newlines in the private key — they must be `\n`).

---

**Q: Discord notifications work but Telegram doesn't.**

A: Check your `TELEGRAM_CHAT_ID`. It must be the numeric chat ID, not the username. Get it by visiting:
```
https://api.telegram.org/bot{YOUR_TOKEN}/getUpdates
```
after sending a message to your bot. The `id` field inside `"chat"` is what you need.

---

**Q: The server on Render stops sending notifications after a while.**

A: Render free tier spins down after 15 minutes of no HTTP traffic. Set up UptimeRobot (free) to ping your Render URL every 5 minutes. See [Deployment — Free Tier Behavior](#112-free-tier-behavior).

---

**Q: ESP32 shows "Disconnected" and won't reconnect.**

A: Check in order:
1. WiFi credentials in the sketch are correct
2. `ws_host_primary` / `ws_host_fallback` are correct (no `https://` prefix, no trailing slash)
3. The server is actually running (check Render dashboard logs)
4. The Cloudflare tunnel is active (if using `moo.qzz.io`)
5. Open Arduino Serial Monitor at 115200 baud for detailed error output

---

**Q: ESP32 OLED shows nothing / garbled text.**

A: Two likely causes:
- Wrong I2C address. Most 0.96" modules are `0x3C`, some are `0x3D`. Upload an I2C scanner sketch to find the actual address, then update `SCREEN_ADDRESS` in the sketch.
- SDA/SCL pins swapped. Double-check GPIO 21 → SDA and GPIO 22 → SCL.

---

**Q: Buzzer makes no sound.**

A: Verify you have a **passive** buzzer (not active). An active buzzer only responds to on/off signals, not PWM frequency — it will be silent when the sketch tries to play a specific frequency. Passive buzzers are usually unmarked; active buzzers often have a small circuit board visible on the bottom.

---

**Q: `[ERROR] Firebase error: PERMISSION_DENIED`**

A: The service account credentials don't match the database URL. Confirm:
1. `obsidian-firebase.json` belongs to `obsidian-8234e`
2. The `databaseURL` in `server.js` is `https://obsidian-8234e-default-rtdb.firebaseio.com`
3. The service account has `Editor` or `Firebase Admin` role in Google Cloud IAM

---

**Q: Messages appear doubled / loading twice.**

A: This happens if `onValue(messagesRef)` is registered more than once (e.g. `initMainApp()` called twice). Ensure the auth flow only calls `initMainApp()` once. The panic logout (`window.location.reload()`) resets all listeners on reload.

---

**Q: The typing indicator never disappears.**

A: The typing state is cleared after 2 seconds of no input. If a user's browser crashes or closes unexpectedly without running cleanup, the `typing/{user}` node stays `true` in Firebase indefinitely. Add a Firebase `onDisconnect` cleanup to fix this:
```js
onDisconnect(ref(db, 'typing/' + currentUser)).set(false);
```

---

**Q: Images uploaded to ImgBB aren't showing up.**

A: ImgBB free accounts have a rate limit. Check:
1. The ImgBB API key (`c3b5dc7293956c433f6b2cd1cdd121f6`) is still valid — log into imgbb.com to verify
2. The file size is under ImgBB's limit (32MB for API uploads)
3. The network request in DevTools → Network tab to see the actual error response

---

**Q: How do I add a third user?**

A: The app is hardcoded for two users — `Sayem` and `Shajeda`. The PIN comparison, presence display, read receipts, and typing indicator all assume exactly two parties. Adding a third user would require significant refactoring of the auth logic, presence watcher, and UI layout.

---

## License

Private project. Not licensed for redistribution.

---

<div align="center">
  <sub>Built with Firebase · Node.js · WebSocket · ESP32</sub>
</div>
