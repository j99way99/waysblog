---
title: 'Building a register: PoS payment constraints in Korea, Japan, and the US'
date: 2026-07-30
permalink: /posts/2026/07/pos-payments-kr-jp-us/
tags:
  - claude
  - payments
  - pos
  - systems
---

I have been building [MarketDay](https://github.com/j99way99/inven-manage-app), an inventory and on-site ordering tool for flea-market and pop-up sellers. Sellers register products, group them into a menu for each event, and share a public menu board that customers order from on their own phones.

The next thing it needed was a register — a proper point-of-sale screen for the person standing behind the table. That turned out to be less about UI than about how payments actually work in different countries, so this post covers both: the constraints I hit, and what I shipped in the first pass.

## The distinction that decides your architecture

My first instinct was wrong. I assumed "card payments in Korea can't be done from a web app," which is too broad. The real split is:

- **Card-not-present (online).** A web app is fine. In Korea you contract with a PG — Toss Payments, PortOne, Inicis, NICE — and you never touch a card number.
- **Card-present (in person).** This is where the constraint lives. Korea routes physical card taps through a VAN (Smartro, KIS, NICE, KICC) and there is no browser API for it.

Everything downstream follows from that one line. A menu board taking pre-orders is a web problem. A register taking a card at the counter is a hardware problem, and how hard it is depends entirely on which country you are in.

## Three countries

| | Korea | Japan | US |
|---|---|---|---|
| Online PG | Toss Payments, PortOne, Inicis | Stripe, GMO PG, PAY.JP, Komoju | Stripe, Square, Adyen |
| In-person | **VAN contract required** — no web control | Rakuten Pay, Square, AirPay | **Stripe Terminal / Square** — driven from web or app |
| In-person difficulty | High (needs a native bridge) | Medium | Low |
| Currency minor units | 0 (₩) | 0 (¥) | 2 (¢) |
| Tax display | Included (VAT 10%) | Usually included (10% / 8% reduced) | **Excluded**, varies by state |
| 3-D Secure | Card-issuer auth window | EMV 3DS **mandatory** | Optional, merchant carries the risk |

The in-person row is the whole story. In the US, Stripe Terminal is a documented API you call from your own code. In Korea, the equivalent is a certified terminal talking to a VAN, and the practical path is an Android device running your web app in a WebView with a native bridge to the terminal SDK. Same product, completely different amount of work.

### Korea

- Business registration plus an e-commerce filing before a PG will sign you. Review takes days to weeks.
- Skipping the wallets (KakaoPay, Naver Pay, Toss Pay) measurably costs you conversion. They are not optional in practice.
- Cash-equivalent methods (bank transfer, virtual accounts) over ₩100,000 require escrow or consumer-protection insurance.
- Cash receipts (현금영수증) are mandatory for certain industries above ₩100,000 per cash transaction — which lands directly on a register that takes cash.
- Cancellation splits into same-day authorization reversal versus post-settlement refund, and partial-cancel support differs per PG.

### Japan

- Method coverage drives conversion: cards, convenience-store payment, bank transfer, cash-on-delivery, PayPay, Rakuten Pay.
- **Convenience-store and bank-transfer payments are order-first, pay-later.** You need to hold stock until payment confirms, which is a fundamentally different inventory model from a register that decrements immediately. If you port a PoS-shaped design into Japanese e-commerce without adding a hold state, your stock counts will drift.
- The Installment Sales Act requires merchants not to retain card numbers. Tokenize, or don't take the payment.
- EMV 3DS is now effectively mandatory for e-commerce merchants under the METI security guidelines.
- A Specified Commercial Transactions Act disclosure page (business name, address, phone, return policy) is a hard requirement, not a nice-to-have.
- JPY is zero-decimal. In Stripe, ¥100 is `amount: 100`. Sending `10000` charges a hundred times too much, and it is an easy mistake to make if you have only worked in USD.

### US

- Sales tax is the real complexity. Rates vary by state, county, and city, and economic nexus rules mean crossing a threshold in a state (commonly $100k or 200 transactions) creates a filing obligation there. Use Stripe Tax, TaxJar, or Avalara. Do not compute this yourself.
- Prices are displayed tax-exclusive, the opposite of Korea and Japan. That is a branch in your price formatting, not a config value.
- 3DS is optional, so chargeback risk stays with the merchant. Budget for dispute handling.
- Tipping is close to mandatory for in-person retail, and non-certified EMV terminals shift fraud liability onto you.

## What that means for the code

Since the in-person story differs so much per country, the register cannot know anything about the hardware. I settled on a port/adapter boundary: the sale flow depends only on interfaces, and each country or device gets an adapter.

```ts
export type PaymentRequest = {
  clientKey: string;        // same key as the sale — this is what makes retries safe
  amount: number;           // integer minor units
  method: 'cash' | 'card' | 'qr';
  installments?: number;
};

export type PaymentResult =
  | { ok: true; transactionId: string; approvalNo?: string }
  | { ok: false; code: string; message: string; retriable: boolean };

export interface PaymentPort {
  readonly id: string;
  readonly methods: readonly PaymentRequest['method'][];
  isAvailable(): Promise<boolean>;
  authorize(req: PaymentRequest): Promise<PaymentResult>;
  cancel(transactionId: string, amount: number): Promise<PaymentResult>;
}
```

The scanner gets the same treatment, and there the pragmatic discovery was that most barcode scanners are HID keyboard devices. They type the code and press Enter. No driver, no WebUSB permission prompt. Human typing and scanner typing are separable by inter-key timing alone, so "works with any scanner you plug in" is mostly a 20-line adapter.

## What I actually shipped

Phase one is deliberately small: **cash only**, no card, no payments table yet.

The useful realization was that I had already written most of the transaction engine. MarketDay's menu board works offline — orders queue in localStorage and sync later — and that sync path already had the two properties a register needs: a unique `client_key` for idempotency, and a guarded stock decrement inside a transaction with an audit row per line. A register is that same engine with a different input and a different tender step.

So `commitPosSale` reuses it. The terminal generates a UUID before payment starts, and a replayed request finds the existing sale instead of selling the stock twice:

```ts
const [existing] = await db
  .select({ id: orderGroups.id })
  .from(orderGroups)
  .where(eq(orderGroups.clientKey, clientKey))
  .limit(1);

if (existing) {
  // Return the original order rather than committing a second one.
}
```

The stock decrement stays conditional, so concurrent sales cannot drive it negative:

```ts
.update(products)
.set({ stock: sql`${products.stock} - ${quantity}` })
.where(and(
  eq(products.id, product.id),
  sql`${products.stock} >= ${quantity}`
))
```

On top of that: an optional `barcode` column with a *partial* unique index on `(created_by, barcode) WHERE barcode IS NOT NULL`, so two sellers can reuse a code and any number of products can leave it blank. Then the register screen itself — text-only product tiles (no base64 image payloads, so it stays fast), category chips, an autofocused scan box where Enter resolves an exact barcode first, a cart with quantity steppers, cash tender with change calculation, and a receipt showing the order number. Two-pane on a tablet, sticky charge bar and a full-screen cart sheet on a phone.

End-to-end on dev: three units across two products, `ORD-000006`, ₩50,600 total, ₩9,400 change, and stock moved 39→37 and 20→19 after a reload. Then a barcode saved on a product and scanned straight into the cart.

## Two failure modes worth designing around

**Authorized but not committed.** The worst outcome in any PoS is taking the money and failing to record the sale — no stock movement, no receipt, an angry customer. The ordering matters: never decrement stock before approval, persist the approval number the moment you have it, and treat the commit as a retriable step keyed on the same idempotency key. Card authorization is also online-only, so an offline register can take cash and nothing else. That is a property of the card networks, not something better code fixes.

**Migrations before code.** I nearly shipped this one badly. The branch adds a column, and `main` auto-deploys. Pushing code that selects `products.barcode` before the production database has that column would have broken product pages that had nothing to do with the feature, because the ORM expands `select()` to every column in the schema. The project log already had an outage from exactly this pattern a few weeks earlier. Migrate the database first, verify, then ship the code — and check which database your env file actually points at before you run anything.

## Next

Phase two extracts the ports properly and adds the keyboard-wedge scanner adapter, so the register stops depending on a focused input. Card comes after that, and by country it will be QR wallet deep links first (web-only, no VAN), then online PG, and native VAN bridging last — the order of increasing pain.

*Payment and tax rules change often and vary by industry and revenue. Treat the above as design-level orientation, not compliance advice; verify against current PG documentation before you sign anything.*
