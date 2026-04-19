# Credits

This project stands on work by others. Specific borrowings:

## NYCU-Chung/my-claude-devteam

- **URL:** https://github.com/NYCU-Chung/my-claude-devteam
- **License:** MIT
- **Borrowed:**
  - Full text of 12 agent definitions, stored as reference files in `kc_claude_memory/ref/` (with `source_url` and `fetched_commit` frontmatter so drift can be checked).
  - 4 safety hooks (`commit-quality`, `branch-protection`, `large-file-warner`, `audit-log`), adapted into `kc_ai_skills/hooks/` with enhanced error messages that include fix instructions.
  - The P9 six-element Task Prompt schema (Goal / Scope / Inputs / Outputs / Acceptance / Boundaries), adopted into `/spec` skill's `spec.md` template.
  - The P7-COMPLETION structured delivery format (solution, changes, impact, self-review, remaining risk), adopted into `/spec` skill's `report.md` template.
- **Not borrowed:** P7/P9/P10 role labels, PUA pressure-mode triggers, frontend-designer / design-quality hooks (not applicable to this user's backend/ML workflow).

## OpenAI Harness Engineering

- **Article:** https://openai.com/index/harness-engineering/ (Feb 2026, Ryan Lopopolo)
- **Borrowed:**
  - `specs/{active,completed}/` split in `/spec` skill — mirrors the `docs/exec-plans/{active,completed}/` pattern.
  - Principle of injecting fix instructions into hook error messages (harness teaches agent how to recover, agent doesn't have to remember).
  - Progressive disclosure via short `MEMORY.md` index + detailed `ref/` files.
  - "Codebase as single source of truth" framing for the spec-driven workflow.

## Karpathy Four Pillars

- **Reference:** https://github.com/forrestchang/andrej-karpathy-skills
- **Borrowed:**
  - Think / Simple / Surgical / Goal-Driven principles as the core coding discipline.
  - Surgical Changes / Simplicity First / Orphan Cleanup rules in `/spec` skill's Implement Stage.

## tanweai/pua (indirectly via NYCU)

- **URL:** https://github.com/tanweai/pua (MIT)
- **Note:** NYCU's P7/P9/P10 methodology is itself adapted from tanweai/pua. Not directly used in this repo, but acknowledged for chain of origin.
