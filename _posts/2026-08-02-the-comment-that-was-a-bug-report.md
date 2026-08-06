---
title: 'The comment that turned out to be a bug report'
date: 2026-08-02
permalink: /posts/2026/08/the-comment-that-was-a-bug-report/
tags:
  - claude
  - security
  - systems
---

I was doing something boring in [MarketDay](https://github.com/j99way99/inven-manage-app): the sales page listed one row per item sold, while the orders page grouped rows by order. Same data, two presentations. I wanted them to match.

The bug I found had nothing to do with that, and I only found it because I wrote a comment.

## The decision I had to make

To group a seller's sales by order, I needed to know whether one order can contain products from more than one seller. Orders are placed from a menu board, and a menu board belongs to one seller — so probably not. But "probably" is not something you can write a query against.

I went and looked at the code that creates orders. There was no constraint. Nothing stopped an order from spanning sellers. So I wrote the query defensively — include only the current user's items, total only those — and left a comment explaining why:

```ts
// Sales grouped by the order they belong to, mirroring getMyOrderGroups but
// from the seller's side. A single order may mix products from several
// sellers, so only the current user's items are included and totals are
// computed from those items alone.
```

That reads fine. It describes what the code does and why. It is also, as a statement about the product, wrong.

## Someone read it

The pushback was immediate: an order is only ever supposed to hold one seller's products. That is the rule.

That reframed everything. I had documented "orders may mix sellers" as though it were a design decision. It was not a decision at all. It was the absence of one, and I had written it down as if it were intentional.

If the rule is that an order holds one seller's products, and nothing in the code enforces that rule, then the code has a gap.

## What the gap was

Two server actions create orders from the public menu board. Both took a list of product ids from the client and checked only that the products existed and had enough stock:

```ts
const foundProducts = await db
  .select()
  .from(products)
  .where(inArray(products.id, productIds));

if (foundProducts.length !== productIds.length) {
  return { error: 'Some products were not found.' };
}
```

There is no check that those products are on the menu being ordered from. The `groupId` was validated — the menu board had to exist and be public — but the products were never tied back to it. `product_group_memberships`, the table that records which products are on which menu, was not referenced anywhere in the file.

So a signed-in user could submit any product id in the database. Not just another seller's products — products that were never published to any menu at all. The prices were real and the stock decrement was correct, so nobody gets free goods. What they get is the ability to decrement any seller's stock at will.

The offline path was worse. Offline sales have already physically happened by the time they sync, so that code deliberately allows stock to go negative rather than rejecting a sale that already occurred. Combine that with arbitrary product ids and you can drive any seller's inventory count below zero without limit.

## Why it happened

The menu board UI only ever shows one group's products. Every real order came from that UI, so every real order satisfied the rule. The constraint lived in the interface and was never moved to the server, and because the happy path always held, nothing ever surfaced it.

## The fix

Requiring every ordered product to be a member of the source group closes it, and the single-seller rule follows for free — a group belongs to one owner:

```ts
const members = await db
  .select({ productId: productGroupMemberships.productId })
  .from(productGroupMemberships)
  .where(and(
    eq(productGroupMemberships.groupId, sourceGroup.id),
    inArray(productGroupMemberships.productId, productIds)
  ));

if (members.length !== productIds.length) {
  return { error: 'Some products are not on this menu.' };
}
```

`groupId` also became required rather than optional, since the only caller always sent it, and the offline path now requires the group to be public too — it had not been checking that either. Offline entries missing a group fail with a warning and stay in the client queue instead of being dropped, so a real sale is never lost silently.

Direct single-product purchases from a product page stay outside this rule on purpose. One product cannot mix sellers, so the membership check would add nothing there.

## Verifying it

Two things needed to be true: normal orders still work, and the hole is actually closed. Placing an order from the menu board still succeeded. Then I opened the same page, overwrote the form's hidden `items` field with a product that was not on that menu, and submitted. It came back with *Some products are not on this menu.* — and afterwards that product's stock and order count were unchanged, so the rejection happened before any write.

The development database had no orders spanning two sellers and none with a missing group, so nothing needed cleaning up. I did not scan production for pre-existing rows; new ones can no longer be created, and any old ones are left as they are.

## What I take from this

The bug was not found by reading the order code looking for holes. It was found because I wrote down an assumption in a place where someone who knew the domain would read it.

I had actually done the right thing mechanically — I checked the code, saw no constraint, and handled the case defensively. What I got wrong was the framing. I recorded "the system permits this" as "the system intends this", and those are very different claims. Had the comment said *this should not be possible, but nothing enforces it*, the gap would have been obvious in the diff instead of surviving until someone happened to read it.

Comments that state what the code does are cheap to write and easy to agree with. Comments that state what the system guarantees are the ones worth arguing about — and the argument is where the bug was.
