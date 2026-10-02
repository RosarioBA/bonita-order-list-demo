# Bonita Order List — Shareable Demo (with real persistence)

This is the second stage of the Bonita order-list project — same validated interaction model as
[`bonita-order-list-mockup`](https://github.com/RosarioBA/bonita-order-list-mockup), but meant to
be deployed somewhere real (GitHub Pages) instead of living inside a Claude Artifact, specifically
so data doesn't get wiped and other people can actually try it and give feedback.

It is **not** the real backend-based product (no login, no shared/multi-device sync, no admin
settings) — see that other repo's README/BRIEF for the full interaction spec, and the
`bonita-real-app-next-steps` planning notes for what the eventual real build looks like. This is
just "the same thing, but it actually saves, and anyone can open the link."

## What's different from the Claude-artifact version

Everything about the app itself is identical — same screens, same catalog, same interaction
model. The only change is **how data is saved**:

- **Before**: `localStorage` only, inside Claude's Artifact viewer. Worked, but data could be lost
  (e.g. a catalog-version bump wiping local edits, or just losing track of which browser/device had
  the data), and the link only really made sense to open from within Claude.
- **Now**: `localStorage` is still the fast, synchronous source of truth for every read/write in
  the app — nothing about the actual app code changed. A thin extra layer mirrors the whole
  `bonita_ol_*` slice of localStorage up to **Firebase (Firestore)** under an anonymous,
  per-browser identity, and restores it automatically if you open the link on a browser/device that
  doesn't have local data yet (or had it cleared). Each person who opens this gets their **own**
  private saved copy — nobody else can see or affect it.
- A small "☁ Saved" / "☁ Saving…" / "⚠ Not saved to cloud" badge in the bottom-right corner shows
  the current sync status. If Firebase isn't set up yet (see below) or the badge shows an error,
  the app still works exactly as before — this is a backup layer, not a hard dependency.
- There's also a **🕑 Order history** button on the order screen — unlike everything else in this
  app, that one *is* shared across everyone (not per-device), since the point of a history is
  seeing what others submitted. It reads from a separate shared Firestore collection and needs the
  extra security rule below to work.

## One-time setup: connecting a real Firebase project

This repo ships with a **placeholder** Firebase config (`FIREBASE_CONFIG` near the top of the
`<script>` block in `index.html`, search for `REPLACE_ME`) — I can't create a Firebase project on
your behalf since it needs your own Google account. Takes about 5 minutes:

1. Go to [console.firebase.google.com](https://console.firebase.google.com) and create a new
   project (free "Spark" plan is plenty for this).
2. In the project, go to **Build → Firestore Database → Create database** — start in **production
   mode** (we'll set the security rule below), pick any region.
3. Go to **Build → Authentication → Get started → Sign-in method → Anonymous → Enable**.
4. In **Project settings → General → Your apps**, click the **</>** (web) icon to register a new
   web app (no Firebase Hosting needed, just the config object). Copy the `firebaseConfig` object
   it gives you.
5. Paste those real values into `FIREBASE_CONFIG` in `index.html`, replacing every `"REPLACE_ME"`.
6. In **Firestore Database → Rules**, replace the default rules with this — the `users` block keeps
   each anonymous user's private backup document to themselves, and the `shared_submissions` block
   lets any signed-in (even anonymous) user read every submitted order and create new ones, but
   never edit or delete someone else's:
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /users/{userId} {
         allow read, write: if request.auth != null && request.auth.uid == userId;
       }
       match /shared_submissions/{submissionId} {
         allow read, create: if request.auth != null;
         allow update, delete: if false;
       }
     }
   }
   ```
7. Commit and push the updated `index.html`.

## Deploying to GitHub Pages

1. Push this repo to GitHub (if it isn't already).
2. In the repo's **Settings → Pages**, set source to **Deploy from a branch**, branch `main`,
   folder `/ (root)`.
3. GitHub gives you a `https://<username>.github.io/<repo-name>/` URL within a minute or two —
   that's the link to share with people for feedback.

No build step, no CI — it's the same plain HTML/CSS/JS file as the other prototype, so pushing a
new commit is the entire deploy process.

## Known limitations (same as the original prototype, see its README for full detail)

No login, no shared backend between different people (each person's data is private to them, by
design per this round's requirements — see the conversation notes if that ever needs to change),
box sizes and some De La Casa bar suppliers still need real numbers filled in via Item Info. This
is a feedback-gathering step, not the final product.
