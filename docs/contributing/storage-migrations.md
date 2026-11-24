# Writing a storage migration

This page shows you how to write the migration code that a `shade` upgrade needs when a stored type has to change shape: which changes need one, which pattern to use, a complete worked example that adds a field to `Invoice`, and how to test it against the WASM that is deployed today.

It is the how-to companion to two other pages. [Upgradeability](../architecture/upgradeability.md) explains how an upgrade replaces the code and leaves storage alone. The [Upgrade runbook](../operations/upgrade-runbook.md) is the procedure for shipping an upgrade to a live network. Read the upgradeability page first if you have not.

## When you need a migration

`upgrade` swaps the contract's code and leaves every stored value byte for byte as it was. The new code must decode those old bytes. When it cannot, every call that touches an old record fails, and only another upgrade can fix it.

### How each kind of type is stored

| Kind of type | Stored as | What identifies a value |
| --- | --- | --- |
| Storage key enum (`DataKey`) | A vector: variant name as a symbol, then the tuple arguments | Variant name, argument count and argument types |
| Unit enum without `#[repr(u32)]`, such as `Role` | A vector holding the variant name as a symbol | Variant name |
| Integer enum with `#[repr(u32)]`, such as `InvoiceStatus` and `InvoicePricingMode` | A bare `u32` | The number |
| Struct, such as `Invoice` and `Merchant` | A map from field-name symbols to values | The exact set of field names, and each field's type |

### Changes that are safe without a migration

| Change | Why it is safe |
| --- | --- |
| Add a new `DataKey` variant | No existing entry uses it. Old code never wrote it, so reads return `None` until new code writes it. |
| Store new data under a new key, keyed by an existing id (for example `InvoiceExternalRef(u64)`) | The existing record is untouched. Treat a missing entry as "not set". |
| Reorder variants of a key enum or a unit enum | Position is not stored; the name is. |
| Append a variant with a new, higher number to a `#[repr(u32)]` enum | Old values keep their numbers. Old code cannot decode the new number, which matters only if you roll back. |
| Rename a variant of a `#[repr(u32)]` enum | Only the number is stored. |
| Reorder fields or change doc comments on a struct | Fields are stored by name. |
| Change a struct that is only returned from a query and never stored, such as `InvoiceFilter` or `InvoicePage` | Nothing on the ledger has its shape. |

### Changes that need a migration

| Change | What happens to existing data |
| --- | --- |
| Add a field to a stored struct, **including an `Option` field** | The old map lacks the new key. Every read of an old record traps. |
| Remove or rename a field of a stored struct | The old map has a key the new struct does not expect. Every read traps. |
| Change a field's type (`u64` → `i128`, `Address` → `Option<Address>`) | The stored value no longer converts. Every read traps. |
| Rename a key enum variant, change its arguments, or move it to another enum | Old entries are stored under a key the new code never builds. They are orphaned: still on the ledger and unreadable. |
| Move a key between persistent, instance and temporary storage | The entry stays where it was written; the new code looks elsewhere. |
| Rename a variant of a unit enum such as `Role` | The stored symbol no longer matches a variant. Every read traps. |
| Renumber a `#[repr(u32)]` variant, or reuse a retired number | Old values silently decode as a different variant. Nothing traps, so it is the most dangerous change on this list. |

> **Warning:** "New optional data" is safe only as a **new key**. An `Option<T>` field added to an existing struct is still a breaking change: the stored map has no entry for it at all, and the SDK does not fill in `None`.

Before you write a migration, check whether a new key does the job. Most "I need one more field on `Merchant`" changes are simpler as a new `DataKey` variant keyed by the same id, which needs no migration at all.

## Why you cannot catch a failed decode

The obvious migration is "try to read the new shape, and if that fails read the old one". It does not work. When a stored value does not match the struct you ask for:

- `env.storage().persistent().get::<_, Invoice>(&key)` traps. It does not return `None`.
- `Invoice::try_from_val(&env, &raw_val)` also traps inside a deployed contract, because the host function that unpacks the map fails and the host turns that into a trap before your code sees an `Err`. In native unit tests the same call can return `Err` instead, so a native-only test can pass for code that traps on chain.

So a migrating reader must decide which shape a record has **before** it decodes it. The patterns below do that either with a separate key or by checking the stored map for a field name.

## Migration patterns

All three patterns are ordinary contract code. Soroban has no built-in migration framework.

### Versioned keys

Write records in the new shape under a new key variant and leave the old key for old records:

```rust
pub fn load_invoice(env: &Env, id: u64) -> Option<Invoice> {
    let s = env.storage().persistent();
    if let Some(invoice) = s.get(&DataKey::InvoiceV2(id)) {
        return Some(invoice);
    }
    let v1: InvoiceV1 = s.get(&DataKey::Invoice(id))?;
    Some(v1.into())
}
```

The key tells you the shape, so no inspection is needed. Every writer must write `InvoiceV2(id)`, and should `remove` the old `Invoice(id)` entry when it does so that the record is not stored twice.

### Lazy migration on read

Keep the same key. Check which shape is stored, convert old records in memory, and let the record reach the new shape the next time it is written. The [worked example](#worked-example-add-a-field-to-invoice) uses this pattern. It detects the shape by checking for a field that only the new struct has. Reading the value as `Map<Symbol, Val>` works for a struct of any shape:

```rust
let raw: Map<Symbol, Val> = storage.get(&key)?;
if raw.contains_key(Symbol::new(env, "external_ref")) {
    return storage.get(&key); // already v2
}
```

A variation writes the converted record back during the read. It migrates records sooner, but it makes every getter a write, which costs more and breaks the guarantee in `contracts/shade/src/components/search.rs` that search reads have no side effects. Convert in memory unless you have a reason not to.

### Admin-only one-shot migration

Add an admin entry point that rewrites a bounded batch of records per call and records how far it got, so that it can be called repeatedly until done and can never skip or repeat a range. Use it on its own when the new shape must be in place before any user traffic, or together with lazy migration so that old records do not stay in the old shape indefinitely.

### Choosing a pattern

| | Versioned keys | Lazy on read | One-shot migrate |
| --- | --- | --- | --- |
| Reader complexity | Two key lookups | One shape check | None once finished |
| Cost to users | An extra read for old records | A map read for every record | None; the admin pays |
| Old shape kept in source | Until every old key is gone | Until every record is rewritten | Until the migration finishes |
| Works for large datasets | Yes | Yes | Yes, in batches |
| Every record migrated at a known point | No | No | Yes |
| Fits a renamed or moved key | Yes | No: same key only | Yes |

The recommended default for a changed struct is **lazy migration on read plus a one-shot `migrate` function**. Lazy reads keep the contract working from the moment of the upgrade. The `migrate` calls finish the job, after which the storage version records that every record is in the new shape.

## Worked example: add a field to `Invoice`

This example adds `external_ref: Option<String>`, an integrator's own order reference, to `Invoice`. It is not merged into `shade`. It was applied to the repository and checked with `cargo test --workspace --all-features`, `cargo clippy --workspace --all-features -- -D warnings` and a `wasm32v1-none` release build, so the code below compiles and its tests pass against the current source.

### 1. Freeze the old struct

In `contracts/shade/src/types.rs`, copy `Invoice` exactly as it is today into a new struct named `InvoiceV1`. This is the decoder for records already on the ledger, so never edit it afterwards:

```rust
/// `Invoice` as storage version 1 wrote it. Frozen: never edit this struct.
#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct InvoiceV1 {
    pub id: u64,
    pub description: String,
    pub amount: i128,
    pub token: Address,
    pub status: InvoiceStatus,
    pub merchant_id: u64,
    pub payer: Option<Address>,
    pub date_created: u64,
    pub date_paid: Option<u64>,
    pub amount_paid: i128,
    pub amount_refunded: i128,
    pub expires_at: Option<u64>,
    pub pricing_mode: InvoicePricingMode,
    pub fiat_pricing: FiatPricingData,
}
```

### 2. Change the struct and write the conversion

Add the field to `Invoice`, and write a `From` conversion that carries every old field over and chooses the value for the new one:

```rust
pub struct Invoice {
    // … the 14 existing fields, unchanged …
    pub fiat_pricing: FiatPricingData,
    /// Integrator's own order reference. Added in storage version 2.
    pub external_ref: Option<String>,
}

impl From<InvoiceV1> for Invoice {
    fn from(v1: InvoiceV1) -> Self {
        Invoice {
            id: v1.id,
            description: v1.description,
            amount: v1.amount,
            token: v1.token,
            status: v1.status,
            merchant_id: v1.merchant_id,
            payer: v1.payer,
            date_created: v1.date_created,
            date_paid: v1.date_paid,
            amount_paid: v1.amount_paid,
            amount_refunded: v1.amount_refunded,
            expires_at: v1.expires_at,
            pricing_mode: v1.pricing_mode,
            fiat_pricing: v1.fiat_pricing,
            external_ref: None,
        }
    }
}
```

The compiler then points you at every `Invoice { … }` literal that needs the new field. In this example that is the four constructors in `contracts/shade/src/components/invoice.rs` (`create_invoice`, `create_fiat_invoice`, `create_invoice_draft` and `create_invoice_signed`), each of which gains `external_ref: None,`.

### 3. Add the version keys

Add two variants to `DataKey`. New variants are always safe:

```rust
    // --- Storage versioning ---
    /// Storage layout version. Absent means version 1.
    StorageVersion,
    /// Highest invoice id that `migrate_invoices` has rewritten.
    InvoiceMigrationCursor,
```

`DataKey` has a 50-variant cap. Check how many variants it has before adding more, and if it is close, start a new key enum for the new keys instead.

### 4. Write the migration component

Create `contracts/shade/src/components/migration.rs` and add `pub mod migration;` to `contracts/shade/src/components/mod.rs`:

```rust
//! Storage layout versioning and the `Invoice` v1 → v2 migration.

use crate::components::core;
use crate::types::{DataKey, Invoice, InvoiceV1};
use soroban_sdk::{Env, Map, Symbol, Val};

/// The storage layout this WASM reads and writes.
pub const CURRENT_STORAGE_VERSION: u32 = 2;

/// Returns the stored layout version. A contract that never stored one was
/// written by code older than versioning, which is version 1.
pub fn get_storage_version(env: &Env) -> u32 {
    env.storage()
        .persistent()
        .get(&DataKey::StorageVersion)
        .unwrap_or(1)
}

pub fn set_storage_version(env: &Env, version: u32) {
    env.storage()
        .persistent()
        .set(&DataKey::StorageVersion, &version);
}

/// Reads an invoice in either layout and returns it in the current one.
///
/// Decoding a stored value into the wrong struct traps the whole call, so
/// the layout is detected first by looking for a field only v2 has. The
/// converted invoice is not written back here: every write path stores the
/// full `Invoice`, and `migrate_invoices` rewrites the rest.
pub fn load_invoice(env: &Env, invoice_id: u64) -> Option<Invoice> {
    let key = DataKey::Invoice(invoice_id);
    let storage = env.storage().persistent();
    let raw: Map<Symbol, Val> = storage.get(&key)?;
    if raw.contains_key(Symbol::new(env, "external_ref")) {
        return storage.get(&key);
    }
    let v1: InvoiceV1 = storage.get(&key)?;
    Some(v1.into())
}

/// Rewrites up to `limit` invoices, continuing from where the previous call
/// stopped, and returns the highest invoice id migrated so far. Once every
/// invoice is in the v2 layout, the storage version becomes 2.
pub fn migrate_invoices(env: &Env, admin: &soroban_sdk::Address, limit: u32) -> u64 {
    core::assert_admin(env, admin);

    let storage = env.storage().persistent();
    let invoice_count: u64 = storage.get(&DataKey::InvoiceCount).unwrap_or(0);
    let mut cursor: u64 = storage
        .get(&DataKey::InvoiceMigrationCursor)
        .unwrap_or(0);
    let end = invoice_count.min(cursor.saturating_add(limit as u64));

    while cursor < end {
        cursor += 1;
        if let Some(invoice) = load_invoice(env, cursor) {
            storage.set(&DataKey::Invoice(cursor), &invoice);
        }
    }

    storage.set(&DataKey::InvoiceMigrationCursor, &cursor);
    if cursor >= invoice_count {
        set_storage_version(env, CURRENT_STORAGE_VERSION);
    }
    cursor
}
```

The stored cursor is what makes `migrate_invoices` safe to call repeatedly. A call never revisits a finished range, and once the cursor reaches `InvoiceCount` further calls do nothing and return the same value.

### 5. Route every read through the loader

Every place that decodes an `Invoice` from storage must call `load_invoice`. A single direct `get::<_, Invoice>` left behind traps on the first old record it meets. List them:

```bash
grep -rn "DataKey::Invoice(" contracts/shade/src --include='*.rs' | grep -v /tests/ | grep -v "\.set("
```

In this example that is four sites:

| File | Function |
| --- | --- |
| `contracts/shade/src/components/invoice.rs` | `get_invoice` |
| `contracts/shade/src/components/invoice.rs` | `get_invoices` |
| `contracts/shade/src/components/search.rs` | `search_invoices_paginated`, twice |

Each one becomes a call to the loader. For example, `get_invoice`:

```rust
pub fn get_invoice(env: &Env, invoice_id: u64) -> Invoice {
    migration::load_invoice(env, invoice_id)
        .unwrap_or_else(|| panic_with_error!(env, ContractError::InvoiceNotFound))
}
```

and the loop in `get_invoices`:

```rust
    for i in 1..=invoice_count {
        if let Some(invoice) = migration::load_invoice(env, i) {
```

Writes need no change. Every write path stores a complete `Invoice`, so any invoice that is written after the upgrade is in the new shape.

### 6. Record the version on fresh deployments

A contract deployed from this WASM has nothing to migrate. In `core::initialize` in `contracts/shade/src/components/core.rs`, after `ContractInfo` is stored, add:

```rust
    // A fresh deployment starts on the current layout; nothing to migrate.
    crate::components::migration::set_storage_version(
        env,
        crate::components::migration::CURRENT_STORAGE_VERSION,
    );
```

A contract that was initialized by older code has no `StorageVersion` entry, and `get_storage_version` reports it as version 1.

### 7. Expose the entry points

Add the two functions to `ShadeTrait` in `contracts/shade/src/shade_interface.rs`:

```rust
    fn get_storage_version(env: Env) -> u32;
    fn migrate_invoices(env: Env, admin: Address, limit: u32) -> u64;
```

and implement them in `contracts/shade/src/shade.rs`, importing the component as `migration as migration_component`:

```rust
    fn get_storage_version(env: Env) -> u32 {
        migration_component::get_storage_version(&env)
    }

    // Not pausable: the upgrade runbook migrates while the contract is paused.
    fn migrate_invoices(env: Env, admin: Address, limit: u32) -> u64 {
        migration_component::migrate_invoices(&env, &admin, limit)
    }
```

Leave `migrate_invoices` without `pausable_component::assert_not_paused`, like `upgrade`. The [Upgrade runbook](../operations/upgrade-runbook.md) keeps the contract paused from the upgrade until verification passes, and the migration has to run inside that window.

Update [`docs/reference/shade-interface.md`](../reference/shade-interface.md) and [`docs/reference/data-types.md`](../reference/data-types.md) in the same PR, as for any interface change.

## Testing the migration

The test has to start from real pre-upgrade state, which means state written by the WASM that is deployed today, not by the code in your branch.

### 1. Commit the deployed WASM as a fixture

Build `shade` from the commit that is deployed on the network you are upgrading, and commit it next to the existing fixtures. A worktree keeps your branch untouched:

```bash
git worktree add ../shade-deployed <deployed-tag>
(cd ../shade-deployed && cargo build -p shade --target wasm32v1-none --release)
cp ../shade-deployed/target/wasm32v1-none/release/shade.wasm contracts/shade/src/tests/fixtures/shade_v1.wasm
git worktree remove ../shade-deployed
```

Replace `<deployed-tag>` with the tag or commit recorded in your deployment artifact record (see [Upgradeability § Versioning](../architecture/upgradeability.md#versioning-and-identifying-a-deployed-build)).

`.gitignore` ignores `*.wasm` except for named fixtures, so add an exception for the new file next to the existing ones:

```text
!contracts/shade/src/tests/fixtures/shade_v1.wasm
```

### 2. Write state with the old WASM, then switch to the new code

Create `contracts/shade/src/tests/test_storage_migration.rs` and add `pub mod test_storage_migration;` to `contracts/shade/src/tests/mod.rs`. The helper deploys the v1 WASM, creates invoices through it, and then puts the current source in its place:

```rust
#![cfg(test)]
//! Upgrades a contract holding v1 invoices to the current code.

use crate::components::migration::CURRENT_STORAGE_VERSION;
use crate::shade::{Shade, ShadeClient};
use crate::types::{DataKey, Invoice, InvoiceFilter};
use soroban_sdk::testutils::Address as _;
use soroban_sdk::{Address, Env, String};

/// The `shade` WASM that is deployed today, built from the release tag.
mod shade_v1 {
    soroban_sdk::contractimport!(file = "src/tests/fixtures/shade_v1.wasm");
}

/// Deploys the v1 WASM, creates `count` invoices through it, then swaps the
/// contract's code for the current source, keeping its storage.
fn upgraded_with_v1_invoices(count: u64) -> (Env, ShadeClient<'static>, Address) {
    let env = Env::default();
    env.mock_all_auths();
    // Instantiating the full shade WASM exceeds the default test budget.
    env.cost_estimate().budget().reset_unlimited();

    let contract_id = env.register(shade_v1::WASM, ());
    let v1 = shade_v1::Client::new(&env, &contract_id);
    let admin = Address::generate(&env);
    v1.initialize(&admin);

    let merchant = Address::generate(&env);
    v1.register_merchant(&merchant);
    let token = env
        .register_stellar_asset_contract_v2(Address::generate(&env))
        .address();
    v1.add_accepted_token(&admin, &token);
    for i in 1..=count {
        let description = String::from_str(&env, "v1 invoice");
        v1.create_invoice(&merchant, &description, &(1_000 * i as i128), &token, &None);
    }

    // The "upgrade": same address, same storage, new code.
    env.register_at(&contract_id, Shade, ());
    let client = ShadeClient::new(&env, &contract_id);
    (env, client, admin)
}
```

Three details in this helper matter:

- **`contractimport!`** generates `shade_v1::Client` and `shade_v1::WASM` from the fixture, so the state is written by the deployed code through its real entry points, not by hand.
- **`reset_unlimited()`** is required. Instantiating the full `shade` WASM exceeds the test environment's default CPU budget, and the test fails with `Error(Budget, ExceededLimit)` before it writes anything.
- **`env.register_at(&contract_id, Shade, ())`** replaces the contract's code at the same address and keeps its storage, which is what `upgrade` does on chain. The new code runs natively, so you get backtraces and coverage. The `upgrade` entry point itself is already covered by `contracts/shade/src/tests/test_upgrade.rs`.

### 3. Assert on reads

Write one test for each behavior the migration promises:

```rust
#[test]
fn v1_invoices_read_through_new_code() {
    let (_env, client, _admin) = upgraded_with_v1_invoices(2);

    assert_eq!(client.get_storage_version(), 1);
    let invoice = client.get_invoice(&2);
    assert_eq!(invoice.amount, 2_000);
    assert_eq!(invoice.external_ref, None);
}

#[test]
#[should_panic]
fn typed_read_of_v1_invoice_traps() {
    let (env, client, _admin) = upgraded_with_v1_invoices(1);

    env.as_contract(&client.address, || {
        let _: Option<Invoice> = env.storage().persistent().get(&DataKey::Invoice(1));
    });
}

#[test]
fn migrate_invoices_rewrites_in_batches_then_bumps_version() {
    let (env, client, admin) = upgraded_with_v1_invoices(3);

    assert_eq!(client.migrate_invoices(&admin, &2), 2);
    assert_eq!(client.get_storage_version(), 1);

    assert_eq!(client.migrate_invoices(&admin, &2), 3);
    assert_eq!(client.get_storage_version(), CURRENT_STORAGE_VERSION);

    // Every record now decodes as the current struct directly.
    env.as_contract(&client.address, || {
        for id in 1..=3u64 {
            let invoice: Invoice = env
                .storage()
                .persistent()
                .get(&DataKey::Invoice(id))
                .unwrap();
            assert_eq!(invoice.amount, 1_000 * id as i128);
        }
    });

    // Calling again after completion is a no-op.
    assert_eq!(client.migrate_invoices(&admin, &2), 3);
}

#[test]
fn list_reads_handle_mixed_layouts() {
    let (_env, client, admin) = upgraded_with_v1_invoices(3);

    // Invoice 1 is rewritten as v2; invoices 2 and 3 are still v1.
    client.migrate_invoices(&admin, &1);

    let all = client.get_invoices(&InvoiceFilter {
        status: None,
        merchant: None,
        min_amount: None,
        max_amount: None,
        start_date: None,
        end_date: None,
    });
    assert_eq!(all.len(), 3);
}

#[test]
fn fresh_deployment_starts_on_current_version() {
    let env = Env::default();
    env.mock_all_auths();
    let client = ShadeClient::new(&env, &env.register(Shade, ()));
    client.initialize(&Address::generate(&env));

    assert_eq!(client.get_storage_version(), CURRENT_STORAGE_VERSION);
}
```

`typed_read_of_v1_invoice_traps` is deliberately a failing read. It proves that the fixture really holds the old shape: if it stops panicking, the fixture no longer matches what is deployed, and the other tests are no longer testing a migration.

`list_reads_handle_mixed_layouts` covers the state the contract is in between the upgrade and the end of the migration, when some records have each shape. It is the test that catches a loop that still decodes `Invoice` directly.

Run them, then the whole suite:

```bash
cargo test -p shade test_storage_migration
cargo test --workspace --all-features
```

### Optional: a full WASM-to-WASM rehearsal

`register_at` runs the new code natively. To run exactly the bytes you will install, build the new `shade` WASM and call the real `upgrade` entry point in a test:

```rust
let new_hash = env.deployer().upload_contract_wasm(NEW_SHADE_WASM);
v1.upgrade(&new_hash);
let client = ShadeClient::new(&env, &contract_id); // calls now run the uploaded WASM
```

Here `NEW_SHADE_WASM` is `include_bytes!` of the release build from your branch. Because that file only exists after a build, keep this test out of the committed suite and run it as part of the [testnet rehearsal](../operations/upgrade-runbook.md#testnet-rehearsal).

## Recording and checking the storage version

`get_storage_version` is how anyone confirms, after an upgrade, which layout a deployed contract's storage is in:

| Value | Meaning |
| --- | --- |
| `1` | No version was ever stored. The contract was initialized by code older than versioning and has not been fully migrated. |
| `2` | Every invoice is in the v2 layout: the contract was initialized by v2 code, or `migrate_invoices` reached the last invoice. |

Bump `CURRENT_STORAGE_VERSION` once per release that changes the storage layout, never per release.

During the upgrade, run the migration after `upgrade` and before unpausing:

```bash
stellar contract invoke --id <shade-id> --network <network> -- get_storage_version
# 1

stellar contract invoke --id <shade-id> --network <network> --source-account admin -- migrate_invoices --admin <admin-address> --limit 50

stellar contract invoke --id <shade-id> --network <network> -- get_storage_version
# 2
```

Replace `<shade-id>`, `<network>` and `<admin-address>` with the values for the deployment you are upgrading. Repeat the `migrate_invoices` call until it returns the current invoice count and `get_storage_version` returns `2`.

Choose `--limit` so that one call fits within the network's per-transaction limits. Each invoice is one read and one write. `stellar contract invoke` simulates the call before submitting it, so a batch that is too large fails in simulation and costs nothing; lower the limit and try again.

## Removing the migration later

`InvoiceV1`, the shape check in `load_invoice` and `migrate_invoices` must stay in the code for as long as any deployment might still hold v1 records. Once `get_storage_version` returns `2` on every network, a later upgrade can make `load_invoice` a plain `get`, delete `InvoiceV1` and `migrate_invoices`, and delete the `shade_v1.wasm` fixture and its tests. Keep `DataKey::StorageVersion` and `get_storage_version` for the next migration.

## Rollback

A migration makes rolling back harder. Once any record is in the v2 shape, the v1 WASM traps when it reads that record, so reinstalling the old hash does not restore service. Plan the rollback before the upgrade, as the [Upgrade runbook § Rollback](../operations/upgrade-runbook.md#rollback) describes: a fix-forward upgrade is usually the only safe option after `migrate_invoices` has run.

## Checklist

- [ ] Every storage change is classified against [the tables above](#when-you-need-a-migration), and a new key was considered first.
- [ ] The old struct is frozen as `<Type>V1`, with a `From` conversion to the new struct.
- [ ] Every read of the changed type goes through one loader, including list and search loops.
- [ ] The loader detects the shape before decoding; it never relies on a failed decode.
- [ ] `migrate_*` is admin-only, works in bounded batches, keeps a cursor and sets the storage version when it finishes.
- [ ] `initialize` records the current storage version.
- [ ] The deployed WASM is committed as a fixture and the tests write state through it.
- [ ] Tests cover a single old record, mixed layouts in a list read, a full batched migration and a fresh deployment.
- [ ] The migration ships in the same WASM as the change that needs it.
- [ ] The interface reference docs are updated, and the runbook steps for `migrate_*` are in the release plan.
