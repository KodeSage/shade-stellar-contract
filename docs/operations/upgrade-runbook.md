# Runbook: Upgrading the `shade` contract

Replace the deployed `shade` contract's WASM with a new build while preserving its on-chain state. This is the highest-risk routine operation the protocol supports — read [Upgradeability](../architecture/upgradeability.md) in full before your first real upgrade; this runbook assumes you already understand *why* each check below exists, not just *that* it exists.

> **Warning:** An upgrade that lands a storage-incompatible change is not something you can undo by reinstalling the old WASM. If the new code has already written data in a shape the old code cannot decode, "rolling back" the executable does not roll back the data — see [Rollback](#rollback).

> **Note:** Only `shade` has an `upgrade` entry point.

## When to use this

- A reviewed, tested fix or feature for `shade` is ready to ship to a live deployment.
- An incident response has identified that the fix requires new contract code, not just a configuration change (configuration changes go through the [Admin Runbook](admin-runbook.md) instead).

Do **not** reach for this runbook for: fee changes, token whitelist changes, role grants, merchant status changes, or oracle reconfiguration — all of those are `set_*`/`grant_role` calls documented in the [Admin Runbook](admin-runbook.md) and do not touch the WASM at all.

## Preconditions

- You hold the current `shade` admin key — see [the upgrade mechanism](../architecture/upgradeability.md#the-upgrade-mechanism).
- `cargo build -p shade --release` succeeds against the commit you intend to ship. **Check this explicitly and do not assume it** — `main` has been broken at points during this repository's history (see [Deployment Runbook § Known-broken baseline](deployment.md#known-broken-baseline-verify-before-you-start)); an upgrade is not the moment to discover the crate doesn't compile.
- `cargo test -p shade` passes in full.

## Pre-flight

### Source and build

- [ ] The commit you're building from is the exact reviewed/merged commit — `git rev-parse HEAD` recorded before you build anything.
- [ ] Working tree is clean (`git status` shows nothing pending).
- [ ] `cargo build --target wasm32-unknown-unknown --release -p shade` succeeds.
- [ ] `stellar contract optimize --wasm target/wasm32-unknown-unknown/release/shade.wasm` succeeds; use the `.optimized.wasm` output, never the plain `.wasm`.
- [ ] Toolchain matches [Prerequisites](../getting-started/prerequisites.md) (Rust ≥ 1.84.0, Stellar CLI 23.x, `soroban-sdk` 23.5.3) — a build from a different toolchain version is not guaranteed to produce a byte-identical, independently-reproducible WASM (see [Upgradeability § Versioning](../architecture/upgradeability.md#versioning-and-identifying-a-deployed-build)).
- [ ] `cargo test --workspace --all-features` (or at minimum `cargo test -p shade`) is fully green.

### Storage compatibility

This is the step most likely to be skipped under time pressure and the one most likely to make an upgrade unrecoverable. Diff `contracts/shade/src/types.rs` between the currently-deployed commit and the new commit, and for every changed `#[contracttype]` item, check it against [Upgradeability's storage compatibility rules](../architecture/upgradeability.md#storage-compatibility-rules):

- [ ] No key-enum variant in `DataKey` was renamed, retyped, or moved to a different enum.
- [ ] No `#[repr(u32)]` integer enum variant (`InvoiceStatus`, `Role`, etc.) was renumbered. New variants, if any, took new, higher numbers.
- [ ] No field was added to, removed from, or renamed on a struct that is already persisted on the live network you're upgrading — or a migration ships in the same WASM (see [Migration patterns](../architecture/upgradeability.md#migration-patterns) and [Writing a storage migration](../contributing/storage-migrations.md)).
- [ ] If the `account` contract's code changed in this release: understand that upgrading `shade` does **not** upgrade any already-deployed `account` instance — `account` has no upgrade entry point at all, and even if it did, `shade` does not currently deploy accounts through its dead `account_factory` component (see [Deployment Runbook](deployment.md#what-actually-gets-deployed)). A merchant `account` fix requires a separate migration plan (new `account` deployment + `set_merchant_account`), independent of this runbook.

### Interface review

- [ ] A reviewed diff of every changed `ShadeTrait` method's signature, return type, and events exists (compare against `docs/reference/shade-interface.md` at the current deployed commit).
- [ ] Breaking changes (removed function, changed parameter types/order, changed return type) are explicitly listed and communicated — see [Communication plan](#communication-plan).
- [ ] `docs/reference/shade-interface.md` and `docs/reference/data-types.md` are updated in the same PR as the interface change, per [CONTRIBUTING.md](../../CONTRIBUTING.md#keeping-reference-docs-in-sync) — do not ship an interface change whose reference docs still describe the old interface.

### Testnet rehearsal

- [ ] The upgrade has been executed against a testnet deployment of `shade` carrying representative state (registered merchants, invoices in more than one status, at least one fee/token/oracle configuration) before this runbook is followed against mainnet.
- [ ] After the rehearsal upgrade, the following were confirmed readable and correct: an existing invoice, an existing merchant, fee configuration for at least one token, accepted-tokens list, oracle configuration (if used), role assignments, and analytics/counters (`get_merchant_volume`, `get_token_analytics`).
- [ ] A representative write (e.g. paying a pre-existing invoice) succeeded against the rehearsal contract after the upgrade.

> **Note:** This repository has no infrastructure for snapshotting or cloning production state onto testnet. If your team needs a rehearsal against a faithful copy of live state rather than freshly-seeded testnet data, that tooling does not exist yet — flag this as a gap for the team rather than skipping the rehearsal step entirely. `contracts/shade/src/tests/test_upgrade.rs::test_state_persists_after_upgrade` is the closest existing automated coverage; it upgrades a test contract instance and asserts `DataKey::Admin` and `DataKey::AcceptedTokens` survive, but it is a unit test with synthetic state, not a rehearsal against realistic data volume.

## Execution

Every command below targets one specific `shade` contract ID on one specific network. Confirm both explicitly before running anything — see the [Deployment Runbook's mainnet cautions](deployment.md#mainnet) for why this matters more than it sounds like it should.

### 1. Announce the maintenance window

Notify merchants/integrators before pausing — see [Communication plan](#communication-plan).

### 2. Pause the contract

```bash
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- pause --admin <admin-address>
```

Required signer: the current admin. `pause` panics with `ContractNotPaused` if already paused (pausing is not idempotent — check first if unsure).

### 3. Verify pause took effect

```bash
stellar contract invoke --id <shade-id> --network <network> -- is_paused
# true
```

Do not proceed on the assumption that step 2 "must have worked" — read the state back.

### 4. Install the new WASM

```bash
stellar contract install --wasm target/wasm32-unknown-unknown/release/shade.optimized.wasm --network <network> --source-account admin
```

> **Note:** On Stellar CLI 25.x, `stellar contract install` is a deprecated alias for `stellar contract upload`; both do the same thing as of this writing. Use whichever your pinned CLI version documents as current — check `stellar contract install --help` / `stellar contract upload --help` on your installed CLI.

This prints the new WASM hash. Record it immediately — this is the value you pass to `upgrade` next and the value you'll need for [rollback](#rollback) if something goes wrong.

### 5. Record the new WASM hash

Write it into your [deployment artifact record](deployment.md#artifact-recording) before proceeding, not after. If step 6 fails or produces an unexpected result, you need this hash on hand immediately, not reconstructed from scrollback.

### 6. Call `upgrade`

```bash
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- upgrade --new_wasm_hash <hash-from-step-4>
```

Required signer: the current admin (`upgrade` reads the admin from storage and requires that exact address's authorization — there is no separate `admin` parameter to the call).

> **Warning:** `upgrade` takes effect immediately and is not pausable-gated.

A successful response confirms the new WASM is live and can read existing storage — `get_admin` reading `DataKey::Admin` correctly is a minimal but real signal that the new code and the old storage are compatible for at least this one key. This does not by itself prove full storage compatibility — continue to the fuller checks in [Verification](#verification).

### 7. Run the storage migration (if this release has one)

Skip this step unless the release changes a stored type and ships a `migrate_*` entry point. Otherwise, run the migration now, while the contract is still paused, and repeat it until it reports completion:

```bash
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- migrate_invoices --admin <admin-address> --limit 50
stellar contract invoke --id <shade-id> --network <network> -- get_storage_version
```

The function name, arguments and the expected version come from the release; `migrate_invoices` is the [worked example](../contributing/storage-migrations.md#recording-and-checking-the-storage-version). Do not unpause until the storage version reads the value the release expects.

### 8. Verify critical state

See [Verification](#verification) below — run the full list, not just the `get_admin` check from step 6, before unpausing.

### 9. Unpause

```bash
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- unpause --admin <admin-address>
```

Only after every item in [Verification](#verification) has passed and [Unpause criteria](admin-runbook.md#unpause-criteria) are met.

### 10. Run the smoke test

Follow the [Deployment Runbook's smoke test](deployment.md#smoke-test) (register → create invoice → pay invoice → verify) against the now-upgraded, now-unpaused contract, using disposable test data on mainnet.

## Verification

After the upgrade and before unpausing, confirm at minimum:

| Check | Command |
|---|---|
| A known invoice reads correctly | `stellar contract invoke --id <shade-id> --network <network> -- get_invoice --invoice_id <known-id>` |
| A known merchant reads correctly | `stellar contract invoke --id <shade-id> --network <network> -- get_merchant --merchant_id <known-id>` |
| Fee configuration is unchanged | `stellar contract invoke --id <shade-id> --network <network> -- get_fee --token <token>` |
| Accepted tokens are unchanged | `stellar contract invoke --id <shade-id> --network <network> -- is_accepted_token --token <token>` |
| Oracle configuration is unchanged (if used) | `stellar contract invoke --id <shade-id> --network <network> -- get_token_oracle --token <token>` |
| Roles are unchanged | `stellar contract invoke --id <shade-id> --network <network> -- has_role --user <address> --role Manager` |
| Analytics/counters are consistent | `stellar contract invoke --id <shade-id> --network <network> -- get_merchant_volume --merchant <address> --token <token>` — compare against the pre-upgrade value you recorded before pausing |
| Storage version matches the release (if it shipped a migration) | `stellar contract invoke --id <shade-id> --network <network> -- get_storage_version` |
| Pause state is correct (still paused, pending unpause) | `stellar contract invoke --id <shade-id> --network <network> -- is_paused` → `true` |
| Deployment/build hash is recorded | Cross-check the hash from step 4 against your artifact log |
| Smoke test passes | See [step 10](#10-run-the-smoke-test) |

Any check that fails here means: do not unpause. Treat it as [incident response](admin-runbook.md#incident-response), not as "proceed and fix later" — the contract is paused specifically so that a bad upgrade doesn't process live payments before you've caught the problem.

## Rollback

### What "rollback" means here

Reinstalling the previous WASM hash and calling `upgrade` again:

```bash
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- upgrade --new_wasm_hash <previous-wasm-hash>
```

This is only meaningful if the previous WASM hash is known and was recorded (see [Deployment Runbook § Artifact recording](deployment.md#artifact-recording)) — you cannot "undo" an upgrade if you didn't keep the old hash.

### When this genuinely restores old behavior

If the new WASM made **no** persisted-storage changes (pure logic fix, no new/removed/renamed fields on a stored struct, no key-enum or integer-enum changes) — reinstalling the old WASM restores the exact prior behavior, because the storage was never touched incompatibly in either direction.

### When rollback does not restore old behavior — and reinstalling the old WASM can make things worse

If the new WASM wrote data in a new format before you rolled back — for example, a new field on a stored struct, or a new key-enum variant now populated for some records — reinstalling the old WASM does **not** convert that data back. The old code doesn't know the new field exists and either ignores it (data silently orphaned) or, if the storage layout changed incompatibly (a renamed/retyped field, not just an added one), the old code's read of that record traps, because the byte layout coming off the ledger no longer matches what the old code's deserializer expects (see [Upgradeability § How storage survives](../architecture/upgradeability.md#how-storage-survives)).

Concretely, "rollback is impossible" in this codebase means one of:

- **State written in a new incompatible format.** Any record created or modified by the new code between `upgrade` (step 6) and the rollback `upgrade` call, if it touches a struct/enum shape the old code can't parse.
- **A migration that already ran.** If the new WASM shipped with a migration ([Migration patterns](../architecture/upgradeability.md#migration-patterns)) and it already executed against some records, those records are now in the new shape permanently — the old WASM cannot re-migrate them backward, because no such reverse-migration code exists unless you specifically wrote one.
- **New authorization or storage assumptions.** If the new code changed what a stored value *means* (not just its shape) — e.g. a role's semantics, a status enum's implied invariants — reinstalling old code that doesn't share that assumption can produce silently wrong behavior rather than an outright trap.

### Mitigation, since prevention is the only real fix here

- **Testnet rehearsal is the primary control** (see [Pre-flight](#pre-flight)) — a rehearsal that actually exercises writes, not just reads, is what catches an incompatible change before it reaches a network where rollback matters.
- **Prefer additive changes.** New `DataKey` variants and new structs (see the "escape hatch" in [Upgradeability § Structs](../architecture/upgradeability.md#structs-the-field-name-set-is-the-identity)) over modifying anything already persisted.
- **If a migration is unavoidable, ship it in the same WASM as the change that needs it**, and design it to be idempotent and safe to run partially (a paused window that ends mid-migration should not corrupt state).
- **If rollback turns out to be impossible after the fact,** the remedy is a forward-fix: a new WASM that either completes the migration correctly or adds compatibility code to read both old and new shapes, not a second attempt at reinstalling the original WASM.
- **Record pre-upgrade state** (the [Verification](#verification) table's values, captured *before* step 6, not just after) specifically so that if you do end up forward-fixing rather than rolling back, you have a known-good baseline to reconcile against.

## Communication plan

- **Before pausing:** announce the maintenance window to merchants/integrators — expected start time, expected pause duration, and what will be unavailable (invoice creation, payments, and every other `assert_not_paused`-gated function — see [Pausable § Blocked vs allowed](../security/pausable.md#blocked-vs-allowed-while-paused); reads remain available throughout).
- **At pause:** send the "upgrade window has started" notification.
- **At completion (success):** send the "upgrade complete, unpaused, service restored" notification, including any breaking interface changes integrators need to handle.
- **At completion (rollback or failure):** send a distinct "upgrade did not complete as planned, contract remains paused / has been rolled back" notification — do not reuse the success template with words changed, since integrators watching for the "safe to resume" signal need this to be unambiguous.
- **If the upgrade cannot complete** (verification fails and rollback is not viable — see [When rollback does not restore old behavior](#when-rollback-does-not-restore-old-behavior--and-reinstalling-the-old-wasm-can-make-things-worse)): escalate per [Admin Runbook § Severity and escalation](admin-runbook.md#severity-and-escalation) — this situation is at minimum High severity (major payment disruption with the contract deliberately left paused) and likely Critical if any state was written in an incompatible format.

## Sign-off checklist

Before step 1 (announcing the window), the following roles confirm readiness:

- [ ] **Protocol engineer** — confirms the build, tests, and storage-compatibility review are complete and accurate.
- [ ] **Security reviewer** — confirms the interface diff and storage-compatibility analysis were independently checked, not only self-reviewed by the author.
- [ ] **Release operator** — confirms they will execute this runbook, have access to the admin key, and have the rollback plan and previous WASM hash on hand before starting.
- [ ] **Product/operations owner** — confirms the maintenance window and merchant communication plan are approved.
- [ ] **Incident/on-call owner** — confirms they are reachable during the window in case verification fails.

## Related pages

- [Upgradeability](../architecture/upgradeability.md) — the mechanism, storage rules, and versioning approach this runbook assumes.
- [Deployment Runbook](deployment.md) — the original deployment this runbook upgrades, including the known-broken-baseline caution that applies here too.
- [Admin Runbook](admin-runbook.md) — incident response and severity levels referenced above.
- [Pausable emergency-stop mechanism](../security/pausable.md) — exactly what is and isn't blocked during the pause window.
- [Admin ownership](../security/admin-and-ownership.md) — who can call `upgrade` and `pause`/`unpause`.

← [Back to Operations](README.md)
