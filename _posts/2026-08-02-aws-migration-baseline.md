---
title: 'Measuring first changed my AWS migration plan'
date: 2026-08-02
permalink: /posts/2026/08/aws-migration-baseline/
tags:
  - claude
  - aws
  - architecture
  - systems
---

[MarketDay](https://github.com/j99way99/inven-manage-app) runs on Next.js and Vercel with Postgres on Neon. The plan for this half of the year is to rebuild its backend on AWS as a way of actually using what I studied for the SAA, so August is a design month: architecture diagram, cost estimate, and the first infrastructure-as-code commits.

Before drawing anything I measured what the app is today. That turned out to matter more than the drawing, because two of my assumptions were wrong.

## What measuring changed

The production database is small — 8.8 MB total. But the shape of it was not what I expected.

```
products         3 rows   272 kB
seller_profiles  1 row    152 kB
teams            2 rows    64 kB
users            2 rows    48 kB
```

Product photos are stored in Postgres as base64 data URLs rather than in object storage. So:

```
products with an image : 3
total image bytes      : 188,069   (~69% of the products table)
largest single image   : 78,631
```

Nearly seventy percent of that table is image data. Applying the caps already in the code — 500 products per user, 250 KB per image — a single fully-loaded seller is bounded at about 125 MB inside the database. At the current average image size it is closer to 31 MB.

**First assumption broken:** I had been thinking about database sizing in terms of order volume, because orders are the thing that grows when the app is used. They are not the driver. Storage tracks *user count*, because each user brings a bounded but large block of image data with them. That changes which axis the cost estimate needs.

The second thing I found by looking for it deliberately: there is no server-side session store, no cache, no in-memory state anywhere. Auth is a self-signed JWT in a cookie. The offline order queue lives in the browser's localStorage. The service worker cache is also browser-side. The only server-side state is Postgres.

**Second assumption broken:** the reference architecture I had sketched from the plan included ElastiCache, because that is what these diagrams usually include. Nothing in the current code needs it. It is an optimisation I might want later, not a prerequisite for migrating. Leaving it out is one fewer always-on resource to pay for.

Neither of these would have come out of guessing. Both came from running three read-only queries.

## Sizing it down for practice

MarketDay is deliberately not open to the public, so there is no traffic that would justify running the architecture from the plan — ECS Fargate, RDS Multi-AZ, ElastiCache, an ALB — at the size it was originally sketched. Those resources bill by the hour whether anyone uses them or not.

So I decided to build the same shape much smaller, and to optimise it for cheap, efficient operation rather than for capacity. Splitting the work by cost rather than by component:

**Move the cheap permanent pieces now.** Images out of Postgres and into S3. At this volume that is cents per month, it removes the base64 overhead — roughly a third of the stored bytes — and it gives a live application a real dependency on AWS. Configuration goes to Parameter Store for the same reason.

**Build the expensive tier as code and run it in bursts.** VPC, ALB, ECS, RDS live as Terraform. Bring the stack up, verify it, capture the evidence, destroy it. Cost becomes per-exercise instead of per-month.

What I like about this is that it stopped feeling like a compromise once I wrote it down. Reproducing a full stack from code on demand *is* the thing infrastructure-as-code is for. A stack that can be recreated and torn down cleanly is arguably a better artifact than an idle environment somebody is paying to keep alive — it demonstrates the same knowledge plus the discipline to not leak resources.

It also decides a design constraint that I would otherwise have gotten wrong: the target is not an optimised always-on architecture. The target is a clean `apply` and a clean `destroy`. That is now a criterion in the Terraform-versus-CDK decision I still have to make.

Feature work continues on Vercel throughout. The two tracks never compete for the same hours.

## What actually costs money

If a practice session is going to cost a few dollars rather than a few tens of dollars, the thing to control is not the hourly rate. It is what stays running after I think I am done.

The one that catches people is the NAT gateway — charged hourly regardless of traffic, plus data processing on top, and easy to leave behind because nothing about it is visible in the app. For practice I would rather avoid it entirely: put tasks in public subnets with tight security groups, and use gateway VPC endpoints for S3, which are free and remove the reason to route that traffic through NAT at all.

The rest of the list is similar in character. Elastic IPs are billed when they are *not* attached to anything, which is the opposite of the intuition. EBS volumes and snapshots outlive the instances they belonged to. Load balancers bill from creation with zero requests.

So the practices that matter are unglamorous: set a budget alert before the first `apply`, tag everything so Cost Explorer can tell me what one session cost, time-box each session so the stack never survives overnight, and verify the teardown in the console rather than trusting the exit code of `destroy`.

One thing I deliberately kept out of the baseline document: dollar figures. Rates and free-tier terms change, and a stale number written down confidently is worse than no number, because later on it reads like a verified fact. The document explains what generates cost; the cost estimate deliverable is where current pricing gets looked up.

## Status

Nothing is built on AWS yet. This is a measurement and a decision, and the honest summary is that August's output so far is a baseline document and a strategy, not infrastructure. The Terraform-versus-CDK choice is still open, and so is whether the Next.js app eventually moves to ECS at all or whether a separate API service takes over the backend — under the burst model that is a diagram question rather than a hosting one, since Vercel keeps serving the app either way.

The next concrete step is small and boring: a budget alert, before anything else gets created.
