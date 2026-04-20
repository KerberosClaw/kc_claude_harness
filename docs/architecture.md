# Architecture (deep dive) — TODO

Placeholder for the full architecture write-up. Will cover:

- How the four layers interact: **skill** → **ref** → **memory** → **hook**
- Data flow: `/spec` spawns artifacts → agent reads refs on trigger → memory indexes state → hooks enforce invariants
- Lifecycle of a spec from fuzzy idea → shipped + archived
- Extension guide: adding a new skill, adding a new hook, per-repo opt-outs

---

## Digital PM Workflow（規劃中）

長期願景：用 AI agent 擔「數位 PM」。一句話需求進來，整條管線自動跑到 work items 建好 + 派好人 + 每日 standup 推送。

```
長官一句話需求
    ↓
[spec skill]       Discovery Stage（forcing questions 釐清痛點）
    ↓
                   → 產 docs/DESIGN.md
    ↓
[spec skill]       Spec Stage → 每個 feature 一套 spec.md / plan.md / tasks.md
    ↓
[pm-sync]          push: 六要素 + tasks.md → Azure DevOps work items（含 hierarchy）
    ↓
[pm-sync]          sprint-plan: 切 sprint + 依賴排序 + capacity-aware 派人
    ↓
[skill-cron]       每日早 9am 跑 /pm-sync daily → Telegram standup
    ↓
[pm-sync]          工程師 state 改動 → 反向更新本地 tasks.md
```

### 現狀 vs gap（2026-04-20 盤點）

| Layer | 已有 | 缺 |
|---|---|---|
| `/spec` Discovery + Spec + Implement + Check + Report | ✅ 完整 | 結尾沒「觸發 pm-sync push」的 hook |
| `/spec` → `/pm-sync` 串接 | ❌ 完全無 | user 要手動執行兩步 |
| `/pm-sync` read | ✅ `sprint <id>` 可用 | `show / daily / filters / sprints list` |
| `/pm-sync` write | ❌ 完全無 | `push / assign / state-update / sprint-plan` 全缺 |
| Sprint 切分演算 | ❌ 零 | effort 估算、capacity、依賴排序 |
| Daily diff | ❌ 零 | 跨日 snapshot cache、diff engine、standup format |
| Engineer roster | ❌ 零 | team profile + capacity 從哪取 |
| Harness orchestration | ❌ 零 | 把 spec / pm-sync / cron 串成 pipeline 的入口 |

### harness 層要做的事

跟 pm-sync 本身的 feature work 分開——harness 負責「串接」，不負責實作 CLI 指令。

- [ ] **`docs/DIGITAL_PM_WORKFLOW.md`** — 完整 workflow 文件，含時序圖、錯誤路徑、人工介入點
- [ ] **`scripts/digital-pm.sh`**（或等效 entry script）— 一鍵 orchestrator：
  - Input: 一句話需求
  - Step 1: invoke `/spec`（user interactive Discovery）
  - Step 2: 等 spec VERIFIED → invoke `/pm-sync push`
  - Step 3: 設 `skill-cron` daily job → Telegram standup
- [ ] **skill-cron 排程模板** — 早 9am 跑 `/pm-sync daily`，結果推 Telegram
- [ ] **memory 紀錄** — 當前活躍 spec / sprint / engineer assignment（以便中斷後續跑時有 context）
- [ ] **install.sh 加 pm-sync 安裝步驟** — 現在 harness install 沒涵蓋 pm-sync

### 依賴的其他 repo 工作

大部分重活在 [`kc_pm_sync`](https://github.com/KerberosClaw/kc_pm_sync) 的 Phase A-D roadmap（見 [`docs/USAGE.md` §6](https://github.com/KerberosClaw/kc_pm_sync/blob/main/docs/USAGE.md)）。harness 只負責把它們串起來。

`kc_ai_skills/spec` 端要做的：
- [ ] `/spec report` 結尾加可選 hook — 印提示「下一步：`/pm-sync push NN-feature-name`」，或互動式問 user 要不要 trigger pm-sync

### 時程粗估

| Phase | 範圍 | 工作量 | 結果 |
|---|---|---|---|
| pm-sync A | read 完整 | ~1 週 | daily standup 自動化、show/filters 可用 |
| pm-sync B | write（push + assign） | ~2-3 週 | spec → work item 端到端可跑（手動 assign）|
| harness orchestration | 入口 script + cron | ~1 週 | 一句話端到端 |
| pm-sync C | sprint-plan 半智慧 | ~1-2 週 | 自動切 sprint |
| pm-sync D | team roster + capacity | ~1 週 | 自動 assign |

**MVP「數位 PM」（A + B + harness）= 3-4 週週末量**。C/D 等 MVP 真的用起來再看需不需要。

### 歡迎 PR

這是個人主 side project 級別的規劃；如果你有興趣接某塊 TODO 直接在對應 repo 開 issue / PR。Phase A / Phase B / harness orchestration 都有明確定義，不需要先對齊願景。
