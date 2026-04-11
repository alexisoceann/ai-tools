# Vines & Acorn App-Design Patterns — Research Findings

Scope: two additional reference apps, orthogonal to the earlier round
(unyt-app / dino-adventure / emergence).

Acorn has three candidate directories:
- `acorn/` — webapp shell (React UI, Electron/Weave packaging), no zome code.
- `acorn-hc/` — 2020 artifact (`Cargo.toml` 2020-12, pre-HDI). Ignored.
- `acorn-happ/` — current zome code (`Cargo.toml` 2024-11). **Surveyed
  as "acorn" below.** Zod models live in `acorn/zod-models`, the
  React/Redux UI in `acorn/web`.

The two apps are each "well developed" in very different ways. Acorn is
several years old, mid-migration from legacy HDK to HDI (integrity
lib.rs is modern, the rest is mostly 0.0.122-era). Vines is recent (0.6)
and sits atop a third-party HDK framework (`zdk` / `@ddd-qc/lit-happ`)
authored by the same person who wrote the app.

---

## vines

### 1. DNA layout

Single DNA (`dThreads`, role `rVines`), **three integrity + four
coordinator zomes** — the largest fan-out observed in either round
(`vines/dna/workdir/dna.yaml:29-53`).

- `threads_integrity` — core domain (beads, threads, PPs, moderation).
  8 entry types, 20 link types
  (`vines/dna/zomes/threads_integrity/src/lib.rs:46-125`).
- `profiles_integrity` — one-line re-export of
  `hc_zome_profiles_integrity` from holochain-open-dev
  (`profiles_integrity/src/lib.rs:1`; git dep at
  `vines/Cargo.toml:29`). Coordinator is a custom fork
  `profiles_alt_coordinator` from the author's zdk (`Cargo.toml:30`).
- `authorship_integrity` — structurally link-only; declares a single
  `Bogus` private entry because "some holochain tools will not work
  properly if there are none"
  (`authorship_integrity/src/lib.rs:27-40`). Real content is two link
  types (`Target`, `Author`) used to preserve original author
  attribution across DNA migrations.
- `zPathExplorer` — 3-line re-export of a generic path browser from zdk
  (`path_explorer/src/lib.rs`). No domain content; lets the UI
  introspect any path tree.

This contradicts the prior-research "one integrity + one coordinator
per DNA" norm. Vines splits zomes for composition/reuse.

### 2. Entry & link type conventions

- `#[hdk_entry_helper]` + `#[serde(rename_all = "camelCase")]`
  struct-level (`threads_integrity/src/entries.rs:46-96`).
- Explicit `required_validations = 3` and
  `visibility = "public"|"private"` on every `#[entry_type]`
  (`threads_integrity/src/lib.rs:47-62`).
- **Entry-polymorphism**: `ThreadsEntry::{AnyBead, EntryBead, TextBead,
  EncryptedBead}` are four separate entry types sharing an inner
  `Bead { pp_ah, prev_bead_ah }` struct (`entries.rs:37-89`). A
  `BaseBeadKind` non-entry enum wraps them at runtime
  (`entries.rs:5-32`).
- Anchors as named string consts (`lib.rs:28-35`).
- Beads get **two parallel time indexes** on commit: per-thread
  `ThreadTimePath` and global `GlobalTimePath`, both via
  `time_indexing::index_item()` from zdk
  (`threads/src/beads/index_bead.rs:7-33`). Time buckets are path
  segments.
- **LinkTag as serialized payload** — the single most distinctive
  link-tag pattern in either round. The `Ban` link-tag carries a
  msgpack-encoded `Vec<ActionHash>` of prior flag-links, and
  validation deserializes and walks each referenced link to verify
  quorum (`validate_link.rs:66-89`). The authorship zome similarly
  packs an `AuthorshipLog` struct via a `obj2Tag()` helper
  (`authorship_coordinator/src/authorship.rs:47-59`).
- Update chains are avoided for most types; PP title updates go via a
  separate `update_pp_title` extern rather than `update_entry`
  (`threads/src/participation_protocols/pp_title.rs`).

### 3. Validation patterns

The elaborate-validation exemplar of the pair. `threads_integrity`
dispatches on `Op` and does cross-entry, chain-walking work on
`StoreEntry` and `RegisterCreateLink`; other variants are permissive
(`threads_integrity/src/validate.rs:7-26`).

What validation actually checks:

1. **PP invariants** (`validation_app_entry.rs:39-88`): at-least-one
   moderator if any flags, non-zero rate limit, at-least-one message
   type, max>min for text length and file size.
2. **Per-bead capability check**
   (`validation_app_entry.rs:91-156`): `must_get_action` +
   `must_get_entry` on the referenced PP, `allowed_agents` check, then
   rate-limit.
3. **Rate limiting via chain walk**
   (`validation_app_entry.rs:159-237`): uses `must_get_agent_activity`
   with `ChainFilter { chain_top: prev_action, limit_conditions:
   UntilTimestamp(since) }` to fetch the author's recent chain, filter
   to bead-create actions, and count beads targeting this PP. **Only
   instance across all five surveyed apps of validation walking the
   author's source chain.**
4. **Bead-type matches PP rules** (`:128-152`) — `AnyBead` rejected if
   `!can_wal`, etc.
5. **Text content check** (`:275-301`): length bounds,
   case-insensitive banned-words filter.
6. **Moderation link validation** (`validate_link.rs:22-89`):
   `Flagged` requires author-is-moderator of the PP. `Banned` requires
   moderator AND the link-tag decodes to a `Vec<ActionHash>` of prior
   flag-links, whose count meets `pp.moderation.allowed_flags`, whose
   target-authors all match the ban target, and whose base addresses
   all match the same PP. Each referenced link is fetched via
   `must_get_action`.

Style: no state machine. Per-op declarative checks that pull
dependencies through `must_get_*` and chain walks. Acknowledged gaps in
comments: cross-zome file-entry validation (`:239-242`), banned-member
check (`:107`).

### 4. Coordinator API shape

Radically different from any of the prior three apps. Fetch externs
**return `ExternResult<()>`** and publish results as signals:

`threads/src/beads/fetch_beads.rs:7-38`:
```
#[hdk_extern]
pub fn fetch_beads(input: GetManyAhInput) -> ExternResult<()> {
   // ... build EntryPulse per record, emit ZomeSignalProtocol::Entry(pulse)
   emit_zome_signal(pulses)?;
   Ok(())
}
```

The UI calls `fetch_beads`, awaits `()`, and separately listens for
signals. `ZomeSignalProtocol` / `EntryPulse` / `LinkPulse` are
standardized in zdk's `zome_signals` crate. `post_commit` calls
`attest_post_commit::<ThreadsEntry, ThreadsLinkType>()`
(`threads/src/callbacks.rs:44-52`), auto-emitting signals for local
commits.

Write externs return **tuples**, not structs:
- `publish_text_bead(TextBead) -> (ActionHash, String, Timestamp)`
  where the tuple is `(bead_ah, global_time_anchor, bucket_time)`
  (`beads/text_bead.rs:10-17`).
- `ascribe_app_entry(ActionHash) -> (Timestamp, AgentPubKey, String)`
  (`authorship_coordinator/src/authorship.rs:70`).

**No `WireRecord<T>` / `EntryRecord<T>` wrapper at all.** The UI gets
raw hashes and fills in the rest from signals.

Other details:
- `std::panic::set_hook(zome_panic_hook)` at the top of every extern.
- `#[feature(zits_blocking)]` on write externs — a pseudo-attribute
  consumed by the `zits` type generator (§5) to pick `callBlocking` vs
  `call` in the generated TS proxy. Not a real Rust feature; the
  `ill_formed_attribute_input` lint is silenced crate-level
  (`threads/src/lib.rs:6`).
- `get_record_local` / `get_record_network` pairs exposed as separate
  externs (`threads.proxy.ts:107-121`).
- `init` grants a `receive_signal` cap to any agent, then emits a
  "done" tip (`threads/src/callbacks.rs:23-40`).

### 5. Serialization boundary handling

Vines is the only app in either round to **generate TS bindings from
Rust source** via `zits`.

- `zits` is a standalone Rust binary pinned at 1.17.1
  (`vines/package.json:6,22`).
- Invoked by `vines/scripts/ts-bindings.sh:5` with `-i <dir>` inputs
  pointing at every integrity and coordinator zome plus `zome_core`,
  `time_indexing`, etc.
- Output: six files per target zome under
  `vines/webcomponents/src/bindings/`: `{zome}.types.ts`, `.fn.ts`,
  `.proxy.ts`, `.integrity.ts`, plus shared `deps.types.ts`. Each is
  marked `/* This file is generated by zits. Do not edit manually */`.
- What is generated:
  - One TS interface per Rust struct, camelCase-converted
    (`threads.types.ts:330-396`):
    `Bead { ppAh: ActionArray; prevBeadAh: ActionArray }` where
    `ActionArray = Uint8Array` (`threads.types.ts:48-62`).
  - A variant-object union per Rust enum matching the default serde
    external-tag shape: `BaseBeadKindVariantTextBead = {TextBead:
    TextBead}` (`threads.types.ts:323-327, 466-472`).
  - A const list of every `#[hdk_extern]` function name per zome
    (`threads.fn.ts:7-83`).
  - A proxy class with one async method per extern, camelCased,
    dispatching to `this.call(...)` or `this.callBlocking(...)`
    depending on whether the source had `#[feature(zits_blocking)]`
    (`threads.proxy.ts:100-160`). Integrity-zome string consts like
    `VINES_DEFAULT_ROLE_NAME` are also pulled through
    (`threads.proxy.ts:4`).
- Because hashes are emitted as `Uint8Array` aliases, the UI wraps
  everything in a **materialize/dematerialize layer**: `materializeBead`
  / `dematerializeBead` convert wire types into `ActionId` / `EntryId` /
  `AgentId` class instances from `@ddd-qc/lit-happ`
  (`viewModels/threads.materialize.ts:212-260`). The UI consumes the
  `*Mat` wrapper types; raw wire types appear only at the boundary.

Load-bearing disciplines implicit in this setup:
- Rust type changes require running `scripts/ts-bindings.sh`; the `*Mat`
  converter silently ignores new fields.
- `#[hdk_extern]` is what zits uses to discover externs;
  `#[feature(zits_blocking)]` is a second hand-maintained signal.
- `EncryptedBead`'s `XSalsa20Poly1305EncryptedData` fields become
  `unknown` because zits can't model them
  (`threads.types.ts:360-363`).

### 6. Test layout

**Vines has no automated tests.** No `tests/` directory, no sweettest
or tryorama dep in `Cargo.toml` or `package.json`. Testing is manual
via the Tauri shell (`src-tauri/`). The most elaborate-validation app
in this survey ships no tests of that validation.

### 7. UI ↔ zome coupling

Layered as **dvm → zvm → perspective**:
- `ZomeViewModel` (zvm) extends `ZomeViewModelWithSignals` from
  `@ddd-qc/lit-happ`, wraps a generated proxy class, subscribes to
  `ZomeSignalProtocol` signals, maintains an observable "perspective"
  (`viewModels/threads.zvm.ts:1-80`).
- `DnaViewModel` (dvm) composes multiple zvms.
- Lit elements bind to the perspective.

Eventual-consistency strategy: most fetches emit signals rather than
returning data. The UI fires `fetch_beads(ahs)`, awaits `()`, and waits
for `EntryPulse` signals. Local-vs-network exposed as
`strategy: GetStrategy` on every fetch input
(`threads.types.ts:88-151`). `fetch_beads` also overlays authorship
from the authorship zome onto each pulse before emitting
(`fetch_beads.rs:22-31`) to handle migrated data.

### 8. Anti-patterns / surprises

- **`Bogus` placeholder entry** in `authorship_integrity` to satisfy
  tools requiring at least one entry type
  (`authorship_integrity/src/lib.rs:27-40`). Removing it breaks the
  build.
- Crate-level `#![allow(non_upper_case_globals, non_snake_case,
  ill_formed_attribute_input, ...)]` because `zits_blocking` isn't a
  real feature (`threads/src/lib.rs:1-6`).
- `ThreadsLinkType::Invalid` is a reserved null placeholder
  (`threads_integrity/src/lib.rs:115`). Sorting or pruning the enum
  changes wire format.
- **`dna.yaml` properties include a ~6KB base64 SVG** as the group icon
  (`dna/workdir/dna.yaml:11`). Properties are checked in
  `genesis_self_check` (`callbacks.rs:7-20`); changing the icon bytes
  changes the DNA hash.
- Profiles: integrity from upstream `hc_zome_profiles_integrity`,
  coordinator from a local fork (`Cargo.toml:29-30`) — mixed upstream
  and fork.
- Link-tag payloads use `decode(&tag_data)` (default msgpack) with no
  schema pin (`validate_link.rs:69-70`). A mismatched encoder
  downstream (`obj2Tag` vs `to_vec_named`) silently fails validation.

---

## acorn

### 1. DNA layout

**The only multi-DNA app in either round.** Two DNAs
(`acorn-happ/happ/workdir/happ.yaml:4-19`):

- `profiles` — singleton (`deferred: false`, `clone_limit: 0`), shared
  across all user projects.
- `projects` — **cloned per project** (`deferred: true`,
  `clone_limit: 999`). Each acorn project is a separate DNA instance.

Each DNA has one integrity + one coordinator zome — the per-DNA pattern
from prior research holds; the orthogonal signal is at the DNA level.

`projects_integrity` declares 9 entry types — `Tag`, `Connection`,
`EntryPoint`, `Outcome`, `OutcomeComment`, `OutcomeMember`, `OutcomeVote`,
`Member`, `ProjectMeta` (`projects_integrity/src/lib.rs:13-35`). The
link-types enum has **one variant: `All`** (`:37-40`). Opposite of
vines' 20-link-types approach — acorn uses a single catch-all link type
and distinguishes via anchor paths, not tags.

### 2. Entry & link type conventions

- `#[hdk_entry_helper]` + `#[serde(rename_all = "camelCase")]` on every
  entry (`projects_integrity/src/project/outcome/entry.rs:8-10`).
- Entries embed **hand-rolled authorship metadata** —
  `creator_agent_pub_key: AgentPubKeyB64`,
  `editor_agent_pub_key: Option<AgentPubKeyB64>`, plus
  `timestamp_created: f64` / `timestamp_updated: Option<f64>` as
  JS-number-compatible floats (`outcome/entry.rs:11-22`).
- **All hashes in entries and signals are B64-wrapped** —
  `ActionHashB64`, `AgentPubKeyB64`, `EntryHashB64` — never raw
  `ActionHash` (`outcome/crud.rs:22, 75-78`).
- One anchor path per entry type, generated by the `hdk_crud` macro:
  `get_outcome_path(LinkTypes::All)` etc. (`outcome/crud.rs:115-118`).
- Link-tag convention: `LinkTag::from(())` — empty
  (`project_meta/crud.rs:67`). Tags carry nothing; anchors distinguish.

### 3. Validation patterns

**Acorn has effectively no on-chain validation.** It ships extensive
*documentation* of what validation should do, but none is wired in.

Evidence:
- `projects_integrity/src/project/mod.rs:11` has
  `// pub mod validate;`. Same in every entry-type `mod.rs`.
- No `#[hdk_extern] fn validate(op: Op)` exists in either integrity
  zome. Grep for `Op::` or `fn validate\b` returns only the dead
  modules.
- The dead `validate.rs` files use **pre-HDI signatures**:
  `fn validate_create_entry_outcome(validate_data: ValidateData) ->
  ExternResult<ValidateCallbackResult>`
  (`outcome/validate.rs:16-43`) — the 0.0.122-era
  `validate_{create,update,delete}_entry_<name>` convention. The
  top-level entry-types enum is modern HDI; the validate bodies were
  never ported.
- `projects_integrity/validation_rules.txt` documents the intended
  rules, including things the author acknowledges can't be done in
  validate ("only ONE [project_meta] can/should be created ... Perform
  during zome call not validate hook" — lines 51-56). That singleton
  check is implemented at zome-call time via
  `simple_create_project_meta`, which does a `GetOptions::network()`
  fetch+count with an acknowledged race
  (`project_meta/crud.rs:29-78`).
- `acorn-happ/happ/tests/projects/src/` targets the dead validate
  functions via `hdk_unit_testing::mock_hdk` + `mockall`, with
  `Cargo.toml` pinning `hdk = "0.0.122"`
  (`tests/projects/Cargo.toml:11`). Incompatible with the current
  zome's HDI. Orphaned.

Headline: the only production-tested acorn DNA ships with a permissive
validate callback — more extreme than dino-adventure/emergence, where
there was at least an explicit permissive `Op` match. Here the callback
is absent entirely.

(`Profile::Status` has a custom `Serialize`/`Deserialize` to emit bare
`"Online"|"Away"|"Offline"` strings as a workaround for a serde quirk
— see §5. Integrity-side but about wire format, not validation.)

### 4. Coordinator API shape

Acorn is built on a published third-party CRUD framework,
`hdk_crud = "0.13.0"` (`acorn-happ/Cargo.lock:551-554`). One macro
invocation per entry type generates create/update/fetch/delete externs:

`projects/src/project/outcome/crud.rs:33-43`:
```
crud!(Outcome, EntryTypes, EntryTypes::Outcome, LinkTypes,
      LinkTypes::All, outcome, "outcome",
      get_peers_content, SignalType);
```

The macro defines `create_outcome`, `fetch_outcomes`, `update_outcome`,
`delete_outcome`; handles anchor-path creation; emits an
`ActionSignal<T>` via `send_remote_signal` after every write
(`projects/src/lib.rs:67-106`); takes `get_peers_content()` as a
callback to determine recipients.

Return shape is `WireRecord<T>` — `{ entry, action_hash, entry_hash,
created_at, updated_at }` with B64-wrapped hashes and Timestamps at
both create and update (`project_meta/crud.rs:70-77`). Fetches return
`Vec<WireRecord<T>>`. Closest to mewsfeed's `EntryRecord` and
dino-adventure's `AuthoredX` from prior research.

Signals: every write emits `ActionSignal<T>` (`{ entry_type, action,
data: WireRecord<T> }`) via `send_remote_signal` to peers returned by
a local `Path::from(MEMBER_PATH)` lookup
(`projects/src/lib.rs:182-209`). No `post_commit` — inline emission.

Custom non-crud externs:
1. **Composite operations** that wrap multiple `do_create`/`do_delete`
   and emit a custom signal variant:
   - `create_outcome_with_connection` (`outcome/crud.rs:103-179`)
     creates outcome + optional connection, emits
     `SignalType::OutcomeWithConnection`.
   - `delete_outcome_fully` (`outcome/crud.rs:201-353`) cascade-deletes
     all linked connections, members, votes, comments, entry points;
     returns a `DeleteOutcomeFullyResponse` listing every deleted hash;
     emits `SignalType::DeleteOutcomeFully`.
   Both variants live in the top-level `SignalType` enum
   (`projects/src/lib.rs:44-65`).
2. **Singleton checks** — `simple_create_project_meta` and
   `check_project_meta_exists` (`project_meta/crud.rs:29-78`).

Ephemeral-presence remote signals (`emit_realtime_info_signal`,
`emit_editing_outcome_signal` at `projects/src/lib.rs:151-180`) commit
nothing — they broadcast presence/who-is-editing entirely via
`send_remote_signal`.

### 5. Serialization boundary handling

Opposite approach from vines: **no code generation, hand-maintained
zod schemas in a workspace package.**

- Rust uses `#[serde(rename_all = "camelCase")]` on all entries,
  hashes as B64 strings inside entries, `f64` timestamps for
  JS-number compatibility.
- `acorn/zod-models` is a separate workspace package with one zod
  schema per Rust struct
  (`acorn/zod-models/src/outcome/outcomeSchema.ts`):
  ```
  export const OutcomeSchema = z.object({
    content: z.string(),
    creatorAgentPubKey: z.string(),
    editorAgentPubKey: z.string().nullable(),
    timestampCreated: z.number().gt(0),
    ...
  })
  export type Outcome = z.infer<typeof OutcomeSchema>
  ```
  The TS type is derived from zod, not from Rust. Field naming and
  nullability are manually mirrored. Zod doubles as a runtime parser.
- `WireRecord<T>` is a hand-written TS mirror
  (`acorn/web/src/api/hdkCrud.ts:11-17`). `callZome.ts` is a 17-line
  wrapper with no type narrowing.
- **`ZomeFnInput<T> { input: T; local?: boolean }`**
  (`acorn/web/src/types/shared.ts:32-35`) — used by
  `createCrudFunctions` for fetches only; writes pass raw payload
  (`web/src/api/hdkCrud.ts:62`). This is the **same shape as
  mewsfeed's** `ZomeFnInput<T>` flagged in prior research — acorn
  rediscovered it independently. The zome-side `hdk_crud::crud!` macro
  maps `local` to `GetOptions::local()`/`network()`.
- **UIEnum pattern** for serde-enum quirks: wraps Rust enums the UI
  treats as opaque strings in a `UIEnum(pub String)` newtype
  (`projects_integrity/src/ui_enum.rs`) with
  `#[serde(from = "UIEnum", into = "UIEnum")]` + manual `From` impls.
  Forces enums like `AchievementStatus` / `RelationInput` /
  `ComputedScope` to serialize as bare strings rather than the default
  `{Achieved: null}` shape
  (`outcome/small_scope.rs:51-76`, `outcome/crud.rs:45-70`).
  **Profile::Status solves the same problem differently** — a
  hand-rolled custom `Serialize`/`Deserialize` impl
  (`profiles_integrity/src/lib.rs:82-107`). Two workarounds for the
  same pitfall in the same repo.

Lockstep points: adding a Rust field requires updating the zod schema
manually. Enum fallback arms swallow unknown variants silently
(`outcome/crud.rs:55-58` — unknown relation becomes
`ExistingOutcomeAsChild`). The UI imports `@msgpack/msgpack/dist`
directly to decode signals (`signalsHandlers.ts:9`).

### 6. Test layout

`acorn-happ/happ/tests/{profiles,projects}/` exists but is **neither
sweettest nor tryorama**. It uses ancient mocked-HDK unit testing:
`hdk = "0.0.122"` with `features = ["mock", "test_utils"]`,
`hdk_unit_testing = "0.1.3"`, `mockall = "0.9"`, `fixt = "0.0.8"`,
`holochain_types = "0.0.26"` (`tests/projects/Cargo.toml:7-32`).

The tests fabricate `ValidateData`, mutate the inner element, and
assert on error enums from the dead validation functions
(`tests/projects/src/project/outcome.rs:14-79`). They cannot compile
against the current zome without re-enabling the commented-out
`validate.rs` modules.

**No sweettest, no tryorama, no multi-agent scenarios, no
consistency-wait harness.** The most complex coordinator logic —
`delete_outcome_fully`'s cascade + signal emission — is entirely
untested.

### 7. UI ↔ zome coupling

- React + Redux + `@holochain/client`. No `@holochain-open-dev` packages.
- No view-model / materialize layer: reducers consume `WireRecord<T>`
  directly, stored keyed by `actionHash`.
- Signal-driven for remote writes. `signalsHandlers.ts` decodes the
  payload, extracts `action` (create/update/delete) and `signalType`,
  and dispatches the same reducer action whether the write was local
  or remote (`signalsHandlers.ts:1-7` comment).
- Custom signal variants drive composite updates atomically:
  `OutcomeWithConnection` updates outcome + connection slices;
  `DeleteOutcomeFully` removes an outcome and all hanging relationships
  in one reducer dispatch.
- Ephemeral state (`RealtimeInfoSignal`, `EditingOutcomeSignal`) flows
  entirely through `send_remote_signal` broadcasts with no chain writes,
  stored in `ephemeral/realtime-info/actions.ts`.
- Fetch API takes optional `local?: boolean` (default `true`) via
  `ZomeFnInput<T>` (`web/src/api/hdkCrud.ts:62-69`). Per-call opt-in.
- `createCrudFunctions<T>` is a TS-side equivalent of the Rust `crud!`
  macro: one factory call per entry type yields
  `{create, fetch, update, delete}` (`web/src/api/hdkCrud.ts:46-90`).
  `projectsApi.ts` augments each CRUD with the custom externs.

### 8. Anti-patterns / surprises

- **The whole `tests/` + `validate.rs` machinery is dead code** but
  looks live. Test Cargo.tomls still declare `hdk = "0.0.122"`. A
  naive code-gen agent sampling these files will produce wrong
  patterns.
- `f64` timestamps inside entries (`outcome/entry.rs:15`). Nothing
  enforces any relationship between the embedded timestamp and the
  Create action's actual `Timestamp`.
- `simple_create_project_meta` takes a non-standard write path
  (`hash_entry` + raw `create_entry` + manual `create_link`) that
  bypasses `do_create` and therefore **emits no signal**, unlike
  every other create in the repo (`project_meta/crud.rs:58-68`).
- `acorn-hc` looks like the obvious acorn-Rust-code directory but is a
  2020 artifact. Alphabetic-directory-selection picks the wrong repo.
- `clone_limit: 999` + `deferred: true` means an agent sees only
  `profiles` until a project is created or joined. Forgetting this
  model leads to "cell not found" failures at startup.
- With no validate running, malformed entries reach the DHT and fail
  at read time — not at commit time. Error surfaces move.
- Two different serde-enum workarounds in the same repo — `UIEnum`
  newtype and hand-rolled `Serialize`/`Deserialize` — suggest the
  pattern was rediscovered twice rather than consolidated.

---

## Orthogonal findings (what these add beyond unyt-app/dino-adventure/emergence)

### Serialization boundary handling

Where vines and acorn diverge most sharply from each other and from the
prior three apps.

**Three distinct strategies observed across the five apps:**

1. **Hand-written TS types mirroring Rust byte-for-byte** (unyt-app,
   dino-adventure, emergence, plus acorn for its shared types). No
   compile-time check that Rust and TS agree.

2. **Hand-written zod schemas, TS types derived from zod** (acorn's
   `zod-models` workspace package). Zod doubles as a runtime validator
   — `OutcomeSchema.parse(incoming)` catches drift at runtime. Still no
   compile-time Rust↔TS check; field naming is a convention both sides
   must apply.

3. **Codegen from Rust via `zits`** (vines only). The only approach
   where adding a Rust field flows to TS without manual intervention.
   Details in the vines §5 above. Key consequences: hashes become
   `Uint8Array` aliases (forcing a materialize layer); enum codegen
   produces the serde external-tag shape directly
   (`FooVariant = {Bar: Bar}`); `XSalsa20Poly1305EncryptedData` degrades
   to `unknown` as an escape hatch (`threads.types.ts:360-363`).

**Specific serde pitfalls the repos work around:**

- **Default serde enum shape.** Rust `pub enum E { A(T), B(T) }` via
  Holochain's msgpack path surfaces on the TS side as `{A: ...}`
  variant-object tags. Acorn works around this **twice, inconsistently**:
  a `UIEnum(pub String)` newtype with
  `#[serde(from = "UIEnum", into = "UIEnum")]` and manual `From` impls
  (`projects_integrity/src/ui_enum.rs`, used by `AchievementStatus`,
  `RelationInput`, `ComputedScope`), and a hand-rolled custom
  `Serialize`/`Deserialize` for `Profile::Status`
  (`profiles_integrity/src/lib.rs:82-107`). Both coerce the enum to a
  bare JSON string. Vines accepts the default shape and handles it on
  the TS side via the generated variant-object types
  (`threads.types.ts:466-472`) plus wrapper-class materialization.
  Neither acorn solution is visible in mewsfeed, unyt, dino, or
  emergence.

- **Hashes as bytes vs B64 strings inside entries.** Acorn commits
  hashes as B64 strings **inside the entry body**
  (`outcome/entry.rs:13,18,22`) so the UI can string-compare directly.
  Vines uses raw `ActionHash` bytes and wraps on arrival. Prior apps
  used raw hashes. Committing B64 strings bakes a UI-friendly encoding
  into the DHT and makes Rust-side hash math require a parse.

- **camelCase propagation.** `#[serde(rename_all = "camelCase")]` at
  the struct level is universal. Vines applies it only to outer types
  (not `Moderation`, for example, which stays snake_case in Rust and
  gets translated by zits — `entries.rs:143-150`). Acorn applies it
  uniformly. No crate-level `rename_all` observed.

- **`Timestamp` vs `f64` vs `number`.** Vines uses `Timestamp`
  (`@holochain/client` i64, microseconds). Acorn uses `f64` inside
  entries for JS-number compat (`outcome/entry.rs:15-16`). mewsfeed
  (prior) used `Timestamp` with bigint coercion. No convergence.

- **Local-vs-network fetch.** Three explicit mechanisms:
  - mewsfeed: `ZomeFnInput<T> { local: bool }` on every fetch.
  - acorn: `ZomeFnInput<T> { input, local?: bool }` on fetches only,
    via `createCrudFunctions<T>` (`web/src/api/hdkCrud.ts:62-69`);
    the `hdk_crud::crud!` macro maps this to `GetOptions::local()` /
    `network()`. Independent rediscovery of mewsfeed's pattern.
  - vines: `strategy: GetStrategy` field on purpose-built input
    structs, PLUS separate local/network externs for common cases
    (`get_record_local`, `get_record_network` —
    `threads.proxy.ts:107-121`).

  Unyt/dino/emergence expose no control — the zome picks. Three of
  five apps expose it, landing on three different shapes.

- **Signal-emission placement.** Acorn emits signals inline inside
  every write extern via `send_remote_signal`
  (`projects/src/lib.rs:163-180`, also produced by `hdk_crud::crud!`).
  Vines uses `post_commit` via `attest_post_commit::<ThreadsEntry,
  ThreadsLinkType>` (`threads/src/callbacks.rs:44-52`) and also emits
  protocol signals from fetch externs (`fetch_beads.rs:33-36`).
  `post_commit` doesn't block the extern return; inline emission can
  send custom cross-entry signals like `OutcomeWithConnection`.

### Patterns confirmed

- Anchor-based indexing as the default. Vines has anchor
  consts (`ROOT_ANCHOR_*` at
  `threads_integrity/src/lib.rs:28-35`) and `hc_zome_profiles` anchors
  via the upstream crate; acorn uses one anchor path per entry type
  driven by `hdk_crud`.
- `#[serde(rename_all = "camelCase")]` on every entry struct. Both
  repos do this everywhere content crosses to the UI.
- `WireRecord<T>`-shaped wrapper is once again the chosen API return
  shape for one of the two apps (acorn). Vines substitutes a signal
  stream, but where acorn does return data, it returns this exact
  struct (hash + entry + timestamps). With unyt-app's hash-only,
  dino-adventure's `AuthoredX`, emergence's `EntryRecord<T>`, and
  acorn's `WireRecord<T>`, the consensus is "some struct wrapping a
  hash and the entry"; the bikeshed is just what's in the wrapper.
- Hand-rolled authorship metadata inside the entry (rather than relying
  on `action.author`) — acorn does this explicitly
  (`outcome/entry.rs:13-14`), matching the pattern prior research
  found in mewsfeed's user-profile linkage.

### Patterns contradicted or refined

- **"Single DNA everywhere" is contradicted by acorn.** A principled
  multi-DNA: `profiles` singleton + `projects` cloned per-workspace
  with `clone_limit: 999` and `deferred: true`. Both shapes are valid;
  the prior "all single-DNA" finding was an accident of sample. The
  skill needs to know multi-DNA with deferred-clone provisioning
  exists.

- **"Single integrity + single coordinator per DNA" is contradicted by
  vines.** Three integrity zomes in one DNA, deliberately split:
  `authorship` is an orthogonal migration-safe side-channel designed
  to drop into any DNA; `profiles` is a direct reuse of upstream
  `hc_zome_profiles_integrity` (keeping it as its own zome avoids
  pulling upstream entries into the caller's integrity enum). These
  are principled **reuse/composition** reasons to split that the
  "zome-merge is best practice" narrative didn't account for. A
  skill that only generates single-zome DNAs forces users to
  inline-copy upstream crates.

- **Validation spectrum refined.** Vines sits in the middle with a
  per-op dispatcher (`validate.rs:7-26`) where some ops are permissive
  and StoreEntry / RegisterCreateLink do real cross-entry work via
  `must_get_*` and `ChainFilter` chain walks. No state machine; the
  invariants are per-entry and per-link. This is a **third style**:
  declarative per-op cross-entry checks with chain walks, distinct
  from both unyt's state enum and dino/emergence's
  minimal-permissive. Acorn sits further toward the minimal end than
  any prior app — no `validate` callback at all, just a spec doc and
  dead 0.0.x code. New minimum is "no on-chain validation, documented."

- **"Emergence is the only app using `@holochain-open-dev/*`" is
  refined.** Vines uses `hc_zome_profiles_integrity` from hod
  partially — integrity only; the coordinator is a local fork
  (`profiles_alt_coordinator` from zdk — `Cargo.toml:29-30`). So "uses
  hod" is a spectrum: whole-zome reuse (emergence), partial-with-fork
  (vines), none (unyt/dino/acorn).

### New patterns not seen before

- **Generated TS bindings via `zits`** — first observed codegen of
  this kind in the ecosystem. Enables a ~80-function API surface
  (`threads.fn.ts:7-83`) that would be impractical to hand-maintain.
- **Materialize/dematerialize boundary layer** — `*Mat` wrapper types
  that box `Uint8Array` hashes in `ActionId`/`EntryId`/`AgentId` class
  instances before the UI consumes them
  (`viewModels/threads.materialize.ts:212-260`). Required because
  `zits` emits hashes as raw `Uint8Array` aliases.
- **Push-based fetch via signals.** `fetch_beads` returns
  `ExternResult<()>` and emits results as `ZomeSignalProtocol::Entry`
  signals (`fetch_beads.rs:7-38`). Fire-and-forget at the call site;
  data arrives asynchronously through the `ZomeViewModelWithSignals`
  subscription. Completely different architectural shape from
  request-reply.
- **LinkTag-as-serialized-payload.** Vines packs a `Vec<ActionHash>`
  into a `Ban` link-tag and an `AuthorshipLog` struct into an `Author`
  link-tag; validation decodes the tag and must-gets each reference.
  No prior app used link tags for more than a flag/category marker.
- **Migration-aware authorship zome.** A whole side-channel zome
  dedicated to storing original-author attribution as links, so
  migrated data keeps pre-migration authorship. `fetch_beads` overlays
  this onto each `EntryPulse` before signalling the UI
  (`fetch_beads.rs:22-31`).
- **Link-type-only zome with placeholder entry.** `authorship_integrity`
  declares a `Bogus` private entry purely to satisfy tools; the content
  is three link types.
- **`hdk_crud` macro-generated CRUD boilerplate** (acorn). One
  `crud!(...)` produces create/update/fetch/delete + anchor path +
  signal emission. Couples to a specific external crate and specific
  wire-type assumptions (`WireRecord<T>`, `ActionSignal<T>`). A skill
  that generates "acorn-style" code without the macro has to unroll it.
- **Multi-DNA with deferred-clone provisioning** (acorn). `projects` is
  cloned per-project via UI-driven `AppClient.createCloneCell()`. None
  of the prior three apps used deferred cloning.
- **Singleton-entry-by-zome-call-check.** Acorn's
  `simple_create_project_meta` does a `GetOptions::network()` lookup
  before creating the singleton. Author acknowledges the race
  (`validation_rules.txt:51-56`) because a true singleton can't be
  enforced from integrity.

### Things that would trip up a naive Claude

- **Commented-out modules that look live.** Acorn's every
  entry-type `mod.rs` has `// pub mod validate;`. Uncommenting to
  "fix a missing module" cascades pre-HDI type errors throughout.
- **Pre-HDI validation signatures masquerading as current.** The dead
  `validate.rs` files use `ValidateData`, `Header::Update(_)`,
  `header.original_header_address`, `WasmError::Guest(_)` — 0.0.x HDK
  types. Sampling them as patterns produces code that won't compile
  against HDI 0.6.
- **Tests pin ancient deps.** `acorn-happ/happ/tests/projects/Cargo.toml`
  pins `hdk = "0.0.122"`, `hdk_unit_testing = "0.1.3"`. `cargo test`
  won't resolve.
- **`ZomeFnInput<T>` is fetch-only in acorn.** `createCrudFunctions`
  wraps fetch payloads in `{input, local}` but passes writes raw
  (`web/src/api/hdkCrud.ts:52-90`). Wrapping writes in the same envelope
  silently fails at the coordinator — the macro's `create_outcome`
  doesn't destructure it.
- **`LinkTag::from(())` vs `LinkTag::new([])` vs
  `LinkTag::from(vec![])`** — different byte sequences. Mixing breaks
  tag decoding in vines-style repos where tags carry payloads.
- **Link-tag msgpack encoder mismatch** — vines' `validate_link.rs:69`
  uses `decode(...)` (default msgpack) to recover a
  `Vec<ActionHash>`. Constructing the tag with a different encoder
  (`to_vec_named`, `serde_json::to_vec`) silently fails validation.
  Generating the create side without auditing the integrity-side
  decoder is a real risk.
- **`#[feature(zits_blocking)]` is not a Rust feature gate.** It's a
  pseudo-attribute read by zits (`beads/text_bead.rs:9`). Removing
  it or replacing with a real feature attribute breaks
  call/callBlocking dispatch in the generated TS proxy with no
  Rust-side error.
- **Crate-level `#![allow(...)]` silences non-optional warnings.**
  Vines' `ill_formed_attribute_input` allow covers the pseudo-feature
  attribute. "Cleaning up" these surfaces errors.
- **Custom `Serialize`/`Deserialize` on integrity entries.**
  `Profile::Status` has a hand-rolled impl
  (`profiles_integrity/src/lib.rs:56-107`) because default derive
  produced `{Online: null}`. "Simplifying" to `#[derive(...)]` breaks
  the wire format.
- **`acorn-hc` is a trap** — 2020 artifact with a tempting Cargo.toml.
  Correct dir is `acorn-happ`.
- **Vines has no tests.** A skill that looks for `tests/` as the source
  of test-harness patterns finds nothing, not even a stub. Injecting a
  new test harness disagrees with the project's actual (manual)
  workflow.
- **`Bogus` placeholder entries are load-bearing.** The `Bogus` entry
  in `authorship_integrity` is not dead code; removing it breaks the
  build.
- **Deferred DNA provisioning.** Acorn's `projects` role is not
  provisioned at install time. Assuming `dna_info()` returns the
  projects DNA at startup gives a wrong cell or none.
- **`profiles` in acorn vs vines are unrelated.** Acorn rolls its own
  `Profile { first_name, last_name, handle, status, ... }`. Vines
  re-exports `hc_zome_profiles_integrity` from holochain-open-dev.
  "Harmonizing" them breaks one side's wire format.
