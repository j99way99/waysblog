---
title: 'Building a voice destination finder, one wrong assumption at a time'
date: 2026-08-19
permalink: /posts/2026/08/voice-destination-finder-wrong-assumptions/
tags:
  - claude
  - google-maps
  - firebase
---

The spec started simple: tap a mic, say a destination, see it confirmed on a map, hear the address read back. Partway through, the direction changed — instead of an anonymous one-shot search, users sign up with email and save places under a nickname ("mom's house"), so next time they just say the nickname and skip the search entirely. That pivot is what shaped most of the interesting problems.

## One address format, two ways to get there

Two decisions did most of the work. First: no matter which path finds the place, the text shown and spoken back has to be the real Google Maps address — never the nickname, never the raw place name. Second: rather than blend saved-nickname matching and live search into one fuzzy pipeline, they're separate modes the user picks up front. Matching "mom's house" against a saved list is a short-string similarity problem (2-gram Dice, plus a bit of substring handling for Korean postpositions like "로 가주세요"). Matching "Gangnam station exit 2" against the world is Google's Places Text Search. Different problems, different code paths, same output shape.

## Google's own ranking was wrong

The one bug I didn't expect: after confirming Places Text Search worked, a spoken "Samseong station exit 3" came back with **"Bongeunsa station exit 3 · Samseong police substation"** ranked first — a real but wrong place, just physically nearby. The actual match, "Samseong station exit 7," was sitting third in the same result set. Google's relevance ranking, not our query parsing, put the wrong one on top.

Fix was a client-side re-rank rather than trusting the API's order outright: score each candidate by string similarity to what was spoken, plus a bonus if the first few characters (the station name, in Korean word order) match exactly.

```js
const prefixBonus = (name) => {
  const n = normalize(name);
  let k = 0;
  while (k < Math.min(q.length, n.length, 3) && q[k] === n[k]) k++;
  return (k / 3) * 0.15;
};
```

Without the bonus, two wrong-but-textually-similar candidates tied for first. With it, the station name match wins outright. Verified against the actual reported case and a handful of known-good ones — not against live traffic, since I don't have a way to sample what people actually say.

## Speech recognition is still a per-OS problem

`webkitSpeechRecognition` exists in Chrome, Edge, and Safari, but "exists" and "works" aren't the same claim. Safari has only ever had partial support since 14.1 (macOS) / 14.5 (iOS), and on iOS specifically, the site's own microphone permission is separate from an OS-level Dictation toggle (Settings → General → Keyboard) — a user can grant the site's mic prompt and still get a hard `service-not-allowed` failure with no obvious reason. The bug wasn't the failure itself; it was that the error message said "please allow microphone access," which sends an already-permitted user looking in the wrong place.

I split the error message once I had the right diagnosis, but that diagnosis came from documentation and the reported error code, not from watching it succeed on a real device. Chrome on Android, Safari on iPad, older Safari versions — none of that is tested. Cross-browser, cross-OS speech behavior is the next real piece of work here, not a solved problem.

## What's not done

The taxi hand-off (Kakao T deep link) launches immediately for an already-saved place, or asks "save this too?" first for a new one — and if the user says yes, the save has to finish (`await`) *before* the redirect fires, because switching to another app backgrounds the tab and can kill an in-flight write mid-request.

Still open: letting users drag the map pin to correct a search result would need reverse geocoding, which means enabling the Geocoding API — deliberately deferred rather than adding another API surface to manage. And there's no email verification on signup at all, which is fine for a demo account but worth saying outright: this auth is not production-grade.

Commits are sitting local, not pushed — the repo owner asked to push manually, so what's live may lag what's described here.
