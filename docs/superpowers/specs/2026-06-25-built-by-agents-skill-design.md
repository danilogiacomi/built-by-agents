# Built by agents — Claude Code plugin/skill

**Date:** 2026-06-25
**Status:** Approved design

## Purpose

Package the "Built by agents" README feature (originally built in the `ccusage`
project) as a reusable Claude Code **plugin** containing a single **skill**, so it can
be dropped into any other repository. When invoked inside a target repo, the skill
scaffolds everything needed to generate and maintain a "Built by agents" section in that
repo's `README.md`: a Bun generator script, a git pre-commit hook, the README markers,
and a `package.json` script — then tells the user the one command to enable it.

The skill writes files; it never silently mutates git config or makes network calls.

## Background — what the feature is (in ccusage)

Three pieces working together:

1. **`scripts/usage-self.ts`** — a Bun script that reads local agent logs
   (`~/.claude/projects/<encoded-cwd>/*.jsonl` for Claude Code + Codex rollouts whose
   `session_meta.cwd` matches the repo), aggregates tokens / active-time / turns /
   tool-calls / models / agents, and rewrites a marked block in `README.md` between
   `<!-- usage:self:start -->` and `<!-- usage:self:end -->`. It no-ops safely when no
   logs are found (fresh clone / CI).
2. **`.githooks/pre-commit`** — regenerates the section and re-stages `README.md` on each
   commit. Guarded: skips silently if `bun` is missing, never blocks a commit.
3. **README markers + setup** — the start/end markers, a `usage:self` entry in the
   `package.json` scripts table, and the `git config core.hooksPath .githooks`
   enablement line.

## Design decisions

| Decision | Choice | Rationale |
|---|---|---|
| Generator runtime | **Bun only** | Copies the proven ccusage script nearly verbatim; lowest complexity. |
| Distribution | **Full plugin + marketplace** | Installable via `/plugin marketplace add` → `/plugin install`; shareable, versioned. |
| Section copy | **Generic default** | Project-agnostic prose so it drops into any repo without editing. |
| Hook enable | **Scaffold only, manual enable** | Never touch `git config` automatically — avoids clobbering husky/lefthook/existing `core.hooksPath`. The skill prints the command and warns about conflicts. |

## Repository layout (this new repo)

```
built-by-agents/                         ← git repo root; serves as the marketplace
├── .claude-plugin/
│   └── marketplace.json                 ← lists the one plugin
├── plugins/
│   └── built-by-agents/
│       ├── .claude-plugin/
│       │   └── plugin.json              ← plugin manifest
│       └── skills/
│           └── built-by-agents/
│               ├── SKILL.md             ← skill instructions (the procedure below)
│               └── assets/
│                   ├── usage-self.ts    ← Bun generator (genericized from ccusage)
│                   └── pre-commit       ← git pre-commit hook (verbatim from ccusage)
├── README.md                            ← install + usage docs
└── LICENSE
```

### `marketplace.json` (shape)

```json
{
  "name": "danilo-skills",
  "owner": { "name": "Danilo Giacomi", "email": "giacomi@netseven.it" },
  "plugins": [
    {
      "name": "built-by-agents",
      "source": "./plugins/built-by-agents",
      "description": "Scaffold a self-updating 'Built by agents' README section into any repo."
    }
  ]
}
```

### `plugin.json` (shape)

```json
{
  "name": "built-by-agents",
  "description": "Scaffold a self-updating 'Built by agents' README section into any repo.",
  "version": "0.1.0",
  "author": { "name": "Danilo Giacomi", "email": "giacomi@netseven.it" },
  "license": "MIT"
}
```

### `SKILL.md` frontmatter (shape)

```yaml
---
name: built-by-agents
description: >
  Scaffold a self-updating "Built by agents" section into a repo's README — a Bun
  generator that reads local Claude Code / Codex agent logs, a pre-commit hook, and the
  README markers. Use when the user wants to show their repo's agent-development
  footprint in the README, or mentions "Built by agents".
---
```

## Bundled assets

### `assets/usage-self.ts`
Copied from ccusage's `scripts/usage-self.ts`. **One change:** in `renderSection()`,
replace the dashboard-specific intro paragraph and the dashboard-specific tip callout
with generic, project-agnostic copy:

> This project is built largely by coding agents. The numbers below are this repo's own
> development footprint, read from the local agent logs.

The tip about cache-reads dominating the total is generic enough to keep (reworded to
drop the "dashboard" references). Everything else is unchanged:
- Claude Code transcript aggregation (`aggregateClaude`)
- Codex rollout aggregation matching repo cwd (`aggregateCodex`)
- `mergeStats`, `timeSpans`, token/duration formatting
- marker constants (`START`/`END`), `replaceSection`, `claudeProjectDir`
- `main()`: resolve log roots from `homedir()`, read globs, merge, rewrite README,
  safe no-op when `!hasData(stats)`.

### `assets/pre-commit`
Copied **verbatim** — it is already generic and safe (bun-guarded, never blocks a commit,
re-stages `README.md`). The one nuance: it calls `bun run --silent usage:self`, which
assumes a `usage:self` npm script exists. The skill ensures that script exists, or (for
repos without `package.json`) the skill patches the hook to call the script path directly.

## Skill procedure (what `SKILL.md` instructs the agent to do)

Invoked inside a **target** repo, the agent:

1. **Preconditions.** Confirm the cwd is a git repo and has a `README.md`. If no README,
   offer to create a minimal stub. Abort cleanly if not a git repo.
2. **Copy generator.** Copy `${CLAUDE_PLUGIN_ROOT}/.../assets/usage-self.ts` →
   `scripts/usage-self.ts`. If it already exists, diff/skip rather than clobber.
3. **Copy hook.** Copy `assets/pre-commit` → `.githooks/pre-commit`; `chmod +x`.
4. **Wire the script.** If `package.json` exists, add
   `"usage:self": "bun run scripts/usage-self.ts"` to `scripts` (skip if already present).
   If there is no `package.json`, patch `.githooks/pre-commit` to invoke
   `bun run --silent scripts/usage-self.ts` directly instead of the `usage:self` alias.
5. **First run.** Run `bun run scripts/usage-self.ts` once to insert the markers and the
   initial section into `README.md`. If `bun` is missing or there are no local logs for
   this repo, report that and continue (the markers will populate on a later run).
6. **Print manual enable step — do NOT run it.** Tell the user to run
   `git config core.hooksPath .githooks`, with a warning to first check for an existing
   husky / lefthook / `core.hooksPath` setup so it isn't overwritten.
7. **Summary.** List the files created/modified and restate the enable step.

## Key principles

- **Idempotent.** Re-invoking detects existing files/markers and skips rather than
  clobbering.
- **Safe.** No `git config` writes, no network, edits confined to the target repo.
- **Honest reporting.** When bun is absent or no logs exist, say so — don't claim the
  section was generated.

## Out of scope (YAGNI)

- Non-Bun runtimes (Node/Python generators).
- Per-project tailored prose generation.
- Automatic git-config / husky integration.
- Multi-skill plugin; this ships exactly one skill.

## Verification

- Lint/format the bundled `usage-self.ts` matches the source project's style where it
  lands (the script is plain TS; no repo-specific tooling required to ship it as an asset).
- Manual end-to-end: add the local marketplace, install the plugin, invoke the skill in a
  throwaway git repo with and without `package.json`, with and without local logs, and
  confirm the four files/edits appear and the enable line is printed (not executed).
