# kc_claude_harness — Claude 工作守則

## 這個 repo 是什麼

Docs-only meta-repo。串起四個 sibling repo 的結締組織：
- `~/dev/kc_ai_skills` — 真正會跑的 skills + hooks（public）
- `~/dev/kc_claude_memory` — Claude 對 user 的記憶 + NYCU agent ref（private）
- `~/dev/kc_pm_kit` — prd-create + prd-breakdown 兩 skill（public）
- 上游：NYCU-Chung/my-claude-devteam（agent prompts 來源）

**沒有可執行程式碼、沒有 secrets。** 任何 install 邏輯出現在 README 之前先確認。

## 結構

```
README.md / README_zh.md / CREDITS.md / LICENSE
docs/architecture.md  — skeletal
```

## 動工前注意

- 預設輸出 **正體中文**。
- **不要反射性加 TL;DR / Overview / 摘要** 框架（見 memory `feedback_collaboration.md`）。
- 改 README 同時記得改 `README_zh.md`，兩邊內容對齊（status / 進度數字 / repo 連結）。
- 引用 sibling repo 狀態前先 `ls ~/dev/kc_<name>/` 或 git log 確認，**不要憑記憶寫**（見 memory `feedback_doc_vs_reality_check.md`）。
- Status 段裡 ✅ / 🚧 / ❌ 是當前現況，不是規劃；改之前對帳。

## 還沒做的東西

兩個已決定 scope、還沒動的 TODO。user 說要動再動，不要主動開工。

- **`install.sh`** — public 了之後 fork 下來要能一鍵裝：
  - clone `kc_ai_skills` 到 `~/dev/`
  - symlink skills + hooks 進 `~/.claude/`
  - seed `~/.claude/branch-protection-skip.txt`
  - bootstrap `~/dev/kc_claude_memory/` 一份 minimal MEMORY.md
  - PAT / token 全走 env var，**不准 hard-code、不准吃 argv**
- **`docs/architecture.md`** — 目前 stub，要填三塊：
  - 四層互動：skill → ref → memory → hook
  - spec 生命週期（`/spec` 從需求釐清到結案那條線）
  - kc_pm_kit 對接點

## Commit 慣例

- 跟 user 其他 repo 一致：`docs:` / `chore:` / `feat:` 前綴。
- Update cadence 是 Togashi 級，不要催進度，也不要硬塞「下一步」。
