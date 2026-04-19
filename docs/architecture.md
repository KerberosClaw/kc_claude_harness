# Architecture (deep dive) — TODO

Placeholder. Will document:

- How the four layers interact: **skill** → **ref** → **memory** → **hook**
- Data flow: `/spec` spawns artifacts → agent reads refs on trigger → memory indexes state → hooks enforce invariants
- Lifecycle of a spec from fuzzy idea → shipped + archived
- pm-sync integration points (when kc_pm_sync lands)
- Extension guide: adding a new skill, adding a new hook, per-repo opt-outs
