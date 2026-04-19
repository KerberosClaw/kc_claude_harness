# kc_claude_harness

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

[English](README.md)

> 把 skills、hooks、memory、PM 整合層用膠帶黏在一起的個人設定，讓 Claude Code 真的能把 code 寫出來。

不是工具。是一份宣言加一些 dotfiles。如果你想找 `brew install` 能裝的東西，這個不是。

## 這裡到底放什麼

這 repo 是**結締組織** — 文件、credits、未來的 install script，把散在其他 repo 的東西串起來。這裡什麼都不會執行。把它想成一個跨 4 個 repo 的系統的 README。

| Piece | 在哪 | 幹嘛用的 |
|---|---|---|
| [kc_ai_skills](https://github.com/KerberosClaw/kc_ai_skills) | 獨立 repo（public） | 12+ 個 skill（`/spec`、`/memory-lint`、`/repo-scan`、`/skill-cron` 等）加 4 個安全 hook。真的能執行的部分都在這。 |
| kc_claude_memory | 獨立 repo（private） | Claude 寫給自己看的筆記都在這。還有 `ref/` 檔 — 12 份從 NYCU 抄來的 agent system prompt，用到才載入。 |
| [NYCU-Chung/my-claude-devteam](https://github.com/NYCU-Chung/my-claude-devteam) | upstream | 上面那些 agent prompt 的來源。MIT 授權，CREDITS.md 有記功。 |
| kc_pm_sync | 獨立 repo（private, skeleton） | `specs/<name>/tasks.md` 跟 Azure DevOps work item 之間的橋。MVP 目標：`/pm-sync sprint`。現在只有目錄結構跟良好的意圖。 |

## 為什麼叫 harness？

從 OpenAI 的 [harness engineering](https://openai.com/index/harness-engineering/) 文章偷來的（2026-02）。那篇的論點：工程師不該再寫 code 了 — 該打造的是讓 agent 寫 code 的環境。工具、回饋循環、記憶佈局、不變式。打字的事交給 agent。

這篇打得有點痛，因為它精準描述了我做了好幾個禮拜但沒有名字的事。我吸收的幾點：

1. **給 agent 一張地圖，不要給 1000 頁手冊。** 塞爆的 `CLAUDE.md` 會把真正要做的事擠掉。100 行的 `MEMORY.md` 當索引、指向需要才讀的 `ref/` 檔 — 這個真的會動。
2. **codebase 是 agent 唯一讀得到的東西。** Slack 討論它看不到。Google Doc 裡的決策它看不到。沒進 repo 就等於沒發生。所以 `/spec` 什麼都寫進 `specs/` — 需求、計畫、tasks、結案報告。
3. **規則寫成 hook，不要寫成請求。** 「拜託不要 commit 機敏資訊」是許願。hook 直接 grep 暫存區的 `sk-*` 發現就 exit 2 才是承重牆。
4. **在安全範圍內讓 agent 跑迴圈。** `skill-cron` 每天跑 banini。`/spec` 的 Implement Stage 跑到 tasks 做完為止。不是全自動，也不是牽線木偶。

## repo 裡面有什麼

```
kc_claude_harness/
├── README.md         — 你現在在這
├── CREDITS.md        — 該謝誰 / 該怪誰
├── LICENSE           — MIT
└── docs/
    └── architecture.md   — 深入解說（TODO，骨架都還沒長肉）
```

零執行檔。真的在動的 code 都在另外那 4 個 repo 裡。

## 安裝（TODO）

```bash
# 某一天
git clone https://github.com/KerberosClaw/kc_claude_harness ~/dev/kc_claude_harness
cd ~/dev/kc_claude_harness && ./install.sh
```

`install.sh` 寫好後會：
- Clone `kc_ai_skills` 到 `~/dev/`
- 把 skills + hooks symlink 進 `~/.claude/`
- 初始化 `~/.claude/branch-protection-skip.txt`（不然 hook 會擋你自己個人 repo 的 direct-to-main）
- Bootstrap `~/dev/kc_claude_memory/` 跟一份最小的 MEMORY.md

計畫是這樣。等 pm-sync 真的能動再說。

## 狀態

**2026-04-20** — Public，但大部分還是 markdown 裡的願景。提早放出來是為了證明我還活著，不是因為這東西可以給別人用了。

什麼能動什麼還不行：
- ✅ [kc_ai_skills](https://github.com/KerberosClaw/kc_ai_skills) 裡的 skills + hooks 是真的，而且我每天在用
- ✅ `ref/` 當基礎的 progressive disclosure 模式也在日常運作中
- 🚧 `kc_pm_sync` 有骨架跟 roadmap，`/pm-sync sprint` MVP 是下一個要出的
- ❌ `install.sh` 不存在。現在想複製這套的話只能手動複製貼上

簡單說：看完宣言、順手偷幾個點子，但不要期待現成可用的配置包

> **更新頻率：** 富奸式佛系更新，請斟酌服用。

## License

MIT。其他誰出力見 [CREDITS.md](CREDITS.md)。
