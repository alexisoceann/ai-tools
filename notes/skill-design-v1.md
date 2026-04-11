# Holochain-Dev Skill — Design v1

**Status:** synthesis of research findings, pre-implementation. Round 2 research on vines + acorn pending; design will be refined when it lands.
**Sources:**
- [holochain-skill-planning.md](holochain-skill-planning.md) — original handoff context and constraints
- [research-app-patterns.md](research-app-patterns.md) — patterns from unyt-app, dino-adventure, emergence
- [research-mewsfeed-friction.md](research-mewsfeed-friction.md) — Claude friction mining from mewsfeed sessions
- [research-vines-acorn-patterns.md](research-vines-acorn-patterns.md) — orthogonal data from vines and acorn *(pending)*
- Memory: `project_skill_hdk_version_check.md`, `project_mewsfeed_quality.md`

## Reference-app weighting (from Eric, 2026-04-10)

- **unyt-app** — most tested, most important for serious development. Highest weight.
- **vines / acorn** — significantly well developed; added as orthogonal data points in round 2.
- **emergence** — solid mid-weight exemplar, middle of the complexity range.
- **dino-adventure** — primarily a *testing app* so don't overweight scope, BUT built by a very skilled Rust coder and senior Holochain team developer, so its patterns are high-signal per-line. Worth including for patterns, not for comprehensive coverage.
- **mewsfeed** — history-mining only, never design exemplar (early-dev anti-patterns; recent fixes show the direction).

---

## Core insight (load-bearing)

The mewsfeed friction mining produced one decisive observation that should drive everything else:

> "The skill probably should NOT try to reteach Claude how to write `get`, `create_entry`, `create_link`, or `update_entry` — those worked fine in the session data without correction. **The friction is at the semantic layer (caching, consistency, zome topology, validation context) not the syntactic layer.**"

This means the skill is **not a Holochain tutorial**. Claude already writes plausible HDK syntax. What Claude consistently gets wrong is:
- Cost model of host functions (calling `must_get_agent_activity` to read one field)
- Side effects of touching integrity-zome code (DNA hash changes)
- Eventual-consistency reasoning (assumes data is always reachable)
- Cross-zome wiring (hardcoded ordinals, missing wrapper structs)
- Source-of-truth discipline (guesses semantics instead of reading source/docs.rs)
- Test hygiene under DHT consistency (`dhtSync` agent lists, timeouts)

**The skill is a semantics encoder, not a syntax teacher.** Every section below should be evaluated against this rule: does it address something Claude is already getting right (skip), or something Claude is getting wrong because the model lacks the context (include).

---

## What the research settled

These are decisions the research already made for us — no further user input needed. **Several of these were revised after round 2 (vines + acorn) added orthogonal data points.**

1. **`@holochain-open-dev` usage is a spectrum.** ~~Only emergence uses it.~~ **Revised:** emergence uses the full stack; vines uses `hc_zome_profiles_integrity` upstream but forks the coordinator; acorn/unyt-app/dino-adventure don't use it. Spectrum: whole-zome reuse → partial-with-fork → none. The skill must not assume the libraries are present, but should recognize partial reuse as a legitimate pattern.

2. **Sweettest is the skill's only test-generation target.** Tryorama is being deprecated. The skill will generate sweettest-only test code. Tryorama tests in existing codebases may be *read* for test-pattern inspiration — what scenarios need covering, how multi-agent flows are structured, where consistency-waits belong — but the skill never emits tryorama output. **Note from round 2:** two of the five surveyed apps (**vines, acorn**) have no functional automated tests at all. Vines has nothing; acorn has orphaned 2020-era mock-HDK tests pinning `hdk = "0.0.122"`. The skill cannot assume existing tests to copy from — it may have to bootstrap a sweettest harness from scratch.

3. **Coordinator return shape: convergent but not universal.** ~~Per-app architectural choice.~~ **Revised:** Four of the five apps (dino-adventure `AuthoredX`, emergence `EntryRecord<T>`, mewsfeed `EntryRecord`, acorn `WireRecord<T>`) return "some struct wrapping a hash + entry + metadata." Only **unyt-app** returns hash-only, and **vines** returns *nothing* from fetches — it publishes via signals. Consensus is "return a wrapper struct"; the bikeshed is what's in it. The skill should default to the wrapper-struct shape and recognize hash-only and signal-push as deliberate minority choices.

4. **Validation has three styles, not two.** ~~Bimodal.~~ **Revised:**
   - **Minimal-permissive** (dino-adventure, emergence, acorn) — the validate callback is empty or absent. Structural integrity only.
   - **Declarative per-op cross-entry checks with chain walks** (vines) — dispatch on `Op`, do real cross-entry work via `must_get_*` and `ChainFilter` on specific ops (`StoreEntry`, `RegisterCreateLink`), leave others permissive. Middle ground.
   - **Comprehensive state machine** (unyt-app) — validation encodes full business logic against a `TxProcessor` walking the agent's entire chain.
   
   The skill should know all three and should not push elaborate validation by default.

5. **Zome architecture is "single-zome default, split for principled reuse."** ~~Single integrity + single coordinator is best practice everywhere.~~ **Revised:** Single-zome is the default — confirmed by dino-adventure, emergence (core), unyt-app, acorn, and mewsfeed's `zome-merge` direction. BUT **vines intentionally splits into three integrity zomes** for two load-bearing reasons:
   - **Third-party zome reuse without entry-enum pollution:** `profiles_integrity` is a direct re-export of `hc_zome_profiles_integrity` — keeping it separate avoids pulling upstream entries into the main zome's `EntryTypes` enum.
   - **Migration-safe side channels:** `authorship_integrity` is a dedicated zome that records original-author attribution as links, so migrated data preserves pre-migration authorship across DNA hash changes.
   
   The anti-pattern is *gratuitous* splitting with cross-zome `call_local_zome()` and hardcoded `ZomeIndex(N)` ordinals (mewsfeed's original design). Principled splits for reuse and migration are *good*. A skill that always generates single-zome DNAs forces users to inline-copy upstream crates.

6. **Multi-DNA architectures exist and are principled** (new finding from acorn). Acorn uses a two-DNA design: a `profiles` singleton shared across all workspaces, plus a `projects` DNA **cloned per workspace** with `clone_limit: 999, deferred: true`. Each cloned DNA is its own cell. The skill must know that multi-DNA with deferred-clone provisioning is a valid architecture, not just an artifact. Implication: `dna_info()` does not return the clone DNA at startup; assuming it does gives "cell not found" errors.

7. **The default serde enum shape is a named pitfall.** (New finding.) Rust `pub enum E { A(T), B(T) }` serialized through Holochain's msgpack path surfaces on the TS side as `{A: ...}` variant-object tags. Acorn has **two different workarounds in the same repo** — a `UIEnum(pub String)` newtype with `#[serde(from/into = "UIEnum")]`, AND a hand-rolled `Serialize`/`Deserialize` for `Profile::Status`. Both coerce to a bare JSON string. Vines embraces the default and handles it on the TS side via the generated variant-object types + wrapper-class materialization. The skill needs to know this pitfall has a name and has multiple valid fixes.

8. **Three serialization strategies across the five apps.** (New, replaces part of what was in round 1.)
   - **Hand-written TS types mirroring Rust byte-for-byte** (unyt-app, dino-adventure, emergence). No compile-time drift check.
   - **Hand-written zod schemas, TS types derived from zod** (acorn's `zod-models` workspace package). Zod doubles as runtime validator — catches drift at runtime.
   - **Codegen from Rust via `zits`** (vines only). The only approach where adding a Rust field flows to TS automatically. Forces a materialize/dematerialize layer because hashes become raw `Uint8Array` aliases.
   
   The skill must detect which strategy a project uses and adapt — specifically, after any Rust type change it must know which lockstep action to trigger (update TS manually / update zod manually / re-run `scripts/ts-bindings.sh`).

---

## Hard constraints to encode as skill behavior

These are tripwires the skill should enforce. Each one comes from a real friction moment in the research.

### A. DNA-hash tripwire (context-aware)
**Rule:** Before editing anything that affects the DNA hash, the skill must recognize the impact and behave differently based on git branch:
- **On a protected branch** (`main`, `master`, `production`, `release/*`): hard stop. The skill refuses to write the change until the user either explicitly opts in for that turn or moves to a dev branch.
- **On a dev/feature branch:** warning only. The skill writes the change but surfaces the DNA-impact in the response.

When a user is getting blocked on a protected branch, the skill should remind them that **moving to a dev branch suppresses the warnings** — that's the documented escape hatch.

**Scope of "DNA-impacting":**
1. Anything under `**/zomes/integrity/**`
2. Any `#[hdk_entry_types]` / `#[hdk_link_types]` / `validate*` definition
3. Any `dna.yaml` file
4. Any file referenced from a `dna.yaml`'s `properties` block (e.g. an embedded image/asset whose bytes are checked in `genesis_self_check`)

**Source:** `research-mewsfeed-friction.md` patterns #2 and #10. Direct user quotes: *"your fix included DNA changes, why?"* and *"no, revert it for now ... we want to fix this problem without having to redeploy the DNA so I don't want a breaking chagne."* Plus `research-vines-acorn-patterns.md` finding that `dna.yaml` properties bytes are part of the hash via `genesis_self_check`.

**How it should work:** When a proposed edit lands on a DNA-impacting path, the skill checks the current branch. If protected, it pauses and offers two paths: (a) explicit per-turn opt-in, or (b) move to a dev branch. If the same fix can be made in coordinator code, UI code, or signals/cache instead, the skill should suggest that route first.

### B. HDK version verification
**Rule:** The skill must detect the HDK/HDI version the target project builds against (read `Cargo.toml` / `Cargo.lock`) and verify any HDK API usage against `https://docs.rs/hdk/<version>` for that version. Never embed HDK API surface as authoritative.

**Source:** Project memory `project_skill_hdk_version_check.md`; reinforced by mewsfeed friction where 0.4 → 0.6 migration introduced `FlatOp<EntryTypes, LinkTypes>`, `GetOptions::local()` (note `.local()` not `::local`), `#[hdk_extern(infallible)] post_commit`, etc. — all syntactic shifts that stale training-data patterns get wrong.

**How it should work:** On invocation, the skill reads `Cargo.toml` to find the `hdk` and `hdi` versions, then includes a "current target version" banner in its working state. When suggesting an HDK API usage, it should fetch the relevant docs.rs page (or instruct the agent to do so) before recommending. The local `../holochain` checkout is research-only context for skill *authoring* — it must not be referenced from inside the shipped skill content as a source of API truth.

### C. Source-over-guessing for semantics (docs.rs only)
**Rule:** When the skill needs to reason about *what a host function actually does* (not just its signature), it must look up the answer from docs.rs at the project's pinned HDK/HDI version, not guess from training data. Order of preference:
1. `https://docs.rs/hdk/<version>` for HDK functions
2. `https://docs.rs/hdi/<version>` for HDI functions (validation context, `Op` variants, etc.)
3. Only as last resort: model recall, explicitly flagged as such

**No local-source fallback.** The skill must NOT read from a local `../holochain` clone even when one is available. Local sources may be the wrong version, may include unmerged changes, and won't be present in users' environments. docs.rs is the only authoritative source the skill cites.

**Source:** mewsfeed friction pattern #1, by far the most frequent correction class. Eric directive 2026-04-11 narrowing to docs.rs only.

**How it should work:** When the skill is asked about cascade behavior, scratch-space merging, `ChainFilter` range semantics, validation context, or any "what does this function actually do" question, it must verify against docs.rs before answering. The skill body should contain a short list of high-risk host functions where guessing has historically been wrong (`must_get_agent_activity`, `get`, `get_links` with various `GetOptions`, `must_get_action`, `must_get_entry`, the cascade interaction with the scratch space) — any of these should always be looked up, never recalled.

### D. Sweettest-only test generation
**Rule:** The skill generates tests using sweettest (Rust) only. Tryorama is being deprecated. If a project has existing tryorama tests, the skill may read them to understand what scenarios already exist and what coverage patterns the team needs, but new test code is always sweettest.

**Source:** Eric directive, 2026-04-10. Tryorama deprecation means any tryorama output the skill generates ages out immediately.

### F. Serialization-boundary diagnostic inversion
**Rule:** When a serialization error surfaces (msgpack decode failures, "invalid type" errors at the Rust↔TS boundary, unexpected `None` deserializations), the skill's default diagnostic order is:
1. **Rust side hasn't been recompiled** since the last type change — is the running wasm stale? Has the DNA been rebuilt? Is the conductor serving the old wasm?
2. **Rust type and TS type have drifted** — is the TS type definition still aligned with the Rust struct? Any `#[serde(rename = ...)]` or `#[serde(tag = ...)]` on the Rust side that the TS side doesn't mirror?
3. **Wrong-side check** — the agent is inspecting the TS side for a bug that's on the Rust side, or vice versa.
4. *Last resort:* msgpack / serde version mismatches. These are almost never the actual cause.

The skill must **never** suggest pinning or bumping msgpack / serde versions as a first-line fix. That's a Claude reflex that wastes cycles on a phantom cause.

**Source:** Eric directive, 2026-04-10. Recurring pattern in actual dev work that the mewsfeed history mining didn't surface prominently because the users there already knew to skip step 4.

### E. From the original planning notes (unchanged)
- TDD required: any zome change comes with a test (in whichever world the project uses)
- Strong typing: TypeScript code uses strong types; Rust avoids `unwrap()`/`expect()` in production paths
- Coordinator vs integrity separation: validation only in integrity zomes
- No Claude footers in commit messages

---

## Semantic knowledge to encode

This is the meat of the skill — the things that aren't obvious from HDK signatures but tripped Claude up repeatedly.

### Eventual consistency reasoning
- Difference between `GetOptions::local()` and `GetOptions::network()`
- Why a `get` can return `None` even when the data exists somewhere on the network (gossip lag, partial arcs)
- **Zero-arc nodes are a first-class deployment target.** A node running with arc-size 0 (typical on phones / battery-conserving devices) does not participate in gossip and has *no* local data — every `get`/`get_links` must go to the network and may be slow or fail. A node running full-arc (typical on a desktop) has the data locally and can answer cheaply. **Apps that target both contexts need to expose local-vs-network choice in their API surface** so the UI can adapt — "give me what you have locally, fast" on a phone vs "fetch from network" on a desktop. Apps that only ever run full-arc don't need this distinction. The skill should treat the question "does this happ deploy to zero-arc nodes" as a load-bearing architecture decision, not a detail.
- UIs must have partial-result paths — a single `Mew not found` cannot cascade into a blank feed
- The dino-adventure-style polling-plus-signals hybrid is one valid approach; the emergence `neededStuff` batched periodic refresh is another; vines's signal-push (fetch returns `()`, data arrives via signal) is a third
- For multi-DNA / cloned-DNA apps (acorn): the DHT model applies *per cell*, and `dna_info()` won't return clone DNAs at startup if `deferred: true`

### Host function cost model
- `must_get_agent_activity` is a *whole-chain fetch*. Never use it to read a single field like a joining timestamp. Cache once per agent or record the value at create time.
- `get_links` followed by `get` for each target is N+1; sometimes a single `get_links_and_load` style helper is better
- Cross-zome calls have measurable overhead; this is part of why single-zome is the default

### Validation context
- `ChainFilter` calculates a min..=max sequence range first, then iterates — it does not walk eagerly
- Inline validation merges the scratch space into the cascade; non-inline validation does not. This affects whether your `must_get_*` calls see the in-flight chain.
- `Op` variants (`Op::StoreRecord`, `Op::StoreEntry`, `Op::RegisterAgentActivity`, `Op::RegisterCreateLink`, etc.) — what each represents and which ones to handle for which checks
- `FlatOp<EntryTypes, LinkTypes>` via `op.flattened()?` is the 0.6+ pattern for dispatching validation by entry/link type
- **Per-op declarative cross-entry validation is a real middle ground** (vines pattern): dispatch on `Op`, leave most variants permissive, do real `must_get_*` work on `StoreEntry` / `RegisterCreateLink` only. Not every validation needs to be a state machine.
- **LinkTag-as-payload** is a valid but fragile pattern (vines): tags can carry msgpack-encoded payloads decoded during validation. The encoder used to construct the tag must match the decoder used to validate it; mixing `to_vec_named` vs default `decode` silently fails.
- **Singleton entries cannot be enforced from `validate`** (acorn `validation_rules.txt:51-56`). The author has to do a network fetch+count from the coordinator with an acknowledged race. The skill should know there is no on-chain singleton primitive.

### Zome architecture
- Default: single integrity zome + single coordinator zome per DNA
- **Principled splits** that the skill should recognize as good:
  - **Third-party zome reuse without entry-enum pollution** — re-export `hc_zome_profiles_integrity` (or similar) as its own zome rather than inlining its entries into your main `EntryTypes` enum (vines pattern)
  - **Migration-safe side-channel zomes** — a dedicated zome for original-author attribution, recorded as links so migrated data preserves authorship across DNA-hash changes (vines `authorship` pattern)
- **Anti-pattern splits** to flag: many small zomes with `hc_call_utils::call_local_zome()` between them and hardcoded `ZomeIndex(N)` ordinals (mewsfeed's pre-merge design)
- Cross-zome calls should use named imports, not numeric indices
- Wrapper structs that gate local-vs-network behavior (`ZomeFnInput<T> { local: bool }`, independently rediscovered in mewsfeed *and* acorn) are a valid pattern but easy to forget — losing the wrapper silently turns network lookups into local-only. **Watch for the asymmetry** — in acorn the wrapper is fetch-only; passing it on writes silently fails because the macro doesn't destructure it.

### Multi-DNA architectures
- Acorn pattern: a singleton DNA (`profiles`, `clone_limit: 0`) plus a per-workspace DNA (`projects`, `clone_limit: 999, deferred: true`) cloned on demand via `AppClient.createCloneCell()`
- Implication: `dna_info()` does not return clone DNAs at startup. Code that assumes "the DNA" exists at install time gives "cell not found"
- The DHT model applies per cell; agents only see data in cells they've joined
- Migration is much harder across cloned cells — see vines's `authorship` zome for the side-channel approach

### Serialization boundaries
- **Default serde enum shape pitfall:** Rust `pub enum E { A(T), B(T) }` surfaces on the TS side as `{A: ...}` variant-object tags. Three valid responses observed:
  1. Embrace it on TS side via wrapper-class materialization (vines)
  2. Coerce to bare strings via `UIEnum(pub String)` newtype with `#[serde(from/into = "UIEnum")]` (acorn)
  3. Coerce via hand-rolled custom `Serialize`/`Deserialize` impl (acorn `Profile::Status`)
  
  Acorn does both #2 and #3 in the same repo — the pattern was independently rediscovered. The skill should know this pitfall has a name.
- **Hashes-in-entries:** Acorn embeds `ActionHashB64` strings inside entry bodies for UI-side string comparison; vines uses raw `ActionHash` bytes and wraps on the UI side. Both valid; pick one and stay consistent.
- **`f64` vs `Timestamp`:** Acorn uses `f64` timestamps embedded in entries for JS-number compatibility; vines uses `Timestamp` (i64 microseconds) and the UI handles bigint coercion. The skill should not silently convert one to the other.
- **`#[serde(rename_all = "camelCase")]` is universal at the struct level** in modern apps, but it's a per-struct attribute — applying it crate-level is unusual and at least one repo (vines) deliberately leaves some inner structs in snake_case to be camelCased by codegen.
- **Custom `Serialize`/`Deserialize` impls on integrity entries are load-bearing.** Replacing them with `#[derive(...)]` to "clean up" silently changes the wire format. Watch for hand-rolled impls before "simplifying."

### Capability grants and signals
- `set_global_access()` in `init` is the typical authorization pattern (see unyt-app)
- Signals can come from `post_commit` (vines, mewsfeed) OR be emitted inline from coordinator extern functions via `send_remote_signal` (acorn). `post_commit` doesn't block the extern return; inline emission lets you send custom cross-entry signal variants like `OutcomeWithConnection`. Both are valid.
- **`post_commit` is required, not optional, when the app uses relaxed commits or commits asynchronously.** Under relaxed/async commits, the `ActionHash` returned by the synchronous extern call can be a *temporary* hash that gets rewritten when the commit actually lands. If you hold ActionHashes as resource IDs (in the UI, in other entries, in links), the temporary hash will go stale. `post_commit` gives you the real hash and lets you signal it to the UI before anyone notices. The alternative is to handle `source_chain_moved` errors at every read site and re-fetch when temporary hashes are rewritten — same problem, opposite end. The skill should know that the choice between these two is real, that "just use the synchronous return value" is wrong under relaxed/async commits, and that the right answer depends on whether the app uses ActionHashes as durable IDs.
- `#[hdk_extern(infallible)] fn post_commit(...)` is the 0.6 signature for the post_commit hook
- **Push-based fetch via signals** is a real architectural option (vines): fetch externs return `ExternResult<()>`, results arrive as `ZomeSignalProtocol::Entry` signals. The UI fires a fetch, awaits `()`, and waits for signals to populate state. Completely different shape from request-reply.
- **Ephemeral state via remote signals only** (acorn `RealtimeInfoSignal`, `EditingOutcomeSignal`): broadcasts that commit nothing. Used for presence, who-is-editing, etc. The skill should know that not every signal corresponds to a chain write.

### Test hygiene
- Sweettest is the only generation target. `await_consistency` (sweettest's equivalent of tryorama's `dhtSync`) must wait for *every* agent whose actions the assertions depend on, not just the writers.
- Sweettest tests live in a separate Rust crate, typically at `tests/` or a sibling `sweettest/` directory. The `TestEnv` fixture pattern (unyt-app) is a good multi-agent harness shape to learn from.
- For multi-agent scenarios, sync after every cross-agent state change, not just at the end
- **Two of the five surveyed apps (vines, acorn) ship no working tests.** When the skill bootstraps a sweettest harness in a project that has none, that's a from-scratch operation, not a "find the existing harness" operation.
- If the project has *only* dead 0.0.x-era tests (acorn pattern), the skill should not try to use them as patterns — they pin `hdk = "0.0.122"` and use pre-HDI types like `ValidateData`

---

## Patterns to present as choices, not prescriptions

These are places where the three reference apps diverged. The skill should describe the tradeoffs and let the developer pick, then enforce consistency once they have.

| Choice | Options | Trade-off summary |
|---|---|---|
| Coordinator return shape | (a) hash-only (`ActionHashB64`) (b) rich record (`AuthoredX` / `WireRecord<T>` / `EntryRecord<T>`) (c) signal-push (extern returns `()`, data arrives via signals) | (a) is least coupled to UI shape, requires UI to follow up; (b) is the convergent default — 4 of 5 apps do some flavor of this; (c) is fire-and-forget, requires the UI to be signal-driven from the start |
| State management | (a) `@holochain-open-dev/stores` async derived stores (b) hand-rolled Svelte runes / module state (c) Lit context (d) Redux + reducers fed by signals (acorn) (e) dvm/zvm/perspective architecture from `@ddd-qc/lit-happ` (vines) | The skill should detect what the project uses and stay in that idiom, not impose one |
| Local vs network strategy | (a) zome exposes both as separate externs (`get_x` + `get_x_local`) (b) zome only network, UI polls + caches (c) wrapper input struct with `local: bool` flag (d) `strategy: GetStrategy` field on input | **Why this matters: zero-arc nodes.** If the happ supports nodes that don't participate in gossip (e.g. mobile / phone / battery-conserving) and therefore don't have data locally for `get`/`get_links` to succeed quickly, **explicit exposure is critical** — the app needs to distinguish "I'm running zero-arc on a phone" from "I'm running full-arc on a desktop" and behave differently per call. If the happ only ever runs full-arc, the zome can pick and the choice is cosmetic. The skill should ask whether zero-arc deployment is in scope before defaulting |
| Validation depth | (a) minimal-permissive / no validate callback at all (b) per-op declarative cross-entry checks with chain walks (vines) (c) comprehensive state-machine (unyt-app) | (a) appropriate when the entry shape *is* the contract; (b) for cross-entry invariants without full state-machine semantics; (c) for protocols with chain-spanning invariants like ledgers |
| Zome split | (a) single integrity + single coordinator (b) split for third-party reuse without entry-enum pollution (c) split for migration-safe side channels | Default is (a). (b) and (c) are the only principled reasons to split. Splitting for "modularity" alone produces mewsfeed-era anti-patterns |
| Serialization strategy | (a) hand-mirrored TS types (b) hand-mirrored zod schemas with TS derived (c) Rust-to-TS codegen via `zits` | (a) is the simplest and most common; (b) gives you a runtime parser as a side effect; (c) is the only one that catches Rust↔TS drift automatically, but forces a materialize layer for hashes |
| Hashes in entries | (a) raw `ActionHash` bytes, wrap on UI side (b) `ActionHashB64` strings, string-compare on UI side | (a) keeps Rust-side hash math cheap; (b) makes the UI simpler but bakes a UI encoding into the DHT permanently |
| Signal emission | (a) from `post_commit` callback (b) inline from extern via `send_remote_signal` (c) from fetch externs as a fire-and-forget result channel (vines) | **Why `post_commit` matters: relaxed commits.** If the app uses **relaxed commits** OR commits asynchronously, the `ActionHash` returned by the synchronous extern call can be a *temporary* hash that gets rewritten when the commit lands. If you hold ActionHashes as resource IDs anywhere, you must use `post_commit` to get the *real* hash and signal that to the UI — otherwise your IDs go stale. The trade-off is symmetric: either use `post_commit` and get correct IDs, **or** handle `source_chain_moved` errors and re-fetch when temporary hashes get rewritten. (b) is for atomic cross-entry signals with the write; (c) is for push-based UIs. The skill should ask whether the app uses (or will use) relaxed/async commits before defaulting |
| Soft delete vs hard delete | (a) `trashed: bool` field, UI hides (b) actual `delete_entry` (c) cascade-delete custom extern that walks all related links (acorn `delete_outcome_fully`) | (a) preserves history; (b) interacts with DHT tombstone semantics; (c) is needed when an entry has many dependent links that would otherwise dangle |
| Architecture: single vs multi-DNA | (a) single DNA (b) multi-DNA: singleton + per-workspace cloned with `clone_limit: N, deferred: true` | (a) is far more common; (b) is the right shape when each "workspace" / "project" / "room" has its own privacy boundary and shouldn't share a DHT |

---

## Anti-patterns to flag

When the skill encounters these in existing code or is tempted to write them, it should warn:

1. Hardcoded zome ordinals across zomes (`ZomeIndex(1)`, `EntryDefIndex(0)` as constants)
2. Multiple small zomes with `call_local_zome()` between them, split for "modularity" rather than reuse/migration reasons
3. `must_get_agent_activity` used to read a single field
4. DNA-impacting changes bundled with UI/cache/test changes in one commit
5. UI code where a single missing-data error cascades into blank-state failure
6. Test code where `await_consistency` is called with a subset of relevant agents
7. Hardcoded `unwrap()` / `expect()` in zome production paths
8. Claude-authored validation that walks the chain instead of using a dedup link recorded at create time
9. Validation logic in coordinator zomes (must be in integrity)
10. Cross-zome calls that drop a wrapper struct controlling local-vs-network fetch
11. **Suggesting `msgpack` or `serde` version pins as a first response to a serialization error** (per Constraint F — almost always a phantom cause)
12. **"Cleaning up" custom `Serialize`/`Deserialize` impls** by replacing them with `#[derive(...)]` — the hand-rolled impl is almost always there to fix the default serde enum shape and removing it silently changes the wire format
13. **Treating commented-out modules as dead code** — `// pub mod validate;` in acorn-style codebases is intentional, not orphaned. Uncommenting cascades pre-HDI type errors throughout
14. **Sampling 0.0.x-era test files as patterns** — repos with `hdk = "0.0.122"` test pinning are using legacy `ValidateData`-based mock testing that won't compile against current HDI
15. **Removing "useless" placeholder entries** (vines `Bogus` in `authorship_integrity`) — they may be load-bearing for tools that require at least one entry type
16. **Wrapping write payloads in the same envelope as fetch payloads** when only fetches use the wrapper (acorn `ZomeFnInput<T>` asymmetry) — the macro doesn't destructure it on writes, silent failure
17. **Cloning a multi-DNA app's clone DNA at startup expecting `dna_info()` to return it** — `deferred: true` means it's not provisioned until the UI explicitly creates the cell
18. **Mixing link-tag encoders** (`to_vec_named` vs default `decode` vs `serde_json::to_vec`) when tags carry serialized payloads — silently fails validation
19. **Embedding a large base64-encoded asset in `dna.yaml` properties** — properties are checked in `genesis_self_check` and the encoded bytes are part of the DNA hash. Changing the asset breaks the hash.

---

## What the skill should explicitly NOT do

- **Not reteach** `create_entry` / `get` / `get_links` / `update_entry` syntax — those work
- **Not embed HDK API as authoritative** — runtime verify against docs.rs/hdk/<version>
- **Not assume `@holochain-open-dev`** libraries are present
- **Not pick one coordinator API shape** as "the" right way
- **Not generate generic Holochain tutorial content** — that's anti-goal #1 from the planning notes
- **Not push elaborate validation** onto every entry type by default
- **Not write a single bundled commit** when DNA + UI + test changes are all in scope

---

## Skill structure (Claude Code specific, v1)

**Length guidance (corrected 2026-04-11):** the prior-session "~150 lines" rule was made up. Official Claude Code guidance is **`SKILL.md` under 500 lines**, with detailed reference material in separate files linked via plain markdown (no special `references/` subdirectory required, no `@import` syntax). Skill bodies are **not eagerly loaded** — they only enter context when the skill is invoked. Referenced files are even lazier — only loaded when the body explicitly tells the agent to read them. **Compaction note:** during auto-compaction, the first ~5K tokens of each recently-invoked skill are preserved, so the order inside `SKILL.md` matters — hard tripwires and constraints belong near the top so they survive compaction; semantic depth and patterns can come later.

The structure below is illustrative — `skill-creator` should make the actual body-vs-references split during its own design pass. The 11-file split I drafted earlier was based on the bogus 150-line constraint and is probably over-aggressive; some of these may collapse back into the main body once the real ceiling is 500 lines.

```
ai-tools/skills/holochain-dev/
├── SKILL.md                              # main body (frontmatter + ~150 lines)
├── references/
│   ├── host-fn-cost-model.md             # "don't use must_get_agent_activity for one field"
│   ├── validation-context.md             # ChainFilter, scratch space, FlatOp, Op variants, the three validation styles
│   ├── eventual-consistency.md           # GetOptions, gossip, partial arcs, UI patterns, zero-arc nodes
│   ├── test-hygiene-sweettest.md         # await_consistency agent lists, timeouts, multi-agent fixtures, bootstrapping from scratch
│   ├── zome-architecture.md              # single-zome default, principled splits, anti-patterns
│   ├── coordinator-shape-choices.md      # hash-only / wrapper-record / signal-push tradeoffs
│   ├── multi-dna.md                      # singleton + cloned DNA pattern, deferred provisioning, dna_info caveats
│   ├── serialization-boundaries.md       # serde enum pitfall, zits codegen as default, diagnostic order
│   ├── signals-and-post-commit.md        # post_commit vs inline emission vs push-based fetch; relaxed-commit / source_chain_moved tradeoff
│   ├── ui-state-options.md               # @hod vs hand-rolled vs Lit context vs Redux vs dvm/zvm
│   └── anti-patterns-by-example.md       # the 19-item flagging list with concrete examples
└── scripts/
    └── detect.sh                         # subcommands: hdk-version | serialization | test-harness | dna-impact-paths | branch-protected
```

The main `SKILL.md` is the brain: trigger conditions, the hard tripwires, and pointers into the references. The references are the long-form semantic content that doesn't need to be loaded every invocation.

---

## Trigger conditions (draft)

The skill should activate when any of these are present in the current task:

- The conversation mentions writing or modifying a Holochain zome (integrity or coordinator)
- Files in the conversation match `**/dnas/**`, `**/zomes/**`, `**/*.dna.yaml`, `**/*.happ.yaml`
- The conversation mentions HDK / HDI / DHT / coordinator zome / integrity zome / sweettest / tryorama
- A Cargo.toml in the project depends on `hdk` or `hdi`
- A package.json depends on `@holochain/client` or `@holochain/tryorama`
- The user says "holochain", "DNA", "zome", "DHT" in a code-task context

---

## Resolved decisions

All open questions resolved 2026-04-11. Recording answers + their implications here so the skill brief can encode them directly.

### Q1 — DNA tripwire enforcement strength → **(c) context-aware**
The skill detects whether the project is on a `main`-like branch (hard stop on integrity-zome / dna.yaml edits) versus a dev/feature branch (warning only). When the user is getting warnings on `main` and finds them obstructive, the skill should explain that **moving to a dev branch suppresses the warnings** — that's the documented escape hatch, not a flag.

**Branch heuristic:** treat `main`, `master`, `production`, `release/*` as protected; everything else as a dev branch. Configurable via skill state if a project uses different conventions.

### Q2 — Reference material loading → **lazy**
The main `SKILL.md` body knows the references exist but doesn't load them by default. The skill routes to the relevant `references/<topic>.md` only when the current task needs that knowledge. With 11 reference files this is a clear win.

### Q3 — Detection scripts → **single file for now**
A single `scripts/detect.sh` (or equivalent) that takes a subcommand argument: `detect.sh hdk-version`, `detect.sh serialization`, `detect.sh test-harness`. Easier to maintain than three sibling files; can split later if any one grows.

### ~~Q4 — Sweettest vs tryorama~~ → **Resolved earlier: sweettest only**
Tryorama is being deprecated. Skill generates sweettest only; tryorama tests in existing repos may be read for pattern inspiration but never written.

### Q5 — `../holochain` local clone → **NO, docs.rs only**
Eric correction: my earlier "local `../holochain` for semantic deep-dives" lean was wrong. The skill should always go to `https://docs.rs/hdk/<version>` and `https://docs.rs/hdi/<version>` for API surface, signatures, AND semantic deep-dives. The local `../holochain` checkout is research-only context for *me* while authoring the skill — never referenced from inside the shipped skill.

This simplifies Constraint C: source-over-guessing means docs.rs, period. No fallback to local source.

### Q6 — Prescriptiveness → **(c) detect-and-stay-in-idiom, with (b) sensible-default fallback**
The skill detects the existing project's choices on each architecture axis (return shape, serialization strategy, signal placement, single-vs-multi-DNA, etc.) and stays in that idiom. Only when starting from a blank slate does it apply sensible defaults and only when the defaults conflict with project signals does it ask.

### Q7 — `hdk_crud` recommendation → **(c) stay neutral**
The skill does not recognize or push `hdk_crud`. It's too coupled to one third-party crate's opinions (`WireRecord<T>`, `ActionSignal<T>`, the `crud!` macro shape) and adopting it commits the project to a specific dependency's semantics. If a project already uses it, the skill works with what's there but doesn't extend its use; for new projects, the skill writes CRUD by hand following the project's chosen return shape.

### Q8 — Serialization default → **`zits` (Eric's fork)**
For new projects, the skill defaults to **`zits` codegen** for the Rust↔TS boundary. Eric is currently maintaining an updated fork in `/home/eric/code/metacurrency/holochain/zits` — that's the version the skill should target once it's ready, but for now the skill can reference upstream `zits` as the default.

**Implications:**
- New projects scaffolded by the skill should include a `scripts/ts-bindings.sh` (or equivalent) that runs `zits` against integrity + coordinator zomes
- The skill should know about the `zits` semantics that the vines research surfaced: hashes become `Uint8Array` aliases (forcing a materialize/dematerialize layer), enums emit the default serde external-tag shape directly, `#[feature(zits_blocking)]` is a load-bearing pseudo-attribute that selects `callBlocking` vs `call` in the generated proxy
- After any Rust type change in a `zits`-based project, the skill must trigger the bindings regeneration step before considering the change complete
- Existing projects using hand-mirrored TS or hand-mirrored zod get respected per Q6 (detect-and-stay-in-idiom); the skill does not migrate them to `zits` unilaterally
- **TODO when Eric's fork lands:** point the skill at the fork's repo URL/version, document any divergence from upstream `zits` semantics

### Q9 — No-tests-at-all case → **(b) write the change AND bootstrap a sweettest harness**
When invoked in a project with no working tests (vines pattern) or only orphaned 0.0.x-era tests (acorn pattern), the skill announces the harness bootstrap explicitly ("this project has no working sweettest harness — I'll add one alongside the change you asked for"), then writes both the zome change and a sweettest test of the new code in the same response. Refusing is too aggressive; silent bootstrap surprises the user.

### Q10 — `dna.yaml` properties tripwire → **YES, expand Constraint A's scope**
Constraint A (DNA-hash tripwire) now also fires on edits to `dna.yaml` and to any file referenced from `dna.yaml`'s properties. The vines pattern (base64-encoded SVG in properties, checked in `genesis_self_check`) means changing the bytes changes the DNA hash — same class of breakage as integrity-zome edits, just less obvious. The skill should treat any edit reaching the properties bytes as DNA-impacting.

---

## Brief for skill-creator (final)

All open questions resolved. The brief to feed to `Skill(skill: "skill-creator", args: ...)`:

> **Skill name:** holochain-dev
>
> **Trigger:** Activate when the user is working with Holochain zomes, DNAs, sweettest tests, or HDK/HDI APIs in any project that depends on `hdk` (Cargo.toml) or `@holochain/client` (package.json). Also activate on `dna.yaml` / `happ.yaml` files or any path under `**/dnas/**` or `**/zomes/**`.
>
> **What it enforces:**
> - **Context-aware DNA-hash tripwire** on integrity-zome edits, `entry/link/validate*` definitions, `dna.yaml`, and any file referenced from `dna.yaml`'s `properties`. Hard stop on protected branches (`main`, `master`, `production`, `release/*`); warning only on dev branches. When blocked, remind the user that moving to a dev branch suppresses warnings.
> - **HDK version detection + docs.rs verification** — read `Cargo.toml`/`Cargo.lock` for `hdk` and `hdi` versions; verify any HDK/HDI API usage against `https://docs.rs/hdk/<version>` and `https://docs.rs/hdi/<version>` before recommending. Never embed HDK API as authoritative. Never read from a local `../holochain` clone.
> - **Source-over-guessing for semantics** — for cascade behavior, scratch-space merging, `ChainFilter` semantics, validation context, `Op` variants, or any "what does this function actually do" question, look up docs.rs at the project's pinned version. Never recall from training data.
> - **Sweettest-only test generation.** If the project has no working sweettest harness (vines case) or only orphaned 0.0.x-era tests (acorn case), the skill announces "this project has no working sweettest harness — I'll add one alongside the change" and bootstraps it from scratch in the same response. TDD: any zome change ships with a sweettest test in the same turn.
> - **Coordinator vs integrity separation:** validation only in integrity zomes
> - **Strong typing:** TS uses strong types; Rust avoids `unwrap()`/`expect()` in zome production paths
> - **Serialization-boundary diagnostic inversion:** when serialization errors surface, the diagnostic order is (1) Rust side hasn't been recompiled (2) Rust↔TS type drift (3) wrong-side check (4) *last resort* msgpack/serde version. Never suggest pinning/bumping msgpack/serde as a first-line fix.
> - **Detect-and-stay-in-idiom** for the existing project's choices on: coordinator return shape, serialization strategy, signal placement (`post_commit` vs inline vs push-based), zome split, single-vs-multi-DNA, hashes-in-entries, ephemeral-state-via-remote-signals
> - **For new projects, default to:** single integrity + single coordinator zome per DNA; wrapper-record coordinator return shape; **`zits` codegen for the Rust↔TS boundary** (Eric is maintaining a fork in `/home/eric/code/metacurrency/holochain/zits`); minimal-permissive validation; `post_commit` signal emission **if** the project uses or will use relaxed/async commits, otherwise inline emission is fine
> - **Architecture questions to ask before defaulting** (these have load-bearing "why"s):
>   - **Local-vs-network exposure:** does the happ deploy to zero-arc nodes (mobile / battery-conserving)? If yes, the API must expose local-vs-network choice so the UI can adapt. If no, the zome can pick.
>   - **Signal emission via `post_commit`:** does the app use relaxed commits or commit asynchronously? If yes, `post_commit` is required to get the real `ActionHash` (synchronous extern returns can be temporary hashes that get rewritten); the alternative is handling `source_chain_moved` errors at every read site.
> - **No Claude footers in commits**
>
> **Reference material location:** `references/` subdirectory, lazily loaded — main body routes to specific reference files when the task needs them.
>
> **Detection logic:** single `scripts/detect.sh` with subcommands (`hdk-version`, `serialization`, `test-harness`, `dna-impact-paths`, `branch-protected`).
>
> **Anti-goals:**
> - Do not generate Holochain tutorial content
> - Do not embed HDK API as authoritative — always verify against `docs.rs/hdk/<version>` and `docs.rs/hdi/<version>`
> - Do not read from local `../holochain` clones — docs.rs only
> - Do not assume `@holochain-open-dev` is present (it's a spectrum: whole-zome reuse → partial-with-fork → none)
> - Do not assume single-zome / single-DNA — both are defaults but principled splits exist (third-party reuse without entry-enum pollution; migration-safe side channels; multi-DNA with deferred-clone provisioning)
> - Do not recognize or push `hdk_crud` — it's too coupled to one third-party crate's opinions
> - Do not "clean up" custom `Serialize`/`Deserialize` impls, commented-out validate modules, placeholder entries, pseudo-feature attributes (`#[feature(zits_blocking)]`), or crate-level `#![allow(...)]` statements — these are often load-bearing
> - Do not suggest msgpack/serde version pins as a first response to serialization errors
> - Do not migrate existing projects from hand-mirrored TS or zod to `zits` unilaterally — respect the project's chosen serialization strategy
>
> **Reference exemplars (for skill to read patterns from when needed):** unyt-app (highest weight, most production-tested), vines, acorn-happ, emergence (mid-weight), dino-adventure (high signal-per-line, small scope) — all sibling clones in `/home/eric/code/metacurrency/holochain/`. Illustrative only, never embedded as content.
>
> **Authoritative background sources** the skill should consult for "how is this supposed to work" questions (in addition to docs.rs for API surface):
> - **Holochain Build Guide:** `https://developer.holochain.org/build` — the canonical developer-facing guide. Use it for conceptual framing, recommended workflows, and "what is the supported way to do X" questions. Treat it as authoritative for build-time concerns and developer-workflow patterns.
> - **Scaffolding tool source:** `/home/eric/code/metacurrency/holochain/scaffolding` (and the `templates/` directory therein). Use it as a source of truth for **base-layer validation patterns** — the scaffolder generates a decent starting point for validation that the skill can lift directly when bootstrapping new entry/link types. **Caveat:** the scaffolder does not go very far on the UI side; treat its UI output as a *starting skeleton only*, not as a pattern to extend. For zome-side validation it's a good starting point; for UI architecture, fall back to the reference exemplars and to the patterns table in the design doc.
>
> **TODO when Eric's `zits` fork lands:** point the skill at the fork's repo URL/version, document any divergence from upstream `zits` semantics surfaced in the vines research.
>
> **Skill body:** Target under 500 lines per official Claude Code guidance (NOT the ~150 from the prior planning notes — that was a made-up number). Order content so that hard tripwires and constraints come first; semantic depth later. Compaction preserves the first ~5K tokens of recently-invoked skills, so high-priority content benefits from being near the top. The body-vs-references split is `skill-creator`'s call; the file list above is illustrative.
