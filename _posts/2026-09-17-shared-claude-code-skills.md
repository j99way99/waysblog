---
title: 'Sharing Claude Code skills across projects'
date: 2026-09-17
permalink: /posts/2026/09/shared-claude-code-skills/
tags:
  - claude
  - tooling
---

I had the same skill files pasted into two repos. `devlog`, `loop-engineering`, three `fable-*` skills — byte-identical copies in `inven-manage-app` and `vet-receipt-app`. Every time I improved one, the other drifted. Today I moved the general ones into a single repo and wired it up so every project loads the same copy.

## The assumption that was wrong

I went in expecting to use `additionalDirectories`. That is the setting I had seen referenced for pointing Claude Code at directories outside the project, and my plan was to point it at a shared skills folder.

It does not do that. Before writing anything I checked what the installed version actually supports, and grepped the CLI binary for the key:

```
permissions.additionalDirectories
additionalDirectoriesForClaudeMd
```

The first is a tool-access allowlist — which directories Claude may read and write. The second is a separate internal thing for CLAUDE.md discovery. Neither is a skill source. `--add-dir` is the same allowlist by another name.

Skills come from exactly three places: the project's `.claude/skills/`, the user-level `~/.claude/skills/`, and plugins. So the supported way to share skills across projects is to publish them as a plugin.

That check cost about two minutes and saved me from building the whole thing on a setting that would have silently done nothing. The plan I wrote down first was wrong, and the only reason I noticed is that I looked instead of assuming.

## What shipped

One repo that is both a marketplace and the plugin it serves:

```
claude-skills/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json    # plugins: [{ name, source: "./" }]
└── skills/<name>/SKILL.md
```

The self-referencing `source: "./"` is what lets a single repo be both. I validated it with `claude plugin validate` on a throwaway copy before creating the real one. Installing at user scope means it loads in every project with no per-project configuration — no submodules, no symlinks, nothing copied:

```bash
claude plugin marketplace add j99way99/claude-skills
claude plugin install claude-skills@way-skills --scope user
```

Seven skills, split by what you are doing rather than what the code is written in: write (`java-spring-backend`), review (`code-review`), diagnose (`debugging`), design (`api-design`, `database-analysis`), plus `devlog` and `loop-engineering`, which are procedures that call the others. Language-specific checklists live in `references/` under the skill that uses them rather than becoming skills of their own. I dropped a planned `aws-architecture` skill after noticing the `aws-dev-toolkit` plugin already covers it — a second set of AWS review criteria is worse than one.

## The duplication had already broken

Promoting the two shared skills meant stripping out everything that only made sense in one app: named domain flows, a currency format, a UI theme convention, hardcoded `pnpm` commands, references to sibling skills that do not exist in the shared repo.

The part I did not expect: `vet-receipt-app`'s copy of `devlog` told Claude to read `logs/LOOPLOG.md` as its primary source, and that file exists only in `inven-manage-app`. The copy had been wrong for however long it had been sitting there. It was not going to fail loudly — it would just quietly write a worse post. Duplication does not stay duplicated; it decays, and you find out late.

## What nearly caught me

Installed plugins are copied into a cache and pinned to the commit they were installed from:

```
~/.claude/plugins/cache/way-skills/claude-skills/0.1.4/
```

Editing the repo does nothing to a running session. I found this only because I checked whether the cache was a copy or a link, and then tested the update path end to end. The first attempt failed:

```
$ claude plugin update claude-skills
✘ Plugin "claude-skills" not found
```

It resolves only the `plugin@marketplace` form. Both facts are now in the repo's README, because I would otherwise have rediscovered them in three weeks while wondering why an edit had no effect.

One thing I did not verify: I could not run a headless `claude -p` session to confirm a skill fires, because the CLI is not separately logged in on this machine. What I confirmed is that `claude plugin details` lists all seven skills at version 0.1.4 and reports their token cost. That is the plugin inventory being read correctly, not a skill actually triggering on a real prompt. Plugin changes apply on the next session, so the real check happens tomorrow.
