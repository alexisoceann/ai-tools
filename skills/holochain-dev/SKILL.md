---
name: holochain-dev
description: >-
  Use this skill for any Holochain development task — writing or modifying integrity zomes, coordinator zomes, DNA bundles, sweettest tests, HDK/HDI Rust code, or `@holochain/client` TypeScript code. Activate even when the user does not say "Holochain" explicitly: any reference to a hApp, DHT, conductor, sweettest, tryorama, integrity zome, coordinator zome, action hash, link type, entry type, validation callback, or files matching `**/dnas/**`, `**/zomes/**`, `*.dna.yaml`, `*.happ.yaml`, or any `Cargo.toml` depending on `hdk`/`hdi` should trigger this skill. Holochain APIs evolved through alpha versions before 2025 and most LLM training data is full of obsolete patterns that look correct but don't compile against current HDK; this skill exists to keep the model from confidently producing broken code. Use it for: avoiding accidental DNA-hash changes, debugging Rust↔TypeScript serialization errors, choosing zome architecture, writing tests under DHT eventual consistency, and any task where "what does this Holochain function actually do?" matters.
---

# holochain-dev

You are working on a Holochain application. Your job is to be useful at the **semantic** layer — caching, eventual consistency, zome topology, validation context, the Rust↔TypeScript boundary, and host function cost — not at the syntactic layer. Claude can already write plausible HDK syntax. The reason this skill exists is that the plausible-looking syntax in your training data is probably from a pre-1.0 alpha version and won't compile, and the *semantics* of the working code are not obvious from the signatures.

> **Current target versions (as of skill authoring):** **HDK `0.6.1-rc.5`**, **HDI `0.7.1-rc.5`**, **holochain `0.6.1-rc.7`**. These are release candidates expected to become 0.6.1 final imminently. Any HDK API you remember from training data that targets `0.0.x`, `0.1.x`, `0.2.x`, `0.3.x`, `0.4.x`, or `0.5.x` is **almost certainly wrong** for current projects. Always read the project's actual `Cargo.toml` for its pinned version and verify against `https://docs.rs/hdk/<that-version>`.

**Read this whole file before starting any Holochain task.** Then read the relevant `references/<topic>.md` files when the task lands on that topic — pointers are at the bottom.

---

## Why training data is poisoned

Holochain's API evolved through many breaking alpha versions before stabilising. Most LLM training data contains code from those alpha versions. The result: your first instinct for HDK syntax, validation patterns, `Op` variants, `GetOptions` shapes, macro names, and even crate names is **probably wrong** in ways that compile-look-plausible but break against current versions. Examples of things that changed under alpha:

- `ValidateData` (old) → `Op` / `FlatOp<EntryTypes, LinkTypes>` (current)
- `Header::Update` (old) → `Action::Update` (current)
- `validate_create_entry_<name>(data: ValidateData)` (old) → `validate(op: Op)` dispatcher (current)
- `GetOptions::content()` / `GetOptions::latest()` (old) → `GetOptions::local()` / `GetOptions::network()` (current)
- `WasmError::Guest(_)` (old) → `wasm_error!()` macro (current)
- `hdk = "0.0.122"` is the pre-HDI legacy world; `hdk = "0.6+"` (paired with `hdi = "0.7+"`) is the current world. The two are not source-compatible. **As of writing, the current target is `hdk = "0.6.1-rc.5"` and `hdi = "0.7.1-rc.5"` (release-candidate, expected to be 0.6.1 final imminently).** Anything you remember from training data that doesn't match the project's pinned version is suspect — verify against `https://docs.rs/hdk/<project-version>` before using it.

**Operating rule:** never recall an HDK or HDI API from training data. Always verify against `https://docs.rs/hdk/<version>` or `https://docs.rs/hdi/<version>` at the project's pinned version. See "Hard tripwires → B" below.

---

## First-invocation orientation (do this once per project per session)

When this skill first activates, run a cheap orientation pass to establish context. Most decisions downstream depend on these answers.

1. **Detect HDK/HDI version.** Read the project's `Cargo.toml` and/or `Cargo.lock` to find the pinned `hdk` and `hdi` versions. State them explicitly in your reply: *"This project targets HDK X.Y.Z, HDI X.Y.Z."* Every subsequent API claim must be verified against `https://docs.rs/hdk/X.Y.Z` and `https://docs.rs/hdi/X.Y.Z`.

2. **Detect git branch.** Run `git branch --show-current`. Treat `main`, `master`, `production`, and any `release/*` branch as **protected** for purposes of the DNA-hash tripwire (see Tripwire A). Anything else is a dev branch where DNA changes are warned-about but not blocked.

3. **Detect nix dev shell.** Look for a `flake.nix` at the project root. If present, then **every command that needs Rust, cargo, holochain CLI, or sweettest must be run inside the dev shell**: prefix with `nix develop -c ` (e.g. `nix develop -c cargo test`, `nix develop -c sweettest`). Outside the shell those commands fail with cryptic dependency errors that are easy to misdiagnose. Never debug a "command not found" or "cargo can't find dependency" error by installing things globally; the answer is almost always "you forgot `nix develop -c`."

You do not need to run all the deeper detection (serialization strategy, test harness, single-vs-multi-DNA, zome topology) up front. Defer those until the current task touches them — see "Detect-and-stay-in-idiom" below.

---

## Hard tripwires (read these every invocation)

These are the failure modes that come up most often. They are ordered roughly by how expensive a mistake is.

### A. DNA-hash tripwire — context-aware

Touching certain files changes the DNA hash. A new DNA hash means the network is a different network: peers can't talk to each other, deployed apps need migration, action hashes already in use point at nothing. Users get *very upset* when this happens by surprise.

**The following changes affect the DNA hash:**
- Any file under `**/zomes/integrity/**`
- Any `#[hdk_entry_types]` / `#[hdk_link_types]` / `#[hdk_extern] fn validate(...)` definition
- Any `dna.yaml` file
- Any file referenced from a `dna.yaml`'s `properties` block — including embedded images, schemas, or any byte payload checked in `genesis_self_check`. The bytes are part of the hash.

**How to behave:**
- **On a protected branch** (`main`, `master`, `production`, `release/*`): refuse to write DNA-impacting changes. Explain *what* the change would do to the DNA hash. Then offer the user two paths: (1) explicit per-turn opt-in if they truly want this, or (2) move to a dev branch where the warnings stop. **Always include the second option** — it's the documented escape hatch, not a flag.
- **On a dev branch:** write the change but always surface the DNA-hash impact in your response. Phrase it like "this is a DNA-hash-impacting change because..." so the user can catch it if you misjudged.
- **Always ask first**: can the same fix be made in coordinator code, UI code, signal handling, or caching? If yes, prefer that path. The DNA tripwire is about avoiding *unnecessary* breakage, not about banning legitimate integrity work.

### B. HDK/HDI API claims must be verified against docs.rs

Never produce HDK or HDI code based on training data alone. For any function call, type name, macro, or attribute you're about to write:

1. Check the project's pinned version (from orientation step 1).
2. Look up `https://docs.rs/hdk/<version>` or `https://docs.rs/hdi/<version>` for the function.
3. If you can't find it on docs.rs at that version, **the function does not exist** at that version. Don't guess a replacement; ask.

**Never read from a local `../holochain` clone**, even when one is present. Local clones may be a different version, may include unmerged changes, won't be present on other developers' machines, and create false confidence. docs.rs is the only authoritative source the skill cites.

The high-risk functions where guessing is historically most wrong — always look these up, never recall:

- `must_get_agent_activity` and `ChainFilter` (range semantics, scratch space interaction)
- `get`, `get_links`, `get_details` with `GetOptions::local()` vs `GetOptions::network()`
- `must_get_action`, `must_get_entry`
- `validate(op: Op)` dispatch and `op.flattened()?` pattern
- `set_global_access`, `set_zome_call_access`, capability grants in `init`
- `#[hdk_extern]`, `#[hdk_extern(infallible)]`, `#[hdk_entry_types]`, `#[hdk_link_types]`

### C. Source-over-guessing for semantic questions

When the question is "what does this function actually *do*?" — not "what's its signature?" — the answer is even more likely to be wrong from training data than the signature is. Cascade behavior, scratch-space merging, `ChainFilter` range calculation, validation context (inline vs non-inline), `Op` variant meanings, gossip lag — all of these have changed across versions and your model recall is unreliable.

> **Two specific Holochain facts that are easy to miss and worth knowing up front:**
>
> 1. **AgentPubKey-as-EntryHash trick (joining timestamp lookups).** AgentPubKey is the *only* Holochain entry type whose `EntryHash` equals the pubkey bytes themselves — no hash transformation. An agent's third genesis entry IS their AgentPubKey, written atomically with the other two genesis entries within microseconds of joining. So `get(EntryHash::try_from(agent_key)?)` returns that genesis record directly, and the record's timestamp is the joining time. **This is the right answer for "I need an agent's joining timestamp" — O(1), no cache, no chain walk, no `must_get_agent_activity`.** Better than caching, better than recording-at-create-time. See `references/host-fn-cost-model.md`. Verify the exact `EntryHash::try_from` constructor against `https://docs.rs/hdi/<version>` because conversion APIs have shifted.
>
> 2. **Scratch space and the "perspective gap" myth.** A common framing of Holochain validation says "inline validators see scratch, non-inline (peer) validators don't, so they might disagree." That framing is **wrong** in the way that matters. Within a zome call, scratch accumulates pending commits — then all commits are written **atomically** to the source chain before any peer can see anything via gossip. Peers always see the same state the author validated against. There is no perspective gap. What scratch is actually for: letting commit B see commit A from the same call via the cascade. The genuinely non-deterministic case is validating against state outside the chain (coordinator-memory, UI-memory) — *not* scratch-space visibility. See `references/validation-styles.md`.

For semantic questions, look it up at docs.rs. If the user has pasted a confusing error or behavior, **ask to verify against the actual host-function semantics at the project's HDK version** before proposing a fix. Don't reach for plausible-sounding explanations.

A good prompt to yourself before every claim: *"Could I be wrong about this because the semantics changed in an alpha version?"* If yes, look it up.

### D. Sweettest is the only test target

Tryorama is being deprecated. Generate test code using **sweettest** (Rust) only. If the project has existing tryorama tests in TypeScript, you may *read* them to understand what scenarios the team needs to cover and how multi-agent flows are structured — but new test code is always sweettest in Rust.

**If the project has no working sweettest harness** (vines pattern: nothing) **or only orphaned legacy tests** (acorn pattern: pinning `hdk = "0.0.122"`, using `ValidateData`, won't compile against current HDI):
- Announce explicitly: *"This project has no working sweettest harness — I'll add one alongside the change you asked for."*
- Then in the same response, write both the zome change AND a sweettest test of the new code.
- Don't refuse the change. Don't silently bootstrap. Announce, then do both.

Sweettest tests live in their own Rust crate, typically at `tests/` or a sibling `sweettest/` directory. Multi-agent fixtures (the `TestEnv` pattern from unyt-app) are a good shape to learn from. Run the tests via `nix develop -c cargo test` or whatever the project's nix-wrapped command is — see references/sweettest-bootstrap.md.

### E. Serialization-boundary diagnostic inversion

When a serialization error appears at the Rust↔TypeScript boundary — msgpack decode failures, "invalid type" errors, unexpected `null`/`None` deserializations — your first instinct is probably wrong. The standard order of suspects is:

1. **The Rust side hasn't been recompiled** since the last type change. Is the running wasm stale? Has the DNA been rebuilt? Is the conductor serving the old wasm? Run `nix develop -c cargo build --release --target wasm32-unknown-unknown` or whatever the project uses, and reload.
2. **The Rust type and the TS type have drifted.** Is the TS type definition still aligned with the Rust struct? Any `#[serde(rename = ...)]`, `#[serde(tag = ...)]`, `#[serde(rename_all = "camelCase")]` on the Rust side that the TS side doesn't mirror? In a `zits`-codegen project, has `scripts/ts-bindings.sh` been re-run?
3. **Wrong-side check.** Are you inspecting the TS side for a bug that's actually on the Rust side, or vice versa?
4. **(Essentially never.)** msgpack or serde version mismatches. **In real-world projects this is almost never the cause** — the only documented case is when an LLM generated the project's `package.json` and picked an obsolete `@msgpack/msgpack` version that disagrees with what `@holochain/client` ships internally. In that one specific case, the fix is `npm ls @msgpack/msgpack` and aligning with `@holochain/client`'s version, **not** bumping arbitrarily. If a human developer hand-wrote the package.json, you are almost certainly chasing the wrong cause and should walk back to step 1.

The skill must **never** suggest pinning or bumping msgpack / serde versions as a first-line fix. That's the reflex to invert. This is also the order to *teach* the user when they're debugging.

---

## Two architecture questions to ask before defaulting

These are decisions whose right answer depends on a "why" that the user knows and you don't. Don't silently pick.

### Q1 — Does this hApp deploy to zero-arc nodes?

A node running with arc size 0 — typical on phones, battery-conserving devices, or anything that doesn't want to participate in gossip — has **no local data**. Every `get`/`get_links` must go to the network and may be slow or fail outright. A node running full-arc (typical desktop) has the data locally and can answer cheaply.

- **If yes (zero-arc is in scope):** the API surface MUST expose local-vs-network choice explicitly so the UI can adapt. The phone says "give me what you have locally, fast"; the desktop says "fetch from network." Patterns: a `local: bool` wrapper on every fetch input, separate `get_x` / `get_x_local` externs, or a `strategy: GetStrategy` field.
- **If no:** the zome can pick its preferred strategy and the choice is cosmetic.

Ask before designing fetch APIs. Default to "yes, expose it" if the user can't say.

### Q2 — Does this hApp use relaxed commits or commit asynchronously?

Under relaxed/async commits, the `ActionHash` returned synchronously by an extern can be a *temporary* hash that gets rewritten when the commit actually lands. If anywhere in the system holds ActionHashes as **durable resource IDs** — in the UI, in other entries, in links — those temporary hashes will go stale.

- **If yes (relaxed/async commits in scope):** signal emission must come from the `post_commit` callback so the *real* hash gets pushed to the UI before anyone tries to use the temporary one. The alternative is handling `source_chain_moved` errors at every read site and re-fetching when temporary hashes are rewritten — pick one, both work, but you have to pick.
- **If no (synchronous commits only):** signals can be emitted inline from coordinator extern functions via `send_remote_signal`, which is simpler and lets you bundle custom cross-entry signal variants atomically with the write.

Ask before designing the signal layer. Default to `post_commit` if the user can't say (it's the safer default — `post_commit` works with both modes; inline emission breaks under relaxed commits).

---

## Detect-and-stay-in-idiom

When you join an existing project, **detect what choices the project has already made and stay in that idiom**. Don't impose a different style unless the user explicitly asks for migration. Axes to detect when the task touches them:

- **Coordinator return shape** — hash-only / wrapper struct (`AuthoredX` / `WireRecord<T>` / `EntryRecord<T>`) / signal-push (extern returns `()`, data arrives as signals)
- **Serialization strategy** — hand-mirrored TS types / hand-mirrored zod schemas / `zits` codegen (look for `scripts/ts-bindings.sh` or `zits` in `package.json`)
- **Signal placement** — `post_commit` callback / inline `send_remote_signal` / fetch-as-signal
- **Zome topology** — single integrity+coordinator (most common) / split for third-party reuse / split for migration-safe side channels
- **Single vs multi-DNA** — most common: single. Multi-DNA exists when each "workspace"/"project"/"room" needs its own DHT (e.g. acorn's `projects` cloned per workspace with `clone_limit: 999, deferred: true`)
- **Hashes-in-entries** — raw `ActionHash` bytes (most common, cheap Rust-side comparisons) / `ActionHashB64` strings (UI-friendly but bakes encoding into the DHT permanently)
- **Validation depth** — minimal-permissive (most common) / per-op declarative cross-entry (vines pattern) / comprehensive state-machine (unyt-app pattern). Only push elaborate validation when the protocol genuinely needs it.

For *new* projects starting from a blank slate, sensible defaults are: single integrity + single coordinator zome per DNA; wrapper-record return shape; `zits` codegen for the Rust↔TS boundary; minimal-permissive validation; `post_commit` signals (the safer default per Q2). But these defaults are *only* applied when there is no existing project signal — never override the user's existing choices.

See `references/architecture-choices.md` for deeper notes on each axis with pros and cons.

---

## Anti-patterns to flag

If you see these in existing code, or are tempted to write them, stop and warn:

1. **Hardcoded zome ordinals** like `ZomeIndex(1)` or `EntryDefIndex(0)` as constants for cross-zome link queries. Use named imports.
2. **Many small zomes with `call_local_zome()` between them**, split for "modularity." This is an old scaffolder pattern; the current direction is single-zome unless you have a *principled* reason to split.
3. **`must_get_agent_activity` used to read a single field** (e.g. joining timestamp). It's a whole-chain fetch. Cache once per agent or record the value at create time as its own link/entry.
4. **DNA-impacting changes bundled with UI/cache/test changes in one commit.** Always split.
5. **UI code where a single missing-data error cascades into blank-state failure.** Eventually-consistent reads can return `None`; partial-result paths are mandatory.
6. **`unwrap()` / `expect()` in zome production paths.** Use `?` and proper error returns.
7. **Validation that walks the chain via `must_get_agent_activity` for dedup of the author's own prior entries**, when a per-author dedup link recorded at create time would do the same job in O(1). **But know what dedup links actually do**: they're an O(1) lookup *for that author's history*, but they do not prevent cross-agent duplicates (different agents can race to create the same entry; integrity validation cannot prove non-existence). EntryHash collision dedupes at the *entry* level (the entry exists once on the DHT) but duplicate Create *actions* with different timestamps and `prev_action`s still propagate. For cross-agent dedup, the only mechanism is a coordinator-side pre-check, which is best-effort and racy. Don't oversell dedup links as a complete solution to duplication.
8. **Validation logic in a coordinator zome.** Validation belongs in integrity only.
9. **Cross-zome calls that drop a wrapper struct** controlling local-vs-network fetch (the silent local-only failure mode).
10. **Suggesting `msgpack` or `serde` version pins as a first response to a serialization error.** This is the inverted reflex from Tripwire E. Almost always a phantom cause.
11. **"Cleaning up" custom `Serialize`/`Deserialize` impls** by replacing them with `#[derive(...)]`. The hand-rolled impl is almost always there to fix the default serde enum shape; removing it silently changes the wire format.
12. **Treating commented-out modules as dead code** and uncommenting them. In acorn-style codebases `// pub mod validate;` is intentional and uncommenting cascades pre-HDI errors.
13. **Sampling test files that pin `hdk = "0.0.122"`** as patterns for new tests. Those are pre-HDI legacy mocks that won't compile against current HDI.
14. **Removing "useless" placeholder entries** like vines's `Bogus` private entry. Some tools require at least one entry type per zome.
15. **Wrapping write payloads in the same `{input, local}` envelope as fetch payloads** when only fetches use the wrapper. Silent failure on writes.
16. **Assuming `dna_info()` returns clone DNAs at startup.** With `deferred: true` they're not provisioned until the UI explicitly creates the cell.
17. **Mixing link-tag encoders** (`to_vec_named` vs default `decode` vs `serde_json::to_vec`) when tags carry serialized payloads. Encoder and decoder must match exactly.
18. **Embedding a large base64 asset in `dna.yaml` properties.** Properties bytes are part of the DNA hash via `genesis_self_check`. Changing the asset breaks the hash.
19. **Generating tryorama test code.** Tryorama is being deprecated; sweettest only.

---

## Anti-goals (things this skill should NOT do)

- **Don't reteach** `create_entry` / `get` / `get_links` / `update_entry` syntax — those work in current Claude.
- **Don't generate generic Holochain tutorial content.** Users have `https://developer.holochain.org/build` for that.
- **Don't embed HDK API surface as authoritative** — always verify against docs.rs at the project version.
- **Don't read from local `../holochain` clones** — docs.rs only.
- **Don't assume `@holochain-open-dev`** is present. It's a spectrum: some projects use the full stack, some use one or two zomes, some don't use it at all.
- **Don't recognize or push `hdk_crud`.** It's a third-party crate with strong opinions; if a project uses it, work with it, but don't extend or recommend it.
- **Don't migrate existing projects** from one serialization strategy / return shape / validation style to another unless the user explicitly asks.
- **Don't push elaborate validation onto every entry type** by default.
- **Don't write a single bundled commit** when DNA + UI + test changes are all in scope. Split them.
- **Don't suggest msgpack/serde version pins** as a first response to serialization errors.

---

## Authoritative background sources

When the user asks "how is this supposed to work?" and the answer isn't obvious from the in-repo code, consult:

1. **`https://docs.rs/hdk/<version>` and `https://docs.rs/hdi/<version>`** — version-pinned API truth. Always cite the version.
2. **Holochain Build Guide:** `https://developer.holochain.org/build` — canonical developer-facing guide for conceptual framing, recommended workflows, and "what is the supported way to do X" questions. Authoritative for build-time concerns and developer-workflow patterns.
3. **Scaffolding tool source:** `https://github.com/holochain/scaffolding` (and the `templates/` directory therein). The scaffolder generates a decent **base-layer validation** starting point that you can lift directly when bootstrapping new entry/link types. **Caveat:** the scaffolder's UI output is a starting skeleton only, not a pattern to extend. For UI architecture, fall back to the reference exemplars below.
4. **Reference exemplar repos** (well-developed real Holochain apps, public on GitHub, useful for "how do real apps do X"):
   - `https://github.com/lightningrodlabs/vines` — push-based signals, `zits` codegen, principled multi-zome split, elaborate validation, no automated tests
   - `https://github.com/lightningrodlabs/acorn` — multi-DNA with cloned per-workspace DNAs, hand-mirrored zod schemas, inline `send_remote_signal`, custom cascade-delete externs
   - `https://github.com/holochain-apps/emergence` — solid mid-complexity exemplar, uses `@holochain-open-dev` packages, batched eventual-consistency polling
   - `https://github.com/holochain/dino-adventure` — small but built by a senior Holochain developer; high signal-per-line. Good for clean-pattern reference, less for architectural scope

Local sibling clones of these repos may be present in some developer setups but you should not depend on them — always cite the GitHub URL.

---

## Where to read more (lazy-loaded references)

When the current task lands in one of these areas, read the referenced file before proceeding. Don't read all of them at once — they're here so the main body can stay tight.

- **DNA hash impact, in detail** → `references/dna-tripwire.md`
- **Eventual consistency, gossip, zero-arc nodes** → `references/eventual-consistency.md`
- **Host function cost model** → `references/host-fn-cost-model.md`
- **Validation styles and `Op` dispatch** → `references/validation-styles.md`
- **Zome architecture: single vs split, multi-DNA** → `references/zome-architecture.md`
- **Serialization boundaries: serde enum pitfall, three strategies, `zits`** → `references/serialization-boundaries.md`
- **Signals, `post_commit`, relaxed commits** → `references/signals-and-post-commit.md`
- **Sweettest harness bootstrap and test hygiene** → `references/sweettest-bootstrap.md`
- **Architecture choices with pros and cons (the full menu)** → `references/architecture-choices.md`
- **Running things in the nix dev shell** → `references/nix-develop.md`
