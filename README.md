# kc_claude_harness

> Personal harness for driving Claude Code — skills, hooks, memory patterns, and PM integration, wired together under one roof.

**English · 繁體中文 (TODO)**

---

## What is this?

An opinionated setup for using Claude Code as a coding agent, organized around the **harness engineering** pattern: engineers don't write code, they design the environment the agent operates in — tools, feedback loops, memory structure, and constraints.

Rather than a single tool, this repo is a **meta-project** documenting and packaging the pieces that work together:

| Piece | Where | Purpose |
|---|---|---|
| [kc_ai_skills](https://github.com/KerberosClaw/kc_ai_skills) | separate repo | 12+ skills: `/spec`, `/memory-lint`, `/repo-scan`, `/skill-cron`, etc. Plus 4 safety hooks. |
| kc_claude_memory | private | Auto-memory directory with triggers + 12 agent `ref/` files |
| [NYCU-Chung/my-claude-devteam](https://github.com/NYCU-Chung/my-claude-devteam) | upstream | Source of the 12 agent reference files (MIT, attributed) |
| kc_pm_sync | planned | Future: bidirectional sync between spec skill output and PM tools (Linear/Jira) |

---

## Why "harness"?

Term borrowed from OpenAI's [Harness Engineering](https://openai.com/index/harness-engineering/) article (Feb 2026). Key ideas applied here:

1. **Give agents a map, not a manual.** Short `MEMORY.md` as an index; detail lives in `ref/` files accessed on demand (progressive disclosure).
2. **Codebase as single source of truth.** Spec-driven development via `/spec` skill — every feature produces `spec.md` / `plan.md` / `tasks.md` / `report.md` in the repo, not in someone's head.
3. **Enforce invariants, not implementations.** Hooks catch common failures (committed secrets, force-push to main, reading huge files) deterministically, with fix instructions baked into the error message.
4. **Progressive trust.** Agents can run autonomously in well-scoped loops (skill-cron for scheduled tasks, spec's Implement Stage for bounded work).

---

## Structure

```
kc_claude_harness/
├── README.md        — this file (vision + pointers)
├── CREDITS.md       — lineage: NYCU devteam, OpenAI, Karpathy, tanweai/pua
├── LICENSE          — MIT
└── docs/
    └── architecture.md  — deep dive (TODO)
```

This repo is deliberately minimal. The executable pieces live in their own repos (see table above). Here is the connective tissue: vision, design notes, install/update scripts (future), and credit.

---

## Installation (future)

Not yet. Currently this is a living document. When stable:

```bash
# planned
git clone https://github.com/KerberosClaw/kc_claude_harness ~/dev/kc_claude_harness
cd ~/dev/kc_claude_harness && ./install.sh
```

`install.sh` will (when written):
- Clone kc_ai_skills into `~/dev/`
- Symlink skills + hooks into `~/.claude/`
- Set up `~/.claude/branch-protection-skip.txt` template
- Bootstrap `~/dev/kc_claude_memory/` with the MEMORY.md template

---

## Status

**2026-04-20** — Private skeleton. Using as a scratchpad while pieces stabilize. Will go public when:
- `kc_pm_sync` lands a working v1
- `install.sh` is tested on a fresh machine
- Bilingual README (EN + 中文)

---

## License

MIT. See [CREDITS.md](CREDITS.md) for third-party attributions.
