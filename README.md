# Claude Harness — The Meta-Repo Behind My Claude Code Setup

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Docs](https://img.shields.io/badge/Docs-only-blue.svg)](docs/)

[正體中文](README_zh.md)

> Personal duct tape holding together the skills, hooks, memory, and PM glue that turn Claude Code into something that actually ships code.

Not a tool. A manifesto plus some dotfiles. If you're hoping for a `brew install` situation, this isn't it.

## What actually lives here

This repo is the **connective tissue** — docs, credits, and (eventually) an install script that ties together the pieces living in other repos. Nothing here executes. Think of it as the README for a system that spans four repos.

| Piece | Where | What it does |
|---|---|---|
| [kc_ai_skills](https://github.com/KerberosClaw/kc_ai_skills) | separate repo (public) | 12+ skills (`/spec`, `/memory-lint`, `/repo-scan`, `/skill-cron`, ...) plus 4 safety hooks. The part that actually runs. |
| kc_claude_memory | separate repo (private) | Where Claude keeps notes about me. Also holds `ref/` files — 12 agent system prompts borrowed from NYCU, loaded on demand. |
| [NYCU-Chung/my-claude-devteam](https://github.com/NYCU-Chung/my-claude-devteam) | upstream | Where those agent prompts came from. MIT, credited in CREDITS.md. |
| [kc_pm_sync](https://github.com/KerberosClaw/kc_pm_sync) | separate repo (public, skeleton) | Bridge between `specs/<name>/tasks.md` and Azure DevOps work items. MVP target: `/pm-sync sprint`. Currently: directory structure and good intentions. |

## Why "harness"?

Stolen from OpenAI's [harness engineering](https://openai.com/index/harness-engineering/) article (Feb 2026). The argument: engineers shouldn't be writing code anymore — they should be building the environment the agent writes code in. Tools, feedback loops, memory layout, invariants. The agent does the typing.

That article hit embarrassingly close to home because it described what I'd been doing for weeks without a name for it. The parts that stuck:

1. **Give agents a map, not a 1000-page manual.** A bloated `CLAUDE.md` crowds out the task. A 100-line `MEMORY.md` as an index pointing at `ref/` files you load on demand — that actually works.
2. **Codebase is the only thing the agent can read.** Slack threads don't exist to it. Google Doc decisions don't exist. If it's not in the repo, it didn't happen. So `/spec` writes everything — requirements, plans, tasks, reports — to `specs/`.
3. **Encode rules as hooks, not requests.** "Please don't commit secrets" is wishful thinking. A hook that greps the staged diff for `sk-*` and `exit 2`s is a load-bearing wall.
4. **Let the agent loop where it's safe.** `skill-cron` runs banini every day. `/spec`'s Implement Stage runs until tasks are done. Not full autopilot. Not a puppet either.

## What's in this repo

```
kc_claude_harness/
├── README.md         — you are here
├── CREDITS.md        — who to credit / who to blame
├── LICENSE           — MIT
└── docs/
    └── architecture.md   — the deep-dive (TODO, still skeletal)
```

Zero executables. All the working code lives in the other four repos.

## Install (TODO)

```bash
# some day
git clone https://github.com/KerberosClaw/kc_claude_harness ~/dev/kc_claude_harness
cd ~/dev/kc_claude_harness && ./install.sh
```

When `install.sh` exists, it'll:
- Clone `kc_ai_skills` into `~/dev/`
- Symlink skills + hooks into `~/.claude/`
- Seed `~/.claude/branch-protection-skip.txt` (so the hook doesn't block you on your own solo repos)
- Bootstrap `~/dev/kc_claude_memory/` with a minimal MEMORY.md

That's the plan. It'll happen after `pm-sync` actually works.

## Status

**2026-04-20** — Public, but still mostly vision-in-markdown. Making it public early so people know I'm alive, not because it's ready for anyone to use.

What's working vs what isn't:
- ✅ The pieces in [kc_ai_skills](https://github.com/KerberosClaw/kc_ai_skills) (skills + hooks) are real and I use them daily
- ✅ The `ref/`-based progressive disclosure pattern is in daily use
- 🚧 `kc_pm_sync` has a skeleton and a roadmap; `/pm-sync sprint` MVP is the next thing I ship
- ❌ `install.sh` doesn't exist. If you want to replicate this, you're copy-pasting by hand for now.

In other words: read the manifesto, steal whatever ideas help, but don't expect a turnkey setup yet.

> **Update cadence:** Togashi-style. Please enjoy responsibly.

## Security Notice

- **This repo is documentation + metadata only.** No executable code, no secrets, no credentials. Clone it, fork it, read it — nothing sensitive transfers.
- **The sibling repos** ([kc_ai_skills](https://github.com/KerberosClaw/kc_ai_skills), `kc_pm_sync`, `kc_claude_memory`) each have their own Security Notice covering PAT storage, hook behaviour, sanitization rules for test fixtures, etc. Read those before wiring anything into a real workflow.
- **Future `install.sh`** (when it exists) will never hard-code credentials — all tokens go through env vars or adapter-level `from_env()` classmethods. Same pattern as pm-sync.
- **Found a real security issue?** Open a GitHub issue on the relevant sub-repo (pm-sync for PAT flow, ai_skills for hook bugs, etc.). Nothing in this meta-repo itself should ever warrant a CVE.

## License

MIT. See [CREDITS.md](CREDITS.md) for everyone else who had a hand in this.
