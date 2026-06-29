# Graph Report - D:/Claude/Unfold  (2026-06-29)

## Corpus Check
- Corpus is ~14,718 words - fits in a single context window. You may not need a graph.

## Summary
- 90 nodes · 94 edges · 12 communities (8 shown, 4 thin omitted)
- Extraction: 99% EXTRACTED · 1% INFERRED · 0% AMBIGUOUS · INFERRED: 1 edges (avg confidence: 0.5)
- Token cost: 0 input · 0 output

## Community Hubs (Navigation)
- [[_COMMUNITY_Card Deck Content|Card Deck Content]]
- [[_COMMUNITY_Game UI & Animations|Game UI & Animations]]
- [[_COMMUNITY_State & Persistence|State & Persistence]]
- [[_COMMUNITY_Deck Toggle & Chips|Deck Toggle & Chips]]
- [[_COMMUNITY_APK Build System|APK Build System]]
- [[_COMMUNITY_App Icon Assets|App Icon Assets]]
- [[_COMMUNITY_Progress Tracking|Progress Tracking]]
- [[_COMMUNITY_Deployment & Hosting|Deployment & Hosting]]
- [[_COMMUNITY_Typography & Styling|Typography & Styling]]
- [[_COMMUNITY_Card Draw Logic|Card Draw Logic]]
- [[_COMMUNITY_Toast & Feedback|Toast & Feedback]]
- [[_COMMUNITY_PWA & Meta Config|PWA & Meta Config]]

## God Nodes (most connected - your core abstractions)
1. `expo` - 13 edges
2. `adaptiveIcon` - 5 edges
3. `scripts` - 5 edges
4. `ios` - 3 edges
5. `android` - 3 edges
6. `App()` - 2 edges
7. `web` - 2 edges
8. `extra` - 2 edges
9. `eas` - 2 edges
10. `styles` - 1 edges

## Surprising Connections (you probably didn't know these)
- `Card Flip Animation` ----> `Unfold`  [EXTRACTED]
   →   _Bridges community 3 → community 1_
- `Unfold` ----> `CATS Data Structure`  [EXTRACTED]
   →   _Bridges community 3 → community 4_
- `Unfold` ----> `Expo EAS (Android APK Build)`  [EXTRACTED]
   →   _Bridges community 3 → community 5_
- `Card Pool` ----> `CATS Data Structure`  [EXTRACTED]
   →   _Bridges community 4 → community 1_

## Import Cycles
- None detected.

## Communities (12 total, 4 thin omitted)

### Community 0 - "Card Deck Content"
Cohesion: 0.11
Nodes (17): projectId, expo, extra, icon, ios, name, orientation, owner (+9 more)

### Community 1 - "Game UI & Animations"
Cohesion: 0.18
Nodes (13): Auto Reset on Deck Completion, Card Flip Animation, Card Pool, Chip Progress Display, Deck Toggle / Chip Filter, Keyboard Shortcuts, localStorage Persistence, Manual Reset (+5 more)

### Community 2 - "State & Persistence"
Cohesion: 0.20
Nodes (9): main, name, private, scripts, android, ios, start, web (+1 more)

### Community 3 - "Deck Toggle & Chips"
Cohesion: 0.22
Nodes (10): Card Deal Animation, conversation-deck.html, index.html, Jason Tjiadi, PWA / Mobile Web App Meta, Single HTML File Architecture, Typography, Unfold (+2 more)

### Community 4 - "APK Build System"
Cohesion: 0.39
Nodes (9): CATS Data Structure, CSS Design System / Color Palette, For Friends Deck, Get to Know You Deck, Hot Takes Deck, Icebreakers Deck, Relationships Deck, This or That Deck (+1 more)

### Community 5 - "App Icon Assets"
Cohesion: 0.29
Nodes (7): android-icon-background.png, android-icon-foreground.png, android-icon-monochrome.png, Expo EAS (Android APK Build), favicon.png, icon.png, splash-icon.png

### Community 6 - "Progress Tracking"
Cohesion: 0.29
Nodes (7): backgroundColor, backgroundImage, foregroundImage, monochromeImage, adaptiveIcon, package, android

### Community 7 - "Deployment & Hosting"
Cohesion: 0.29
Nodes (7): dependencies, expo, expo-status-bar, react, react-native, react-native-webview, sharp

## Knowledge Gaps
- **33 isolated node(s):** `styles`, `name`, `slug`, `version`, `orientation` (+28 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **4 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `expo` connect `Card Deck Content` to `Progress Tracking`?**
  _High betweenness centrality (0.063) - this node is a cross-community bridge._
- **Why does `android` connect `Progress Tracking` to `Card Deck Content`?**
  _High betweenness centrality (0.029) - this node is a cross-community bridge._
- **What connects `styles`, `name`, `slug` to the rest of the system?**
  _33 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Card Deck Content` be split into smaller, more focused modules?**
  _Cohesion score 0.1111111111111111 - nodes in this community are weakly interconnected._