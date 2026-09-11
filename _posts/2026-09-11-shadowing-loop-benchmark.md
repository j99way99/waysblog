---
title: 'The benchmark was never the native speaker'
date: 2026-09-11
permalink: /posts/2026/09/shadowing-loop-benchmark/
tags:
  - claude
  - systems
  - nextjs
  - firebase
  - performance
---

Most shadowing apps I looked at share the same loop: listen to a native speaker, repeat, record, compare against the original. The benchmark is fixed, so the question the app answers is always "how close am I to native?"

shadowing-loop asks a different question. You type in your own text, read it out loud five times a day for one to five days, then hear your first take next to your last. A three-day plan means fifteen readings of the same passage, first compared with fifteenth. The benchmark isn't a native speaker — it's you, a few days ago. None of the apps I reviewed made that comparison the core experience, so it was the one thing the MVP had to protect.

## Building fast enough to find out if it works

What I actually wanted to know wasn't whether I could build this — it was whether comparing your own recordings across repetition is useful at all. So the first version was a single-page app on Firebase Auth, Firestore, and Storage. Six screens, all logic client-side, managed services doing auth/db/storage so I could spend the time on plan, session, and recording state instead.

## What "fast to build" cost later

Once the core flow worked, loading felt slow, and the cause was structural: JS parse → Firebase init → `onAuthStateChanged` → Firestore WebChannel handshake → `listPlans` → `getDailyPlan` per day, each step waiting on the last. Firestore's `persistentLocalCache` helped repeat visits but did nothing for the first load — that's the one that matters for anyone trying the idea for the first time. Tuning wasn't going to fix a chain of sequential round trips; the architecture had to change.

I built shadowing-loop-next in a separate repo — Next.js 15 App Router, Drizzle, Neon Postgres, Cloudflare R2 — replacing the round-trip chain with one server-side query and one HTML response. Kept the Firebase version untouched and deployed both to compare:

- Lighthouse: 81 → 100
- Largest Contentful Paint: 4.9s → 1.5s (−69%)
- Requests: −54%
- Transferred data: −60%

## The bug the migration surfaced

Moving the code also moved where decisions got made. Firestore rules checked ownership but never validated the payload, so completion logic — read the attempt counter, add one, write it back — lived entirely on the client. A duplicate submission could double-count an attempt. That's not a cosmetic bug here: the whole product is "compare attempt 1 to attempt 15," and a wrong counter breaks the one thing shadowing-loop is for.

In the Next.js version, auth, usage limits, and completion checks moved server-side, and the read-increment-write got replaced with a conditional update:

```sql
UPDATE ...
SET ...
WHERE status = 'pending'
```

Ran it under concurrent transactions to confirm a session can only complete once.

## Status

What's actually proven so far: the compare-past-self flow can be built, made fast, and made correct under concurrency. What isn't proven yet: whether reading the same passage fifteen times and hearing yourself change is a useful way to learn — that was the original question, and getting the architecture right was a precondition for testing it honestly, not a substitute for it.
