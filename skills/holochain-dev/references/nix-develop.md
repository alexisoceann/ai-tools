# Running things in the nix dev shell

## The rule

If the project has a `flake.nix` file at its root, **every command that needs Rust, cargo, the holochain CLI, sweettest, or any other dev dependency must be run inside the dev shell**. The way to do that without entering an interactive shell is to wrap the command:

```sh
nix develop -c <your command here>
```

Examples:

```sh
nix develop -c cargo build
nix develop -c cargo test
nix develop -c cargo test -- --nocapture
nix develop -c cargo build --release --target wasm32-unknown-unknown
nix develop -c hc app pack workdir
nix develop -c bash scripts/ts-bindings.sh
```

This is **not optional**. The dev shell sets up the right Rust toolchain version, the right `wasm32-unknown-unknown` target, the right linker, the right holochain binaries, and any other native dependencies the project needs. None of those are present in the host environment by default.

## Why LLMs get this wrong

Out of the box, an LLM that sees a build error like:

```
error: linker `cc` not found
```

or

```
error[E0463]: can't find crate for `core`
note: the `wasm32-unknown-unknown` target may not be installed
```

or

```
error: could not find `Cargo.toml` in `/...`
```

will reach for "global" fixes: install Rust via rustup, install the wasm target via `rustup target add`, install gcc, etc. **All of these are wrong.** They don't fix the problem (the next command will hit the next missing dependency), and they pollute the host system with toolchain versions that may conflict with the project's pinned toolchain.

The right fix is *always* the same: prepend `nix develop -c`. The flake declares everything the project needs and the dev shell makes it available. If a command works inside an interactive `nix develop` shell, it will also work via `nix develop -c <cmd>`.

## How to detect a flake project

```sh
test -f flake.nix && echo "uses nix flake"
```

If `flake.nix` exists, assume nix is the supported development environment. Most Holochain projects from 2024 onward use nix flakes.

Some older projects use `nix-shell` (the pre-flake interface). If you see a `shell.nix` instead of (or in addition to) a `flake.nix`, the equivalent wrapper is:

```sh
nix-shell --run "<your command>"
```

But check what the project's README or scripts/ directory does — most projects have a one-liner script like `scripts/test.sh` that already wraps the right invocation.

## Common gotchas

### "I'll just install rustup globally"

No. This breaks the toolchain pinning the flake provides. The flake might be on Rust 1.80 because that's what the holochain crate version requires; your global rustup might be on 1.85 and miscompile. Always go through the flake.

### "I'll cache the dev shell with `nix develop` once and then run plain commands"

This works inside a single interactive shell session, but the moment you exit and return, the environment is gone. For one-shot commands (which is what you're usually doing as an agent), use `nix develop -c` every time. It's slightly slower on first invocation while nix evaluates the flake, but it's fast on subsequent invocations because the result is cached.

### "I'll use direnv to auto-activate"

direnv with `.envrc` containing `use flake` is a fine pattern for interactive development, and projects that use it often have an `.envrc` in the repo. If direnv is set up *and* you've allowed the directory, then plain `cargo test` works inside the project directory because direnv has activated the environment for you. **But don't assume it's set up.** If you're not sure, fall back to `nix develop -c <cmd>`.

### "First run takes forever"

`nix develop -c <cmd>` on a fresh machine has to download and build everything in the flake. This can take several minutes. Don't interpret the wait as a hang; it's first-run cache population. Subsequent invocations are fast.

### "It works in my terminal but not when the agent runs it"

The agent's shell environment is different from a developer's interactive shell. direnv and shell aliases don't carry over. Always use `nix develop -c <cmd>` explicitly in commands the agent runs, even if the user is using direnv interactively.

## When the project does NOT use nix

If there's no `flake.nix` and no `shell.nix`, the project is using whatever toolchain is installed globally. In that case, plain `cargo test` etc. work — but verify the project's README to see what toolchain version is expected, and warn the user if their global toolchain seems out of date relative to the project's `rust-toolchain.toml` (if present).

This is rare for current Holochain projects. Most use nix.
