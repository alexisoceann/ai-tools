# Zome architecture: single vs split, multi-DNA

## Default: single integrity + single coordinator zome per DNA

For new projects, the default is one integrity zome and one coordinator zome per DNA. This is the right shape for most apps. The reasons are:

- **Cross-zome calls have measurable overhead.** Each call goes through the wasm boundary. In hot paths this adds up.
- **Many small zomes with cross-zome calls is the historical scaffolder anti-pattern.** Old Holochain scaffolders generated separate integrity+coordinator pairs per entity (a `mews` zome, a `follows` zome, a `likes` zome, etc.) with `hc_call_utils::call_local_zome()` between them. This is what mewsfeed inherited and what its `zome-merge` branch is collapsing. The friction is real.
- **Hardcoded zome ordinals are a real risk.** When zomes need to refer to each other's entry/link types in cross-zome link queries, the easy path is a hardcoded `ZomeIndex(N)` constant. mewsfeed had `MEWS_ZOME_INDEX = 1, FOLLOWS_ZOME_INDEX = 2, ...` in `agent_to_notifications.rs` and changing the zome order in `dna.yaml` would silently break link queries. Single-zome eliminates this entire class of bug.

## Principled splits — when *to* split

Not all splits are bad. Vines deliberately splits its DNA into three integrity zomes, and the splits are principled:

### Split #1 — Third-party zome reuse without entry-enum pollution

If you want to use a published third-party zome like `hc_zome_profiles_integrity` from holochain-open-dev, you *could* inline its entry types into your main zome's `EntryTypes` enum. But that's invasive — the third-party crate's entries become part of your enum, your validation has to handle them, and upgrading the third-party crate forces you to update your zome.

The cleaner pattern is to keep the third-party zome as its *own* integrity zome in your DNA. Your main zome's `EntryTypes` stays clean. The upstream crate is responsible for its own validation. Your DNA still ships as one bundle, but the integrity zomes are isolated. Vines does this for `hc_zome_profiles_integrity` (`vines/dna/zomes/integrity/profiles/src/lib.rs`).

This is the right pattern when you want to reuse upstream integrity logic without forking it.

### Split #2 — Migration-safe side channels

If you anticipate a future DNA-hash change, you can put state that needs to survive the migration into a *separate* integrity zome whose only job is to record that state as links. Vines has `authorship_integrity` for exactly this — a zome that records original-author attribution as `Author` and `Target` links so migrated data preserves who wrote what across DNA hash changes.

The trick is that this side-channel zome can be added to a future DNA bundle alongside re-imported entries from the old DNA, and the link-only structure means the new DNA can preserve attribution that the new entries would otherwise lose.

This is a real pattern, but it requires you to be thinking about migrations at the start. If your app is small and migrations are years away, don't pre-emptively split.

### What's NOT a principled split

- **"Modularity"** as a goal in itself. Each entity is its own zome. This is the mewsfeed-era anti-pattern. Don't do this.
- **"Cleaner code organization."** Use Rust modules within a single zome for that. `pub mod beads; pub mod participation_protocols;` etc.
- **"Performance."** Splitting *adds* cross-zome call overhead, it doesn't remove anything.

## Multi-DNA architectures

Most hApps are single-DNA. But multi-DNA exists and is the right shape for some use cases. Acorn is the exemplar.

### The acorn pattern

Acorn is a project-management hApp where each "project" is its own private workspace shared with project members. Acorn ships with two DNAs:

1. **`profiles`** — a singleton DNA, shared across all users of the hApp. Holds user profiles (name, avatar). One cell per user, joined at install time. `clone_limit: 0`.
2. **`projects`** — a *cloned-per-project* DNA. Each acorn project is a separate clone of this DNA. `clone_limit: 999, deferred: true`.

The UI calls `AppClient.createCloneCell()` to instantiate a new `projects` clone whenever the user creates a new project, joining a private DHT for that project's members only.

### Why use this shape

- **Privacy boundaries.** Each project's DHT is isolated from other projects. Members of project A cannot see project B's data, even if both exist in the same conductor.
- **Independent data lifecycles.** Projects can be archived, deleted, or migrated independently.
- **Bounded gossip cost.** Each agent only gossips with peers in projects they're members of, not with the entire hApp's userbase.

### Caveats and gotchas

- **`dna_info()` does not return clone DNAs at startup.** With `deferred: true`, the clone cells aren't provisioned until the UI explicitly creates them. Code that assumes "the projects DNA exists at install time" gives "cell not found" errors.
- **The DHT model applies per cell.** Eventual consistency, gossip lag, partial arcs — all of these apply *within* a clone cell. Two cells of the same DNA don't share state.
- **Migration is much harder across cloned cells.** If you need to change the DNA, every existing clone needs to be migrated. The vines `authorship_integrity` side-channel pattern is one way to mitigate this.
- **Profile data has to live in the singleton DNA.** Otherwise users would need a separate profile per project, which is bad UX. Acorn's split (profiles singleton + projects cloned) is the natural shape.

### When NOT to use multi-DNA

- When a single shared DHT is fine. Most apps. Don't reach for multi-DNA without a privacy/lifecycle/scale reason.
- When you'd be tempted to put "common" entry types in one DNA and "specific" types in another. That's just refactoring across DNA boundaries — use one DNA with modules instead.

## Cross-zome calls — when you must

If you have a principled split and cross-zome calls are necessary, use **named imports**, never numeric indices. The HDK provides `call` for cross-zome calls within the same cell. The exact signature evolved across versions — verify against `https://docs.rs/hdk/<version>`.

Wrapper structs that gate local-vs-network behavior (`ZomeFnInput<T> { local: bool }`, independently rediscovered in mewsfeed and acorn) are a common pattern for cross-zome fetch calls. **Watch out for the asymmetry:** in acorn the wrapper is fetch-only; passing it on writes silently fails because the macro doesn't destructure it. Always check how the project's existing cross-zome calls are shaped before adding a new one.
