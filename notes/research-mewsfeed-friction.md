# Mewsfeed Claude-History Friction Findings

## Framing

Data mined: three staged directories

- `/tmp/mewsfeed-claude-history/mewsfeed/` — main project, 61 top-level entries, ~130 `.jsonl` files including subagent threads, ~54 MB total
- `/tmp/mewsfeed-claude-history/mewsfeed-fishy/` — fork working on `holo-web-conductor` integration branch, subagents only (no top-level main-session jsonl files), ~7.7 MB
- `/tmp/mewsfeed-claude-history/mewsfeed-kangaroo/` — kangaroo wrapper, 1 main session jsonl + subagent, ~684 KB

191 `.jsonl` files total. Date range of content spans Jan 2026 through late March 2026. The most recent substantive Holochain-specific work clusters around March 19–27, 2026 on branches `main-0.6`, `lick-fixes`, and `zome-merge`. Much of Jan–Feb is the 0.4→0.6 migration on branches `main-0.4` → `main-0.6`. The mewsfeed-fishy work is almost entirely browser-extension / host-function plumbing rather than zome work.

Sampling approach: enumerated sessions, read `aiTitle` summaries to cluster by topic, then drilled into the Holochain-specific sessions line-by-line. I skipped the browser-extension infrastructure sessions (linker, hwc, Firefox ext, deploy scripts) except where they touched zome behavior. Within sessions I used grep to find verbatim user-text lines (interrupt messages, "no", "wait", "actually") and read those turns directly. I did not read subagent research threads exhaustively; I sampled the research summaries which are typically the last line of each subagent file.

HDK/HDI version in play for the recent work: **HDI 0.7.0 / HDK 0.6.0** on branch `main-0.6`, confirmed in `de011c7d` session subagent prompt and in the migration reference subagent `3d619308/agent-a96324b.jsonl:101`. Earlier sessions were on HDI 0.5.1 / HDK 0.4.1.

Anti-pattern note confirmed by the data: mewsfeed's DNA originally had **5 integrity zomes + 6 coordinator zomes** with cross-zome calls via `call_local_zome()` AND **hardcoded zome ordinal indices** (`MEWS_ZOME_INDEX = 1`, etc.) inside `agent_to_notifications.rs` — see `2e88c037-332a-4d31-bf84-5b6f509df1d2.jsonl:607`. A large merge effort on the `zome-merge` branch collapsed this down to essentially one integrity + one coordinator zome. Old sessions where Claude is faithfully working inside this anti-pattern still count as friction because the codebase itself was pushing Claude into wrong-shaped solutions.

---

## Recurring friction patterns

### 1. Claude assumes Holochain semantics instead of reading source

This is the single most frequent category in the high-signal corrections. Claude proposes a fix based on a plausible mental model of how a Holochain host-fn behaves, the user stops him and says "check the actual implementation," and the subagent then returns a corrected understanding that contradicts the first guess.

- **`de011c7d-83cc-4a9e-a339-e111f4dc9e4d.jsonl:304`** — duplicate-mewmew validation bug. User: "WAIT. Can you confirm that this behavior is what Holochain does in its implementation of must_get_agent_activity? (look in ../holochain) I'm not at all confident that that's the correct action to take." Claude had been about to ship a fix based on its assumption that `ChainFilter` walks backward eagerly; the dispatched subagent research (`subagents/agent-aac1c0f78dec29ef4.jsonl:111`) came back with a multi-paragraph corrected answer explaining that `ChainFilter` calculates a min..=max sequence *range* first, then iterates, and that in inline validation the scratch space is merged in. Claude's original fix would have been wrong for the scratch-space case.

- **`8dabc126-860d-429a-baa7-846b0582c0a1.jsonl:46`** — msgpack deserialization error from the profiles zome. Claude's first guess blamed the chain state. User: "Your assumption is wrong. Please look into exactly what the AgentActivity structure is supposed to be. I'm quite sure that our host_fn is passing back illegal format to the zome call when its own request to the linker times out... In that case I think we shouldn't be handing back invalid data, we should be handing back an error code." Claude had no idea that the real problem was in the host-fn layer returning a mangled `AgentActivity` on timeout. And at `:109` the user follows up: "For number 2, what does holochain do? (look in ../holochain)" — same pattern, user forces Claude to check source.

- **`8dabc126...jsonl:186`** — after a piecemeal fix, user: "actually, do this as a plan. There may be many other places where this kind of error handling needs to be done correctly to match how Holochain does it, and it should work the same way for all host fn, not be one off per host_fn." Claude was fixing one symptom; user pushed for systematic matching of Holochain's real behavior across all host functions.

- **`835670c5-4da2-41a7-9387-1f251a255e56.jsonl:651`** — "Profile link target is not of ActionHash" error investigation. Claude proposed multiple theories including prior-DNA drift. User: "Well, of all your options, the only possible one is 2, because we haven't changed the profiles coordinator, nor has there even been a 'prior DNA'. So, what you get to do prior to us fixing validation in the profiles, is look deep into the Mewsfeed code to see if anywhere it's passing something to the profiles zome that would account for this." Claude's theory of the world included non-existent state transitions; user constrained the search space.

**Takeaway**: Claude reaches for plausible-sounding Holochain explanations without ever opening `../holochain` or the cargo-registry sources. The users in these sessions have a sibling clone of the holochain repo specifically so Claude can look at source, and users repeatedly have to tell Claude to use it.

### 2. Claude proposes breaking DNA changes when a non-breaking fix exists

- **`de011c7d-83cc-4a9e-a339-e111f4dc9e4d.jsonl:292`** — User verbatim: "no, revert it for now (you can copy the change to a side branch for later) but we want to fix this problem without having to redeploy the DNA so I don't want a breaking chagne." Claude had already modified validation logic in an integrity zome, not realizing that means a new DNA hash.

- **`835670c5-4da2-41a7-9387-1f251a255e56.jsonl:487`** — "your fix included DNA changes, why?" Followed at `:501`: "actually it should be a DHT item we can control. Why isn't there validation that would prevent that from happening. I would like to know why that's happening. AND, lets move the non-DNA modifying change back to the main worktree so we can test that now." And at `:609`: "move the non-dna fix first, Then continue the research." The user had to give Claude a three-step plan because Claude was conflating a UI/caching fix with a speculative integrity-zome change.

**Takeaway**: Claude does not treat "this touches an integrity zome" as a tripwire. Any change to integrity zome code, entry types, link types, or validation alters the DNA hash and therefore requires redeployment, migration, or a side-branch. Users treated this as a hard constraint and Claude ignored it by default.

### 3. Wrong assumptions about DHT data locality / eventual consistency

- **`4b67de53-94c1-4056-ad2d-93a3ca47bd82.jsonl:1021, :1067, :1197, :1224`** — session titled "Investigate linker 500 errors in deploy logs". Multiple corrections in a row:
  - `:1021`: "No they are not stale. I deleted logs, I even deleted the full instances. The are happening live."
  - `:1067`: "NO. THe problem is that that shouldn't be happening AT ALL because the linker and the conductors are running right now. So why are we getting those errors. THey should ONLY be happening if the there were network errors but there arent'... I think these are false errors introduced by the changes to hwc when you added in the error handling."
  - `:1197`: "no npm run start creates a brand new network new seed with new conductors in that network. I go and create two new agents, and then one agent can find data of another agent in the explore/discover route, and it works. But that's not working in hwc any more. explor only shows local data."
  - `:1224`: "no. that's not the issue. We just started totally fresh. ANd we just published data. You saw the publishes. LOok in the conductors logs."
  
  Claude kept reaching for stale-state / cache / timeout explanations. User kept having to point out that the symptoms were incompatible with those explanations and demand Claude look at conductor logs.

- **`6a39e696-cc8d-43bf-899c-04a55ceb144e.jsonl:3`** — opener: "In the testing deployment, my machine with both conductors crashed. I'm able to restart the conductors and linker, and re establish connection, but now none of the non-local resource exist anywhere to be seen... the UI does not degrade gracefully. It doesn't show my own tweets or that I tried to comment on tweets that I can find right now." The root issue is that `mews::get_followed_creators_mews_with_context` returns a `Guest` error `"Mew not found"` and the UI treats a single failure as fatal. This is a classic eventually-consistent-DHT design gap: a zome that assumes all referenced data is reachable, and a UI that has no "partial result" path.

- **`14e0cde6-619a-4f6a-9862-c8f36ffa5416.jsonl:3`** — "look at the uncommitted changes made to the DNA. In the happy path case with hwc doing network queries always for getlinks that return data, could any of these changes return different data than before the changes were made." Note how precisely the user frames the question — they understand that `GetOptions::local` vs `GetOptions::network` materially changes semantics. This is exactly the corner that Claude would otherwise be sloppy about.

**Takeaway**: Claude default-assumes that "data exists in the DHT → you can get it back." Real Holochain apps have to reason about: local cache vs network fetch (`GetOptions::local` vs `GetOptions::network`), whether an agent has gossiped their chain yet, partial arcs, and the fact that a `must_get_*` call in validation sometimes returns `UnresolvedDependencies` and sometimes returns an error. Claude reaches for cache/stale-state explanations even when the user has already ruled them out.

### 4. Sweettest / Tryorama: `dhtSync` called with wrong agent set

- **`f4e3255c-7796-494a-bce4-746632b0f89d.jsonl:19, :37`** — intermittent CI failures on Ubuntu. The test file `tests/src/mewsfeed/follows/follower-to-creators.test.ts` created 6 agents (`alice, bob, carol, john, steve, mary`) and then called `await dhtSync([alice, bob], alice.cells[0].cell_id[0]);` before querying — only waiting for 2 of 6 agents to be consistent. Claude correctly identified the race and changed the call to `dhtSync([alice, bob, carol, john, steve, mary], alice.cells[0].cell_id[0], undefined, 120000)`. This was a mewsfeed-authored bug, not a Claude bug, but it's exactly the kind of bug a Holochain skill should help catch by default. The code was present in multiple test files in the same pattern.

- **Note**: This is one of the rare cases where Claude got it right on the first pass. Worth learning from the prompt shape: user gave a specific CI URL, Claude web-fetched the log, identified the race, and fixed it without needing source deep-dive.

### 5. Hardcoded zome indices in cross-zome link queries (pre-merge)

- **`2e88c037-332a-4d31-bf84-5b6f509df1d2.jsonl:607`** (the architectural plan) explicitly calls this out:
  > **CRITICAL: agent_to_notifications.rs uses hardcoded zome indices:** `MEWS_ZOME_INDEX = 1, FOLLOWS_ZOME_INDEX = 2, LIKES_ZOME_INDEX = 3, AGENT_PINS_ZOME_INDEX = 4`. These reference specific LinkType ordinals for cross-zome link queries. After merge, all link types will be in a single integrity zome so indices change.
  
  This is pre-existing mewsfeed code, not something Claude wrote, but Claude has to navigate around it in most sessions that touch notifications. The existence of this anti-pattern in the codebase is itself the thing the skill should warn about: if a newer Holochain app is tempted to reach across zomes with a raw ordinal, that's almost always wrong.

- **Subagent `de011c7d/.../agent-a9a0d32b8b3489494.jsonl` prompt at line 84 of parent session**: Claude asked for help finding a dynamic way to replace `(ZomeIndex(1), EntryDefIndex(0))` in an integrity-zome validation with something that could be looked up at runtime. The question itself is a symptom that the original code was brittle by design.

### 6. `ZomeFnInput<T>` wrapper forgotten on cross-zome calls

- **`b5dc2e41-4272-485d-a2c2-a06a49dce034/subagents/agent-ab74c2b.jsonl:1`** — research prompt: "The key question is: are cross-zome calls to the profiles zome wrapping their inputs in a ZomeFnInput with the local flag? The new profiles coordinator likely expects ZomeFnInput<T> wrapping. If cross-zome callers pass raw inputs without the ZomeFnInput wrapper, they might fail or only return local data." This is mewsfeed-specific convention (defined in `crates/hc_zome_input/`) — a wrapper struct with a `local: bool` flag that tells the target zome whether to use `GetOptions::local` or network. Callers that forget the wrapper silently get local-only results. Confirmed failure mode: "searching for other agents' profiles returns nothing, but the local agent's profile is found."

- The TypeScript side calls `wrapInput({...})` in most places (seen throughout `f4e3255c` test file content) but not all; the wrap was added specifically to fix the "local only" regression.

### 7. `must_get_agent_activity` used as a general chain-walk primitive

- **`4061dbf1-3091-473d-9f51-5279a1d1d34f.jsonl:3, :34`** — `get_joining_timestamp_for_agent` is expensive because it calls `must_get_agent_activity` (full chain fetch) just to extract the first action's timestamp. User: "what is the data that's pulled out of that call, just the joining timestamp? And if sow why do we need the full agent activity? Couldn't we just get the very first entry?" And at `:46`: "the call only needs to be made ONCE in the lifecycle of learning about an agent. So I think it should be done as it's own single cache." This is the kind of performance trap where a host function's semantics match the *name* but not the *cost*, and a novice reaches for it reflexively. Claude didn't spot this until the user flagged it.

### 8. Claude forgets what it just changed / missed diffs

- **`1cd80c35-e78f-4e92-9d3b-1bbf1abce8d0.jsonl:1179`** — "you missed the getagentactivity change. That's the big one that I need separated out." User was asking Claude to separate concerns into multiple commits and Claude silently dropped the most important change.

- **`2e88c037-332a-4d31-bf84-5b6f509df1d2.jsonl:955`** — "why didn't you merge in the ping zome?" Claude had left out ping from the zome-merge plan. User at `:962`: "haha I think that the whole purpose of the ping call is to find out if the wasms are loaded, so it SHOULD be part of the main mews wasm. Look at the codebase and see where it's called to confirm this." Claude had treated ping as a standalone health-check and didn't think about *why* a ping zome exists in a wasm-loaded-on-demand runtime.

### 9. Validation logic put in the wrong spot (entry validation reaching out to link space)

- In the mewmew validation code (`de011c7d`, integrity zome `dnas/mewsfeed/zomes/integrity/mews/src/mew.rs` lines 38–94 as read into the session), the code does `must_get_agent_activity` inside the `Op::StoreRecord` branch to look backward through the author's chain for prior mewmews of the same original. This is structurally brittle (uses hardcoded `ZomeIndex(1), EntryDefIndex(0)` to identify mew entries) and fragile under scratch-space semantics (see friction #1). Recent fix direction visible in the session: collapse the indices by merging all link/entry types into one integrity zome, and on the validation side, instead of walking the chain, record a dedup link at create time. This fix direction is not fully landed in the sessions I read but the user's framing at `:304` is the seed.

### 10. Claude wraps zome changes in DNA-migration-impacting scope creep

Related to #2 but distinct: when asked to optimize *one* thing (avatar caching, mew fetching) Claude bundles zome signature changes into the same commit.

- **`835670c5...jsonl:487`**: "your fix included DNA changes, why?" — direct.
- Several sessions show users having to explicitly ask Claude to "move the non-DNA changes to one branch and the DNA changes to another."

---

## Things Claude did well

- **DhtSync race fix** (`f4e3255c-7796-494a-bce4-746632b0f89d.jsonl`) — one-shot correct diagnosis and fix from a CI URL.
- **Holochain 0.4 → 0.6 migration reference document** (`3d619308.../subagents/agent-a96324b.jsonl:101`) — Claude's summary of Emergence's 0.6 setup is accurate and dense: HDI 0.7.0 / HDK 0.6.0 in `Cargo.toml`, `holonix.url = "github:holochain/holonix/main-0.6"`, the `hdk_entry_types` / `hdk_link_types` / `hdk_extern` macro triple, the `FlatOp` validation dispatch pattern, the `post_commit` signal-emission pattern, and the `@holochain/client ^0.20.0` / `@holochain-open-dev/profiles ^0.600.0` client versions. This was produced as a sub-agent research task from a clear prompt ("Explore the ../emergence project which is a Holochain 0.6 Svelte app") and came out essentially ready to use as migration documentation. The prompt shape matters: the user pointed at a working reference codebase (Emergence) rather than asking Claude to recall patterns from training data.
- **Architectural comprehension of the multi-zome layout** (`2e88c037.../subagents/agent-a1094a2b539bba2ad.jsonl` prompted at parent session line 406) — Claude correctly enumerated the 5 integrity + 6 coordinator zome topology, all cross-zome calls from mews into follows/likes/agent_pins, the shared-types crates, and the hardcoded zome-index footgun in `agent_to_notifications.rs`. The plan at session line 607 is structurally sound — it identifies the dna.yaml update, the hc_call_utils removal, the zome-index reassignment, and the corresponding UI callZome name changes.
- **must_get_agent_activity deep dive** (`de011c7d.../subagents/agent-aac1c0f78dec29ef4.jsonl:111`) — produced a careful, accurate summary of ChainFilter range calculation, scratch-space merging during inline validation, and the difference between inline and non-inline validation cascade construction. This is worth preserving verbatim as Holochain reference material; it cites specific files and line numbers in the holochain crate. **But note**: this was only produced *after* the user forced Claude to verify. The good output came from a directed sub-agent research task, not from Claude's first-guess behavior.

Pattern in what worked: clear, directed research tasks pointing at specific Holochain repo paths, with "do NOT write code, just research" instructions, consistently produced accurate Holochain explanations. Claude's first-guess behavior when reasoning directly in a conversational turn was much worse.

---

## Highest-signal recent moments

These are moments where the user made an unambiguous "this is the right way" correction that a future skill should preempt.

1. **Check source instead of guessing** — `de011c7d-83cc-4a9e-a339-e111f4dc9e4d.jsonl:304` — "WAIT. Can you confirm that this behavior is what Holochain does in its implementation of must_get_agent_activity? (look in ../holochain) I'm not at all confident that that's the correct action to take." Direct instruction to verify against the actual holochain source clone at `../holochain`, which these users keep sibling-clone-style for exactly this purpose. A skill should know that `../holochain` exists in the typical layout.

2. **Don't touch the DNA for what should be a UI/caching fix** — `835670c5-4da2-41a7-9387-1f251a255e56.jsonl:487` — "your fix included DNA changes, why?" And `:501` — "actually it should be a DHT item we can control. Why isn't there validation that would prevent that from happening... AND, lets move the non-DNA modifying change back to the main worktree so we can test that now."

3. **Non-breaking over breaking, always prefer** — `de011c7d-83cc-4a9e-a339-e111f4dc9e4d.jsonl:292` — "no, revert it for now (you can copy the change to a side branch for later) but we want to fix this problem without having to redeploy the DNA so I don't want a breaking chagne."

4. **Systematize error handling across all host functions, not per-callsite** — `8dabc126-860d-429a-baa7-846b0582c0a1.jsonl:186` — "actually, do this as a plan. There may be many other places where this kind of error handling needs to be done correctly to match how Holochain does it, and it should work the same way for all host fn, not be one off per host_fn."

5. **Understand why a zome exists in the first place** — `2e88c037-332a-4d31-bf84-5b6f509df1d2.jsonl:962` — "haha I think that the whole purpose of the ping call is to find out if the wasms are loaded, so it SHOULD be part of the main mews wasm. Look at the codebase and see where it's called to confirm this." Claude was ready to leave ping separate because "it's a standalone health check" — a surface reading that missed the Holochain-specific motivation (per-zome wasm lazy-loading).

6. **`must_get_agent_activity` is expensive; don't call it for a single field** — `4061dbf1-3091-473d-9f51-5279a1d1d34f.jsonl:34, :46` — "why do we need the full agent activity? Couldn't we just get the very first entry?" and "the call only needs to be made ONCE in the lifecycle of learning about an agent. So I think it should be done as its own single cache."

7. **The ZomeFnInput<T> wrapper convention** — `b5dc2e41.../agent-ab74c2b.jsonl:1` — "are cross-zome calls to the profiles zome wrapping their inputs in a ZomeFnInput with the local flag? ... If cross-zome callers pass raw inputs without the ZomeFnInput wrapper, they might fail or only return local data." This is a mewsfeed-specific convention but the *general* lesson is: projects often have thin wrapper types that gate local-vs-network fetching, and losing the wrapper silently turns network lookups into local-only.

8. **Zome architecture: fewer zomes is usually better** — `2e88c037-332a-4d31-bf84-5b6f509df1d2.jsonl:3` — "what is the value of having so many different zomes in mewsfeed? Wouldn't it be better to join them into one and thus preventing so many different wasms being installed and having to do the cross-zome calling? That seems like it slows things down a bunch." This is the user teaching Claude the current best-practice: the old scaffolder output (many small zomes) is actively harmful for performance because each wasm is loaded and instantiated separately and cross-zome calls have measurable overhead. The full zome-merge plan is in the same session.

9. **When tests are flaky, check `dhtSync` agent lists** — `f4e3255c-7796-494a-bce4-746632b0f89d.jsonl:19, :37` — the pattern is: created 6 agents but only synced 2. Always sync all agents whose actions the assertions care about. Also add explicit timeouts (120s is used here) because CI is slower than local.

10. **Look at the running system's actual logs** — `4b67de53-94c1-4056-ad2d-93a3ca47bd82.jsonl:1067, :1224` — repeated instruction "check the conductor and linker logs" after Claude conjectured about stale state. Claude's instinct to reach for cache/timeout explanations when presented with "this should work" is strong enough that the user had to correct it several times in one session.

---

## Implications for the skill (raw, not synthesized)

- The skill probably needs to address the reflex to fix Holochain bugs without reading Holochain source. When a user has `../holochain` as a sibling checkout, that's a signal Claude should use it aggressively. Validation semantics, `must_get_*` semantics, and cascade behavior are not something to guess about.

- The skill probably needs a tripwire: "touching integrity zome code / entry types / link types = DNA hash changes = breaking". Before editing integrity zomes, explicitly acknowledge whether the user is on a branch where breakage is OK.

- The skill probably needs to know that *mewsfeed-era* Holochain apps were scaffolded to use multiple small zomes with cross-zome calls, and that the current direction (as of the `zome-merge` branch) is to collapse those into one integrity + one coordinator zome for performance. It should also know that this means dropping `hc_call_utils::call_local_zome()` and any hardcoded zome ordinals.

- The skill probably needs to warn about `must_get_agent_activity` cost — it's a whole-chain fetch and should never be used to retrieve a single field like "joining timestamp". The alternative patterns are: fetch only what's needed (`get(first_action_hash)`, cache once per agent, or record the value at create-time as its own link/entry).

- The skill probably needs to cover Tryorama / Sweettest hygiene: `dhtSync` must include every agent whose actions the assertions depend on; CI-safe timeouts need to be explicit (the mewsfeed pattern uses 120000 ms); `await_consistency` equivalents are required before cross-agent reads.

- The skill probably needs to cover the lazy/partial DHT read story: what does the UI do when `get`/`get_links` returns `None` because the data hasn't gossiped yet? A "Mew not found" Guest error should not cascade into a blank feed. Graceful degradation is the norm, not the exception.

- The skill probably needs to encode the `ZomeFnInput<T> { local: bool, ... }` pattern or at least know that in mewsfeed-style codebases, wrapper structs control local-vs-network fetching and losing the wrapper silently breaks network-reachable-data queries.

- The skill probably needs to distinguish "the host_fn layer returned garbage" from "the zome's own logic is wrong". When decoding errors mention msgpack byte arrays like the `8dabc126` deserialization trace, the skill should know to suspect the host-fn boundary (e.g., browser-extension ribosome, hwc) before suspecting zome code.

- The skill probably needs to know the actual HDI 0.7.0 / HDK 0.6.0 API surface (the 0.6 patterns that Claude's own migration-reference subagent correctly summarized in `3d619308.../agent-a96324b.jsonl:101`): `#[hdk_entry_types]` with `#[unit_enum(UnitEntryTypes)]`, `#[hdk_link_types]`, `FlatOp<EntryTypes, LinkTypes>` via `op.flattened()?` in `validate`, `#[hdk_extern(infallible)] fn post_commit(...)` for signals, `GetOptions::local()` (note `.local()` not `::local`), `@holochain/client ^0.20.0` and `@holochain-open-dev/profiles ^0.600.0` on the UI side. Earlier 0.4/0.5 syntax is now wrong and anything the model picks up from pre-0.5 documentation is stale.

- The skill probably needs to know about the `main-0.6` branch convention in `holonix`, `p2p-shipyard`, `hc_zome_profiles_integrity`, `hc_zome_profiles_coordinator`, and `hc_zome_file_storage_coordinator` — migration bugs in the session history were partly caused by dependency branches not moving in lockstep.

- The skill probably should NOT try to reteach Claude how to write `get`, `create_entry`, `create_link`, or `update_entry` — those worked fine in the session data without correction. The friction is at the semantic layer (caching, consistency, zome topology, validation context) not the syntactic layer.

- The skill probably needs to encode the "many small commits, please" instruction (`1cd80c35...:1179`, `835670c5...:609`). Mewsfeed users consistently want DNA changes, UI changes, test changes, and cache/signal changes separated so each can be reviewed and tested independently. Claude's default of "one fix = one commit" bundles things together that should be split.
