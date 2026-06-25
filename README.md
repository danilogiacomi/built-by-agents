# built-by-agents

A Claude Code plugin that scaffolds a **self-updating "Built by agents" section** into any
repository's `README.md` — showing that repo's own coding-agent development footprint
(tokens, active time, turns, tool calls, models) read from your local Claude Code and
Codex logs.

It installs three things into the target repo:

- `scripts/usage-self.ts` — a Bun generator that reads this machine's local agent logs and
  rewrites a marked block in `README.md`.
- `.githooks/pre-commit` — regenerates the section and re-stages `README.md` on each
  commit (bun-guarded, never blocks a commit, safe in CI / fresh clones).
- README markers (`<!-- usage:self:start -->` … `<!-- usage:self:end -->`) and a
  `usage:self` npm script.

> Requires [Bun](https://bun.sh) on the developer's machine to run the generator. The hook
> no-ops silently where bun or local logs are absent.

## Install

Add this repo as a plugin marketplace, then install the plugin:

```sh
# in Claude Code
/plugin marketplace add danilogiacomi/built-by-agents     # or a local path / git URL
/plugin install built-by-agents
```

For local development, point the marketplace at the checkout directly:

```sh
/plugin marketplace add /Users/danilo/Progetti/built-by-agents
/plugin install built-by-agents
```

## Use

In any git repo you want the section in, invoke the skill:

```
/built-by-agents:built-by-agents
```

The skill copies the generator and hook, wires the `usage:self` npm script (or patches the
hook if there's no `package.json`), runs the generator once, and prints the one command to
enable the hook:

```sh
git config core.hooksPath .githooks
```

It deliberately does **not** run that command for you — so it can't clobber an existing
husky / lefthook / `core.hooksPath` setup.

## Layout

```
.claude-plugin/marketplace.json          # marketplace catalog
plugins/built-by-agents/
  .claude-plugin/plugin.json             # plugin manifest
  skills/built-by-agents/
    SKILL.md                             # the scaffold procedure
    assets/
      usage-self.ts                      # Bun generator (copied into target repos)
      pre-commit                         # git hook (copied into target repos)
```

## Credit

Extracted from the [`ccusage`](https://github.com/danilogiacomi/ccusage) project, where the
"Built by agents" section first appeared.

<!-- usage:self:start -->

## 🤖 Built by agents

This plugin dogfoods itself. The numbers below are the footprint of the single coding-agent
session that built this repo, rendered by its own `scripts/usage-self.ts`.

| Metric | Value |
|---|---|
| **Total tokens** | **6.7M** |
| Token breakdown | 100.6K output · 32.6K input · 169.4K cache-write · 6.4M cache-read |
| Agent time | ~44m active (2h 25m wall-clock) |
| Turns | 102 assistant turns · 39 tool calls |
| Agents / models | Claude Code — claude-opus-4-8 |
| As of | 2026-06-25 |

> 💡 Most of those tokens are *cache reads* — re-reading the growing conversation each
> turn — which is why the total dwarfs the tokens actually written.

<!-- usage:self:end -->

## License

MIT
