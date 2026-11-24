# Choosing a ticketing model

Shade's codebase contains two unrelated ticketing implementations: the event module that used to live in the `shade` hub, and the standalone `ticketing` contract that `ticketing_factory` deploys. This page compares them feature by feature, documents the hub's bulk-purchase discount in full, and recommends which to use for common kinds of event. It is for integrators and contributors deciding how to sell and track tickets.

> **Warning:** Neither model is a supported, deployable product today. Since [ADR 0007](../../../docs/architecture/decisions/0007-core-gateway-scope.md), the `shade` hub has no ticketing entry points: the former hub module lives in [`example/ticketing/src/from_shade/`](../../ticketing/src/from_shade/) behind the `from-shade` feature and does not compile (`cargo build -p ticketing --features from-shade` fails with 3 errors). The standalone contract builds and its tests pass, but everything under `example/` is reference material that CI does not test. Read this page as a guide to what each design does, not as an API you can call on a live network.

## Status of each model

| | Former hub module | Standalone `ticketing` contract |
|---|---|---|
| Source | [`example/ticketing/src/from_shade/event.rs`](../../ticketing/src/from_shade/event.rs), entry points in [`entrypoints.rs`](../../ticketing/src/from_shade/entrypoints.rs) | [`example/ticketing/src/lib.rs`](../../ticketing/src/lib.rs) and [`example/ticketing_factory/src/lib.rs`](../../ticketing_factory/src/lib.rs) |
| Callable on the `shade` contract | No. The functions were removed from the hub's interface. | Not applicable: a separate contract. |
| Builds | No, with `--features from-shade` (unresolved `ContractError` import, private `DataKey`) | Yes |
| Tests | `src/from_shade/tests/` is behind the same feature and does not run | `cargo test -p ticketing -p ticketing_factory` passes (38 and 10 tests) |
| Depends on the gateway | Yes. It reads the hub's merchant registry, token whitelist and merchant accounts, and routes platform fees through `shade::components::platform_fee`. Restoring it means putting it back inside `shade`. | No. It never calls the hub. |

## Feature comparison

| Capability | Former hub module | Standalone `ticketing` |
|---|---|---|
| **Who creates an event** | A registered, active Shade [merchant](../../../docs/glossary.md#merchant) (`create_event`), in a token on the hub whitelist | Any address that authorizes (`create_event`). A factory-deployed instance has no owner, so anyone can create events in it. |
| **Primary sale payment** | On-chain. `purchase_ticket` takes the current price from the buyer. | None. `issue_ticket` and `issue_tiered_ticket` are organizer-only and move no tokens; collect payment some other way. |
| **Pricing** | One `ticket_price` per event, in one token | `Tier.price` per tier, stored for display only. The contract never charges it. |
| **Tiers** | No | Yes. `add_tier` with its own `max_supply`; the sum of tier supplies cannot exceed `max_capacity`. See [Ticket tiers](ticket-tiers.md). |
| **Dynamic pricing** | Yes. `configure_dynamic_pricing` sets an early-bird discount until `early_bird_end` and a late markup after it, both in basis points. | No |
| **Bulk discounts** | Yes. `purchase_tickets_bulk`, see [below](#bulk-purchase-discounts). | No |
| **Capacity** | `capacity` (required, at least 1) against `sold` | `max_capacity: Option<u64>` against active (non-refunded) tickets; `None` is unbounded |
| **Resale and royalties** | `resell_ticket` (seller and buyer both sign), plus a listing flow (`list_ticket`, `cancel_ticket_listing`, `buy_ticket_from_listing`) that was never exposed as an entry point. Royalty is `Event.royalty_bps`, paid to the merchant account. | `resell_ticket` (seller and buyer both sign) after the organizer calls `set_resale_config`. Royalty is `ResaleConfig.royalty_bps` in `ResaleConfig.payment_token`, paid to the organizer. |
| **Free transfer** | No transfer function; ownership changes only through resale | `transfer_ticket`, with no payment and no royalty |
| **Check-in** | No | `check_in` (organizer only), `verify_ticket` against a 32-byte QR hash, `CheckInRecord`. See [Ticketing contract reference](../contracts/ticketing.md). |
| **Waitlists** | No | `join_waitlist` once the event is at capacity; `refund_ticket` issues a ticket to the first entry. See [Event waitlists](event-waitlists.md). |
| **Refunds** | Only by cancelling the whole event | `refund_ticket` by the holder: marks the ticket refunded and frees a slot. It moves no tokens. |
| **Cancellation** | `cancel_event_and_batch_refund` refunds every ticket's `purchase_price` from the merchant's **wallet** address in one transaction | `cancel_event` sets `EventState::Cancelled` and refunds nothing |
| **Platform fees** | Yes, on `purchase_ticket` through the hub's fee split (`PlatformFeeRouteKind::TicketPurchase`). Not on bulk purchases, resales or royalties. | None |
| **Settlement** | Merchant account contract, for single purchases and royalties | Organizer address, for royalties only |
| **Analytics and events** | `purchase_ticket` records merchant analytics and emits `PlatformFeeRoutedEvent` and `TicketPurchasedEvent`. `purchase_tickets_bulk` records and emits nothing. | Own events only (`TicketIssuedEvent`, `TicketCheckedInEvent`, `TicketResoldEvent`, and others); nothing in hub analytics |
| **Search** | `search_events` with `EventFilter` (merchant, cancelled, date range, minimum seats left) | `get_event_tickets`, `get_event_tiers`, `get_waitlist` and counts; no filtered search |
| **Pause and upgrade** | Inherited from the hub: entry points checked `assert_not_paused` | Neither. An instance is immutable once deployed and has no admin. |
| **Deployment** | Part of the single hub contract | One contract per `deploy_event_contract` call, at a random address, recorded as an `EventRef` |

### Gaps in the standalone contract

These were confirmed by running the contract, and matter if you build on it:

- `issue_tiered_ticket` does not check whether the event is cancelled, so tiered tickets can still be issued for a cancelled event. `issue_ticket` does check.
- A refunded ticket still passes `verify_ticket` with `valid: true` and can still be checked in.
- `transfer_ticket` works on tickets for a cancelled event, and on refunded tickets.
- `start_time` and `end_time` are stored but never enforced: tickets can be issued and checked in after the event has ended.
- Every issuance scans every ticket in the contract to keep QR hashes unique, so issuing gets more expensive as a contract accumulates tickets.

## Data types

The two models use different types with overlapping names. Nothing converts between them.

| Concept | Former hub module (`from_shade/types.rs`) | Standalone (`ticketing/src/lib.rs`) |
|---|---|---|
| Event identity | `Event.id: u64`, plus `merchant_id: u64` from the merchant registry | `Event.event_id: u64`, plus `organizer: Address` |
| Event timing | `date` (creation time) and `event_date` (must not be in the past) | `start_time` and `end_time` (`end_time >= start_time`) |
| Event price | `ticket_price: i128` and `token: Address` | None on the event. `Tier.price` per tier, informational. |
| Event capacity | `capacity: u32` and `sold: u32` | `max_capacity: Option<u64>`; tickets counted from the event's ticket list |
| Event state | `cancelled: bool` and `refunds_processed: bool` | `state: EventState` (`Active` or `Cancelled`) |
| Dynamic pricing | `early_bird_end`, `early_bird_discount_bps`, `late_markup_bps` on `Event` | None |
| Royalty | `royalty_bps: u32` on `Event`, set at creation | `ResaleConfig { event_id, payment_token, royalty_bps }`, set separately |
| Ticket identity | `Ticket.id: u64`, `event_id`, `owner: Address` | `Ticket.ticket_id: u64`, `event_id`, `holder: Address` |
| Ticket payment record | `purchase_price: i128`, `minted_at: u64` | None |
| Ticket check-in | None | `qr_hash: BytesN<32>`, `checked_in: bool`, `check_in_time: Option<u64>` |
| Ticket tier and refund | None | `tier_id: Option<u64>`, `refunded: bool` |
| Resale listing | `TicketListing { ticket_id, seller, price }` | None; resale is one direct call |
| Errors | `EventError`, codes 120–129, plus hub `ContractError` codes | `TicketingError`, codes 1–21 |
| Storage keys | `EventKey` in the hub's storage | Private `DataKey` in each instance's own storage |

> **Warning:** Event and ticket IDs both start at 1 in every contract. The same number means different things in the hub, in each standalone instance, and across standalone instances. Always store the contract address alongside an ID.

## Bulk purchase discounts

`purchase_tickets_bulk` belongs to the former hub module only. The standalone contract has no bulk purchase.

```rust
pub fn purchase_tickets_bulk(
    env: Env,
    event_id: u64,
    buyer: Address,
    quantity: u32,
    shade_token: Address,
    merchant_account: Address,
)
```

Source: [`example/ticketing/src/from_shade/event.rs#L446-L490`](../../ticketing/src/from_shade/event.rs#L446-L490), exposed in `entrypoints.rs` behind `assert_not_paused`.

### Discount tiers

The discount depends only on `quantity` in the one call ([`group_discount_bps`, `event.rs#L432-L444`](../../ticketing/src/from_shade/event.rs#L432-L444)):

| `quantity` | Discount | Basis points |
|---|---|---|
| 1–4 | None | 0 |
| 5–9 | 5% | 500 |
| 10–19 | 10% | 1,000 |
| 20 or more | 15% | 1,500 |

Splitting a purchase across calls gets the discount for each call's own quantity: two calls of 4 get no discount, one call of 8 gets 5%.

### How the price is calculated

```text
unit_price = current ticket price (early-bird or late-markup applied)
gross      = unit_price × quantity
discount   = gross × discount_bps / 10_000   (integer division, rounds down)
net        = gross − discount
```

- The group discount is applied **on top of** dynamic pricing. During an early-bird window both discounts apply; after it, the late markup is applied first and then the group discount.
- The discount rounds down, so any fraction of a base unit is paid by the buyer. For example, 5 tickets at 999 is a gross of 4,995, a discount of 249 (4,995 × 500 / 10,000 = 249.75), and a net of 4,746.
- The buyer pays `net` in **one** transfer from `buyer` to `merchant_account`. No platform fee is taken.

### Parameters

| Parameter | Role |
|---|---|
| `event_id` | The event. A missing event panics with `InvoiceNotFound` (hub error 8), not `EventNotFound`. |
| `buyer` | Pays and must authorize. |
| `quantity` | Number of seats. `0` panics with `InvalidAmount` (7). |
| `shade_token` | The token contract the transfer is made in. |
| `merchant_account` | The address that receives `net`. |

> **Security:** `shade_token` and `merchant_account` are taken from the caller and never checked. The function does not compare `shade_token` with the event's `token` or the hub whitelist, and does not compare `merchant_account` with the event merchant's account. A buyer can pay in a worthless token, or pay themselves, and still increase `sold`. This must be fixed before the module is restored: read the token from `Event.token` and the destination from `merchant::get_merchant_account(env, event.merchant_id)`, and drop both parameters.

### What a bulk purchase records

Only `Event.sold` changes, increased by `quantity`. The function creates no `Ticket` records, and adds nothing to `get_event_tickets` or `get_user_tickets`. As a result:

- Bulk buyers hold nothing they can resell or present.
- `cancel_event_and_batch_refund` iterates the event's ticket list, so it never refunds a bulk purchase.
- No `TicketPurchasedEvent`, `PlatformFeeRoutedEvent` or analytics record is produced.

### Maximum quantity per call

There is no fixed per-call limit. The only checks are `quantity > 0` and `sold + quantity <= capacity`; going over capacity panics with `InvalidAmount` (7), not `EventSoldOut`. Because the function performs one transfer and one storage write whatever the quantity, its cost does not grow with `quantity` (see [Resource costs](../reference/resource-costs.md)). In practice the limit is the seats left, at most `u32::MAX`. `gross` saturates at `i128::MAX` instead of overflowing, but `gross × discount_bps` is not saturating and panics if a price and quantity are large enough to overflow `i128`.

## Recommendations

Because the hub has no ticketing today, the realistic way to sell paid tickets through Shade is to take the payment with the core gateway, using an [invoice](../../../docs/concepts/invoice-lifecycle.md) per order, and record the ticket in a standalone `ticketing` instance once the invoice is paid. That keeps fee routing, merchant accounts and analytics in the supported gateway, and check-in in the ticketing contract.

| Scenario | Recommended | Why |
|---|---|---|
| Small meetup, free or RSVP only | Standalone, one deployed instance | `issue_ticket` plus `check_in` is all you need, and there is no payment to collect. |
| Small paid meetup | Shade invoice per order, then standalone `issue_ticket` | Payment, fees and analytics stay in the gateway; the ticket gets a QR hash for the door. |
| Large concert with ticket classes | Standalone with tiers, one instance per event | `add_tier`, `issue_tiered_ticket`, `check_in` and waitlists exist only here. A fresh instance per event keeps the QR uniqueness scan short. Take payment through invoices and enforce the [gaps above](#gaps-in-the-standalone-contract) off-chain or fix them first. |
| Recurring events (a weekly series, a season) | Standalone, one instance per season or series | One instance holds many events, but issuance cost grows with every ticket in the instance, so start a new one periodically. |
| Early-bird pricing or group discounts | Neither ships it | Only the former hub module has them, and it does not build. Price the invoice off-chain, or restore and fix the module. |
| Resale with an organizer royalty | Standalone | `set_resale_config` and `resell_ticket` work. Note that `transfer_ticket` lets holders bypass the royalty. |

## Maintainers' intent

The recorded decisions are:

- **[ADR 0007](../../../docs/architecture/decisions/0007-core-gateway-scope.md)** limits `contracts/` to the payment gateway. Ticketing failed its test ("does a merchant need it to accept a payment, settle it, refund it, or see it on a dashboard?"), so the hub module moved to `example/` and the hub lost its ticketing entry points. Code in `example/` is reference material, may drift from the gateway, and is not a supported API.
- **[ADR 0005](../decisions/0005-separate-escrow-and-ticketing-factories.md)** keeps `ticketing_factory` as its own factory, separate from escrow, and rejects deploying ticket contracts from the hub.

Taken together: ticketing is outside the gateway, and hub ticketing is not coming back into `shade` under the current scope. ADR 0007 moved the standalone `ticketing` and `ticketing_factory` crates into `example/` as well, so both models now have the same status: reference implementations that nobody maintains as a product. No ADR chooses one model over the other, plans to merge them, or promotes either to a supported contract. A proposal to do any of those needs a new ADR.

There is no migration path between the models. They share no storage, and moving an event from one to the other means issuing its tickets again.

## Related pages

- [Event ticketing, dynamic pricing, and resale](event-ticketing.md): the former hub module in detail.
- [Ticketing contract reference](../contracts/ticketing.md): the standalone contract and factory.
- [Ticket tiers](ticket-tiers.md) and [Event waitlists](event-waitlists.md): standalone features in detail.
- [Ticket transfers, refunds, and event cancellation](ticket-transfers-and-refunds.md): standalone ticket ownership and refund rules.
- [Choosing a subscription model](choosing-a-subscription-model.md): the same comparison for subscriptions.
