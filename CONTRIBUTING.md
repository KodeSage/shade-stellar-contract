# Contributing to Shade Stellar Contract

Thank you for your interest in contributing to this project! This document takes you from picking up a numbered issue to getting a PR merged, end to end, so you shouldn't need to ask a maintainer what to do next at any step along the way.

## Finding and claiming an issue

Almost all work in this repository starts from a numbered GitHub issue — look at merged PR titles and branch names in the repository history and you'll see the pattern (`docs/465-466-467-468-ops-runbooks`, `feat/campaign-kyc-verification-system-324`, and so on): a contributor picked up one or more open, unassigned issues and referenced their numbers in the branch name and PR.

1. Browse the repository's [Issues tab](https://github.com/ShadeProtocol/shade-stellar-contract/issues) for an open issue with no assignee and no linked in-progress PR.
2. Comment on the issue to claim it before you start writing code — this avoids two people duplicating the same work. A short "I'd like to work on this" is enough.
3. **Expected turnaround:** once you've claimed an issue, aim to open a draft PR (or at least push a branch with visible progress) within **7 days**. If an issue has been claimed for more than 7 days with no activity, maintainers may unassign it and let someone else pick it up — this isn't a penalty, it just keeps issues from sitting blocked behind an inactive claim. If you need more time, say so in a comment on the issue before the window closes.
4. If you want to work on something that doesn't have an open issue yet (a new feature, a larger refactor), open an issue describing it first and wait for a maintainer to weigh in before investing significant time — this avoids spending effort on something that won't be merged as proposed.

## Getting Started

1. Fork the repository
2. Clone your fork: `git clone https://github.com/ShadeProtocol/shade-stellar-contract.git`
3. Create a new branch from `contrib` following the [branch naming convention](#branch-naming) below, e.g. `git checkout -b feat/your-feature-name-123 origin/contrib` (where `123` is the issue number)
4. Make your changes
5. Run the [pre-PR checklist](#pre-pr-checklist)
6. Commit your changes (see [Commit Guidelines](#commit-guidelines))
7. Push to your fork: `git push origin feat/your-feature-name-123`
8. Open a Pull Request **against the `contrib` branch** using the [PR template](#pull-request-process), linking the issue you claimed. A PR against any other branch is closed automatically unless you are a repository admin.

## Branch naming

Name branches `<type>/<short-description>`, optionally suffixed with the issue number(s) the branch closes, matching this repository's actual history:

| Type | Use for | Example |
|---|---|---|
| `feat/` | A new feature or capability | `feat/campaign-kyc-verification-system-324` |
| `fix/` | A bug fix | `fix/invoice-rounding-error-212` |
| `docs/` | Documentation-only changes | `docs/465-466-467-468-ops-runbooks` |
| `refactor/` | Code restructuring with no behavior change | `refactor/platform-fee-routing` |
| `test/` | Adding or updating tests only | `test/invoice-expiry-coverage` |
| `chore/` | Tooling, CI, formatting, maintenance | `chore/update-soroban-sdk` |

These prefixes mirror the [commit type](#commit-guidelines) you'll use most in that branch's commits — a `docs/` branch should contain almost entirely `docs:` commits, and so on. When a branch addresses multiple numbered issues at once (common for a batch of related docs pages), list every issue number, hyphen-separated, as this repository's own history does.

## What belongs in this repository

`contracts/` holds only the payment gateway: the `shade` hub and the per-merchant `account` contract. Before adding to `shade`, ask whether a merchant needs it to accept a payment, settle it, refund it, or see it on a dashboard. If not, it belongs in `example/`, a separate workspace that CI does not run against. See [ADR 0007](docs/architecture/decisions/0007-core-gateway-scope.md).

## Development Setup

### Prerequisites

- Rust (stable toolchain)
- Cargo
- `wasm32v1-none` target: `rustup target add wasm32v1-none`
- Stellar CLI (if needed for contract deployment)

### Building the Project

```bash
# Build all contracts
cargo build --workspace --target wasm32v1-none --release

# Build a specific contract
cargo build -p shade --target wasm32v1-none --release
```

### Running Tests

```bash
# Run all tests
cargo test --workspace --all-features

# Run tests for a specific contract
cargo test --manifest-path contracts/shade/Cargo.toml
```

### Code Formatting

```bash
# Format all code
cargo fmt --all

# Check formatting
cargo fmt --all -- --check
```

### Linting

```bash
# Run clippy
cargo clippy --workspace --all-features -- -D warnings
```

### Using the Makefile

The [`Makefile`](Makefile) wraps the most common commands above:

```bash
make fmt    # cargo fmt --all
make lint   # cargo clippy --workspace --all-features -- -D warnings
make test   # cargo test --workspace --all-features
make build  # cargo build --workspace --target wasm32v1-none --release
make fixtures  # rebuild the account WASM that shade's tests deploy
```

These are exactly the commands CI runs in [`.github/workflows/ci.yml`](.github/workflows/ci.yml) — running them locally before pushing means CI should not surprise you.

## Pre-commit Hooks

This project uses pre-commit hooks to ensure code quality. Install and set them up:

```bash
# Install pre-commit (requires Python)
pip install pre-commit

# Install the hooks
pre-commit install

# Run hooks manually
pre-commit run --all-files
```

The hooks will automatically:

- Validate YAML files
- Format Rust code
- Run clippy checks
- Check for merge conflicts

## Commit Guidelines

### Commit Message Format

We follow conventional commit format:

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types

- `feat`: A new feature
- `fix`: A bug fix
- `docs`: Documentation only changes
- `style`: Code style changes (formatting, missing semicolons, etc.)
- `refactor`: Code refactoring
- `test`: Adding or updating tests
- `chore`: Maintenance tasks

### Examples

```
feat(contract): add new token transfer function

fix(test): correct assertion in hello_world test

docs: update README with installation instructions
```

## Pre-PR Checklist

Run this checklist locally before opening (or re-requesting review on) a PR — it matches what CI actually enforces, so a clean run here means CI should pass too:

- [ ] `make fmt` (or `cargo fmt --all -- --check` to verify without rewriting) — no formatting diffs.
- [ ] `make lint` (`cargo clippy --workspace --all-features -- -D warnings`) — zero warnings.
- [ ] `make test` (`cargo test --workspace --all-features`) — full workspace suite passes, not just the crate you touched.
- [ ] **Snapshot check:** `cargo test` generates `test_snapshots/` directories as a side effect (see [Running the test suite § Test snapshots](docs/getting-started/running-tests.md#test-snapshots)). This repository does **not** commit these — confirm `git status` shows no `test_snapshots/` path before staging anything. If one slipped into a previous commit, remove it rather than leaving it in the PR.
- [ ] **Docs checks:** if the PR adds or changes a page under `docs/`, it satisfies the [Documentation Style Guide's review checklist](docs/contributing/documentation-style-guide.md#review-checklist) (accuracy against code, working links, spell-check, correct template, terminology, style, and index updated). If the PR changes a public interface, it also satisfies [Keeping reference docs in sync](#keeping-reference-docs-in-sync) below.
- [ ] **Changelog:** if the PR changes anything an integrator or operator could notice, add an entry under `## [Unreleased]` in [`CHANGELOG.md`](CHANGELOG.md), and prefix it `**BREAKING:**` if it matches the [breaking-change criteria](docs/operations/releases.md#breaking-changes-for-integrators).
- [ ] Commit messages follow the [Commit Guidelines](#commit-guidelines).

## Pull Request Process

1. **Run the [Pre-PR Checklist](#pre-pr-checklist)** above.
2. **Fill out the [PR template](.github/pull_request_template.md)** — GitHub pre-fills it automatically when you open the PR; complete every section (summary, linked issue, test evidence, and documentation updates) rather than deleting the ones that feel optional.
3. **Link the issue** you claimed (`Fixes #123`, or from the PR's Development sidebar). A workflow closes it automatically when the PR is merged into `contrib`.
4. **Request review** from a maintainer.
5. **Respond to feedback** — see [Review expectations](#review-expectations) below for what happens next.

## Review expectations

- **Approvals required:** at least one maintainer approval before merge. A PR touching contract logic in `contracts/*/src/` that moves funds or changes authorization (payments, refunds, withdrawals) should get a second, independent look before merge — the same "not just self-reviewed by the author" principle the [Upgrade Runbook](docs/operations/upgrade-runbook.md) applies to upgrades.
- **Response times:** maintainers aim to leave an initial review (approval, requested changes, or at least a comment) within a few business days of a PR being marked ready for review. If you haven't heard anything after a week, a polite ping on the PR is welcome — it's not pushy, it's often just what was needed to surface a stalled review queue.
- **Handling requested changes:** push additional commits to the same branch rather than force-pushing over history mid-review — reviewers may have left line comments anchored to specific commits, and a force-push can orphan those threads. Once a reviewer's comments are addressed, reply on each thread (or resolve it, if your review tooling allows) so the reviewer can tell at a glance what's been handled versus what's still open, then re-request review.
- **Merge method:** PRs are merged via GitHub's standard merge commit (see the `Merge pull request #NNN from ...` commits in this repository's history) — you don't need to squash or rebase your branch yourself before merge unless a maintainer asks for it to clean up a messy commit history.

## Documentation Contributions

If your PR adds or changes a page under `docs/`, follow the [Documentation Style Guide](docs/contributing/documentation-style-guide.md) and use the matching template from [`docs/contributing/templates/`](docs/contributing/templates/). Every docs PR must satisfy the [review checklist](docs/contributing/documentation-style-guide.md#review-checklist) before it can be merged.

For a deeper understanding of how the pieces you're documenting fit together before you write about them, read the [Architecture overview](docs/architecture/overview.md) and the [testing guide](docs/getting-started/running-tests.md) — a docs PR that misdescribes a cross-contract call or a test convention is harder to catch in review than one that's simply incomplete.

### Architecture Decision Records

A PR that makes a significant design decision (new or restructured contract, factory, or component; storage, upgrade, or cross-contract call changes; privileged admin/fee flows; new dependencies; or reversing a past decision) must add an ADR under [`docs/architecture/decisions/`](docs/architecture/decisions/README.md) using the [template](docs/architecture/decisions/template.md), and add it to the index.

### Docs site and quality checks

Preview the docs site locally with `make docs-serve` (or `mdbook serve docs`) and build it with `make docs`. Docs PRs run link checks in CI; see [Docs tooling](docs/contributing/docs-tooling.md) for how to run them locally.

### Keeping reference docs in sync

A PR changing a **public interface** must update the matching reference docs in the same PR — "public interface" means any of the following:

- `ShadeTrait` (adds, removes, or renames a function) → update `docs/reference/shade-interface.md`.
- A public type in `contracts/shade/src/types.rs` → update `docs/reference/data-types.md`.
- The public surface of the `account` contract (`contracts/account/src/interface.rs`) → update `docs/contracts/account.md`.
- A new or changed `#[contractevent]` → update `docs/reference/events.md`.
- A new or changed `#[contracterror]` variant → update `docs/reference/errors.md`.

This keeps the API reference trustworthy for integrators who read the docs instead of the Rust source.

### PR Checklist

Before submitting a PR, ensure:

- [ ] Code follows the project's style guidelines
- [ ] All tests pass locally
- [ ] Code is properly formatted (`cargo fmt`)
- [ ] No clippy warnings (`cargo clippy`)
- [ ] Documentation is updated (if needed) — see [Keeping reference docs in sync](#keeping-reference-docs-in-sync)
- [ ] `CHANGELOG.md` is updated (if needed) — see [Releases](docs/operations/releases.md)
- [ ] Commit messages follow the guidelines
- [ ] PR description is clear and complete

## Code Style

- Follow Rust standard formatting (enforced by `cargo fmt`)
- Follow Rust naming conventions
- Use meaningful variable and function names
- Add comments for complex logic
- Keep functions focused and small
- Write tests for new functionality

## Testing Guidelines

- Write unit tests for all new functions
- Test edge cases and error conditions
- Ensure test coverage doesn't decrease
- Use descriptive test names

## Reporting Issues

When reporting issues, please include:

- Description of the issue
- Steps to reproduce
- Expected behavior
- Actual behavior
- Environment details (Rust version, OS, etc.)
- Relevant logs or error messages

## Code of Conduct

- Be respectful and inclusive
- Welcome newcomers and help them learn
- Focus on constructive feedback
- Respect different viewpoints and experiences

## Questions?

If you have questions, feel free to:

- Open an issue for discussion
- Ask in pull request comments
- Contact the maintainers

## Related reading

- [Documentation Style Guide](docs/contributing/documentation-style-guide.md) — tone, formatting, and terminology rules for anything under `docs/`.
- [Architecture overview](docs/architecture/overview.md) — how the contracts, factories, and components fit together before you change or document any of them.
- [Running the test suite](docs/getting-started/running-tests.md) — test layout, filtering, snapshot behavior, and the checklist for adding a new test module.

Thank you for contributing to the development! 🎉

<br />

