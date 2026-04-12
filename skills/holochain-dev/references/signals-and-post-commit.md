# Signals, `post_commit`, and the relaxed-commit tradeoff

## What signals are

Holochain signals are messages emitted from a zome (or remote zome) to a connected client (or another agent). The UI subscribes to a stream of signals from the conductor and reacts to them. Signals are *not* committed to the DHT — they're ephemeral messages, not chain state.

Three places signals can come from:

1. **`post_commit` callback** — fired *after* a successful commit. Has access to the real (non-temporary) action hash. Asynchronous, doesn't block the extern that triggered the commit.
2. **Inline from a coordinator extern via `send_remote_signal` or `emit_signal`** — fired during the extern call, before the commit returns. Synchronous with the call.
3. **Push-based fetch result channels** (vines's pattern) — a fetch extern returns `()` and emits its result as a signal asynchronously. The UI fires the fetch and waits for the signal to populate state.

All three are valid. Picking between them depends on whether the app uses **relaxed/async commits**.

## The relaxed-commit tradeoff (the load-bearing why)

Holochain supports two commit modes:

- **Synchronous commits** (the default for simple apps). When you call `create_entry`, the action is committed to the source chain *before* the call returns. The `ActionHash` you get back is the real, final hash. You can use it as a durable resource ID immediately.
- **Relaxed / asynchronous commits**. When you call `create_entry` with a relaxed-commit option (or in an `unsafe_block`), the call returns *before* the commit lands. The `ActionHash` you get back is **temporary** — it's a placeholder that will be rewritten when the commit actually completes. If you store that temporary hash anywhere durable, it goes stale.

This matters because **signal placement choice depends on which mode the app uses**.

### If the app uses synchronous commits only

Either signal style works. Inline `send_remote_signal` is simpler — you can bundle custom cross-entry signal variants atomically with the write (acorn does this with `OutcomeWithConnection` and `DeleteOutcomeFully` signals — see `acorn/projects/src/lib.rs:44-209`). `post_commit` works too but adds an extra hop.

### If the app uses relaxed/async commits

`post_commit` is **required**, not optional. Here's why:

- Inline signal emission happens *before* the commit lands. The signal carries the temporary hash.
- The UI receives the signal and stores the temporary hash as a resource ID.
- The commit lands (sometime later) and the real hash is different.
- Now the UI's stored ID is wrong. Every read using that ID fails or returns stale data.

`post_commit` runs *after* the commit lands, so it has access to the real hash. The signal it emits carries the real hash, and the UI can either (a) wait for the post_commit signal before storing the ID, or (b) listen for post_commit signals to *update* a previously stored temporary ID.

The alternative is to handle `source_chain_moved` errors at every read site and re-fetch when temporary hashes get rewritten. Both approaches work; pick one and be consistent. The skill should *ask* whether the app uses relaxed commits before designing the signal layer.

**Default for new projects:** use `post_commit`. It works under both commit modes (synchronous and relaxed), so it's the safer default. Inline emission only works under synchronous commits, so picking it locks you out of relaxed commits later without a refactor.

## `post_commit` signature

In current HDK (verify against `https://docs.rs/hdk/<version>`):

```rust
#[hdk_extern(infallible)]
pub fn post_commit(actions: Vec<SignedActionHashed>) -> () {
    // Inspect actions, emit signals, etc.
}
```

The `infallible` modifier matters — `post_commit` must not return an error because there's nothing the system can do if it fails (the commit has already happened). Any error must be handled internally (logged, swallowed, etc.).

`post_commit` receives a *batch* of actions, not one at a time. If multiple commits happen in a single zome call, you get them all at once.

## Push-based fetch via signals (vines's pattern)

Vines's coordinator design is unusual: fetch externs return `ExternResult<()>` and emit their results as `ZomeSignalProtocol::Entry` signals (`vines/dna/zomes/coordinator/threads/src/beads/fetch_beads.rs`). The UI fires `fetch_beads(hashes)`, awaits `()`, and then waits for the signal stream to populate its state.

This is fire-and-forget at the call site, and the UI is entirely signal-driven. It composes well with the rest of vines's architecture (the `ZomeViewModelWithSignals` from `@ddd-qc/lit-happ` subscribes to all signals automatically) but it's a big architectural commitment — the UI must be designed for it from the start.

**When to use this pattern:** when the UI is already signal-driven for everything else. Don't introduce it piecemeal.

## Ephemeral state via remote signals only (acorn pattern)

Not every signal corresponds to a commit. Acorn uses `send_remote_signal` to broadcast presence and "who's editing this outcome right now" information that *never* gets committed to the chain — it's purely ephemeral. The signals reach connected peers, the UI reflects them, and they vanish when the connection drops.

Use this for:
- Presence indicators ("Alice is online")
- Real-time editing locks ("Bob is editing this entry, don't conflict")
- Cursor positions, typing indicators, etc.

These have no DHT footprint. They're free in the sense that they don't grow the chain or trigger validation, but they're *not* reliable — peers that aren't currently connected don't see them.

## Capability grants and signals

For a remote agent to send a signal *to* an agent, the receiving agent must have granted the `receive_signal` capability. This is typically done in `init`:

```rust
#[hdk_extern]
pub fn init(_: ()) -> ExternResult<InitCallbackResult> {
    // Grant receive_signal to all agents (illustrative — verify against docs.rs)
    // ...
    Ok(InitCallbackResult::Pass)
}
```

Vines does this with `set_global_access` (`vines/dna/zomes/coordinator/threads/src/callbacks.rs:23-40`). Verify the exact API against `https://docs.rs/hdk/<version>`. unyt-app uses `set_global_access` for role-based caps in `init`.
