# Architecture choices: the full menu

Each section below is one architectural axis. The skill should **detect** which option an existing project uses and stay in that idiom. For new projects starting from a blank slate, the **defaults** noted here apply, but only when there is no existing signal to follow.

---

## Coordinator return shape

**Question:** when a coordinator extern creates or fetches a record, what does it return?

**Options:**

- **(a) Hash-only.** Returns `ActionHashB64` or `ActionHash`. The UI must follow up with a `get` to fetch the entry. Used by unyt-app.
- **(b) Wrapper struct.** Returns a struct holding the hash, the entry, and metadata (timestamps, author, etc.). Examples: `AuthoredX` (dino-adventure), `WireRecord<T>` (acorn), `EntryRecord<T>` (emergence/mewsfeed). **The most common pattern.**
- **(c) Signal-push.** The extern returns `ExternResult<()>` and emits the result as a signal. The UI listens to the signal stream. Used by vines.

**Default for new projects: (b)**, the wrapper struct. Most apps converge here. Pick a struct shape and stay consistent within the project.

**Watch out for:** mixing shapes within one project. Pick one and apply it to all externs.

---

## Validation depth

**Question:** how much business logic should the integrity zome's validate callback enforce?

**Options:**

- **(a) Minimal-permissive.** Validate callback returns `Valid` for everything; only structural integrity matters. Most apps. dino-adventure, emergence, acorn.
- **(b) Per-op declarative cross-entry checks.** Dispatch on `Op`, do real cross-entry work via `must_get_*` on a small number of high-stakes ops. Vines.
- **(c) Comprehensive state machine.** Validation encodes full business logic and chain-spanning invariants. unyt-app.

**Default for new projects: (a)**. Don't push elaborate validation onto every entry type — that's a category error. Move to (b) when you discover real cross-entry invariants. Move to (c) only when the protocol genuinely requires chain-spanning logic (ledgers, multi-phase transactions, smart agreements).

---

## Zome topology

**Question:** how should integrity and coordinator zomes be split within a DNA?

**Options:**

- **(a) Single integrity + single coordinator.** The default. Use Rust modules within each for code organization.
- **(b) Split for third-party reuse.** Keep an upstream integrity zome (e.g., `hc_zome_profiles_integrity`) as its own zome to avoid polluting your main `EntryTypes` enum. Vines pattern.
- **(c) Split for migration-safe side channels.** Dedicated zome whose only job is to record cross-DNA state as links (e.g., original-author attribution). Vines `authorship` pattern.

**Default for new projects: (a)**. Splits should be principled. **Anti-pattern:** splitting for "modularity" alone — that's the mewsfeed-era scaffolder mistake.

**Watch out for:** hardcoded zome ordinals (`ZomeIndex(N)`) when zomes need to refer to each other. Use named imports.

---

## Single vs multi-DNA

**Question:** does the hApp need more than one DNA?

**Options:**

- **(a) Single DNA.** One DHT, all data shared among all peers. The default.
- **(b) Multi-DNA: singleton + cloned.** A singleton DNA for shared data (e.g., user profiles) plus a per-workspace cloned DNA with `clone_limit: N, deferred: true`. Each clone is its own private DHT. Acorn pattern.

**Default for new projects: (a)**. Use multi-DNA only when you have a real privacy boundary, lifecycle isolation need, or scale concern. Reaching for multi-DNA without one of those is overkill.

**Caveats for (b):**
- `dna_info()` doesn't return clone DNAs at startup with `deferred: true`. The UI must explicitly create cells.
- Migration is much harder across cloned cells.
- The eventual-consistency model applies *per cell*.

---

## Local vs network fetch exposure

**Question:** should the API surface let the UI choose between local and network fetches?

**Options:**

- **(a) Hide it.** The zome picks `GetOptions::local()` or `GetOptions::network()` per call; the UI doesn't see the choice.
- **(b) Wrapper input struct with `local: bool` flag.** Every fetch input is wrapped: `ZomeFnInput<T> { input: T, local: bool }`. mewsfeed and acorn (independently rediscovered).
- **(c) Separate externs.** `get_x` (network) and `get_x_local` (local) as distinct functions. dino-adventure.
- **(d) Strategy field on input.** A `strategy: GetStrategy` enum with explicit variants. Vines.

**Default for new projects: depends on whether the hApp deploys to zero-arc nodes.** If yes, expose local-vs-network choice — pick (b), (c), or (d) based on team taste. If the hApp is desktop-only and full-arc-only, (a) is fine. Ask the user.

**Why this matters:** see `eventual-consistency.md`. Zero-arc nodes have no local data; every `get` must hit the network. If you don't expose the choice, the UI can't adapt to its node's arc.

**Asymmetry trap for (b):** in acorn the wrapper is fetch-only. Writes pass payloads raw. Wrapping a write in the same envelope causes silent failure because the macro doesn't destructure it. Always check how existing writes are called.

---

## Signal placement

**Question:** where should signals be emitted from?

**Options:**

- **(a) `post_commit` callback.** Fired after commit lands; has access to the real (non-temporary) action hash. Asynchronous.
- **(b) Inline `send_remote_signal` from a coordinator extern.** Fired during the extern call. Lets you emit custom cross-entry signal variants atomically with the write.
- **(c) Push-based fetch via signals.** Fetch externs return `()` and emit results as signals. Vines pattern.

**Default for new projects: (a)** because it works under both synchronous and relaxed commit modes. (b) only works under synchronous commits — picking it locks you out of relaxed commits later without a refactor.

**Why this matters: see `signals-and-post-commit.md` for the full relaxed-commit tradeoff.** Under relaxed commits, the synchronous extern's returned `ActionHash` can be a *temporary* hash that gets rewritten when the commit actually lands. If anywhere holds ActionHashes as durable resource IDs, you must use `post_commit` to get the real hash. The alternative is handling `source_chain_moved` errors at every read site.

**Ask the user** whether the app uses or will use relaxed/async commits before designing the signal layer.

---

## Hashes-in-entries

**Question:** when an entry references another entry by hash, store raw bytes or B64 strings?

**Options:**

- **(a) Raw `ActionHash` bytes.** Cheap Rust-side comparisons. The UI wraps hashes in convenience types when needed.
- **(b) `ActionHashB64` strings.** UI can string-compare directly. **Bakes a UI-friendly encoding into the DHT permanently** — Rust-side hash math now requires a parse on every comparison.

**Default for new projects: (a)**. Acorn picked (b); it works but requires consistency throughout the codebase and you can't undo it without a DNA migration.

---

## Timestamps in entries

**Question:** when an entry has a timestamp field, what type?

**Options:**

- **(a) `Timestamp`** — i64 microseconds, the HDK type. UI handles bigint coercion.
- **(b) `f64`** — JS-number-compatible. Loses precision past 2^53 microseconds; fine for app lifetimes.

**Default for new projects: (a)**. (b) is acorn's choice. Don't silently convert one to the other across the boundary.

**Note:** action timestamps are *always* `Timestamp` regardless of what entries embed. Don't confuse the embedded timestamp with the action's actual create timestamp.

---

## Soft delete vs hard delete

**Question:** when a user "deletes" something, what happens?

**Options:**

- **(a) `trashed: bool` field**, UI hides trashed records. Preserves history.
- **(b) Actual `delete_entry`** call. Interacts with DHT tombstone semantics.
- **(c) Custom cascade-delete extern** that walks all related links and deletes them too. Acorn's `delete_outcome_fully` pattern, needed when an entry has many dependent links that would otherwise dangle.

**Default for new projects: depends on the data model.** If "delete" should be reversible or auditable, use (a). If deletion is final and there are no dependent entities, use (b). If there are dependent entities that need to be cleaned up too, use (c). Ask the user about requirements.

---

## Serialization strategy

**Question:** how is the Rust↔TS type boundary maintained?

**Options:**

- **(a) Hand-mirrored TS types.** Simplest; no tooling.
- **(b) Hand-mirrored zod schemas, TS derived.** Acorn pattern. Runtime validation as a side effect.
- **(c) `zits` codegen.** Vines pattern. Auto-derives TS from Rust source.

**Default for new projects: (c)**. `zits` is the only strategy where adding a Rust field flows to TS automatically. See `serialization-boundaries.md` for the full discussion of pros and cons (especially the `Uint8Array` materialize layer that `zits` forces).

**Eric is currently maintaining an updated fork of `zits`.** Once the fork is the source of truth, point new projects at it. For now, upstream `zits` is the default.

---

## State management on the UI side

**Question:** how does the UI organize state derived from zome calls and signals?

**Options:**

- **(a) `@holochain-open-dev/stores`** async-derived stores. Heavyweight but gives you `asyncDerived`, `toPromise`, lazy-loading helpers. Used by emergence.
- **(b) Hand-rolled Svelte runes / module state.** Simpler; works with Svelte 5. dino-adventure.
- **(c) Lit context.** Used by unyt-app's Lit-native UI.
- **(d) Redux + reducers fed by signals.** Acorn.
- **(e) `@ddd-qc/lit-happ` dvm/zvm/perspective architecture.** Vines.

**Default: detect what the project uses and stay in that idiom.** Don't impose a state-management style on an existing project. For new projects, pick based on the UI framework: Svelte → (b), Lit → (c) or (e), React → (d) or hand-rolled.
