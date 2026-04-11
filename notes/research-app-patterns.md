# Holochain App-Design Patterns — Research Findings

This document surveys three production Holochain applications to identify common patterns, design conventions, and coupling points between coordinator zome APIs and UI state management. The three repos surveyed are **unyt-app**, **dino-adventure**, and **emergence**, each representing different complexity and scale.

---

## unyt-app

**Type:** Complex economic/transactional system. Implements RAVE (Resource Agreement and Value Exchange) engine for bilateral transactions with phases: negotiation → commitment → acceptance → completion.

### 1. DNA Layout

- **Single DNA:** `alliance`
- **Zomes:**
  - **Integrity:** `transactor` (single integrity zome)
  - **Coordinator:** `transactor` (corresponding coordinator)
- **Notes:** Substantial business logic is outsourced to a shared Rust crate (`crates/rave_engine`) containing types, validation helpers, and transaction processing logic. Entry and link types are defined in the engine and re-exported.

### 2. Entry & Link Type Conventions

Entry types are extensive and reflect the RAVE flow:
- Negotiation phase: `Proposal`, `CodeTemplate`, `SmartAgreement`, `RAVE`
- Transactional phase: `Commitment`, `Accept`, `Receipt`
- Rejection phase: `Reject`, `Reclaim`
- System/metadata: `GlobalDefinition`, `GlobalUnits`, `UnitDefinition`, `LaneBasicProperties`, `LaneDefinition`, `DataBlob`, `ConversionTable`, `DocDef`, migration entries

Link types are task-specific and include:
- `SendReceiverDepositNotification` — notification for incoming deposits
- `SendExecutorParkedLinkNotification` — for execution requests
- `InitAgent` — tracking network participants at genesis

**Pattern:** Entries and links are named semantically for their domain role, not generically. The schema is deeply coupled to the RAVE transaction model. All type definitions centralize in integrity modules; coordinator imports and uses them without modification.

See: `unyt-app/dnas/alliance/zomes/integrity/transactor/src/entries/mod.rs:28-69`

### 3. Validation Patterns

Validation is **comprehensive and state-aware.** Examples:

- **Agent identity checks:** Proposal validation verifies that author and counterparty are distinct (lines 16-22 in proposal/mod.rs).
- **Timestamp ordering:** Proposal timestamps must not exceed action timestamps (lines 24-31).
- **Chain logic:** Validates that previous proposals exist and form a valid chain, with rules about counterparty roles (lines 35-65).
- **Ledger validation:** Complex state validation; transactions are validated against the full agent activity chain and a custom `TxProcessor` (lines 68-74).
- **Immutability rules:** Updates are explicitly forbidden for certain entry types (e.g., `Proposal update is not allowed`).

Validation is **not minimal** — it encodes business rules (e.g., balances, transaction sequencing, authorization). All validation occurs in integrity zomes; coordinator zomes have no validation responsibility.

See: `unyt-app/dnas/alliance/zomes/integrity/transactor/src/entries/proposal/mod.rs`

### 4. Coordinator API Shape

**Return types:** Coordinator functions return low-level hashes (e.g., `ActionHashB64`) or task-specific result wrappers. Functions do not return full entry records; the UI must fetch those separately if needed.

**Example extern functions:**
- `create_proposal(ProposalInput) -> ActionHashB64` — creates and returns only the action hash
- `create_commitment(CommitmentInput) -> ActionHashB64`
- `create_accept(AcceptInput) -> ActionHashB64`

Operations are **granular CRUD**—each phase of the transaction has explicit create/read/list functions organized under submodules (`externs/core/actions.rs`, `externs/rave/`, etc.). No "high-level" transaction operations; the UI orchestrates the flow.

**Capability grants:** Init function uses `set_global_access()` to grant specific zome functions to agents with certain roles (e.g., "by_notary", "by_participant"). This is an authorization pattern not visible in the zome signature itself.

See: `unyt-app/dnas/alliance/zomes/coordinator/transactor/src/externs/core/actions.rs:51-143`

### 5. Sweettest Layout

Tests live in a separate crate at `sweettest/` using Rust + Tokio + async/await.

**Structure:**
- `tests/configuration_and_datablob.rs` — integration tests for multi-agent setups
- `tests/unit_types/` — modular tests for unit types, lanes, phase transitions
- `tests/migration/` — tests for state migration logic

**Patterns:**
- `TestEnv` fixture that manages a multi-agent scenario (progenitor + developer + participants)
- `await_consistency()` calls to wait for DHT synchronization
- Tests call coordinator functions directly via the test client; assertions check state afterward
- Comprehensive scenario testing: agents propose, commit, accept, verify ledger state

See: `unyt-app/sweettest/tests/configuration_and_datablob.rs:11-24`

### 6. UI ↔ Zome Coupling

**State management:** The unyt-app UI is built with **Lit Web Components** and uses **LitContext** for dependency injection, not Svelte stores. See `context.ts`: a logger and locale context provider.

**API layer:** Coordinator calls are wrapped in a singleton `HolochainService` (at `ui/white-label/src/services/holochain-service.ts`). This service:
- Manages the `AppClient` connection lifecycle
- Provides a `callZome` method for all coordinator calls
- Integrates with Sentry for metrics

**Signal handling:** Not directly visible in the sampled files, but post_commit hooks in the coordinator send remote signals via `send_remote_signal()` for transaction state changes (see `post_commit.rs:42-53`). The UI would listen to these.

**Return-type design:** Coordinator functions return only hashes. The UI must follow up with get operations to fetch entry details. This suggests the API is **not designed for the UI's convenience**—it's designed for the zome's logic. The UI is responsible for state assembly.

See: `unyt-app/ui/white-label/src/services/holochain-service.ts:16-31`, `unyt-app/dnas/alliance/zomes/coordinator/transactor/src/post_commit.rs`

### 7. Surprising / Non-obvious Patterns

- **Rave engine abstraction:** Entry types and validation logic live in a shared crate (`crates/rave_engine`). This is unusual; most Holochain apps define types inline in integrity zomes. It allows type reuse and shared validation logic but makes the zome hard to audit independently.
- **Complex state machine in validation:** The `TxProcessor` (used in validation) is a stateful validator that processes transactions across an agent's entire chain. This is computationally expensive during validation and suggests the RAVE protocol is itself the source of truth, not just a consequence of entry structure.
- **No visible membrane proof logic:** Though the coordinator references "notary agents" in init, there's no explicit membrane proof in the code sampled.

---

## dino-adventure

**Type:** Minimal collaborative game. Users create dinosaurs, form adventure groups, nest batches of dinosaurs. Very simple state model.

### 1. DNA Layout

- **Single DNA:** `dino_adventure`
- **Zomes:**
  - **Integrity:** `dino_adventure`
  - **Coordinator:** `dino_adventure`
- **Organization:** Much simpler than unyt-app. All types and validation inline.

### 2. Entry & Link Type Conventions

Entry types:
- `Dino` — dinosaur entity with name and kind (enum of ~15 types)
- `Adventure` — group activity
- `NestBatch` — batch of nests
- `Nest` — individual nest

Link types:
- `AllDinos` — from anchor to all dinos
- `AllAdventures` — from anchor to all adventures
- `MyAdventures` — from agent to their adventures
- `AdventureNestBatches` — from adventure to its nests
- `NestBatchNests` — from batch to individual nests

**Pattern:** Simple anchor-based indexing. Link types are named after the relationship they represent (`X` to `Y` or `AllX`). No sophisticated tagging schemes.

See: `dino-adventure/dnas/dino_adventure/zomes/integrity/dino_adventure/src/lib.rs:23-31`

### 3. Validation Patterns

Validation is **minimal and permissive:**

- `validate_create_dino()` — accepts all (TODO comment in code)
- `validate_update_dino()` — rejects all updates
- `validate_delete_dino()` — rejects all deletes
- Link validation checks that targets are valid hashes and reference Dino entries, nothing more

No agent identity checks, timestamp validation, or state-machine logic. Validation is **about structural integrity only**, not business logic.

See: `dino-adventure/dnas/dino_adventure/zomes/integrity/dino_adventure/src/dino.rs:30-57`

### 4. Coordinator API Shape

Coordinator functions are **granular CRUD with richer returns:**

```rust
create_dino(dino: Dino) -> ExternResult<AuthoredDino>
get_all_dinos() -> ExternResult<Vec<AuthoredDino>>
get_all_dinos_local() -> ExternResult<Vec<AuthoredDino>>
```

Return type `AuthoredDino` is a struct containing the entry, its action hash, and author—everything the UI needs to display and reference it. Compare to unyt-app which returns only hashes.

Functions also explicitly offer **local vs. network** query strategies via separate function names (`get_all_dinos` for network, `get_all_dinos_local` for local). This is a deliberate API affordance for UIs managing eventual consistency.

See: `dino-adventure/dnas/dino_adventure/zomes/coordinator/dino_adventure/src/dino.rs:5-52`

### 5. Sweettest Layout

Tests are **TypeScript/Vitest** (not Rust), using `@holochain/tryorama`:

```typescript
await runScenario(async (scenario) => {
  const [alice, bob] = await scenario.addPlayersWithApps([appSource, appSource]);
  const record = await createDino(alice.cells[0]);
  await dhtSync([alice, bob], alice.cells[0].cell_id[0]);
  const allDinos = await bob.cells[0].callZome({...});
});
```

Tests use tryorama's scenario abstraction to add agents, sync DHT, and make zome calls. Much more approachable than Rust tests and closer to how the JS client actually works.

See: `dino-adventure/tests/src/dino_adventure/dino_adventure/dino.test.ts`

### 6. UI ↔ Zome Coupling

**State management:** **Svelte 5 runes-based reactive state**, hand-rolled. See `dino-adventure/ui/src/api/dino.svelte.ts`:

```typescript
let dinoState: Record<AgentPubKeyB64, AuthoredDino> = $state({});
let dinosFirstLoad = $state(false);

export const getDinoState = () => dinoState;
export const createDino = async (dino: Dino): Promise<AuthoredDino> => {
  return callZome({...});
};

const fetchDinos = async (): Promise<void> => {
  // Every 5 calls, fetch from network; otherwise fetch locally
  if (networkRefreshCounter % 5 === 0) {
    authoredDinos = await callZome<AuthoredDino[]>({...fn_name: "get_all_dinos"});
  } else {
    authoredDinos = await callZome<AuthoredDino[]>({...fn_name: "get_all_dinos_local"});
  }
  // Merge new dinos into state
  dinoState = {...dinoState, ...newDinos};
};

// Signal handlers update state
signalHandler.addSignalHandler("dino_adventure:EntryCreated:Dino", (dino, action) => {
  dinoState[encodeHashToBase64(action!.hashed.content.author)] = {...};
});

// Polling + signal-driven updates
setInterval(() => fetchDinos(), 5000);
```

**Pattern:** No external store library. State is a module-level variable. UI code imports and uses it directly. Polling + signals drive updates. The 5-call heuristic (1 network query per 5 calls) is a pragmatic eventually-consistent pattern.

**Signal routing:** Signals are decoded by `SignalHandler` class and dispatched to keyed callbacks. The callback key includes zome name, signal type, and entry type (e.g., `dino_adventure:EntryCreated:Dino`).

See: `dino-adventure/ui/src/api/dino.svelte.ts`, `dino-adventure/ui/src/api/common.svelte.ts:77-167`

### 7. Surprising / Non-obvious Patterns

- **Local vs. network strategy in the API:** The coordinator exposes two functions (`get_all_dinos` and `get_all_dinos_local`) rather than using a parameter. This is a deliberate affordance for the UI.
- **Polling + signals hybrid:** The UI polls every 5 seconds but only hits the network 1-in-5 times. Signals trigger immediate updates. This is a lightweight approach to eventual consistency.
- **AuthoredDino return type:** Unlike unyt-app, this coordinator returns rich records. The design assumes the UI wants the full context.

---

## emergence

**Type:** Social platform. Users create and attend sessions (group activities), manage spaces, share notes, create maps, manage proxy agents, and interact via relations and feeds.

### 1. DNA Layout

- **Single DNA:** `emergence`
- **Zomes:**
  - **Integrity:** `emergence`, `file_storage`, `profiles`
  - **Coordinator:** `emergence`, `file_storage`, `profiles`
- **Organization:** Three coordinated zomes with different responsibilities. Emergence handles core entities; file_storage for media; profiles (from @holochain-open-dev) for agent metadata.

### 2. Entry & Link Type Conventions

Entry types:
- `Session` — a group activity with type, title, leaders, duration, amenities, etc.
- `Space` — a venue with capacity, tags, stewards
- `Note` — a written artifact
- `Map` — a spatial site map
- `ProxyAgent` — an agent representing another (delegation)

Link types:
- `AllSessions`, `AllSpaces`, `AllMaps`, `AllProxyAgents` — anchor-based indices
- `SessionUpdates`, `SpaceUpdates`, `NoteUpdates`, `MapUpdates`, `ProxyAgentUpdates` — history chains
- `TimeWindows` — time slots associated with sessions
- `Settings` — per-user settings
- `Relations` — edges in a relational graph
- `SessionUpdates`, `SpaceUpdates`, etc. — from hash to updates (version chain pattern)

**Pattern:** More elaborate than dino-adventure. Uses both anchor indexing and update chains. Entries have semantic "Update" links forming a temporal history.

See: `emergence/dnas/emergence/zomes/integrity/emergence/src/lib.rs:18-44`

### 3. Validation Patterns

Validation is **present but still minimal.** Each entry has a `validate_create_*` function that currently returns `ValidateCallbackResult::Valid`. The code doesn't appear to have sophisticated business logic validation yet.

One exception: the entry types include a `trashed` boolean on sessions, suggesting soft-delete semantics, but no explicit deletion validation visible in sampled code.

See: `emergence/dnas/emergence/zomes/integrity/emergence/src/lib.rs:57-164`

### 4. Coordinator API Shape

The coordinator is wrapped by a **client class** (`EmergenceClient`) that sits between the UI and the zome calls. This client:

```typescript
async createSession(sessionType, title, ..., leaders): Promise<EntryRecord<Session>>
async updateSession(update: UpdateSessionInput): Promise<EntryRecord<Session>>
async deleteSession(actionHash): Promise<void>
async getSessions(): Promise<Array<InfoSession>>
```

Returns rich objects: `EntryRecord<T>` from `@holochain-open-dev/utils`, wrapping entry + metadata. No hash-only returns; the API is **designed for client convenience**.

Functions are higher-level than dino-adventure's CRUD—e.g., `getSessions()` returns `InfoSession` records that bundle `original_hash`, `record`, and `relations`, a computed join.

See: `emergence/ui/src/emergence-client.ts:104-147`

### 5. Sweettest Layout

Tests are **TypeScript/Vitest** using `@holochain/tryorama` (same as dino-adventure):

```typescript
const [alice, bob] = await scenario.addPlayersWithApps([appSource, appSource]);
await scenario.shareAllAgents();
const sessionRecord = await createSession(alice.cells[0], {...});
```

Tests are organized by entity (session.test.ts, space.test.ts, etc.) and cover multi-agent scenarios with DHT sync.

See: `emergence/tests/src/emergence/emergence/`

### 6. UI ↔ Zome Coupling

**State management:** Emergence uses **Svelte stores + @holochain-open-dev/stores**. The main store is `EmergenceStore`, a class managing multiple writable stores:

```typescript
export class EmergenceStore {
  sessions: Writable<Array<InfoSession>> = writable([]);
  spaces: Writable<Array<Info<Space>>> = writable([]);
  notes: Writable<Array<Info<Note>>> = writable([]);
  // ...
  uiProps: Writable<UIProps> = writable({...});
  settings: Writable<Settings> = writable({...});

  async getSessions(): Promise<Array<InfoSession>> {
    const sessions = await this.client.getSessions();
    // Update stores
    this.sessions.update(oldSessions => {...});
  }
}
```

**Lazy loading:** Uses `toPromise()` and `asyncDerived()` from `@holochain-open-dev/stores` to expose derived/computed stores with async status:

```typescript
const notes = writable(new HoloHashMap<ActionHash, Info<Note> | undefined>());
setInterval(async () => {
  if (neededStuff.notes) {
    const stuff = await client.getStuff(neededStuff);
    notes.update(oldNotes => {...});
  }
}, 10000);
```

**Signal handling:** Not explicitly visible in sampled code, but the pattern suggests signals from coordinator post_commit hooks would feed store updates.

**File storage:** Integrates `@holochain-open-dev/file-storage` client to manage media. Downloads are cached in a `HoloHashMap`.

See: `emergence/ui/src/stores/emergence-store.ts:105-200`

### 7. Surprising / Non-obvious Patterns

- **Dual library usage:** Uses both `@holochain-open-dev` packages (profiles, file-storage, stores, utils) **and** hand-rolled client (`EmergenceClient`). Not fully committed to the ecosystem.
- **HoloHashMap everywhere:** Extensive use of `HoloHashMap` for agent-keyed and hash-keyed collections, suggesting the app deals with complex multi-agent state.
- **Async stores with periodic polling:** `neededStuff` batching pattern—entities register what they need, and a 10-second interval polls for batch updates. This is a different take on eventual consistency than dino-adventure's naive polling.
- **Trashed flag instead of deletion:** Sessions have a `trashed: boolean` field, suggesting soft-delete semantics (UI hides trashed, but doesn't prune them). No explicit validation of this rule in the sampled code.

---

## Cross-cutting Observations

### Common across all three

1. **Single DNA per app.** All three apps use exactly one DNA with one integrity and one coordinator zome pair (or more, in emergence's case, but logically organized around the same DNA). None use multi-DNA architectures.

2. **Anchor-based indexing.** All three use anchor patterns for "get all X" queries (via Path/link).

3. **Minimal or permissive validation.** Entry validation is either non-existent (dino-adventure, emergence TODO stubs) or domain-specific and expensive (unyt-app). None use membrane proofs or sophisticated agent onboarding checks.

4. **Coordinator as thin wrapper.** Coordinators are primarily CRUD layers; business logic lives in integrity (unyt-app) or is deferred to the UI (dino-adventure, emergence).

5. **Returning more than hashes helps the UI.** dino-adventure and emergence return rich records (`AuthoredDino`, `EntryRecord<T>`); unyt-app returns only hashes and requires follow-up queries. The difference in API design is striking.

6. **Svelte dominates the UI.** Two of three apps (dino-adventure, emergence) use Svelte with hand-rolled or library-provided stores. unyt-app uses Lit. No React or Vue.

7. **Eventual consistency is UI responsibility.** None of the zomes provide guarantees about DHT consistency. The UIs implement polling, local-vs-network strategies, or signal-driven updates to manage it.

### Varies meaningfully

1. **Validation complexity.** Ranges from empty (dino-adventure) to extremely complex (unyt-app). No middle ground; either the protocol is simple or it's hard.

2. **Coordinator return types.** unyt-app returns hashes; dino-adventure and emergence return rich records. This suggests the architectural decision is made per-app, not per-framework.

3. **Testing language.** unyt-app uses Rust/Sweettest; dino-adventure and emergence use TypeScript/Tryorama. Reflects maturity: unyt-app is older and internal; dino-adventure and emergence adopted JS testing tools.

4. **Library adoption.** emergence uses `@holochain-open-dev/*` packages; dino-adventure and unyt-app do not. No clear pattern.

5. **Zome count.** dino-adventure and unyt-app have 2 zomes (integrity + coordinator); emergence has 6 (3 DNA pairs). emergence is larger and more modular.

### Open question: @holochain-open-dev usage

**Search result:** Only **emergence** actively uses `@holochain-open-dev` packages:
- `@holochain-open-dev/profiles` — for agent profiles (though only profiles, not the full store ecosystem)
- `@holochain-open-dev/file-storage` — for media storage
- `@holochain-open-dev/stores` — for `AsyncReadable`, `asyncDerived`, `toPromise`
- `@holochain-open-dev/utils` — for `EntryRecord` wrapper

**dino-adventure** and **unyt-app** do not depend on `@holochain-open-dev` at all.

**Hand-rolled equivalent in dino-adventure:**
- State is plain objects and module-level variables
- Polling is a naked `setInterval()`
- Signal routing is a custom `SignalHandler` class with a keyed callback map
- No async store derivations; computed state is done at the component level

**Assessment:** `@holochain-open-dev` packages are **not idiomatic** across the three surveyed apps. It's one valid choice (emergence uses it) but not required. Hand-rolled state (dino-adventure, unyt-app) is equally valid and simpler. The skill should **not assume** the libraries are always present; it should present them as an optional pattern.

### Things that would trip up a naive Claude

1. **Coordinator functions may return only hashes, not entries.** A naive agent might assume `create_x()` returns the full entry; it often returns just an `ActionHashB64`. The UI must follow up with a `get()` call. This is not a bug; it's a design choice. unyt-app does this consistently.

2. **"Minimal" validation is a feature, not a bug.** dino-adventure and emergence have validation functions that just return `Valid` with no checks. This is intentional: the protocol doesn't need complex validation. Do not assume every entry type needs elaborate validation logic.

3. **Local vs. network strategy is a UI concern, not a zome concern.** dino-adventure explicitly exposes `get_all_dinos()` and `get_all_dinos_local()`. This is not a quirk; it's a deliberate affordance for UIs managing eventual consistency. The zome is **not** responsible for hiding this detail.

4. **Signals are sent from post_commit, not from the extern functions themselves.** unyt-app's coordinator functions return hashes; signals announcing the transaction are sent later from `post_commit()`. The UI must listen for signals, not expect them synchronously.

5. **Stores can be module-level variables, not always instances.** dino-adventure's state is just a module-level record `let dinoState: Record<...> = $state({})` exported as a function. No class, no store framework. This is valid Svelte 5 runes state.

6. **Entry types are often defined in integrity zomes and imported by coordinator zomes.** The coordinator does not re-declare them; it uses the types from integrity via a prelude or explicit import. See unyt-app's `pub mod prelude` in the integrity zome.

7. **Rave engine (unyt-app) is a weird outlier.** Types and validation are shared in a crate. This is not typical Holochain; it's a specific unyt-app pattern. Do not assume all apps do this.

8. **Sweettest vs. Tryorama are different testing worlds.** unyt-app uses Rust/Sweettest; dino-adventure and emergence use TypeScript/Tryorama. A Claude agent generating tests should know which world the app targets. The syntax and patterns are very different.

9. **Links have semantic meaning in the data model.** Link types are not arbitrary; they encode relationships (e.g., `AdventureNestBatches` means "adventure contains nest batches"). A naive agent might treat link types as interchangeable. They are not.

10. **InitAgent links and anchor-based indexing are separate patterns.** unyt-app uses `InitAgent` links to track who joined the network. dino-adventure and emergence use anchor-based indexing (Path.from("all_dinos")). Both are valid; different apps use different patterns.

---

## Architectural Open Questions (for future skill design)

1. **Should the skill guide the choice between hash-returning and record-returning coordinators?** unyt-app (hash) vs. dino-adventure (record) is a fundamental architectural difference. What are the tradeoffs?

2. **How should the skill handle validation complexity?** Should it guide towards minimal validation (appropriate for simple protocols) or comprehensive validation (required for complex protocols)? The three apps span the spectrum.

3. **Should the skill teach @holochain-open-dev as the standard or as one option?** Only emergence uses it. Is it worth teaching as canonical, or should the skill present hand-rolled stores as equally valid?

4. **Is local-vs-network a zome concern or a UI concern?** dino-adventure encodes it in the zome API. Unyt-app and emergence hide it (the UI just polls). Where should the skill guide developers?

5. **How much validation should happen in the coordinator vs. integrity?** unyt-app puts all validation in integrity; dino-adventure has minimal validation anywhere. Is this a spectrum, or are there hard rules?

---

**Survey completed.** The patterns above represent the surface-level design of three production Holochain applications at varying scales of complexity. All three are functional and actively developed, suggesting these patterns are viable starting points for new Holochain app development.
