# Unfold — Firebase & Improvement Design Spec

**Date:** 2026-06-29  
**Project:** Unfold Conversation Card Game  
**Stack:** Single HTML file (vanilla HTML/CSS/JS), Vercel, Android APK (Expo/EAS)  
**Scope:** Four priority improvements: lobby screen, visual polish, multiplayer sync, content management

---

## 1. Context & Constraints

The game lives in a single `index.html` file. All improvements must stay compatible with this architecture — Firebase is loaded via CDN `<script>` tags, no build step. If Firebase is unavailable offline, the game falls back to the bundled deck content and `localStorage` persistence.

Primary use case: **friends sitting together in the same room on one device.** All UX decisions optimize for this scenario first.

---

## 2. Lobby / Intro Screen

### Entry flow

A new intro screen appears before the existing deck-select screen.

```
┌─────────────────────────────────┐
│                                 │
│           🃏 Unfold             │
│      a conversation game        │
│                                 │
│  ┌───────────────────────────┐  │  ← Primary CTA (full width, accented)
│  │    ▶  Play Live           │  │
│  └───────────────────────────┘  │
│                                 │
│  ┌─────────────┐ ┌───────────┐  │  ← Secondary row (online only)
│  │ Host Session│ │Join Session│  │
│  └─────────────┘ └───────────┘  │
│                                 │
└─────────────────────────────────┘
```

**Play Live** is the hero action — visually dominant, styled in the game's rust accent colour. The two online options are smaller, equal-weight, and visually grouped below with an "online" label to signal the distinction.

### End screen

When a deck runs out (or all cards are seen), an end screen replaces the card view:

> *"You unfolded 24 cards together."*

Options: **Play Again** (same deck, reset) · **New Deck** (back to deck select) · **View Stats** (if signed in).

---

## 3. Google SSO Integration

### Play Live — recommended sign-in gate

When the user taps **Play Live**, a sign-in interstitial appears before the game:

```
┌─────────────────────────────────┐
│   Sign in to save your progress │
│                                 │
│  Track which questions you've   │
│  already answered — across      │
│  devices and sessions.          │
│                                 │
│  [G  Continue with Google]      │  ← Primary
│                                 │
│  [  Play as Guest  ]            │  ← Secondary, smaller/muted
└─────────────────────────────────┘
```

- **Signed in:** card history stored in Firebase per UID. Persists until the user taps **Reset** in settings. Works across any device where they sign in.
- **Guest:** falls back to `localStorage` (today's behaviour). Clears with browser data.

The guest option is always available — SSO is recommended, never mandatory for Play Live.

### Host / Join Session — required sign-in

Online sessions require Google sign-in. The sign-in step is part of the session creation / join flow, not a separate interstitial.

### Firebase Auth setup

- Anonymous auth: not used (we want persistent identity)
- Provider: `GoogleAuthProvider`
- Persistence: `browserLocalPersistence` (stays signed in across sessions)
- Scope: `email` + `profile` only — no calendar, drive, etc.

---

## 4. Visual Polish (on-brand)

Retain: dark parchment palette (`#1c1410`, `#f4ead5`, rust `#c14a2b`), Space Mono + Georgia serif pairing, card-as-surface motif, grain texture. Improvements within that brand:

| Element | Current | Improved |
|---|---|---|
| Card deal | Instant appear | Slide-up with slight bounce, shadow depth shift |
| Deck color | Chip only | Subtle full-screen wash when deck is active |
| Button press | Basic | Slight scale-down + shadow pop on tap |
| Progress bar | Fill | Eased fill, micro-pulse on milestone (50%, done) |
| Card flip | CSS transform | Retain, add `perspective` depth + timing curve tweak |
| Mobile haptics | None | `navigator.vibrate(10)` on deal/answered/skip |
| Sound | None | Optional muted-by-default card-flip sound (toggle in settings) |

No new typefaces, no new colours outside existing palette, no layout restructuring.

---

## 5. Multiplayer — Host-Controlled Room Sync

### Data model (Firebase Realtime Database)

```
rooms/
  {roomCode}/           # 4-letter code, e.g. "WXYZ"
    hostUid: string
    deckKey: string
    currentCardIdx: number
    status: "lobby" | "playing" | "ended"
    createdAt: timestamp
    participants/
      {uid}: { displayName, joinedAt }
```

### Host flow

1. Sign in with Google
2. Tap **Host Session** → select deck → room created with auto-generated 4-letter code
3. Share code verbally/visually (displayed large on screen)
4. Tap **Start** when everyone is in the lobby
5. Host sees the card + controls: **Answered** / **Skip** (same as solo play)
6. Tapping either advances `currentCardIdx` and writes to DB

### Participant flow

1. Sign in with Google
2. Tap **Join Session** → enter 4-letter code
3. Land on read-only mirror of the host's card view
4. Card updates in real time when host advances
5. No controls — participant view shows card text and deck theme only

### Room cleanup

Rooms are auto-deleted after 24h of inactivity via a Firebase Realtime Database rule with `indexOn` timestamp. No Cloud Functions needed — client sets `createdAt`, rules enforce TTL.

---

## 6. Centralized Deck Content (Firestore)

### Data model

```
decks/
  {deckId}/
    name: string
    colorVar: string        # e.g. "--gold"
    status: "active" | "archived"
    version: number
    updatedAt: timestamp
    cards: [
      { id, text, category? }
    ]
```

### Behaviour

- On app load: fetch all `status == "active"` decks from Firestore
- Merge with bundled decks (bundled = offline fallback + seed data)
- If Firestore fetch fails: use bundled content silently
- Archived decks disappear from deck select immediately on next load
- Jason manages decks from the Firebase console — no code change or redeploy needed

### Migration

Initial seed: the 7 existing decks from the HTML are written to Firestore once (migration script or manual console entry). The bundled copy in the HTML stays as the offline fallback.

---

## 7. PWA (iOS + Android Home Screen)

The game is already deployed on Vercel. Adding PWA support gives iOS users an "Add to Home Screen" icon without the App Store.

### Changes to `index.html`

```html
<!-- already present -->
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">

<!-- add -->
<link rel="manifest" href="/manifest.json">
<script>
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js');
  }
</script>
```

### `manifest.json`

```json
{
  "name": "Unfold",
  "short_name": "Unfold",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#1c1410",
  "theme_color": "#1c1410",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

### Service worker (`sw.js`)

Cache-first for the HTML, CSS, and bundled JS. Network-first for Firebase calls. This gives offline play when Firebase is unreachable.

### Distribution summary

| Channel | Platform | Install method |
|---|---|---|
| Vercel URL | All | Browser |
| PWA | iOS + Android | Add to Home Screen |
| APK (EAS) | Android | Sideload |

---

## 8. Build Order (incremental, each phase shippable)

| Phase | Scope | Firebase dependency |
|---|---|---|
| 1 | Lobby screen + end screen | None |
| 2 | Google SSO (Play Live gate) | Firebase Auth |
| 3 | Visual polish pass | None |
| 4 | PWA manifest + service worker | None |
| 5 | Firestore deck content | Firestore |
| 6 | Multiplayer room sync | Realtime Database |

Phases 1, 3, and 4 can be deployed without any Firebase setup. Phase 2 unlocks progress tracking. Phases 5–6 are the more complex backend work.

---

## 9. Open Questions / Deferred

- Firebase project creation and credentials (Jason to create; API key goes in HTML via CDN config block)
- Room code collision handling (low-probability for small user base — simple retry on write is sufficient)
- Guest-to-account migration (if user plays as guest then signs in, existing localStorage progress is not merged — acceptable for v1)
- Apple ID alternative to Google SSO (deferred — Google covers the primary use case)
