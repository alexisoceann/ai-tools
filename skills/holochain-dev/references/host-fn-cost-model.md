# Host function cost model

LLMs reach for plausible-sounding host functions without considering what they actually cost. The cost model for HDK host functions is not obvious from the signatures, and getting it wrong can make a zome usably correct but unusably slow.

## The expensive ones

### `must_get_agent_activity`

**This is a whole-chain fetch.** It pulls every action on an agent's source chain (or every action in a `ChainFilter` range). It is expensive in proportion to the size of the chain. It is the single most-misused host function in the codebases studied.

**Wrong reflexes to avoid:**

- Calling `must_get_agent_activity` to read a *single field* like a joining timestamp. This pulls the whole chain to extract one value. Real example: `mewsfeed`'s `get_joining_timestamp_for_agent` did exactly this and was a measurable performance bottleneck.
- Calling `must_get_agent_activity` to *check* for the existence of a single prior action when a dedup link recorded at create time would do the same job in O(1).
- Calling `must_get_agent_activity` inside validation when the validation only needs to check one specific prior entry — `must_get_action` on a known hash is much cheaper.

#### The right answer for "I need an agent's joining timestamp": the AgentPubKey-as-EntryHash trick

This is genuinely non-obvious Holochain domain knowledge that no LLM training data reliably contains. It's the right answer for the "joining timestamp" use case and it generalizes to anything you'd want from the third genesis entry.

**The fact:** `AgentPubKey` is the **only** Holochain entry type whose `EntryHash` equals the pubkey bytes themselves — with no hash transformation applied. Every other entry's hash is a transformation of its bytes; AgentPubKey is the exception.

**Why this matters:** when an agent joins a network, they create three genesis entries — `Dna`, `AgentValidationPkg`, and `AgentPubKey` — atomically and within microseconds of each other. The `AgentPubKey` entry is the third one. Combining the two facts:

1. You already know an agent's pubkey for any action they've authored (it's `action.author()`)
2. The pubkey **IS** the EntryHash of the third genesis entry
3. So you can `get(EntryHash::try_from(agent_key)?)` to fetch that genesis record directly
4. The record's timestamp is the joining time (within microseconds of the actual join, since all three genesis entries are atomic)

```rust
pub fn get_joining_timestamp(agent: AgentPubKey) -> ExternResult<Timestamp> {
    let entry_hash = EntryHash::try_from(agent)?;  // pubkey IS the entry hash
    let record = get(entry_hash, GetOptions::default())?
        .ok_or(wasm_error!(WasmErrorInner::Guest("agent not found".into())))?;
    Ok(record.action().timestamp())
}
```

**This is O(1).** No cache, no new entries, no chain walk, no `must_get_agent_activity`, no record-at-create-time link. None of those alternatives are needed because the data is already present and directly addressable. It is strictly better than:

- Caching (the cache is a workaround for an expensive lookup that doesn't need to be expensive)
- `must_get_agent_activity` + `.first()` (whole-chain fetch for one timestamp)
- Recording the joining timestamp at create time as a separate link (the entry already exists; you'd be duplicating data)

**Local-vs-network sensitivity:** like any `get`, this lookup is sensitive to the calling node's arc. On a full-arc node it's a local fetch; on a zero-arc node (mobile) it goes to the network. See `eventual-consistency.md`.

**Verify the exact API against docs.rs:** the conversion between `AgentPubKey` and `EntryHash` (whether it's `EntryHash::try_from(agent)`, `EntryHash::from(agent)`, `agent.into()`, or some other form) has shifted across HDK versions. The *fact* (AgentPubKey == its own EntryHash) is stable; the conversion API surface is what to look up at `https://docs.rs/hdi/<version>` before shipping.

#### When `must_get_agent_activity` is actually appropriate

When you genuinely need to walk a range of an agent's chain — e.g., "find all bead-creates in the last 5 minutes for rate-limiting" (vines's pattern in `validation_app_entry.rs`). Even then, scope the `ChainFilter` as tightly as you can.

**`ChainFilter` semantics gotcha:** `ChainFilter` calculates a `min..=max` action sequence range *first*, then iterates. It is not lazy. The cost is proportional to the range size, not to the filter predicate's matching set. If you want to walk back 50 actions, set `take(50)` or `until_timestamp(...)`; don't use a predicate filter and hope it short-circuits.

**On scratch space and the "perspective gap" myth:** an outdated framing of validation says "must_get_agent_activity called inline sees scratch, called non-inline doesn't, so the two contexts give different answers." This framing is **misleading**. Within a zome call, scratch accumulates pending commits — but all commits in the call are then written **atomically** to the source chain before any peer can see anything via gossip. By the time a peer is validating any action from your call, the entire batch is on the chain. Peers always see the same state the author saw. There is no perspective gap. See `validation-styles.md` for the full discussion. Scratch matters within a single zome call (so commit B can see commit A in the cascade if both are in the same call), but it does **not** create a determinism gap between author and peer validators.

### `get` and `get_links` over the network

These are slow on a zero-arc node (every call hits the network) and fast on a full-arc node (the data is local). See `eventual-consistency.md` for why this matters for API design.

`get_links` followed by `get` for each target is **N+1 queries**. If you can fetch a batched representation in one call, do it. Vines builds a single `EntryPulse` per record and emits the whole batch as one signal — see `vines/dna/zomes/coordinator/threads/src/beads/fetch_beads.rs`.

### Cross-zome calls

Cross-zome calls within the same DNA have *measurable* overhead. They go through the wasm boundary on each side. This is part of why "single integrity + single coordinator zome per DNA" is the default architecture (see `zome-architecture.md`). Don't split a zome to make cross-zome calls "for modularity" — it's a real performance cost.

## The cheap ones

### `must_get_action`, `must_get_entry`

These fetch a *single* action or entry by its hash. They're cheap relative to chain walks. Use them whenever you need a specific record and you have its hash.

### `query`

`query` against the local source chain (with appropriate filters) is fast — it's a local SQL query. Use it for "what entries of type X has this agent created?" type questions. It does *not* see other agents' chains; it's local-only.

### Direct entry creation, link creation

`create_entry`, `create_link`, `update_entry`, `delete_entry` are all relatively cheap. The expensive part of a write is the validation that runs at commit time (and again at validation receipt time), not the host call itself.

## Diagnostic guideline

If a zome is slow, the first place to look is "how many `must_get_agent_activity` calls am I making per request?" If the answer is anything other than zero or one tightly-scoped one, that's the bottleneck.

The second place to look is N+1 patterns: a `get_links` followed by a per-target `get` in a loop. Batch where possible.

The third place is cross-zome calls in hot paths.

The fourth place — and only after the above three are clean — is anywhere else.
