# SDD Progress Ledger

Baseline commit: ed6d8e0

## Tasks
- Task 1: Screen Management + Lobby Screen — PENDING
- Task 2: End Screen — PENDING
- Task 3: Firebase Auth Setup — PENDING
- Task 4: Play Live SSO Gate — PENDING
- Task 5: Firebase Progress Persistence — PENDING
- Task 6: Animations & Micro-Interactions — PENDING
- Task 7: PWA Manifest + Service Worker — PENDING
- Task 8: Firestore-Backed Deck Loading — PENDING
- Task 9: Host a Session — PENDING
- Task 10: Join a Session — PENDING
- Task 11: Deploy to Vercel — PENDING
- Task 1: complete (commits ed6d8e0..9081472, review clean). Minor notes: ghost class undefined (gap in spec, not a bug), initGame not idempotent (downstream tasks should guard), showToast order safe.
- Task 2: complete (commits 9081472..f27f7c5, review clean). Low: BOM fixed inline. Low: .end-stat CSS unused (spec artifact). Very Low: placeholder text, non-issue.
- Task 3: complete (commits f27f7c5..00ab25d, review clean). No issues.
- Task 4: complete (commits 00ab25d..6409f98, review clean). Low: avatar broken-img edge case. Low: 900ms no spinner.
- Task 5: complete (commits 6409f98..ab0bad4, review PASS_WITH_NOTES). Medium (false alarm): reviewer flagged guest path not calling buildPool/updateChips — but initGame() does this synchronously, so no bug. Low: saveState() localStorage write missing try/catch on QuotaExceededError. Low: seen stored as JSON string not native Firestore map (intentional design). Low: auth state race (handled by onAuthReady caller).
- Task 6: complete (commits ab0bad4..7ea5e4e). Card deal bounce (cardDeal keyframes), body background-color wash on draw, button press scale+shadow transition, progress bar cubic-bezier, haptics in draw() and markCard(). Reviewer caught animation-fill-mode:forwards regression (blocked .card.flipped transition); fixed by removing forwards. Final clean commit: 7ea5e4e.
- Task 7: complete (commits 7ea5e4e..d2838fd). manifest.json + sw.js created. index.html: manifest link in head + SW registration at end of script. Note: implementer introduced UTF-8 double-encoding corruption (read file as Windows-1252, re-saved as UTF-8); fixed in d2838fd by encoding round-trip reversal (UTF-8 decode → Windows-1252 encode → UTF-8 decode, strip BOM).
