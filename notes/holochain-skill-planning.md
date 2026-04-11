# Holochain Skill Planning — Handoff Notes

**Created:** 2026-04-10
**Source:** Continued from a Claude Code session in `/home/eric/code/metacurrency/holochain/moss`
**Purpose:** Carry planning context into work happening in this `ai-tools` repo.

---

> **Update 2026-04-10:** Research phase complete. Findings in [research-app-patterns.md](research-app-patterns.md) and [research-mewsfeed-friction.md](research-mewsfeed-friction.md). Synthesized design brief in [skill-design-v1.md](skill-design-v1.md) — this planning doc is now historical context; the design doc is the working artifact.

## Big picture

Eric is building `ai-tools` as a public repo collecting various AI-agent assets — skills, plugins, MCP servers, etc. — relevant to the Holochain ecosystem. The first concrete deliverable under discussion is a **Holochain development skill** for Claude Code.

## Decisions reached

### 1. Research app-development history 

   - Look at reference repos as example apps and distill patterns for building holochain apps.
   - Look at claude-coding history i.e. inside (~/.claude/projects/) for the work that has been done in those apps over time and analyze what has worked with claude and what hasn't as input into the skill-creator tool.

### 1.5  UI state management drives DNA development

  - The patterns used in the DNA, especially the coordinator zome, have strong effects on how the UI state-management code needs to be developed.
  - This skill needs to help developers manage constraints that aren't usual in centralized tooling, i.e. failure-mode and performance constraints that are different when designing for distributed systems.
  - Many of the examples use @holochain-open-dev npm libraries for lazy-loading and other state management and we need to discuss whether to use those patterns in the skill or not.

### 2. iterate a plan based on this research.

### 3. Repo location: `ai-tools` is the right home (for now)

### 4. Skills are Claude-specific; plan for portability anyway

The "skill" file format (frontmatter, `Skill` tool, `.claude/skills/` directory, plugin packaging) is **Claude Code-specific**. Other agents (Cursor, Aider, Cline, Continue, Copilot, Windsurf) each have their own incompatible mechanisms (`.cursorrules`, `CONVENTIONS.md`, `.clinerules`, etc.). MCP is cross-agent but solves a different problem (tool/data exposure, not procedural knowledge). `AGENTS.md` is an emerging informal convention but not standardized.

**Implication for `ai-tools` structure:** The long-term right shape is **canonical content as plain markdown + thin per-agent adapter layers**. So a Holochain skill might look like:

```
ai-tools/
  skills/
    holochain-dev/
      content/              # vendor-neutral markdown — the actual knowledge
        zome-scaffolding.md
        sweettest-testing.md
        validation-patterns.md
      claude-code/          # Claude Code skill wrapper (frontmatter + Skill tool integration)
      cursor/               # .cursorrules / .mdc files pointing at content/
      agents-md/            # AGENTS.md snippet
      README.md
```

For v1, **don't generalize prematurely** — build the Claude Code version, find out what knowledge actually matters by using it, then refactor to the layered structure once the content has proven itself. Vendor-neutral docs written speculatively tend to be bland and help no agent in particular.

### 4. Distribute as a Claude Code plugin from day one (eventually)

Once the skill is stable, package it as a Claude Code plugin so users can `claude plugin install <repo>` rather than manually copying files. This makes the "where does it live" question much less load-bearing because installation works the same regardless of host repo.

For initial development, a loose `.claude/skills/` folder is fine.

## Hard constraints to encode in any Holochain skill

These come from Moss's `CLAUDE.md` and apply broadly to Holochain dev with Claude:

- **Test-driven development required** — write or update a sweettest test alongside any zome change.
- **Strong typing** — Any TypeScript code should use strong types; Rust should avoid `unwrap()`/`expect()` in production paths.
- **Coordinator vs integrity zome separation** — validation logic only in integrity zomes.
- **No Claude footers in commit messages** (per Eric's global preference).

## How to bootstrap the actual skill

Use the `skill-creator` skill (already installed in Eric's Claude Code setup):

```
Skill(skill: "skill-creator", args: "<brief>")
```

A good brief covers: skill name, trigger conditions, what the skill enforces, reference material (local Holochain source paths, example zomes in Moss like `dnas/group/` and `example/dnas/`), what the skill produces when invoked, and what to avoid (generic Holochain tutorial content). Keep the skill body under ~150 lines; put long reference material in `references/` files.

A draft brief is in the prior conversation — recreate it from the constraints above when ready.

## Reference:

For developing the skill and When the skill needs example code, these are the production-quality references:

[scaffolding](../../scaffolding)
[kando](../../kando)
[presence](../../presence)
[emergence](../../emergence)
[acorn](../../acorn)
[mewsfeed](../../mewsfeed)
[dino-adventure](../../dino-adventure)

(Paths assume `ai-tools` and above repos assumes are siblings, which they currently are.)
