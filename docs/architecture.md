# Architecture (deep dive) — TODO

Placeholder for the full architecture write-up. Will cover:

- How the four layers interact: **skill** → **ref** → **memory** → **hook**
- Data flow: `/spec` spawns artifacts → agent reads refs on trigger → memory indexes state → hooks enforce invariants
- Lifecycle of an idea from one-line ask → shipped + archived
- Extension guide: adding a new skill, adding a new hook, per-repo opt-outs

---

## Digital PM Workflow（規劃中）

長期願景：用 AI agent 擔「數位 PM」。一句話需求進來，整條管線跑到 work items 建好 + 派好人 + 每日 standup 推送。

```
長官一句話需求
    ↓
[/spec skill]      Discovery → 產 docs/DESIGN.md / spec.md / plan.md
    ↓
[/prd-create]      （未實作）spec/design 收斂成 PRD markdown
    ↓
[/pm-sync] A       PRD → vertical slice plan.md（quiz user / wave / blocked_by）
    ↓
[/pm-sync] B       Push slices to ADO（parent / Predecessor / assignee + fingerprint idempotent）
    ↓
[/pm-sync daily]   （未實作）snapshot diff → standup-friendly format
    ↓
[skill-cron]       排 daily 9am 跑 standup → Telegram
```

### 現狀 vs gap（2026-04-29 盤點）

| Layer | 已有 | 缺 |
|---|---|---|
| `/spec` Discovery + Spec + Implement + Check + Report | ✅ kc_ai_skills 內完整 | 結尾沒「觸發 prd-create / pm-sync」hook |
| `/prd-create`（spec/design → PRD markdown） | ❌ 還沒寫 | 整個 skill 待寫；可參考 `vibe-grimoire/create-prd` 內化 |
| `/pm-sync` Workflow A（PRD → slice plan） | ✅ v1.0.1 stable | — |
| `/pm-sync` Workflow B（push to ADO） | ✅ v1.0.1 stable，含 Step B0 production-confirm gate | — |
| `/pm-sync daily`（snapshot diff for standup） | ❌ | stateful；caller 維護 `~/.pm-sync.snapshots/<date>.json`，Claude 對話內讀寫 |
| Multi-platform（GitHub / Jira / Redmine / 禪道） | ❌ | 各寫獨立 SKILL section（不抽 abstraction） |
| Cross-skill orchestration（一鍵跑全 pipeline） | ❌ | harness 自己的 install + 串接 prompt |
| `skill-cron` 排定 `/pm-sync daily` Telegram | ⚠️ skill-cron 在 kc_ai_skills，但 daily 還沒做 | 等 daily 出來才能接 |

### harness 層要做的事

跟 sub-repo 各自的 feature work 分開——harness 負責「串接 + 規範」，不負責實作 skill 本身。

- [ ] **`docs/DIGITAL_PM_WORKFLOW.md`** — 完整 workflow 文件，含時序圖、錯誤路徑、人工介入點
- [ ] **Cross-skill orchestrator** — 一鍵串：`/spec` → `/prd-create` → `/pm-sync` A → B → `skill-cron`。可能是 install 進 `~/.claude/skills/` 的 meta-skill，也可能就是一份引導 prompt
- [ ] **install.sh** — clone sub-repos、symlink skills/hooks、bootstrap memory、初始化 branch-protection-skip
- [ ] **memory 紀錄** — 當前活躍 spec / push session / sprint context（中斷後續跑時有 context）

### 依賴的其他 repo 工作

| Repo | 待做 |
|---|---|
| `kc_pm_sync` | `/pm-sync daily` workflow（stateful snapshot diff）+ multi-platform expansion（GitHub / Jira / Redmine / 禪道）|
| `kc_ai_skills` | `/prd-create` skill（spec/design → PRD markdown）+ `/spec report` 結尾選擇性 hook 觸發下游 |

### 時程

無精細估算 — 所有 sub-piece 都靠 weekend project 推進，估算容易過時。看 sub-repo 自己的 issue 動態。

### 歡迎 PR

這是個人主 side project 級別的規劃。每塊 TODO 在對應 repo 開 issue / PR，不需要先對齊願景。
