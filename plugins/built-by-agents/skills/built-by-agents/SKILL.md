---
name: built-by-agents
description: >
  Scaffold a self-updating "Built by agents" section into a repo's README. Installs a Bun
  generator that reads local Claude Code / Codex agent logs, a git pre-commit hook that
  keeps the section fresh, and the README markers. Use when the user wants to show their
  repo's agent-development footprint (tokens, time, turns) in the README, or mentions
  "Built by agents".
---

# Built by agents

Scaffold the "Built by agents" machinery into the **current** repository: a Bun script
that reads this machine's local agent logs and rewrites a marked section of `README.md`,
plus a pre-commit hook that regenerates it on each commit.

This skill only **scaffolds files and reports** the one enable step. It never runs
`git config` and never makes network calls.

## Bundled assets

Two files ship with this skill, in its `assets/` directory:

- `usage-self.ts` — the Bun generator (Bun-only; uses `Bun.Glob`/`Bun.file`/`Bun.write`).
- `pre-commit` — the git hook (POSIX sh; bun-guarded, never blocks a commit).

Resolve the assets directory before copying. Try, in order:

1. `$CLAUDE_PLUGIN_ROOT/skills/built-by-agents/assets` if `$CLAUDE_PLUGIN_ROOT` is set.
2. Otherwise find it under the plugin cache:
   `find ~/.claude/plugins -type d -path '*built-by-agents/skills/built-by-agents/assets' | head -1`

## Procedure

Do these in order, in the target repo's root (the current working directory).

1. **Preconditions.**
   - Confirm the cwd is inside a git repo (`git rev-parse --is-inside-work-tree`). If not,
     stop and tell the user to run this from a git repo.
   - Confirm a `README.md` exists. If it doesn't, offer to create a minimal stub
     (`# <repo name>\n`) and proceed only if the user agrees.

2. **Copy the generator.** Copy the resolved `assets/usage-self.ts` → `scripts/usage-self.ts`
   (create `scripts/` if needed). If `scripts/usage-self.ts` already exists, show the user a
   diff and ask before overwriting — do not clobber silently.

3. **Copy the hook.** Copy the resolved `assets/pre-commit` → `.githooks/pre-commit`
   (create `.githooks/` if needed) and make it executable (`chmod +x .githooks/pre-commit`).
   If it already exists, ask before overwriting.

4. **Wire the npm script.**
   - If `package.json` exists: add `"usage:self": "bun run scripts/usage-self.ts"` to its
     `scripts` object (skip if a `usage:self` script is already present). Preserve formatting.
   - If there is **no** `package.json`: the `usage:self` alias won't exist, so edit
     `.githooks/pre-commit`, changing the line
     `bun run --silent usage:self` → `bun run --silent scripts/usage-self.ts`.

5. **First run.** Run `bun run scripts/usage-self.ts` once to insert the markers and the
   initial section into `README.md`.
   - If `bun` is not installed, report that and skip this step (the markers will be created
     on the first run once bun is available).
   - If the script prints "no local agent logs for this repo", report that honestly — the
     section will populate once this repo has Claude Code / Codex activity. Do not claim a
     section was generated when it wasn't.

6. **Print the enable step — do NOT run it.** Tell the user to enable the hook with:

   ```sh
   git config core.hooksPath .githooks
   ```

   Warn them to first check for an existing hook manager: run
   `git config --get core.hooksPath` and check for `.husky/` or `lefthook.yml`. If any
   exist, setting `core.hooksPath` would override them — in that case advise calling the
   generator from their existing pre-commit instead of switching `hooksPath`.

7. **Summary.** List exactly what was created/modified:
   - `scripts/usage-self.ts`
   - `.githooks/pre-commit`
   - `package.json` (added `usage:self`) — or the hook patch, if no `package.json`
   - `README.md` (markers + section) — or note it's pending bun/logs
   Then restate the enable command from step 6.

## Notes

- **Idempotent.** Re-running detects existing files and the README markers and skips or
  asks rather than duplicating.
- **What the generator reads.** Claude Code transcripts at
  `~/.claude/projects/<encoded-cwd>/*.jsonl` and Codex rollouts under `$CODEX_HOME`/`~/.codex`
  whose `session_meta.cwd` matches this repo. It is read-only against those logs.
- **Safe in CI / fresh clones.** With no local logs the generator leaves `README.md`
  untouched and the hook exits 0, so it never blocks a commit.
