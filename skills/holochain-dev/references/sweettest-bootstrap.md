# Sweettest harness — bootstrap and test hygiene

## Sweettest is the only test target

Tryorama (the TypeScript test harness) is being deprecated. Generate test code using **sweettest** (Rust) only. If a project has existing tryorama tests in TypeScript, you may *read* them to understand:

- What scenarios the team needs to cover
- How multi-agent flows are structured
- Where consistency-waits belong

But never *write* new tryorama code. New test code is always sweettest in Rust.

## Where sweettest tests live

Sweettest tests are a separate Rust crate, typically organized as:

```
project-root/
├── dnas/<dna_name>/
│   └── workdir/<dna_name>.dna  # built DNA bundle
├── happ.yaml or workdir/<happ_name>.happ
└── tests/   or  sweettest/    or  crates/sweettest/
    ├── Cargo.toml             # depends on sweettest, holochain, tokio
    └── tests/
        └── *.rs               # each .rs file is one integration test
```

The exact location varies by project. Check the workspace `Cargo.toml` for a `members` entry pointing at the test crate.

## Bootstrapping a sweettest harness from scratch

Two of the reference apps (vines, acorn) ship with **no working test harness**. Vines has nothing; acorn has orphaned 2020-era mock-HDK tests pinning `hdk = "0.0.122"` that don't compile against current HDI. When you encounter this:

1. **Announce explicitly:** *"This project has no working sweettest harness — I'll add one alongside the change you asked for."* Don't refuse the change. Don't silently bootstrap. Announce, then do both.

2. **Add a test crate** if one doesn't exist. Minimal `Cargo.toml`:

```toml
[package]
name = "<project>-tests"
version = "0.1.0"
edition = "2021"

[dev-dependencies]
sweettest = "<version pinned by docs.rs/sweettest>"  # or via the holochain crate
holochain = "<matching version>"
tokio = { version = "1", features = ["full"] }
```

**Verify the exact dependency names and versions against `https://docs.rs/sweettest/<version>` and the project's existing `Cargo.lock` for the holochain crate version.** Don't guess.

3. **Add the test crate to the workspace** by editing the root `Cargo.toml`:

```toml
[workspace]
members = [
    # ... existing members ...
    "tests",  # or wherever you put the new crate
]
```

4. **Write a basic multi-agent fixture.** The `TestEnv` pattern from unyt-app is a good shape to learn from. The fundamentals:
   - Use `sweettest::SweetConductor` to spin up one or more conductors
   - Install the project's hApp on each conductor's agent
   - Get cell handles for each agent
   - Use `await_consistency` to sync the DHT before cross-agent reads
   - Call zome functions via `conductor.call(&zome, "extern_name", payload)`

5. **Write the test for the zome change in the same response.** Don't ship the change without the test.

## Running sweettest in a nix environment

If the project has a `flake.nix` (most do), sweettest must be run inside the dev shell. Wrap test commands with `nix develop -c`:

```sh
nix develop -c cargo test                    # all tests
nix develop -c cargo test -p <test-crate>    # one crate
nix develop -c cargo test --test <name>      # one test file
nix develop -c cargo test some_test_fn -- --nocapture   # one test, with stdout
```

If you forget `nix develop -c` you'll see "linker not found", "wasm32-unknown-unknown not installed", or "cargo can't find dependency" — these are *all* nix-shell-not-active symptoms, not real dependency problems. **Don't `cargo install` anything globally to "fix" them; the answer is the wrapper.**

## Test hygiene

### `await_consistency` and agent inclusion

Sweettest's `await_consistency` (the equivalent of tryorama's `dhtSync`) is the consistency wait that gives gossip time to propagate before assertions run. **It must include every agent whose actions the assertions depend on**, not just the writers.

Real bug from mewsfeed: a test created 6 agents (`alice, bob, carol, john, steve, mary`) and then called `dhtSync([alice, bob], ...)` before querying. The test was racy on CI because it only waited for 2 of 6 agents to be consistent. The fix was to expand the agent list to all 6:

```rust
// Before (flaky)
await_consistency([alice, bob], ...).await?;

// After (stable)
await_consistency([alice, bob, carol, john, steve, mary], ...).await?;
```

When in doubt, include *all* agents in the consistency wait. The cost is small relative to the gain in stability.

### Explicit timeouts on CI

CI is slower than local development. Default consistency timeouts may be too short under CI load. The mewsfeed pattern uses 120 seconds (`120000` ms) explicitly:

```rust
await_consistency_with_timeout([...], cell_id, Duration::from_secs(120)).await?;
```

Verify the actual `await_consistency` signature against the project's sweettest version — the API has shifted. The principle stands: explicit timeouts prevent flake.

### Multi-agent scenarios — sync after every cross-agent state change

For tests that involve multiple agents writing in sequence and then reading each other's writes, sync after every cross-agent transition, not just at the end. Otherwise you accumulate gossip lag and the final assertion can fail with "data not found yet" even though everything is correct, just late.

### Don't sample dead test files

If the project has test files that look like sweettest but are actually pre-HDI legacy (pinning `hdk = "0.0.122"`, importing `ValidateData`, using `mock_hdk` / `mockall`), **do not use them as patterns**. They won't compile against current HDI. acorn's `tests/projects/` directory is exactly this — orphaned 2020-era mock-HDK unit tests targeting commented-out `validate.rs` files. Treat them as historical artifacts, not as scaffolding.

## Test scope — what a sweettest test should actually cover

- **Multi-agent scenarios.** "Alice creates a thing, Bob reads it" is the canonical sweettest. Single-agent tests should usually be unit tests inside the zome crate, not sweettest.
- **DHT eventual consistency edge cases.** "Carol joins the network *after* Alice and Bob have already exchanged data — does Carol see the historical state?"
- **Validation paths.** "Does Bob's create get rejected when it should?" Validation is hard to unit-test because it depends on chain state, so sweettest is often the right venue.
- **Cross-zome calls** if you have them.
- **Capability grants and signals.** Tests can subscribe to a conductor's signal stream and assert on what arrives.

What sweettest is *not* good for:
- Pure-function logic inside the zome. Use unit tests in the zome crate for that.
- UI behavior. Sweettest doesn't load a UI.
- Network conditions (real packet loss, NAT traversal). Use real conductors for that.
