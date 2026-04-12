# Eventual consistency, gossip, and zero-arc nodes

## The mental model

A Holochain DHT is **eventually consistent**, not immediately consistent. When agent A commits an entry on their source chain, that entry is *immediately* visible to A's local conductor — but it has to propagate to other agents through gossip before B can see it. Gossip is asynchronous and depends on network conditions, peer availability, and arc coverage.

This means:

- **A `get` can return `None` even when the data exists somewhere on the network.** B may simply not have heard about it yet. The data isn't lost; it just hasn't gossiped to B's slice of the DHT.
- **A `get_links` can return a partial set.** Some links may have gossiped, others not.
- **Reading your own writes is reliable.** A's conductor has A's source chain locally; A always sees A's own commits immediately.
- **Reading another agent's writes is *not* reliable** until gossip has settled.

## The arc model

Every agent in a Holochain DHT has an **arc**, which is the slice of the DHT keyspace they hold. An arc of 1.0 means "I hold the whole DHT" (full-arc); an arc of 0.0 means "I hold nothing" (zero-arc); intermediate values mean "I hold this fraction." Most desktops run with full or near-full arcs and hold most data locally. Most phones and battery-conserving devices run with zero or very small arcs.

This has a load-bearing consequence for API design that LLMs almost always miss:

- **A full-arc node can answer almost any `get` from local cache, fast.** `GetOptions::local()` works because the data is local.
- **A zero-arc node has no local data.** Every `get`, `get_links`, `get_details` must go to the network. `GetOptions::local()` will return `None` for *everything*. The node must use `GetOptions::network()` (or whatever the version-current equivalent is — verify against docs.rs/hdk/<version>) to get any results at all.

If your hApp targets *only* full-arc nodes (e.g., a desktop-only app), you can hardcode network/local choices in the zome and the UI doesn't need to think about it. If your hApp targets *both* contexts (e.g., a chat app with a desktop client and a mobile client), the API surface MUST expose local-vs-network choice so the UI can adapt per call.

## How to expose local-vs-network choice

Three patterns observed across real apps:

1. **Wrapper input struct** (mewsfeed, acorn): every fetch input is wrapped in `ZomeFnInput<T> { input: T, local: bool }`. The coordinator extracts `local` and uses it to pick `GetOptions::local()` vs `GetOptions::network()`.
2. **Separate externs** (dino-adventure): the coordinator exposes both `get_all_dinos` (network) and `get_all_dinos_local`. The UI calls whichever it wants.
3. **Strategy field on input** (vines): each fetch input has a `strategy: GetStrategy` enum with `Local`, `Network`, etc. variants.

All three are valid. The skill should detect which the project uses and stay in idiom. For new projects, the wrapper-input-struct approach is the most common and easiest to extend.

**Watch for the asymmetry trap:** in acorn the wrapper is *fetch-only*. Writes pass payloads raw. If you wrap a write payload in the same envelope, the macro doesn't destructure it and the write silently fails. When in doubt, look at how the project's existing writes are called.

## How UIs cope with eventual consistency

You cannot make Holochain reads synchronous. The UI has to be designed for partial / late / missing data from the start. Patterns from the reference apps:

- **Polling + signals hybrid** (dino-adventure): the UI polls every N seconds, but only hits the network 1 in M of those polls (e.g., 1 in 5). Signals from `post_commit` trigger immediate updates between polls.
- **Batched periodic refresh** (emergence's `neededStuff` pattern): the UI maintains a "what I need" registry, and a single timer fetches a batch every N seconds.
- **Push-based / signal-driven** (vines): the UI fires `fetch_x` and ignores the return value; results arrive asynchronously as `ZomeSignalProtocol::Entry` signals from the coordinator. The UI's state is entirely driven by signal subscription.

In all three cases, the UI **must** have a "partial result" rendering path. If your view assumes "I have all the data or I have none," a single missed `get` cascades into a blank screen. The mewsfeed bug `6a39e696-cc8d-43bf-899c-04a55ceb144e` is a real example: a single `Mew not found` from `get_followed_creators_mews_with_context` made the entire feed blank.

## Multi-DNA / cloned-DNA caveats

If the hApp uses cloned DNAs (acorn pattern: `clone_limit: 999, deferred: true`), each clone is its own DHT cell. The eventual-consistency model applies *per cell*. A user joining a new "project" cell sees no data until gossip catches up — and `dna_info()` does not return clone DNAs at startup, only the singleton DNAs. Code that assumes "the DNA exists at install time" gives "cell not found" errors.

## What this means for tests

Sweettest tests need explicit consistency waits (`await_consistency`) before asserting on data written by another agent. The wait must include *every agent whose actions the assertions depend on*, not just the writers. CI is slower than local — explicit timeouts (mewsfeed pattern uses 120 seconds) prevent flake. See `sweettest-bootstrap.md`.
