# DNA hash tripwire — what changes the hash, and why it matters

## Why this matters

A Holochain DNA's hash is a content-addressed identity. Two agents are on the same network if and only if their DNAs hash to the same value. The instant any byte that contributes to that hash changes, the new DNA is a *different network*. Peers running the old DNA cannot talk to peers running the new one. Action hashes already in use point at chains in a network nobody is in anymore. Apps deployed to users need a migration story. Users get very upset when this happens by surprise.

The hash is computed over a deterministic structure that includes:

- Each integrity zome's compiled wasm bytes
- The DNA's `properties` block (raw bytes, including any embedded assets)
- The set of integrity zome names and their declared entry/link types
- The DNA's `origin_time` and `network_seed` (when set)

It does *not* include:

- Coordinator zomes — these can be swapped out without changing the hash. Only integrity zomes affect identity.
- The UI bundle
- Tests
- README files, comments, or anything else not compiled into wasm

## Things that affect the DNA hash

1. **Any file under `**/zomes/integrity/**`.** Even cosmetic changes — renaming a private struct, reordering match arms, changing a comment that gets included via `include_str!` — recompile the wasm and may change the hash. *Always assume an integrity-zome edit changes the hash unless you can prove otherwise.*

2. **`#[hdk_entry_types]` or `#[hdk_link_types]` enum changes.** Adding, removing, reordering, or renaming a variant changes both the wasm and the declared type set. New entry types can sometimes be added at the end without breaking peers running the old DNA in *some* configurations, but the hash always changes. Treat any change here as breaking.

3. **`fn validate(op: Op)` and any helper it calls.** Validation logic lives in the integrity zome's wasm. Changing it changes the wasm.

4. **`dna.yaml` itself.** Changes to `name`, `network_seed`, `origin_time`, `properties`, the list of zomes, the wasm hashes recorded for each zome, or the `clone_limit` value all change the DNA hash.

5. **Any file referenced from `dna.yaml`'s `properties` block.** This is the gotcha that catches people. If `properties` includes a base64-encoded asset (an SVG icon, a JSON schema, anything), the *bytes of that asset* are part of the DNA hash via `genesis_self_check`. Swapping the icon for a slightly different one breaks the hash. Vines does this with a 6KB base64 SVG group icon — see `vines/dna/workdir/dna.yaml` and `vines/dna/zomes/integrity/threads/src/callbacks.rs`.

## Things that do NOT affect the hash

- Anything under `**/zomes/coordinator/**`
- Tests, fixtures, sweettest crates
- The UI bundle, TS bindings, generated `*.ts` files
- `package.json`, `Cargo.toml` for the workspace root, README, license files
- The order of fields *inside* an entry struct (these don't affect the macro-generated `EntryDef`, only the wasm bytes — but the wasm bytes still change, so see #1 above)

## What to do when the user asks for a fix that touches the DNA

1. **Check the branch.** `git branch --show-current`. If it's `main`, `master`, `production`, or `release/*`, the tripwire is in hard-stop mode.

2. **Ask: can the same fix be made elsewhere?** Coordinator code, UI code, signal handling, caching, post_commit hooks, or a side-channel zome (vines's `authorship` pattern) can often achieve the same user-facing outcome without touching integrity. *Always look for a non-DNA-impacting path first.* If you find one, propose it.

3. **If the fix really needs an integrity-zome change:**
   - On a protected branch: refuse to write the change yet. Explain the impact. Offer two options:
     - "I can write this if you confirm you want a DNA-hash change for this turn"
     - "Or you can move to a dev branch (`git checkout -b feature/<name>`) and the warnings will stop"
   - On a dev branch: write the change but include a clear "this is a DNA-hash-impacting change because <reason>" line in your response so the user can catch it if your judgment was off.

4. **Never bundle DNA changes with non-DNA changes in one commit.** Split them. The user wants to be able to review and revert DNA changes independently of cosmetic UI fixes.

## The "side channel" escape hatch

Sometimes you genuinely need new persistent state that's tied to existing data, but the existing integrity zome is locked down. Vines's `authorship_integrity` zome is a good pattern to learn from: it's a *separate* integrity zome whose only job is to record original-author attribution as links. New side-channel zomes can be added to a DNA without breaking the existing zomes (the DNA hash *does* change because the integrity zome set changes, but the existing zomes' contents are unchanged), and the side channel can record new state about old data.

This isn't a way to avoid the tripwire — adding a new integrity zome still changes the hash — but it's a way to add new state without modifying the wasm of zomes that other parts of the system depend on. For migrations specifically, side-channel zomes mean migrated data can preserve original-author attribution across DNA-hash changes, which is otherwise hard.
