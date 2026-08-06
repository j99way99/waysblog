---
title: 'The architecture diagram had to be two diagrams'
date: 2026-08-04
permalink: /posts/2026/08/architecture-diagram-and-vpc-cidr/
tags:
  - claude
  - aws
  - architecture
  - terraform
  - systems
---

The [previous post](/posts/2026/08/aws-migration-baseline/) ended with a measurement and a strategy: [MarketDay](https://github.com/j99way99/inven-manage-app) stays on Vercel as the always-on environment, cheap permanent pieces move to AWS now, and the expensive tier gets built as code and run in short bursts. This post is the next step — the actual diagram, and the address plan underneath it. Both live in [marketday-infra](https://github.com/j99way99/marketday-infra).

One correction first. That post said the Terraform-versus-CDK choice was still open. It was closed the same day, in favour of Terraform, mostly because CDK's L2 constructs create supporting resources implicitly — `new ec2.Vpc()` provisions a NAT gateway per availability zone by default, which is precisely the cost I am trying to design around. You can turn that off, but only if you already know it is there.

## The diagram had to be two diagrams

I sat down to draw one architecture diagram and immediately could not.

The two tracks never run at the same time. The permanent track is S3 for product images and Parameter Store for config, running against the live app, costing cents per month. The burst track is VPC, ALB, ECS and RDS, applied and destroyed within a single session. Drawing them as one picture would show a system that has never existed and never will.

So it is two pages. Page one is what runs today. Page two is what comes up for an hour and then gets torn down, framed in a box labelled with that lifecycle. Splitting it turned out to make both pages easier to read, because each one only has to answer one question.

This is the sort of thing that looks obvious written down and was not obvious while staring at a blank canvas.

## Absence needs annotating

The more useful decision was about what to draw that is not there.

A reviewer looking at this diagram will notice there is no NAT gateway, no ElastiCache, no SQS, and no Multi-AZ standby. Without a note, every one of those reads as something I forgot. So each absence carries its reason on the page:

- **NAT gateway** — the classic idle-cost leak. Tasks sit in public subnets with tight security groups instead.
- **ElastiCache** — nothing in the codebase needs it. There is no server-side session or cache; auth is a self-signed JWT in a cookie, so the web tier is already stateless.
- **SQS** — the offline order queue already carries a client-generated idempotency key with a unique constraint behind it, so adopting SQS later is a transport swap, not a redesign.
- **Multi-AZ RDS** — drawn as a greyed-out box marked "production would have this, not provisioned", because it is a design-level topic and not a running cost here.

The security-group trade is the one I want to be honest about. Putting tasks in public subnets to avoid NAT means subnet placement is no longer doing the isolation work — security groups are, alone. That is cheaper and tears down more reliably, but it makes one over-permissive rule the single point of exposure. The design note says to keep the ALB → ECS → RDS chain strictly source-group-referenced and never CIDR-based, which is the part that actually holds the line.

## Valid XML is not a correct diagram

I generated the `.drawio` file as XML rather than drawing it by hand, then checked that it parsed and that no shape referenced a missing shape. Both passed. I took that as done.

It was not done. Rendering it in a browser and looking at it turned up three routing problems on the second page alone. An arrow crossed the VPC title. One line passed straight through the RDS box on its way somewhere else. Worst, the edge from the second ECS task to the database was routed as a straight horizontal line between the two ECS boxes — so the picture said the two tasks talk to each other, which is not the design and not true.

That last one is the reason this section exists. A structurally valid file can still communicate something false, and no parser will tell you. The only check that catches it is looking at the rendered output. I had Claude Code drive a browser to render and screenshot each page, which made the iteration quick, but the useful part was simply not trusting the validator.

## Reading images straight from S3

The baseline left an open question about whether the app reads images from S3 directly or through CloudFront. Direct.

At three images and 188 KB total, a CDN in front of them buys nothing measurable, and it brings a cache-invalidation story along with it. The permanent track is supposed to stay small enough that it is obviously worth running. CloudFront is not dropped from the project — it stays a burst-track exercise, where the interesting part is mirroring the cache-policy split the service worker already defines: immutable for hashed build assets, revalidating for menu documents.

Still open, and it does not change the architecture: public-read bucket versus presigned URLs.

## Choosing a CIDR is collision avoidance, not sizing

The VPC is `10.20.0.0/16`.

I expected this to be a capacity question and it is not. VPC and subnet CIDRs cannot be resized after creation, and unused address space is free, so there is no reward for choosing tightly. The only real cost of a bad choice is having to rebuild. That makes the question "what will this collide with later?"

The obvious candidates all fail that test. `10.0.0.0/16` is the default in every tutorial and module example, which makes it the most likely thing on the other end of any future peering connection. `172.31.0.0/16` is the AWS default VPC in every region — peering with the default VPC is a plausible thing to try during a practice session, and picking this range makes it impossible. `192.168.0.0/16` overlaps typical home and office networks, which breaks VPN scenarios.

`10.20.0.0/16` avoids all three and leaves `10.10` and `10.30` free if dev and prod stacks ever exist.

Subnets are a `/20` per tier per availability zone:

| Tier | AZ 1 | AZ 2 | Built now |
|---|---|---|---|
| Public | `10.20.0.0/20` | `10.20.16.0/20` | Yes |
| Private — app | `10.20.32.0/20` | `10.20.48.0/20` | Reserved |
| Private — data | `10.20.64.0/20` | `10.20.80.0/20` | Reserved |

4,091 usable addresses per subnet is absurd for a steady state of two Fargate tasks. It is also free, and the opposite mistake — running out mid-scale-test and rebuilding the VPC — is the expensive one.

The private tiers are reserved and not built. The operating model puts tasks in public subnets, so they are not needed for a working stack, but a later session may well want the private-subnet-plus-endpoints pattern, which is the more production-shaped design and the better exercise. Reserving the ranges costs nothing today; finding out later that there is no room does not.

That decision lives in `variables.tf` rather than in a module default, so it is visible at the top of the config:

```hcl
# Addressing is pinned here rather than left to a module default because VPC and
# subnet CIDRs cannot be resized after creation.
variable "vpc_cidr" {
  description = "VPC CIDR. Avoids 10.0.0.0/16 (tutorial default) and 172.31.0.0/16 (AWS default VPC) so peering stays possible."
  type        = string
  default     = "10.20.0.0/16"
}
```

Availability zones come from the `aws_availability_zones` data source instead of being hardcoded. AZ names like `ap-northeast-2a` are mapped per account, so a hardcoded name is not portable across accounts.

Routing is deliberately minimal: one internet gateway, one route table shared by both public subnets, no NAT gateway, no Elastic IP, and a gateway VPC endpoint for S3 because it is free and attaches to the route table rather than billing per hour.

## Status

Still nothing running on AWS. No resources are declared yet, so `terraform plan` creates nothing, and `terraform validate` passing only says the configuration is well-formed — it says nothing about whether this works, because it has never been applied.

The address plan has been checked the only way it can be checked before that: a script confirming every subnet sits inside the VPC, that none of them overlap, and that the range does not collide with the three I was trying to avoid. That is arithmetic, not evidence.

August's remaining deliverable is the cost estimate, which needs two figures — a hypothetical always-on monthly cost, and the actual cost of one practice session. The second is the one that gets paid. I am still keeping dollar figures out of the repository documents until that estimate is written against current pricing, since AWS rates and free-tier terms change and a confidently stale number is worse than no number.

Before any of that, the budget alert. It was the boring next step at the end of the last post and it is still the gate in front of the first `apply`.
