---
title: 'archify vs Mermaid for the same diagram'
date: 2026-09-17
permalink: /posts/2026/09/archify-vs-mermaid/
tags:
  - claude
  - tooling
---

I wanted diagrams for shadowing-loop-next, a Next.js app on Vercel with Postgres on Neon and audio recordings on Cloudflare R2. Two options: [archify](https://github.com/tt-a1i/archify), a Claude Code skill that renders polished standalone HTML, or plain Mermaid. I drew the same system both ways and deleted archify the same day.

## What each one produced

**archify** gave me one architecture diagram as a 711 KB HTML file: dark and light themes, pan and zoom, search, guided views like "recording upload", PNG and SVG export, and three summary cards underneath. The layout was clean — no crossing lines, the main request path on a single row. It also passed all nine of its own layout checks with zero warnings.

**Mermaid** gave me three diagrams in one note: the architecture plus two sequence diagrams, one for uploading a recording and one for playing it back. It renders directly in Obsidian and on GitHub. The architecture layout was worse; the database drifted to the top corner and two edges ran diagonally.

On looks alone, archify wins.

## Why I still picked Mermaid

**archify's diagram was wrong, and its checks passed.** The core of the upload design is that the browser sends audio straight to R2 with a presigned URL, never through the server. archify routed that dashed line along the bottom edge of the Vercel boundary and put its label inside the box. At a glance, the picture says the upload goes through Vercel. The nine checks measure geometry — crossings, clearances, corridors — so a line in a misleading place is invisible to them. A good-looking diagram that states the wrong thing is worse than an ugly correct one, because people trust it.

**The sequence diagrams explained the design better than either architecture picture.** The part of this system that actually matters is that completing a session claims the row with a conditional update, so a double submit cannot count twice. In Mermaid that is a few lines anyone can read:

```
A->>DB: UPDATE sessions WHERE status='pending'
alt claim succeeded (1 row)
  A->>DB: increment day and plan counters
else already completed (0 rows)
  A-->>B: no counter change
end
```

No box-and-arrow layout shows that, however polished.

**The cost was an order of magnitude apart.** These are rough estimates from how much was read and written, not measured usage: the one archify diagram took around 15–20k tokens, covering its instructions, schemas, a JSON spec, validation, rendering, and a screenshot review. The three Mermaid diagrams took a few thousand. Drawing each sequence in archify would have meant repeating that whole loop per diagram.

**Mermaid stays editable where the diagram lives.** It is text inside the note, so a change is a one-line edit and a readable diff. The archify output is a generated artifact I would regenerate from a JSON file kept somewhere else.

## Mermaid's real weakness

Mermaid fails silently. A bad line gives you a broken diagram, not an error. That is the one thing archify's pipeline did better, so I kept that part: before saving, every Mermaid block now goes through a headless Chrome render that reports which block failed and on which line.

## When I would reach for archify again

If I need one presentation-grade architecture picture for a portfolio page, the polish is real. But I would read every edge against the code before showing it to anyone, because passing its checks tells you the layout is tidy, not that the diagram is true.
