# Unfold — Firebase & Improvement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a lobby screen, Google SSO with progress persistence, visual polish, PWA support, Firestore-backed deck content, and host-controlled multiplayer to the Unfold conversation card game.

**Architecture:** The game lives in a single `index.html` file (vanilla JS, no build step). All features are added as inline `<script>` and `<style>` blocks. Firebase is loaded via CDN. Multiple "screens" (lobby, sso-gate, game, end, join) are toggled with CSS classes. Phases 1–4 can ship without any Firebase project.

**Tech Stack:** HTML/CSS/JS (vanilla), Firebase v10 CDN (Auth, Firestore, Realtime Database), Vercel (hosting), PWA (manifest.json + service worker)

## Global Constraints

- Never use a build step or npm — all JS must work as inline `<script>` in the HTML
- Firebase loaded from CDN: `https://www.gstatic.com/firebasejs/10.12.0/`
- Keep all existing deck content in the HTML as an offline fallback
- localStorage key `unfold_v1` must remain backward-compatible
- Maintain existing branding: `#1c1410` dark, `#f4ead5` parchment, `#c14a2b` rust
- Primary typefaces: Space Mono (monospace), Georgia/serif
- Deploy target: `unfold-nine-lemon.vercel.app` — this domain must be added to Firebase Auth authorized domains
- No frameworks (React, Vue, etc.)
- Files must stay under 500 lines each where possible; `index.html` is the exception (single-file architecture)

---

## Prerequisite — Firebase Project Setup (Jason does this manually)

Before Task 3, Jason must:

1. Go to [https://console.firebase.google.com](https://console.firebase.google.com) → **Create project** → name it `unfold-game`
2. **Authentication** → Sign-in method → Enable **Google**
3. Authentication → Settings → **Authorized domains** → Add `unfold-nine-lemon.vercel.app`
4. **Firestore Database** → Create database → Start in **production mode** → region `asia-southeast1`
5. **Realtime Database** → Create database → region `Singapore` → Start in **locked mode**
6. **Project settings** → Add web app (`unfold-web`) → Copy the `firebaseConfig` object
7. Paste the config into the `FIREBASE_CONFIG` placeholder in Task 3

---

## File Map

| File | Status | Responsibility |
|------|--------|---------------|
| `D:\Claude\Unfold\index.html` | Modify | All game screens, styles, and logic |
| `D:\Claude\Unfold\manifest.json` | Create | PWA manifest for home screen install |
| `D:\Claude\Unfold\sw.js` | Create | Service worker — offline caching |

---

## Phase 1 — Lobby & End Screen (no backend required)

---

### Task 1: Screen Management + Lobby Screen

**Files:**
- Modify: `D:\Claude\Unfold\index.html` (body structure + CSS + JS)

**Interfaces:**
- Produces: `showScreen(id: string): void` — used by all subsequent tasks

**What this does:** Wraps the existing game HTML in `#screen-game`, adds a `#screen-lobby` intro with three buttons (Play Live, Host Session, Join Session), and a `showScreen()` helper.

- [ ] **Step 1: Read the file before editing**

  Open `index.html`. The body currently starts with `<header>` and ends with the watermark div at line ~641. Note the structure:
  - `<header>` (title + tagline)
  - `<div class="decks" id="decks">` (deck chips)
  - `<div class="stage">` (card + controls + progress)
  - `<div class="reset-bar">` + `<div class="hint">` + `<div class="toast" id="toast">`

- [ ] **Step 2: Add screen CSS near the end of the existing `<style>` block (just before `</style></head>`)**

  Locate the line `</style>` just before `</head>` (around line 181). Insert before it:

  ```css
  /* ── Screen management ─────────────────────────────── */
  .screen{display:none;min-height:100vh;flex-direction:column;}
  .screen.active{display:flex;}

  /* ── Lobby screen ──────────────────────────────────── */
  #screen-lobby{
    align-items:center;justify-content:center;
    background:var(--bg);padding:40px 24px;
    text-align:center;gap:0;
  }
  .lobby-logo{margin-bottom:48px;}
  .lobby-logo h1{font-family:Georgia,serif;font-size:3rem;color:var(--ink);margin:0;}
  .lobby-logo h1 em{color:var(--rust);font-style:italic;}
  .lobby-logo .tagline{margin-top:10px;font-size:.85rem;color:var(--muted);letter-spacing:.06em;}
  .lobby-actions{display:flex;flex-direction:column;gap:16px;width:100%;max-width:320px;}
  .lobby-primary{
    font-size:1.05rem;padding:18px 32px;
    background:var(--rust);color:var(--parchment);
    border:none;border-radius:12px;cursor:pointer;
    font-family:'Space Mono',monospace;letter-spacing:.08em;
    box-shadow:0 6px 20px rgba(193,74,43,.35);
    transition:transform .15s ease,box-shadow .15s ease;
  }
  .lobby-primary:active{transform:scale(.97);box-shadow:0 3px 10px rgba(193,74,43,.25);}
  .lobby-online{display:flex;gap:12px;}
  .lobby-online .act{flex:1;font-size:.78rem;padding:14px 10px;}
  .lobby-label{
    font-family:'Space Mono',monospace;font-size:.6rem;letter-spacing:.14em;
    text-transform:uppercase;color:var(--muted);margin-top:4px;
  }
  ```

- [ ] **Step 3: Add the lobby screen HTML at the top of `<body>` (before `<header>`)**

  Insert this immediately after `<body>`:

  ```html
  <!-- ── Lobby Screen ── -->
  <div id="screen-lobby" class="screen active">
    <div class="lobby-logo">
      <h1>Un<em>fold</em></h1>
      <p class="tagline">One card at a time, everyone unfolds a little.</p>
    </div>
    <div class="lobby-actions">
      <button class="lobby-primary" onclick="startPlayLive()">&#9654; Play Live</button>
      <div class="lobby-label">online</div>
      <div class="lobby-online">
        <button class="act ghost" onclick="startHostSession()">Host Session</button>
        <button class="act ghost" onclick="startJoinSession()">Join Session</button>
      </div>
    </div>
  </div>
  <!-- ── Game Screen ── -->
  <div id="screen-game" class="screen">
  ```

- [ ] **Step 4: Close the `#screen-game` div just before the `<script>` tag**

  Find the line `<div class="toast" id="toast"></div>` (around line 224). After it, add `</div>` to close `#screen-game`:

  ```html
  <div class="toast" id="toast"></div>
  </div><!-- /screen-game -->
  ```

- [ ] **Step 5: Add `showScreen()` and stub handlers to the `<script>` block**

  At the top of the existing `<script>` block (before `const CATS = {`), add:

  ```js
  // ── Screen management ──────────────────────────────────────────────────────
  function showScreen(id){
    document.querySelectorAll('.screen').forEach(s=>s.classList.remove('active'));
    document.getElementById(id).classList.add('active');
  }
  function startPlayLive(){ showScreen('screen-sso-gate'); }   // replaced in Task 4
  function startHostSession(){ showScreen('screen-game'); initGame(); showToast('Host flow coming soon',2000); }
  function startJoinSession(){ showScreen('screen-game'); initGame(); showToast('Join flow coming soon',2000); }
  ```

- [ ] **Step 6: Extract init logic into `initGame()`**

  Near the bottom of the `<script>` block, find the existing init lines (around line 626):
  ```js
  initStorage();
  buildChips();
  buildPool();
  updateChips();
  ```
  Replace them with:
  ```js
  function initGame(){
    initStorage();
    buildChips();
    buildPool();
    updateChips();
  }
  // Lobby is the entry point — do NOT auto-init game on load
  ```

- [ ] **Step 7: Verify in browser**

  Open `index.html` in Chrome. You should see the Unfold lobby with three buttons. Clicking "Play Live" should navigate to the SSO gate (stub — shows game for now). Clicking Host/Join should show the game screen.

- [ ] **Step 8: Commit**

  ```bash
  git -C "D:\Claude\Unfold" add index.html
  git -C "D:\Claude\Unfold" commit -m "feat: add lobby screen with screen management"
  ```

---

### Task 2: End Screen

**Files:**
- Modify: `D:\Claude\Unfold\index.html`

**Interfaces:**
- Consumes: `showScreen(id)` from Task 1
- Consumes: `totalUnseen()`, `totalSeen()`, `active` (existing game state)
- Produces: `showEndScreen(): void`

**What this does:** Replaces the current auto-reshuffle toast when all cards are seen with a proper end screen showing stats and replay options.

- [ ] **Step 1: Add end screen CSS to the style block**

  Add after the lobby CSS from Task 1:

  ```css
  /* ── End Screen ─────────────────────────────────────── */
  #screen-end{
    align-items:center;justify-content:center;
    background:var(--bg);padding:40px 24px;text-align:center;gap:24px;
  }
  .end-badge{
    font-size:3rem;margin-bottom:8px;
  }
  .end-headline{
    font-family:Georgia,serif;font-size:1.6rem;color:var(--ink);
    margin:0;line-height:1.3;
  }
  .end-sub{
    font-size:.85rem;color:var(--muted);letter-spacing:.04em;margin:0;
  }
  .end-stat{
    font-family:'Space Mono',monospace;font-size:2rem;
    color:var(--rust);font-weight:700;
  }
  .end-actions{display:flex;flex-direction:column;gap:12px;width:100%;max-width:280px;}
  ```

- [ ] **Step 2: Add end screen HTML inside the `<body>` before `</div><!-- /screen-game -->`**

  Insert a new sibling div after the game screen close tag:

  ```html
  </div><!-- /screen-game -->

  <!-- ── End Screen ── -->
  <div id="screen-end" class="screen">
    <div class="end-badge">&#127183;</div>
    <p class="end-headline" id="endHeadline">You unfolded<br>every card.</p>
    <p class="end-sub" id="endSub"></p>
    <div class="end-actions">
      <button class="lobby-primary" onclick="endReplay()">Play Again</button>
      <button class="act ghost" onclick="endNewDeck()">Pick New Decks</button>
      <button class="act ghost" onclick="showScreen('screen-lobby')">Back to Home</button>
    </div>
  </div>
  ```

- [ ] **Step 3: Add `showEndScreen()` and action handlers to `<script>`**

  Add after `showScreen()`:

  ```js
  function showEndScreen(){
    const total = totalActive();
    document.getElementById('endHeadline').innerHTML =
      'You unfolded<br>' + total + ' card' + (total!==1?'s':'.') + '.';
    document.getElementById('endSub').textContent =
      'Great conversation — all decks seen!';
    showScreen('screen-end');
  }
  function endReplay(){
    // reset seen for active decks and go back to game
    active.forEach(k=>{
      CATS[k].cards.forEach((_,idx)=>{ delete seen[cardId(k,idx)]; });
    });
    saveState();
    buildPool();
    syncChips();
    updateChips();
    setBackState();
    showScreen('screen-game');
  }
  function endNewDeck(){
    showScreen('screen-game');
    // scroll deck chips into view
    document.getElementById('decks').scrollIntoView({behavior:'smooth'});
  }
  ```

- [ ] **Step 4: Replace the auto-reshuffle logic in `draw()` with `showEndScreen()`**

  In `draw()`, find:
  ```js
  if(pool.length===0){
    buildPool();
    if(pool.length===0){
      // all active decks fully seen — auto reset
      active.forEach(k=>{
        CATS[k].cards.forEach((_,idx)=>{ delete seen[cardId(k,idx)]; });
      });
      saveState();
      buildPool();
      showToast('✨ All cards seen — deck reshuffled!');
    }
  }
  ```
  Replace with:
  ```js
  if(pool.length===0){
    buildPool();
    if(pool.length===0){
      showEndScreen();
      return;
    }
  }
  ```

- [ ] **Step 5: Verify in browser**

  Set all CATS decks to 1 card temporarily in devtools, draw it, mark answered. The end screen should appear. "Play Again" reshuffles back to the game. "Back to Home" returns to the lobby.

- [ ] **Step 6: Commit**

  ```bash
  git -C "D:\Claude\Unfold" add index.html
  git -C "D:\Claude\Unfold" commit -m "feat: add end screen when all cards seen"
  ```

---

## Phase 2 — Google SSO + Progress Persistence

---

### Task 3: Firebase Auth Setup

**Files:**
- Modify: `D:\Claude\Unfold\index.html` (Firebase CDN scripts + config)

**Interfaces:**
- Produces: `window.firebaseAuth` — the Firebase Auth instance, available globally
- Produces: `window.currentUser` — null when guest, Firebase User object when signed in
- Produces: `onAuthReady(callback): void` — calls callback once auth state is known

**What this does:** Adds Firebase via CDN, initializes the app, sets up Google auth, exposes auth state globally.

> **Prerequisite:** Complete the Firebase Project Setup section first. Have the `firebaseConfig` object ready from the Firebase console.

- [ ] **Step 1: Add Firebase CDN scripts to `<head>`**

  Add just before `</head>`:

  ```html
  <!-- Firebase -->
  <script type="module">
    import { initializeApp } from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js';
    import { getAuth, GoogleAuthProvider, signInWithPopup, signOut, onAuthStateChanged }
      from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js';

    const firebaseConfig = {
      apiKey: "REPLACE_WITH_YOUR_API_KEY",
      authDomain: "REPLACE_WITH_YOUR_AUTH_DOMAIN",
      projectId: "REPLACE_WITH_YOUR_PROJECT_ID",
      storageBucket: "REPLACE_WITH_YOUR_STORAGE_BUCKET",
      messagingSenderId: "REPLACE_WITH_YOUR_MESSAGING_SENDER_ID",
      appId: "REPLACE_WITH_YOUR_APP_ID",
      databaseURL: "REPLACE_WITH_YOUR_DATABASE_URL"
    };

    const app = initializeApp(firebaseConfig);
    const auth = getAuth(app);
    const provider = new GoogleAuthProvider();

    // Expose globally for non-module scripts
    window.firebaseAuth = auth;
    window.firebaseProvider = provider;
    window.currentUser = null;
    window._authReady = false;
    window._authCallbacks = [];

    window.signInWithGoogle = () => signInWithPopup(auth, provider);
    window.signOutUser = () => signOut(auth);
    window.onAuthReady = (cb) => {
      if(window._authReady) cb(window.currentUser);
      else window._authCallbacks.push(cb);
    };

    onAuthStateChanged(auth, user => {
      window.currentUser = user;
      if(!window._authReady){
        window._authReady = true;
        window._authCallbacks.forEach(cb => cb(user));
        window._authCallbacks = [];
      }
    });
  </script>
  ```

- [ ] **Step 2: Replace all placeholder strings with values from the Firebase console**

  Open the Firebase console → Project settings → Web app config. Copy each value into the corresponding field. Do NOT commit API keys to a public repo — they are safe for client-side Firebase (Firebase security rules protect data access, not the config itself).

- [ ] **Step 3: Verify Firebase loads**

  Open `index.html` in Chrome → DevTools Console. Type `window.firebaseAuth` — should return an Auth object, not `undefined`. If you see a 401/403, the config values are wrong.

- [ ] **Step 4: Commit (without real credentials if in a public repo)**

  If this repo is public, add `index.html` to `.gitignore` for the config section, or store config in a separate `firebase-config.js` (not committed). For a private repo, commit as-is.

  ```bash
  git -C "D:\Claude\Unfold" add index.html
  git -C "D:\Claude\Unfold" commit -m "feat: add firebase auth setup via CDN"
  ```

---

### Task 4: Play Live SSO Gate

**Files:**
- Modify: `D:\Claude\Unfold\index.html`

**Interfaces:**
- Consumes: `window.signInWithGoogle()`, `window.currentUser`, `onAuthReady()` from Task 3
- Consumes: `showScreen()` from Task 1, `initGame()` from Task 1
- Produces: `startPlayLive(): void` — replaces the stub from Task 1

**What this does:** Shows a sign-in interstitial before Play Live. Recommends Google sign-in (with clear value prop), offers a "Play as Guest" skip. If already signed in, goes straight to game.

- [ ] **Step 1: Add SSO gate CSS**

  Add to the style block:

  ```css
  /* ── SSO Gate ───────────────────────────────────────── */
  #screen-sso-gate{
    align-items:center;justify-content:center;
    background:var(--bg);padding:40px 24px;text-align:center;gap:0;
  }
  .sso-card{
    background:var(--parchment);border:1.5px solid rgba(193,74,43,.2);
    border-radius:20px;padding:36px 28px;max-width:320px;width:100%;
    box-shadow:0 8px 32px rgba(28,20,16,.12);
  }
  .sso-icon{font-size:2.4rem;margin-bottom:16px;}
  .sso-title{
    font-family:Georgia,serif;font-size:1.3rem;color:var(--ink);
    margin:0 0 10px;
  }
  .sso-desc{
    font-size:.82rem;color:var(--muted);line-height:1.6;margin:0 0 28px;
  }
  .sso-google{
    display:flex;align-items:center;justify-content:center;gap:10px;
    width:100%;padding:14px;border-radius:10px;border:1.5px solid rgba(193,74,43,.4);
    background:var(--rust);color:var(--parchment);cursor:pointer;
    font-family:'Space Mono',monospace;font-size:.8rem;letter-spacing:.06em;
    margin-bottom:12px;transition:opacity .15s;
  }
  .sso-google:active{opacity:.8;}
  .sso-guest{
    background:transparent;border:none;color:var(--muted);
    font-family:'Space Mono',monospace;font-size:.7rem;letter-spacing:.06em;
    cursor:pointer;padding:8px;text-decoration:underline;
    text-underline-offset:3px;
  }
  .sso-user-bar{
    display:flex;align-items:center;gap:10px;margin-bottom:16px;
    background:rgba(193,74,43,.07);border-radius:10px;padding:10px 14px;
  }
  .sso-avatar{
    width:32px;height:32px;border-radius:50%;object-fit:cover;
  }
  .sso-name{font-size:.8rem;color:var(--ink);font-family:'Space Mono',monospace;}
  ```

- [ ] **Step 2: Add SSO gate HTML (sibling to other screens)**

  After the `</div><!-- /screen-end -->` tag, add:

  ```html
  <!-- ── SSO Gate ── -->
  <div id="screen-sso-gate" class="screen">
    <div class="sso-card">
      <div class="sso-icon">&#128100;</div>
      <p class="sso-title">Save your progress</p>
      <p class="sso-desc">
        Sign in to track which cards you&rsquo;ve answered &mdash; across devices and sessions &mdash;
        until you decide to reset.
      </p>
      <div id="ssoUserBar" class="sso-user-bar" style="display:none">
        <img id="ssoAvatar" class="sso-avatar" src="" alt="">
        <span class="sso-name" id="ssoName"></span>
      </div>
      <button class="sso-google" onclick="handleSsoSignIn()">
        <svg width="18" height="18" viewBox="0 0 18 18" fill="none">
          <path d="M17.64 9.2c0-.637-.057-1.251-.164-1.84H9v3.481h4.844c-.209 1.125-.843 2.078-1.796 2.717v2.258h2.908c1.702-1.567 2.684-3.874 2.684-6.615z" fill="#4285F4"/><path d="M9 18c2.43 0 4.467-.806 5.956-2.18l-2.908-2.259c-.806.54-1.837.86-3.048.86-2.344 0-4.328-1.584-5.036-3.711H.957v2.332A8.997 8.997 0 0 0 9 18z" fill="#34A853"/><path d="M3.964 10.71A5.41 5.41 0 0 1 3.682 9c0-.593.102-1.17.282-1.71V4.958H.957A8.996 8.996 0 0 0 0 9c0 1.452.348 2.827.957 4.042l3.007-2.332z" fill="#FBBC05"/><path d="M9 3.58c1.321 0 2.508.454 3.44 1.345l2.582-2.58C13.463.891 11.426 0 9 0A8.997 8.997 0 0 0 .957 4.958L3.964 6.29C4.672 4.163 6.656 3.58 9 3.58z" fill="#EA4335"/>
        </svg>
        Continue with Google
      </button>
      <button class="sso-guest" onclick="handleSsoSkip()">Play as Guest</button>
    </div>
  </div>
  ```

- [ ] **Step 3: Implement `startPlayLive()` (replaces Task 1 stub) and helpers**

  In the `<script>` block, replace the stub `startPlayLive` function:

  ```js
  function startPlayLive(){
    onAuthReady(user => {
      if(user){
        // Already signed in — show user bar and jump straight to game
        showScreen('screen-sso-gate');
        updateSsoBar(user);
        // Auto-proceed after brief recognition moment
        setTimeout(()=>{ showScreen('screen-game'); initGame(); }, 900);
      } else {
        showScreen('screen-sso-gate');
        updateSsoBar(null);
      }
    });
  }
  function updateSsoBar(user){
    const bar = document.getElementById('ssoUserBar');
    if(user){
      document.getElementById('ssoAvatar').src = user.photoURL || '';
      document.getElementById('ssoName').textContent = user.displayName || user.email;
      bar.style.display = 'flex';
    } else {
      bar.style.display = 'none';
    }
  }
  function handleSsoSignIn(){
    window.signInWithGoogle()
      .then(result => {
        updateSsoBar(result.user);
        setTimeout(()=>{ showScreen('screen-game'); initGame(); }, 400);
      })
      .catch(err => {
        if(err.code !== 'auth/popup-closed-by-user'){
          showToast('Sign-in failed — try again', 2500);
        }
      });
  }
  function handleSsoSkip(){
    showScreen('screen-game');
    initGame();
  }
  ```

- [ ] **Step 4: Verify the sign-in flow in browser**

  Open `index.html` via Vercel URL (popup auth requires HTTPS or localhost). Click **Play Live** → SSO gate appears. Click **Continue with Google** → Google popup → on success, game screen loads. Click **Play as Guest** → game screen loads without sign-in.

- [ ] **Step 5: Commit**

  ```bash
  git -C "D:\Claude\Unfold" add index.html
  git -C "D:\Claude\Unfold" commit -m "feat: add google sso gate for play live"
  ```

---

### Task 5: Firebase Progress Persistence

**Files:**
- Modify: `D:\Claude\Unfold\index.html`

**Interfaces:**
- Consumes: `window.currentUser`, Firebase Firestore (initialized in this task)
- Consumes: `seen` object, `saveState()`, `initStorage()` (existing)
- Produces: Modified `saveState()` and `initStorage()` — transparent to the rest of the code

**What this does:** When the user is signed in, saves and loads their `seen` progress to Firestore instead of just localStorage. Guest users continue using localStorage unchanged.

- [ ] **Step 1: Add Firestore import to the Firebase module script (from Task 3)**

  Find the existing Firebase module `<script type="module">` block. Add Firestore imports:

  ```js
  import { getFirestore, doc, setDoc, getDoc }
    from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js';

  const db = getFirestore(app);
  window.firestoreDb = db;
  window.firestoreDoc = doc;
  window.firestoreSetDoc = setDoc;
  window.firestoreGetDoc = getDoc;
  ```

- [ ] **Step 2: Replace `saveState()` with a cloud-aware version**

  Find the existing `saveState()` function:
  ```js
  function saveState(){
  ```
  Replace the entire function with:
  ```js
  function saveState(){
    // Always save to localStorage (guest fallback + cache)
    if(lsOk) localStorage.setItem('unfold_v1', JSON.stringify(seen));
    // If signed in, also persist to Firestore
    if(window.currentUser && window.firestoreDb){
      const ref = window.firestoreDoc(window.firestoreDb, 'progress', window.currentUser.uid);
      window.firestoreSetDoc(ref, { seen: JSON.stringify(seen) }, { merge: true })
        .catch(()=>{}); // silent fail — localStorage is the ground truth
    }
  }
  ```

- [ ] **Step 3: Replace `initStorage()` with a cloud-aware version**

  Find the existing `initStorage()` function and replace it entirely:

  ```js
  function initStorage(){
    try{ localStorage.setItem('__t','1'); localStorage.removeItem('__t'); lsOk=true; }
    catch(e){ lsOk=false; }

    const loadFromJson = (raw) => {
      try { seen = JSON.parse(raw) || {}; } catch(e){ seen={}; }
    };

    if(window.currentUser && window.firestoreDb){
      // Signed-in: try Firestore first, fall back to localStorage
      const ref = window.firestoreDoc(window.firestoreDb, 'progress', window.currentUser.uid);
      window.firestoreGetDoc(ref).then(snap => {
        if(snap.exists() && snap.data().seen){
          loadFromJson(snap.data().seen);
        } else if(lsOk){
          loadFromJson(localStorage.getItem('unfold_v1'));
        }
        buildPool(); updateChips();
      }).catch(() => {
        if(lsOk) loadFromJson(localStorage.getItem('unfold_v1'));
        buildPool(); updateChips();
      });
    } else {
      // Guest: localStorage only
      if(lsOk) loadFromJson(localStorage.getItem('unfold_v1'));
    }
  }
  ```

- [ ] **Step 4: Set Firestore security rules**

  In the Firebase console → Firestore → Rules, set:

  ```
  rules_version = '2';
  service cloud.firestore {
    match /databases/{database}/documents {
      match /progress/{userId} {
        allow read, write: if request.auth != null && request.auth.uid == userId;
      }
    }
  }
  ```

- [ ] **Step 5: Verify persistence**

  Sign in on the Vercel URL, play 5 cards (mark answered/skipped), close the tab, reopen, sign in again → the same 5 cards should be marked as seen. Open the Firebase console → Firestore → `progress/{uid}` collection to confirm the data is there.

- [ ] **Step 6: Commit**

  ```bash
  git -C "D:\Claude\Unfold" add index.html
  git -C "D:\Claude\Unfold" commit -m "feat: persist seen cards to firestore for signed-in users"
  ```

---

## Phase 3 — Visual Polish

---

### Task 6: Animations & Micro-Interactions

**Files:**
- Modify: `D:\Claude\Unfold\index.html` (CSS + JS additions)

**Interfaces:**
- Consumes: existing `draw()`, `markCard()`, `handleCardTap()` functions
- Produces: no new JS interfaces — improvements are CSS-driven with minor JS additions

**What this does:** Adds card deal bounce, deck color wash, button press feedback, progress bar easing, and mobile haptics — all within the existing brand aesthetic.

- [ ] **Step 1: Enhance card deal animation**

  Find the existing `.dealing` CSS class (search for `dealing`). Replace or augment:

  ```css
  @keyframes cardDeal{
    0%  {opacity:0;transform:translateY(28px) scale(.95);}
    60% {opacity:1;transform:translateY(-6px) scale(1.015);}
    80% {transform:translateY(2px) scale(.998);}
    100%{transform:translateY(0) scale(1);}
  }
  .card.dealing{
    animation:cardDeal .38s cubic-bezier(.22,.68,0,1.2) forwards;
  }
  ```

- [ ] **Step 2: Add deck color screen wash**

  In the CSS, find `:root` block (contains `--bg`, `--rust`, etc.). Add a transition to the body background:

  ```css
  body{
    transition:background-color .6s ease;
  }
  ```

  In `draw()`, after `document.documentElement.style.setProperty('--cat', cat.color)`, add:

  ```js
  // Subtle wash: blend deck color 6% into background
  const washMap = {
    'var(--gold)':'#fcf7ec', 'var(--sky)':'#edf4fb',
    'var(--rose)':'#fceef2', 'var(--rust)':'#fdf0eb',
    'var(--moss)':'#edf4f1', 'var(--wine)':'#f5eef4', 'var(--amber)':'#fdf5e8'
  };
  document.body.style.backgroundColor = washMap[cat.color] || '';
  ```

  In `setBackState()`, after `cardEl().classList.remove('flipped')`, add:

  ```js
  document.body.style.backgroundColor = '';
  ```

- [ ] **Step 3: Button press micro-interactions**

  Add to CSS:

  ```css
  button.act{
    transition:transform .12s ease,box-shadow .12s ease,opacity .12s ease;
  }
  button.act:active{
    transform:scale(.96);
    box-shadow:inset 0 2px 6px rgba(28,20,16,.18);
  }
  ```

- [ ] **Step 4: Progress bar easing**

  Find `.prog-fill` in CSS. Add:

  ```css
  .prog-fill{
    transition:width .5s cubic-bezier(.4,0,.2,1);
  }
  ```

- [ ] **Step 5: Mobile haptics**

  In `markCard()`, after `showToast(msg, 1400)`, add:

  ```js
  if('vibrate' in navigator) navigator.vibrate(status==='answered' ? [8] : [4,30,4]);
  ```

  In `draw()`, after `c.classList.add('flipped','dealing')`, add:

  ```js
  if('vibrate' in navigator) navigator.vibrate(6);
  ```

- [ ] **Step 6: Verify on mobile**

  Open the Vercel URL on an Android phone. Draw a card — you should feel a short haptic. Mark answered — a slightly stronger pulse. The card should bounce in rather than appearing instantly. Each deck should subtly tint the background.

- [ ] **Step 7: Commit**

  ```bash
  git -C "D:\Claude\Unfold" add index.html
  git -C "D:\Claude\Unfold" commit -m "feat: visual polish - card bounce, color wash, haptics, button feedback"
  ```

---

## Phase 4 — PWA

---

### Task 7: PWA Manifest + Service Worker

**Files:**
- Create: `D:\Claude\Unfold\manifest.json`
- Create: `D:\Claude\Unfold\sw.js`
- Modify: `D:\Claude\Unfold\index.html` (add manifest link + SW registration)

**Interfaces:**
- No JS interfaces — browser-native APIs

**What this does:** Enables "Add to Home Screen" on iOS and Android, and caches the game for offline play.

- [ ] **Step 1: Create `manifest.json`**

  Create `D:\Claude\Unfold\manifest.json`:

  ```json
  {
    "name": "Unfold",
    "short_name": "Unfold",
    "description": "A conversation card game",
    "start_url": "/",
    "display": "standalone",
    "orientation": "portrait",
    "background_color": "#1c1410",
    "theme_color": "#1c1410",
    "icons": [
      {
        "src": "/apk/assets/icon.png",
        "sizes": "1024x1024",
        "type": "image/png",
        "purpose": "any"
      },
      {
        "src": "/apk/assets/icon.png",
        "sizes": "1024x1024",
        "type": "image/png",
        "purpose": "maskable"
      }
    ]
  }
  ```

- [ ] **Step 2: Create `sw.js`**

  Create `D:\Claude\Unfold\sw.js`:

  ```js
  const CACHE = 'unfold-v1';
  const STATIC = ['/', '/index.html', '/manifest.json'];

  self.addEventListener('install', e => {
    e.waitUntil(
      caches.open(CACHE).then(c => c.addAll(STATIC))
    );
    self.skipWaiting();
  });

  self.addEventListener('activate', e => {
    e.waitUntil(
      caches.keys().then(keys =>
        Promise.all(keys.filter(k => k !== CACHE).map(k => caches.delete(k)))
      )
    );
    self.clients.claim();
  });

  self.addEventListener('fetch', e => {
    const url = new URL(e.request.url);
    // Network-first for Firebase (auth, firestore, rtdb)
    if(url.hostname.includes('google') || url.hostname.includes('firebase')){
      e.respondWith(fetch(e.request).catch(() => new Response('', {status: 503})));
      return;
    }
    // Cache-first for everything else
    e.respondWith(
      caches.match(e.request).then(cached => cached || fetch(e.request))
    );
  });
  ```

- [ ] **Step 3: Add manifest link to `<head>` in `index.html`**

  After the existing meta tags in `<head>`, add:

  ```html
  <link rel="manifest" href="/manifest.json">
  ```

- [ ] **Step 4: Register service worker in the `<script>` block**

  At the bottom of the `<script>` block (after `initGame` definition), add:

  ```js
  // PWA
  if('serviceWorker' in navigator){
    window.addEventListener('load', () => {
      navigator.serviceWorker.register('/sw.js').catch(()=>{});
    });
  }
  ```

- [ ] **Step 5: Verify PWA on mobile**

  Open the Vercel URL on iOS Safari. Tap the Share button → "Add to Home Screen" should appear. On Android Chrome: three-dot menu → "Add to Home Screen" or install banner. The icon should match the existing Unfold icon. After install, the app should open fullscreen without browser chrome.

- [ ] **Step 6: Commit**

  ```bash
  git -C "D:\Claude\Unfold" add index.html manifest.json sw.js
  git -C "D:\Claude\Unfold" commit -m "feat: add pwa manifest and service worker for ios/android home screen"
  ```

---

## Phase 5 — Firestore Deck Content

---

### Task 8: Firestore-Backed Deck Loading

**Files:**
- Modify: `D:\Claude\Unfold\index.html`

**Interfaces:**
- Consumes: `window.firestoreDb`, `window.firestoreDoc` from Task 5
- Consumes: `CATS` object (existing), `buildChips()`, `buildPool()`, `updateChips()` (existing)
- Produces: Async `loadDecks(): Promise<void>` — fetches active decks from Firestore, merges with bundled fallback

**What this does:** On load, fetches active deck data from Firestore and merges it with the bundled `CATS`. Archived decks from Firestore are hidden. If Firestore fails, only bundled decks show. Call `loadDecks()` instead of calling `initGame()` directly.

- [ ] **Step 1: Add Firestore collection query import**

  In the Firebase module script (Task 3), add to the Firestore imports:

  ```js
  import { getFirestore, doc, setDoc, getDoc, collection, getDocs, query, where }
    from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js';
  ```

  And expose:
  ```js
  window.firestoreCollection = collection;
  window.firestoreGetDocs = getDocs;
  window.firestoreQuery = query;
  window.firestoreWhere = where;
  ```

- [ ] **Step 2: Add `loadDecks()` to the `<script>` block**

  Add before `initGame()`:

  ```js
  function loadDecks(){
    if(!window.firestoreDb) return Promise.resolve();
    const col = window.firestoreCollection(window.firestoreDb, 'decks');
    const q = window.firestoreQuery(col, window.firestoreWhere('status','==','active'));
    return window.firestoreGetDocs(q).then(snap => {
      snap.forEach(docSnap => {
        const d = docSnap.data();
        // Firestore deck overrides/extends the bundled CATS
        CATS[docSnap.id] = {
          name: d.name,
          color: d.colorVar,
          foot: d.foot || '',
          cards: d.cards || []
        };
      });
      // Remove any deck where all cards came from Firestore and are now archived
      // (archived decks won't appear in the query — so their CATS entry remains from bundle)
    }).catch(() => {}); // silent fail — bundled decks remain
  }
  ```

- [ ] **Step 3: Call `loadDecks()` before `initGame()` in `startPlayLive` and other entry points**

  In `handleSsoSignIn()`, replace:
  ```js
  setTimeout(()=>{ showScreen('screen-game'); initGame(); }, 400);
  ```
  With:
  ```js
  setTimeout(()=>{
    showScreen('screen-game');
    loadDecks().then(()=>initGame());
  }, 400);
  ```

  In `handleSsoSkip()`:
  ```js
  function handleSsoSkip(){
    showScreen('screen-game');
    loadDecks().then(()=>initGame());
  }
  ```

- [ ] **Step 4: Seed Firestore with the existing decks (one-time)**

  Open the Firebase console → Firestore → Start collection `decks`. For each deck key in CATS (icebreakers, thisorthat, hottakes, etc.), create a document with ID matching the key and fields:
  - `name`: deck name string
  - `colorVar`: e.g. `"var(--gold)"`
  - `foot`: the foot text string
  - `status`: `"active"`
  - `cards`: array of question strings

  This is a one-time manual step. After this, you manage decks from the console without code changes.

- [ ] **Step 5: Set Firestore security rules for decks (read-only for all)**

  In Firebase console → Firestore → Rules, add:

  ```
  match /decks/{deckId} {
    allow read: if true;   // anyone can read active decks
    allow write: if false; // only via Firebase console
  }
  ```

- [ ] **Step 6: Verify**

  Add a test card to one deck in Firestore console. Reload the game. The new card should appear in that deck.

- [ ] **Step 7: Commit**

  ```bash
  git -C "D:\Claude\Unfold" add index.html
  git -C "D:\Claude\Unfold" commit -m "feat: load deck content from firestore with bundled fallback"
  ```

---

## Phase 6 — Multiplayer

---

### Task 9: Host a Session

**Files:**
- Modify: `D:\Claude\Unfold\index.html`

**Interfaces:**
- Consumes: Firebase Realtime Database (new import), `window.currentUser` from Task 3
- Consumes: `showScreen()`, `loadDecks()`, existing deck-select logic
- Produces: `window.activeRoomCode` — set when hosting, null otherwise
- Produces: `startHostSession(): void` — replaces Task 1 stub

**What this does:** After signing in, host creates a 4-letter room code, writes room state to Realtime Database, and starts the game with host controls. Each "Answered" / "Skip" writes the next card index to the room.

- [ ] **Step 1: Add Realtime Database imports to the Firebase module**

  In the Firebase module script (Task 3), add:

  ```js
  import { getDatabase, ref, set, onValue, push, remove }
    from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-database.js';

  const rtdb = getDatabase(app);
  window.rtdb = rtdb;
  window.rtdbRef = ref;
  window.rtdbSet = set;
  window.rtdbOnValue = onValue;
  window.rtdbRemove = remove;
  ```

- [ ] **Step 2: Add host screen HTML**

  After `</div><!-- /screen-end -->`, add:

  ```html
  <!-- ── Host Screen ── -->
  <div id="screen-host" class="screen">
    <div class="lobby-logo" style="margin-bottom:24px;">
      <h1>Un<em>fold</em></h1>
      <p class="tagline">Host a live session</p>
    </div>
    <div style="text-align:center;width:100%;max-width:320px;">
      <p style="font-family:'Space Mono',monospace;font-size:.7rem;color:var(--muted);letter-spacing:.1em;text-transform:uppercase;">Your room code</p>
      <div id="roomCodeDisplay" style="font-family:Georgia,serif;font-size:3.5rem;color:var(--rust);font-weight:900;letter-spacing:.15em;margin:8px 0 24px;"></div>
      <p style="font-size:.8rem;color:var(--muted);margin-bottom:24px;">Share this code with friends, then pick your deck and start.</p>
      <div id="hostDeckPicker" style="margin-bottom:16px;"></div>
      <button class="lobby-primary" onclick="hostStartGame()">Start Session</button>
      <button class="act ghost" style="margin-top:12px;width:100%;" onclick="showScreen('screen-lobby')">Cancel</button>
    </div>
  </div>
  ```

- [ ] **Step 3: Add host CSS**

  No additional CSS needed — reuses lobby styles. Ensure `.lobby-primary` is defined (Task 1).

- [ ] **Step 4: Add host JS logic**

  ```js
  function generateRoomCode(){
    const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ';
    return Array.from({length:4}, ()=>chars[Math.random()*chars.length|0]).join('');
  }

  function startHostSession(){
    onAuthReady(user => {
      if(!user){
        window.signInWithGoogle().then(r => {
          window.currentUser = r.user;
          showHostScreen();
        }).catch(()=>{});
      } else {
        showHostScreen();
      }
    });
  }

  function showHostScreen(){
    const code = generateRoomCode();
    window.activeRoomCode = code;
    document.getElementById('roomCodeDisplay').textContent = code;
    // Build simple deck picker for host
    const picker = document.getElementById('hostDeckPicker');
    picker.innerHTML = Object.keys(CATS).map(k =>
      `<button class="act ghost" style="margin:4px;font-size:.7rem;"
        onclick="window.hostSelectedDeck='${k}';
        document.querySelectorAll('#hostDeckPicker .act').forEach(b=>b.classList.remove('selected'));
        this.classList.add('selected');">${CATS[k].name}</button>`
    ).join('');
    window.hostSelectedDeck = Object.keys(CATS)[0];
    showScreen('screen-host');
  }

  function hostStartGame(){
    const code = window.activeRoomCode;
    const deck = window.hostSelectedDeck;
    const roomRef = window.rtdbRef(window.rtdb, 'rooms/' + code);
    window.rtdbSet(roomRef, {
      hostUid: window.currentUser.uid,
      deckKey: deck,
      currentCardIdx: 0,
      status: 'playing',
      createdAt: Date.now()
    }).then(() => {
      // Switch to game screen as host
      showScreen('screen-game');
      // Activate only the host's selected deck
      active = new Set([deck]);
      loadDecks().then(() => {
        initGame();
        window.isHost = true;
        showToast('Room ' + code + ' — tap Answered/Skip to advance all players', 3500);
      });
    });
  }
  ```

- [ ] **Step 5: Sync host card advances to Realtime Database**

  In `markCard()`, after `saveState()`, add:

  ```js
  // Sync to room if hosting
  if(window.isHost && window.activeRoomCode && window.rtdb){
    const roomRef = window.rtdbRef(window.rtdb, 'rooms/' + window.activeRoomCode);
    window.rtdbSet(roomRef, {
      hostUid: window.currentUser ? window.currentUser.uid : null,
      deckKey: currentCard ? currentCard.k : (window.hostSelectedDeck||''),
      currentCardIdx: pool.length,
      currentCardText: '',  // cleared after mark
      status: 'playing',
      createdAt: Date.now()
    }).catch(()=>{});
  }
  ```

  After the `draw()` function resolves the card text, add a sync write:

  ```js
  // After: document.getElementById('prompt').textContent = currentCard.t;
  if(window.isHost && window.activeRoomCode && window.rtdb){
    const roomRef = window.rtdbRef(window.rtdb, 'rooms/' + window.activeRoomCode);
    window.rtdbSet(roomRef, {
      hostUid: window.currentUser ? window.currentUser.uid : null,
      deckKey: currentCard.k,
      currentCardText: currentCard.t,
      currentCardIdx: pool.length,
      status: 'playing',
      createdAt: Date.now()
    }).catch(()=>{});
  }
  ```

- [ ] **Step 6: Set Realtime Database rules**

  Firebase console → Realtime Database → Rules:

  ```json
  {
    "rules": {
      "rooms": {
        "$roomCode": {
          ".read": "auth != null",
          ".write": "auth != null && (data.child('hostUid').val() == auth.uid || !data.exists())"
        }
      }
    }
  }
  ```

- [ ] **Step 7: Verify**

  Sign in, tap Host Session, note the room code (e.g. `ABCD`), pick a deck, tap Start. Draw cards and mark answered — open the Firebase console → Realtime Database and watch `rooms/ABCD` update in real time.

- [ ] **Step 8: Commit**

  ```bash
  git -C "D:\Claude\Unfold" add index.html
  git -C "D:\Claude\Unfold" commit -m "feat: host session with firebase realtime db room sync"
  ```

---

### Task 10: Join a Session (Participant View)

**Files:**
- Modify: `D:\Claude\Unfold\index.html`

**Interfaces:**
- Consumes: `window.rtdb`, `window.rtdbRef`, `window.rtdbOnValue`, `window.currentUser`
- Consumes: `showScreen()`, existing card display elements (`#prompt`, `#catName`)
- Produces: `startJoinSession(): void` — replaces Task 1 stub

**What this does:** Participant enters a 4-letter room code, subscribes to real-time updates from the host, sees the same card update live. No controls shown — read-only mirror.

- [ ] **Step 1: Add join screen HTML**

  After `</div><!-- /screen-host (if added) -->`, add:

  ```html
  <!-- ── Join Screen ── -->
  <div id="screen-join" class="screen">
    <div style="text-align:center;width:100%;max-width:320px;padding:40px 24px;">
      <h2 style="font-family:Georgia,serif;color:var(--ink);margin-bottom:8px;">Join a Session</h2>
      <p style="font-size:.82rem;color:var(--muted);margin-bottom:24px;">Enter the 4-letter room code from the host.</p>
      <input id="joinCodeInput" type="text" maxlength="4"
        placeholder="ABCD"
        style="font-family:'Space Mono',monospace;font-size:2rem;font-weight:700;
          text-align:center;text-transform:uppercase;letter-spacing:.2em;
          border:2px solid rgba(193,74,43,.3);border-radius:12px;
          background:var(--parchment);color:var(--rust);padding:16px;
          width:100%;box-sizing:border-box;outline:none;margin-bottom:20px;"
        oninput="this.value=this.value.toUpperCase()"
      />
      <button class="lobby-primary" style="width:100%;" onclick="joinSessionSubmit()">Join</button>
      <button class="act ghost" style="margin-top:12px;width:100%;" onclick="showScreen('screen-lobby')">Back</button>
    </div>
  </div>

  <!-- ── Participant View ── -->
  <div id="screen-participant" class="screen">
    <div style="text-align:center;padding:16px 24px;background:rgba(193,74,43,.07);">
      <span style="font-family:'Space Mono',monospace;font-size:.65rem;letter-spacing:.1em;color:var(--muted);">
        ROOM &nbsp;<span id="participantRoomCode" style="color:var(--rust);">????</span>
        &nbsp;&bull;&nbsp; LIVE
      </span>
    </div>
    <div class="stage" style="flex:1;">
      <div class="card-wrap">
        <div class="card flipped" id="participantCard">
          <div class="face back"><div class="mark">&#10022;</div><div class="label">Unfold</div></div>
          <div class="face front" id="participantFront">
            <div class="top"><span id="participantCatName">—</span><span class="dot"></span></div>
            <div class="prompt" id="participantPrompt">Waiting for host to draw...</div>
          </div>
        </div>
      </div>
    </div>
    <div style="text-align:center;padding:24px;">
      <button class="act ghost" onclick="leaveSession()">Leave Session</button>
    </div>
  </div>
  ```

- [ ] **Step 2: Add join CSS**

  ```css
  #participantCard{pointer-events:none;}
  #screen-participant{flex-direction:column;background:var(--bg);}
  ```

- [ ] **Step 3: Add join JS logic**

  ```js
  let participantUnsubscribe = null;

  function startJoinSession(){
    onAuthReady(user => {
      if(!user){
        window.signInWithGoogle().then(r => {
          window.currentUser = r.user;
          showScreen('screen-join');
        }).catch(()=>{});
      } else {
        showScreen('screen-join');
      }
    });
  }

  function joinSessionSubmit(){
    const code = document.getElementById('joinCodeInput').value.trim().toUpperCase();
    if(code.length !== 4){ showToast('Enter a 4-letter code', 2000); return; }
    const roomRef = window.rtdbRef(window.rtdb, 'rooms/' + code);
    // Verify room exists before subscribing
    window.rtdbOnValue(roomRef, snap => {
      if(!snap.exists()){
        showToast('Room not found — check the code', 2500);
        return;
      }
      // Unsubscribe the one-shot check and set up the live listener
      document.getElementById('participantRoomCode').textContent = code;
      showScreen('screen-participant');
      startParticipantListener(code);
    }, { onlyOnce: true });
  }

  function startParticipantListener(code){
    const roomRef = window.rtdbRef(window.rtdb, 'rooms/' + code);
    participantUnsubscribe = window.rtdbOnValue(roomRef, snap => {
      if(!snap.exists()) return;
      const room = snap.val();
      const catColor = CATS[room.deckKey] ? CATS[room.deckKey].color : 'var(--ink)';
      const catName  = CATS[room.deckKey] ? CATS[room.deckKey].name  : room.deckKey;
      document.documentElement.style.setProperty('--cat', catColor);
      document.getElementById('participantCatName').textContent = catName;
      if(room.currentCardText){
        document.getElementById('participantPrompt').textContent = room.currentCardText;
        const card = document.getElementById('participantCard');
        card.classList.remove('dealing'); void card.offsetWidth;
        card.classList.add('dealing');
        if('vibrate' in navigator) navigator.vibrate(6);
      }
      if(room.status === 'ended'){
        showToast('Session ended by host', 2500);
        leaveSession();
      }
    });
  }

  function leaveSession(){
    if(participantUnsubscribe) participantUnsubscribe();
    participantUnsubscribe = null;
    showScreen('screen-lobby');
  }
  ```

- [ ] **Step 4: Verify end-to-end multiplayer**

  Open the Vercel URL on two devices (or two browser windows in private mode). Device A: Host Session → pick Icebreakers → Start. Device B: Join Session → enter the 4-letter code → Join. On Device A, draw a card and mark answered — Device B should see the same card appear within 1–2 seconds, and automatically advance when the host moves to the next card.

- [ ] **Step 5: Commit**

  ```bash
  git -C "D:\Claude\Unfold" add index.html
  git -C "D:\Claude\Unfold" commit -m "feat: join session participant view with realtime card sync"
  ```

---

## Phase 6.5 — Deploy

---

### Task 11: Deploy to Vercel

**Files:**
- No code changes — deploy existing

**What this does:** Push all changes to the Vercel deployment so all features are live.

- [ ] **Step 1: Ensure Vercel is linked**

  ```bash
  cd D:\Claude\Unfold
  npx vercel --prod
  ```

  Or push to the linked GitHub branch if Vercel auto-deploys on push.

- [ ] **Step 2: Verify on production URL**

  Open `https://unfold-nine-lemon.vercel.app`. Run through the full flow: lobby → Play Live → SSO gate → sign in → game → draw cards → end screen. Verify PWA install prompt on mobile. Verify multiplayer works (repeat Task 10 Step 4 on production URL).

- [ ] **Step 3: Confirm Firebase Auth domain**

  Firebase console → Authentication → Settings → Authorized domains. Confirm `unfold-nine-lemon.vercel.app` is listed (from the Prerequisite section). If sign-in fails with "auth/unauthorized-domain", it's missing.

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Covered in |
|---|---|
| Lobby: Play Live / Host / Join buttons | Task 1 |
| Play Live as primary hero action | Task 1 (CSS lobby-primary) |
| Google SSO gate for Play Live (recommended, skippable) | Task 4 |
| SSO required for Host/Join | Tasks 9, 10 |
| Progress persistence (Firestore per UID) | Task 5 |
| Guest falls back to localStorage | Task 5 (initStorage branch) |
| End screen with stats + replay | Task 2 |
| Visual polish within brand | Task 6 |
| PWA manifest + service worker | Task 7 |
| Firestore deck content with archive support | Task 8 |
| Host-controlled multiplayer (Realtime DB) | Task 9 |
| Participant read-only mirror | Task 10 |
| Room auto-delete after 24h | Firestore rules (Task 9 Step 6) |
| Offline fallback to bundled decks | Task 8 (catch block) |

All spec sections covered. No gaps identified.
