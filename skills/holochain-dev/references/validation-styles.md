# Validation styles and `Op` dispatch

## The validation callback

Every integrity zome can declare a single `validate(op: Op)` callback. The conductor calls it on every action that the zome could be authoritative for, both at commit time on the local source chain and at validation-receipt time when an action arrives from another agent.

The current pattern (HDK 0.6+ / HDI 0.7+; current target HDK 0.6.1-rc.5, HDI 0.7.1-rc.5) for dispatching validation by entry/link type is:

```rust
#[hdk_extern]
pub fn validate(op: Op) -> ExternResult<ValidateCallbackResult> {
    // Use op.flattened()? to get a typed FlatOp<EntryTypes, LinkTypes>
    match op.flattened()? {
        FlatOp::StoreEntry(...) => { ... }
        FlatOp::StoreRecord(...) => { ... }
        FlatOp::RegisterCreateLink(...) => { ... }
        // ...
        _ => Ok(ValidateCallbackResult::Valid),
    }
}
```

**Verify the exact API against `https://docs.rs/hdi/<version>` before writing the dispatcher.** The shape of `Op`, `FlatOp`, the available variants, and the right method to convert have all changed across versions. The above is illustrative — confirm against docs.rs.

## Three validation styles observed in real apps

There is no "right" amount of validation. Different protocols want different things. The skill should *not* push elaborate validation onto every entry type — that's a category error. The three styles below are all valid; pick the one that matches the protocol.

### Style 1 — Minimal-permissive

Most apps. The validate callback returns `Valid` for nearly everything; only structural integrity is enforced (does the link target type match what the link declaration expected? does the entry deserialize?). dino-adventure, emergence, and acorn all sit here. Acorn pushes this further — it has *no* `validate` callback at all. The spec doc says what validation *should* do; the actual dead `validate.rs` modules are commented out.

**When this style is appropriate:** when the protocol is "the entry shape *is* the contract." If the schema is enough, you don't need elaborate logic. Most CRUD apps fit here.

**Watch out for:** singleton entries (only one allowed), uniqueness constraints, agent-identity checks. These all want validation but **cannot be enforced from `validate`** because validation runs against fragments of state without a global view. If you need a singleton, do a network fetch+count from the coordinator (with an acknowledged race, like acorn's `simple_create_project_meta`); there is no on-chain singleton primitive.

### Style 2 — Per-op declarative cross-entry checks (vines)

The middle ground. The validate callback dispatches on `Op`, leaves most variants permissive, and does real cross-entry work on a small number of high-stakes ops (`StoreEntry`, `RegisterCreateLink`). Inside those, it uses `must_get_action`, `must_get_entry`, and `must_get_agent_activity` (with tightly-scoped `ChainFilter`s) to verify invariants like:

- "the referenced parent record exists and is the right type"
- "the author is in the moderator set"
- "the rate limit hasn't been exceeded in the last N minutes"

Vines (`vines/dna/zomes/integrity/threads/src/validation_app_entry.rs`) is the canonical example. It does PP capability checks, rate limiting via chain walks, bead-type-matches-PP-rules checks, banned-words filters, and moderation link validation — all declaratively, no state machine.

**When this style is appropriate:** when there are real cross-entry invariants to enforce but they're per-op rules, not a state machine over the whole chain.

**Watch out for:** the chain-walk inside validation. It works but it's the most expensive part of validation; scope the `ChainFilter` tightly. Walk only as far back as the rule actually requires.

**Scratch space and the "perspective gap" myth.** It's tempting to think of inline (author-side) and non-inline (peer-side) validation as "seeing different things" — they don't, in any way that matters for correctness. Within a zome call, scratch accumulates pending commits. Then **all commits in the call are written atomically to the source chain before any peer can see anything via gossip.** A peer validating any action from that call sees the entire batch already on the chain — the same state the author validated against.

So: peers can't see things you haven't committed, but they aren't validating things you haven't committed either. There is no perspective gap to worry about.

What scratch space *is* actually for: within a single zome call, if commit B references commit A in the same call, B's inline validation needs to see A in order to verify the reference. The scratch holds A so the cascade can resolve it. After the call's commits are written atomically, the scratch is gone but the actions are durable on the chain — so peers can resolve the same dependency by getting it from the DHT.

What you actually need to avoid: validating against state that exists only in coordinator-memory or UI-memory (anything not on the chain). *That* is the genuinely non-deterministic case. Scratch is **not** the same as coordinator-memory; scratch is a staging area for actions that *will* be on the chain by the time the call returns. If your validation depends only on chain state plus scratch, every validator will reach the same verdict.

### Style 3 — Comprehensive state machine (unyt-app)

Validation encodes the full business logic. unyt-app's `transactor` integrity zome runs every action through a `TxProcessor` that walks the agent's entire chain and enforces a transaction state machine: proposal → commitment → acceptance → completion, with balance checks, ledger validation, and authorization rules. Updates are explicitly forbidden for many entry types ("Proposal update is not allowed"). Validation here *is* the protocol — the chain is the source of truth and validation enforces every invariant the protocol requires.

**When this style is appropriate:** when you have chain-spanning invariants like a ledger, a smart agreement, a multi-phase transaction. The protocol's correctness depends on validation being comprehensive.

**Watch out for:** validation cost. Each new action triggers a full state replay. Expensive but unavoidable for protocols of this kind.

## `Op` variants — what each represents

Roughly (verify against docs.rs/hdi/<version>):

- `Op::StoreRecord` — a record is being stored on the DHT (action + optional entry).
- `Op::StoreEntry` — an entry is being stored. Typically the place to validate entry-type-specific rules.
- `Op::RegisterAgentActivity` — an action is being registered as part of an agent's activity. Typically the place to validate cross-action invariants on a single agent's chain.
- `Op::RegisterCreateLink` — a link is being created. Typically the place to validate link-type-specific rules and link tags.
- `Op::RegisterDeleteLink` — a link is being deleted. Validate authorization.
- `Op::RegisterUpdate` — an entry is being updated. Validate that updates are allowed and that the new content satisfies invariants.
- `Op::RegisterDelete` — an entry is being deleted. Validate authorization.

**Each variant runs separately during dependency resolution.** A single new entry causes multiple Op-callback invocations (once for `StoreRecord`, once for `StoreEntry`, etc.). Don't duplicate logic — pick the *one* variant that's the natural place to enforce each rule.

## LinkTag-as-payload — fragile but valid

Vines packs structured data into link tags: a `Ban` link tag carries a msgpack-encoded `Vec<ActionHash>` of the prior flag links that justify the ban, and validation deserializes the tag and walks each referenced link to verify quorum. This is a real and valid pattern (`vines/dna/zomes/integrity/threads/src/validate_link.rs:66-89`).

**The fragility:** the encoder used to construct the tag must match the decoder used to validate it. Mixing `to_vec_named` vs default `decode` vs `serde_json::to_vec` silently fails — the tag bytes don't deserialize and the validation rejects the link with no clear reason. If you're touching a link tag that carries a payload, audit *both* the construction site and the validation site to confirm they use the same codec.

## Don't put validation in the coordinator

Validation belongs in integrity zomes only. A common Claude reflex is to add a "guard" check at the top of a coordinator extern ("if the user isn't a moderator, return an error"). That's not validation — it's UX. The integrity zome must enforce the rule independently, because peers will see the action via gossip and validate it without going through the coordinator extern at all.

## What integrity validation cannot do

Integrity validation is **local and deterministic per action**. It runs against a single action plus the dependencies that action explicitly references (via `must_get_*`) on the author's source chain. This makes it powerful for chain-walk-based and structural rules, but there are hard limits on what it can prove.

**Things integrity validation cannot enforce:**

1. **Singleton entries** ("only one of these can ever exist"). No validator has a global view of the DHT, so none can prove non-existence. Acorn's `validation_rules.txt:51-56` documents this directly: the singleton check has to live in the coordinator with a network fetch+count and an acknowledged race. There is no on-chain singleton primitive.

2. **Cross-agent uniqueness** ("no two agents can create the same X"). Two agents racing to create the same entry both pass validation locally, both gossip their action, and both end up on the DHT. EntryHash collision dedupes at the *entry* level (the entry itself only exists once in the DHT cache because identical content hashes the same), but the *Create actions* are distinct — different timestamps, different `prev_action`s, different action hashes. They both exist and both propagate.

3. **Anything outside the chain.** Validation has no access to coordinator-memory, UI state, the wall clock beyond what the action itself records, the network state, or any other agent's data not explicitly fetched via `must_get_*`. If your invariant depends on any of these, it cannot be enforced by integrity.

**What "dedup links" actually buy you:**

A common pattern is "use a per-entity dedup link recorded at create time instead of walking the chain." This is genuinely cheaper for the case it covers — but be honest about what it covers:

- **Per-author dedup is feasible.** A dedup link from `(agent, content_key) → action_hash` is O(1) to look up via `get_links`, and a single author cannot easily race themselves. You can validate-by-presence of the link, or coordinator-side check + structural integrity rule.
- **Cross-agent dedup is not feasible from validation alone.** Two agents can both `get_links` on the dedup anchor, both see no existing link, and both create their entry. They are both valid by every per-action check. The only mechanism for resolving this is best-effort coordinator-side checks plus reader-side tiebreak (e.g., earliest action hash wins). Don't claim integrity validation prevents the duplicate from being created — claim only that readers can ignore the loser.

So when recommending dedup links as an alternative to chain walks: scope the recommendation to per-author dedup, and explicitly call out that cross-agent dedup needs a different mechanism.
