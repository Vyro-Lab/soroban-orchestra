# Soroban Band

**Multi-contract integration test harness for Soroban smart contracts.**

Band gives Soroban developers the ability to deploy, wire, and test complex multi-contract systems in a single environment — no manual boilerplate, no hand-rolled auth trees, no guessing which cross-contract paths your tests actually cover. It handles everything from dependency-ordered deployment and auth-chain simulation to pre-funded token fixtures, mock oracles with configurable failure modes, and property-based fuzzing across contract boundaries.

If you are building a dApp where Contract A calls Contract B calls Contract C, Band is the test infrastructure you are currently writing by hand.

---

## Why Band Exists

Soroban's built-in test utilities handle single-contract unit tests well. `Env::default()`, `register_contract`, and `#[contractimpl]` test patterns work beautifully when your world is one contract.

Real dApps are never one contract.

A lending protocol has a pool contract, a collateral manager, a price oracle adapter, and one or more SEP-41 token wrappers. A DEX has a router, a factory, pair contracts, and fee accumulators. Every one of these systems requires the same painful setup: deploy five contracts in the right order, pass each contract's address to the ones that reference it, construct auth trees that model realistic user flows, fund test accounts with the right token balances, and hope you remembered to test the path where the oracle returns stale data while a user is mid-liquidation.

Every team writes this scaffolding from scratch. The boilerplate is nearly identical across projects, but there is no shared infrastructure. No fixtures. No cross-contract coverage metrics. No way to fuzz a multi-contract state machine.

Band is the missing layer between `soroban-sdk` unit tests and live testnet deployment. It wraps the SDK's existing test utilities in a higher-level framework — it never forks, replaces, or hides them.

---

## Features

- **Dependency-aware deployment** — declare your contracts and their cross-references, and Band topologically sorts, deploys, and wires them in the correct order with a single `build()` call
- **Auth tree simulation** — model, inspect, and test complex authorization chains across contract boundaries, including delegated auth, multi-hop auth, and intentional auth failure scenarios
- **Auth trace inspection** — after any cross-contract call, dump a human-readable trace showing exactly which auth checks fired, in what order, with what arguments, and whether they passed or failed
- **SEP-41 token fixtures** — deploy fully compliant tokens in one line with pre-funded balances, custom decimals, and admin configuration; includes fee-on-transfer and rebasing variants for edge case testing
- **Oracle mock fixtures** — static prices, time-sequenced price feeds, configurable failure modes (stale data, missing feeds, extreme deviations), and adjustable decimal precision
- **Snapshot and restore** — checkpoint environment state before a complex operation and restore it to test alternative paths without re-deploying the entire contract graph
- **Time travel** — advance ledger timestamps and sequence numbers to test time-locked logic, expiration windows, and cooldown periods
- **Cross-contract call graph tracking** — instrument every test to record which contracts called which functions on which other contracts, then report untested interaction paths
- **Interaction coverage metrics** — measure what percentage of each contract's public interface has been exercised via cross-contract calls, with CI-gating support for minimum thresholds
- **Property-based testing** — strategy generators for all Soroban types, stateful multi-contract fuzzing with automatic shrinking, and a library of common DeFi invariants
- **Scenario DSL** — proc macro attributes that let you declare a full multi-contract environment in a few lines, with the macro expanding to all deployment and wiring boilerplate
- **Scenario composition** — define reusable environment fragments and compose them into larger test scenarios without duplication
- **Zero runtime cost** — everything is a dev-dependency; nothing ships to chain

---

## Requirements

- Rust 1.74 or higher
- `soroban-sdk` 21.x or higher (Band tracks the latest stable SDK release)
- `stellar-cli` installed for WASM compilation (optional — Band can also register contracts natively for faster test execution)
- `cargo` with workspace support

---

## Installation

Add Band to the `[dev-dependencies]` section of your contract crate's `Cargo.toml`. It is never a production dependency.

Feature flags control which capabilities are compiled. The core orchestration and fixture library are always included by default.

### Feature Flags

Band uses feature flags to selectively enable functionality and manage dependencies.

| Flag | Description | Included in `default` |
|------|-------------|-----------------------|
| `fixtures` | Enables the pre-built fixture library (tokens, oracles, accounts) | Yes |
| `wasm` | Enables WASM compilation via `stellar-cli` (requires `stellar-cli` in PATH) | No |
| `proptest` | Enables property-based testing strategies and stateful fuzzing | No |
| `macros` | Enables the procedural macro scenario DSL | No |
| `report` | Enables interaction tracking and coverage reporting | No |
| `full` | Enables all features above | No |

---

### Layer 1: Bandtion Core
The foundation. A fluent builder API for constructing multi-contract test environments. You declare contracts and their dependencies. Band builds a directed acyclic graph, detects circular dependencies at compile time, deploys contracts in topological order, and passes cross-references automatically. Includes the auth tree simulator, snapshot/restore, and ledger time manipulation. Everything else in Band depends on this layer.

### Layer 2: Fixture Library
Pre-built, configurable test components for patterns that appear in nearly every Soroban integration test. SEP-41 tokens with pre-funded balances and multiple behavioral variants. Oracle mocks with static prices, time-sequenced feeds, and every failure mode you need to test against. Named test accounts with human-readable aliases and persona presets. Common contract mocks for liquidity pools, governance modules, timelocks, flash loan providers, and proxy contracts. Every fixture follows the same pattern: sensible defaults, fluent builder for customization, assertion helpers specific to the component.

### Layer 3: Property-Based Testing
Strategy generators for all core Soroban types, with "realistic" composite strategies that generate valid multi-contract states (a user whose balance always covers the amount being transferred, an oracle price that respects decimal precision, a swap amount that fits within pool reserves). The stateful testing engine generates random sequences of valid operations across contract boundaries and checks invariants after each step. When a violation is found, the engine shrinks the failing sequence to the smallest reproduction case and persists the seed for CI regression testing.

### Layer 4: Coverage & Reporting
Instruments the test environment to record every cross-contract invocation. Builds a directed call graph where nodes are contract–function pairs and edges are invocations. After a test suite runs, reports which interaction paths were exercised and which were missed. Tracks auth path coverage separately — which authorization tree shapes have been tested. Outputs to terminal, JSON, DOT, or Mermaid for visualization. Integrates with `cargo test` to print a coverage summary after every run, with optional fail-below-threshold for CI gating.

### Layer 5: Scenario DSL
Proc macro attributes that eliminate the remaining ceremony. A single attribute on a test function declares which contracts to deploy, which tokens to create, which accounts to fund, and what oracle prices to set. The macro expands to the full orchestration boilerplate. Scenarios are composable — define a `lending_pool_with_two_tokens` fragment once and reference it across dozens of tests. Parameterization support lets you run the same scenario across a matrix of configurations.

---

## How It Works

### The Core Loop

A typical Band test follows this flow:

1. **Declare** — list your contracts, tokens, oracles, and accounts using the builder API or the scenario DSL
2. **Build** — Band resolves dependencies, deploys contracts in order, wires cross-references, funds accounts, and validates the environment
3. **Act** — invoke contract functions through Band's environment, which transparently tracks all cross-contract calls and auth events
4. **Assert** — use standard Rust assertions plus Band's fixture-specific helpers to verify outcomes
5. **Report** — after all tests complete, Band aggregates call graphs and coverage metrics

### Auth Tree Simulation

The most valuable and most complex piece. When Contract A invokes Contract B on behalf of a user, Soroban requires a properly constructed authorization tree. In production, the wallet builds this. In tests, you must simulate it.

Band models auth trees as explicit, inspectable data structures. You describe the authorization chain you expect, and Band either confirms it matches what the contracts require or tells you exactly where the mismatch is — which node in the tree, which contract, which function, which argument.

You can also deliberately construct invalid auth trees to test your contracts' error handling: missing auth, wrong signer, expired auth entries, insufficient permissions. This is the class of bug that usually isn't caught until testnet.

### Cross-Contract Coverage

After your test suite runs, Band tells you:

- Which contracts called which other contracts, and through which functions
- Which cross-contract call paths were exercised and which were not
- Which authorization tree shapes were tested
- Which contracts were deployed but never called (indicating test gaps or unnecessary setup)
- A coverage percentage for each contract pair's interaction surface

This is the information that turns "we have tests" into "we have confidence."

---

## Fixture Catalog

### SEP-41 Tokens
Standard compliant tokens with configurable name, symbol, decimals, and admin. Pre-fund any number of addresses at deployment time. Assertion helpers for balance and allowance checks. Includes behavioral variants: **fee-on-transfer** (for testing contracts that must handle deflationary tokens) and **rebasing** (for testing contracts that must handle externally changing balances).

### Oracle Mocks
Set static prices per asset pair, or configure price sequences that change as ledger time advances. Simulate every failure mode your contracts should handle: stale prices, missing feeds, extreme deviations, and zero-confidence responses. Configurable decimal precision and confidence intervals.

### Account Personas
Named test accounts with presets: **whale** (large balances for testing at scale), **dust** (minimum balances for testing edge cases), **multi-sig** (multi-signature accounts requiring n-of-m approval). Every account gets a human-readable alias that appears in auth traces and coverage reports instead of raw addresses.

### Common Contract Mocks
**Liquidity Pool** — configurable reserves, swap execution, LP token minting. **Governance** — proposal creation, voting with configurable quorum and threshold. **Timelock** — delayed execution with configurable delay periods. **Flash Loan Provider** — for testing flash loan receiver contracts. **Proxy/Upgradeable** — for testing contracts that interact with upgradeable contract patterns.

---

## Property-Based Testing

Band's property testing goes beyond single-contract fuzzing. It generates sequences of operations that span contract boundaries and checks invariants across the entire system after each step.

### Built-In Invariants
Conservation of value — no tokens created or destroyed outside of mint/burn operations. No negative balances. Monotonic sequence numbers. Oracle price freshness — no contract reads a price older than the configured staleness threshold. Pool constant product — reserves maintain the k = x × y invariant after every swap.

### Custom Invariants
Define your own invariants as closures that receive the full environment state. Check token balances, oracle prices, pool reserves, and governance state simultaneously. Band runs your invariants after every step in a stateful property test.

### Shrinking
When a property violation is found, Band minimizes the failing sequence to the smallest set of operations that still triggers the bug. The failing seed is persisted to disk and automatically replayed in CI to prevent regressions.

---

## Report Formats

| Format | Use Case |
|--------|----------|
| Terminal | Quick summary after `cargo test` — interaction coverage percentage, untested paths, orphan contracts |
| JSON | Machine-readable call graph and coverage data for CI dashboards and custom tooling |
| DOT | Visual call graph importable into Graphviz for documentation and architecture reviews |
| Mermaid | Visual call graph embeddable in GitHub markdown and Notion docs |
| HTML | Self-contained single-file report with call graphs, coverage metrics, property test results, auth tree visualizations, and timing data |

---

## Crate Structure

Band is a Cargo workspace. The main crate re-exports everything. Sub-crates are split so you only compile what you use.

| Crate | Purpose |
|-------|---------|
| `soroban-band` | Main entry point, re-exports all sub-crates |
| `soroban-band-core` | Env builder, auth simulation, snapshot/restore, call tracking |
| `soroban-band-fixtures` | Token, oracle, account, and contract pattern fixtures |
| `soroban-band-proptest` | Property testing strategies, stateful engine, invariant library |
| `soroban-band-macros` | Proc macro crate for the scenario DSL |
| `soroban-band-report` | Coverage aggregation, HTML/JSON/DOT/Mermaid report generation |
| `soroban-band-cli` | `cargo soroban-band` subcommand for scaffolding and reports |

---

## Design Principles

**Additive, not replacement.** Band wraps `soroban-sdk` test utilities. It never forks, bypasses, or hides them. A developer who knows `Env::default()` and `register_contract` is already at home.

**Convention over configuration.** Sensible defaults for everything — token decimals, oracle prices, initial balances, auth patterns. Override only what your specific test cares about.

**Composition over inheritance.** Fixtures are building blocks, not base classes. Combine a token fixture with an oracle fixture and an account fixture without inheriting from anything.

**Fail loud, fail early.** If an auth tree is misconfigured, Band tells you which node is wrong, not just that something panicked. If a dependency cycle exists, you learn at build time with a message naming every contract in the cycle.

**Zero runtime cost.** Dev-dependency only. Nothing from Band ever ships to chain or touches a production deployment.

---

## Compatibility

Band is built on top of `soroban-sdk` test utilities and tracks the latest stable SDK release. It uses `stellar-cli` for optional WASM compilation. It does not depend on any specific IDE, CI system, or deployment tool.

Band-managed tests are standard `#[test]` functions. They run with `cargo test`. They integrate with every Rust test runner, CI pipeline, and coverage tool that already works with Cargo.

---

## Roadmap

**Phase 1 — Foundation.** Bandtion core, auth tree simulation, basic token and oracle fixtures, snapshot/restore, time travel. The minimum viable harness that replaces your hand-written boilerplate.

**Phase 2 — Fixtures & Coverage.** Expanded fixture library (fee-on-transfer tokens, oracle failure modes, LP pools, governance mocks), cross-contract call graph tracking, interaction and auth path coverage, CI gating.

**Phase 3 — Property Testing.** Strategy generators for all Soroban types, stateful multi-contract fuzzing, invariant library, shrinking, seed persistence.

**Phase 4 — DSL & Polish.** Proc macro scenario DSL, scenario composition and parameterization, HTML report generation, CLI scaffolding tool, comprehensive documentation and migration guide.

---

## License

MIT
