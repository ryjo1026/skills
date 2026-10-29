# skills

Public, agent-agnostic [Agent Skills](https://agentskills.io) (`<name>/SKILL.md`). Works with any agent that reads the format (Claude Code, Codex, Cursor, Gemini CLI, OpenCode, Amp, ...).

This repo is one skill **source** among several my [dotfiles](https://github.com/ryjo1026/dotfiles#agent-skills) aggregate into `~/.agents/skills/` — see there for the source-merge mechanics and per-machine gating.

## Skills

| Skill | What it does |
| --- | --- |
| [`worktree`](worktree/SKILL.md) | Create an externally-openable git worktree at a predictable sibling path — usable by any editor/multiplexer, unlike agent-native worktree features |

## Use without the router

Any standard consumption path works:

```sh
npx skills add ryjo1026/skills --skill worktree   # per-skill install, 70+ agents
# or just clone and symlink into ~/.agents/skills/
```
