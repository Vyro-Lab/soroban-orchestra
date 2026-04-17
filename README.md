# Soroban Band

**A multi-contract integration test harness for Soroban smart contracts.**

Band gives Soroban developers the ability to deploy, wire, and test complex multi-contract systems in a single, reproducible environment, with no manual boilerplate, no hand-rolled auth trees, and no guessing which cross-contract paths your tests actually cover.

If you are building a dApp where Contract A calls Contract B calls Contract C, Band is the test infrastructure you are currently writing by hand.

---

## Why Band?

Soroban's built-in test utilities work well for single-contract unit tests. Real dApps are never one contract.

A lending protocol has a pool, a collateral manager, an oracle adapter, and SEP-41 token wrappers. A DEX has a router, factory, pair contracts, and fee accumulators. Every team writes the same scaffolding from scratch: dependency-ordered deployment, auth trees, funded accounts, mock oracles, cross-contract coverage.

Band is the missing layer between `soroban-sdk` unit tests and live testnet deployment. It wraps the SDK's existing test utilities; it never forks or replaces them.

---

## Features

- **Dependency-aware deployment**, declare contracts and cross-references; Band topologically sorts, deploys, and wires them with a single `build()` call
- **Auth tree simulation**, model, inspect, and test complex authorization chains across contract boundaries, including delegated and multi-hop auth
- **Auth trace inspection**, dump a human-readable trace of every auth check after any cross-contract call
- **SEP-41 token fixtures**, deploy compliant tokens in one line with pre-funded balances; includes fee-on-transfer and rebasing variants
- **Oracle mock fixtures**, static prices, time-sequenced feeds, and configurable failure modes (stale, missing, extreme deviation)
- **Snapshot and restore**, checkpoint environment state and restore it to test alternative paths without redeploying
- **Time travel**, advance ledger timestamps and sequence numbers to test time-locked logic
- **Cross-contract call graph tracking**, record which contracts called which functions, and report untested paths
- **Interaction coverage metrics**, measure what percentage of each contract's public interface has been exercised, with CI-gating support
- **Property-based testing**, strategy generators for all Soroban types, stateful multi-contract fuzzing with automatic shrinking
- **Scenario DSL**, proc macro attributes that declare full multi-contract environments in a few lines
- **Zero runtime cost**, dev-dependency only; nothing ever ships to chain

---

## Who This Is For

- Soroban developers building multi-contract dApps (DeFi protocols, DEXs, governance systems, NFT platforms)
- Security auditors evaluating multi-contract interaction surfaces
- Protocol teams maintaining long-lived contract systems with regression testing needs
- Open-source contributors and ecosystem tool builders

---

## Requirements

- Rust 1.74 or higher
- `soroban-sdk` 21.x or higher (tracks the latest stable SDK release)
- `stellar-cli` (optional; only required for WASM compilation)
- `cargo` with workspace support

---

## Installation

Add Band to the `[dev-dependencies]` section of your contract crate's `Cargo.toml`:

```toml
[dev-dependencies]
soroban-band = "0.1"
```

Band is never a production dependency.

### Feature Flags

| Flag        | Description                                                              | Default |
|-------------|--------------------------------------------------------------------------|---------|
| `fixtures`  | Pre-built fixture library (tokens, oracles, accounts)                    | Yes     |
| `wasm`      | WASM compilation via `stellar-cli` (requires `stellar-cli` in PATH)      | No      |
| `proptest`  | Property-based testing strategies and stateful fuzzing                   | No      |
| `macros`    | Procedural macro scenario DSL                                            | No      |
| `report`    | Interaction tracking and coverage reporting                              | No      |
| `full`      | Enables all features above                                               | No      |

---

## How It Works

A typical Band test follows this flow:

1. **Declare**, list your contracts, tokens, oracles, and accounts
2. **Build**, Band resolves dependencies, deploys contracts, wires cross-references, funds accounts
3. **Act**, invoke contract functions through Band's environment, which tracks all cross-contract calls
4. **Assert**, use standard Rust assertions plus Band's fixture-specific helpers
5. **Report**, Band aggregates call graphs and coverage metrics after the suite runs

---

## Crate Structure

Band is a Cargo workspace. The main crate re-exports everything; sub-crates are split so you only compile what you use.

| Crate                    | Purpose                                                        |
|--------------------------|----------------------------------------------------------------|
| `soroban-band`           | Main entry point, re-exports all sub-crates                    |
| `soroban-band-core`      | Env builder, auth simulation, snapshot/restore, call tracking  |
| `soroban-band-fixtures`  | Token, oracle, account, and contract pattern fixtures          |
| `soroban-band-proptest`  | Property testing strategies, stateful engine, invariants       |
| `soroban-band-macros`    | Proc macro crate for the scenario DSL                          |
| `soroban-band-report`    | Coverage aggregation and report generation                     |
| `soroban-band-cli`       | `cargo soroban-band` subcommand                                |

---

## Development Stages

Band ships in four staged public releases. Each stage is a complete, usable product that builds on the previous one. Adopting an earlier stage never requires code changes when a later stage ships.

### Stage 1: Foundation
The minimum viable harness that replaces hand-written multi-contract boilerplate. Core orchestration, dependency-aware deployment, auth tree simulation, SEP-41 token fixtures, snapshot/restore, and time travel.

### Stage 2: Observe & Harden
Expanded fixtures for real-world edge cases and cross-contract coverage tracking. Oracle mocks with failure modes, fee-on-transfer and rebasing tokens, account personas, liquidity pool and governance mocks, call graph tracking, and terminal coverage output.

### Stage 3: Fuzz & Verify
Property-based testing that finds bugs manual tests miss. Strategy generators for all Soroban types, stateful multi-contract fuzzing, automatic shrinking, seed persistence for CI replay, and a built-in invariant library.

### Stage 4: Automate & Scale
Ergonomics and scale. Scenario DSL via proc macros, scenario composition and parameterization, JSON/DOT/Mermaid/HTML reports, CI threshold gating, and the `cargo soroban-band` CLI for scaffolding.

---

## Design Principles

- **Additive, not replacement.** Band wraps `soroban-sdk` test utilities; it never forks or hides them.
- **Convention over configuration.** Sensible defaults for everything. Override only what your test cares about.
- **Composition over inheritance.** Fixtures are building blocks, not base classes.
- **Fail loud, fail early.** Misconfigured auth trees and dependency cycles produce precise, actionable diagnostics.
- **Zero runtime cost.** Dev-dependency only. Nothing ever ships to chain.

---

## Compatibility

Band-managed tests are standard `#[test]` functions. They run with `cargo test` and integrate with every Rust test runner, CI pipeline, and coverage tool that already works with Cargo. Band does not depend on any specific IDE, CI system, or deployment tool.

---

## Contributing

Contributions are welcome. Band is designed to grow with the Soroban ecosystem, and community-contributed fixtures, invariants, and strategies are a first-class part of the roadmap.

**Before contributing:**

1. Check the [open issues](../../issues) and the current stage roadmap to see where help is most valuable.
2. For non-trivial changes, open an issue to discuss the design before writing code.
3. Read the design principles above; contributions that conflict with them are unlikely to be merged.

**Getting started:**

```bash
git clone https://github.com/your-org/soroban-band.git
cd soroban-band
cargo test --workspace --all-features
```

**Pull request guidelines:**

- Keep PRs focused. One feature or fix per PR.
- Add tests for new behavior. Band is a testing library; untested code sets a bad example.
- Update relevant documentation and the CHANGELOG.
- Run `cargo fmt` and `cargo clippy --workspace --all-features` before submitting.
- New fixtures must follow the fixture pattern standard: sensible defaults, fluent builder, assertion helpers.

**Scope:**

Band is strictly an integration test harness. Deployment tooling, IDE plugins, monitoring, and similar adjacent features are out of scope. Please refer such proposals to other tools.

---

## License

MIT
