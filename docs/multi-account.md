# 多開 Claude Code — 同一台機器並存多個訂閱帳號

先把「多開」拆成兩件事，因為卡住的通常是後面那件：

- **同一個帳號、多個終端機視窗同時跑** —— 本來就可以，不用任何設定。額度算在帳號頭上，多開只是消耗得比較快。
- **兩個以上各自獨立的訂閱帳號，在同一台機器上同時開著** —— 這個要設定，而且直覺的做法會失敗。這份文件講的是這個。

適用範圍：macOS，Claude Code 2.1.x 長期日常使用中驗證。憑證儲存那段用的是 macOS 鑰匙圈，換平台要換工具；但下面六個坑的成因是 Claude Code 自己的行為，換平台一樣會遇到。

---

## 為什麼設個 `CLAUDE_CONFIG_DIR` 不夠

直覺解法是給每個帳號一個 `CLAUDE_CONFIG_DIR`，各自一份設定目錄。方向是對的，但只做這件事會連撞四個坑，前三個都會讓人以為「這功能根本不支援」。

### 坑一：憑證在系統鑰匙圈裡是共用的，設定目錄不隔離它

同一台 macOS 上所有 Claude Code session 共讀同一個鑰匙圈項目 `Claude Code-credentials`。設定目錄分開了，這一份還是共用的，而且**它的優先順序高於環境變數帶進去的 token** —— 所以第二個帳號開起來，坐在裡面的還是第一個帳號。（官方 repo 上的回報：[anthropics/claude-code#20553](https://github.com/anthropics/claude-code/issues/20553)，目前仍 open）

解法是兩件事一起做：

```bash
security delete-generic-password -s "Claude Code-credentials"
```

然後**不要再跑裸的 `claude`，也不要在裡面 `/login`**。一登就把那個共用項目寫回來、隔離當場破功。破了就再刪一次。

憑證改成每個帳號各存一份長效 token 在鑰匙圈的獨立項目裡，啟動時用環境變數餵進去。token 用 `claude setup-token` 產，效期一年。

### 坑二：`claude` 如果被設成 alias，前綴指派餵不進去

假設 `claude` 是這種複合 alias：

```bash
alias claude='some-sync-command; command claude'
```

那麼 `SOME_VAR=xxx claude` 展開之後，指派只黏在 `some-sync-command` 上，真正的 claude 拿不到。解法是包成 subshell、用 `export`、並且用 `command` 直接呼叫執行檔繞過 alias。

### 坑三：全新設定目錄沒有 onboarding 旗標，每次都跳登入選單

新的 `.claude.json` 少了 `hasCompletedOnboarding`，開起來就是「Select login method」，環境變數裡的 token 直接被無視。補上：

```json
{ "hasCompletedOnboarding": true, "lastOnboardingVersion": "<跑 claude --version 拿到的版號>" }
```

**版號要對齊當下實際安裝的版本**。沿用從別處抄來的舊版號，會因為版本落差再次觸發 onboarding。

### 坑四：全新設定目錄的 `settings.json` 是裸的，不繼承既有帳號

新目錄的 `settings.json` 預設只有模型、佈景那幾項 —— 沒有 hooks，也沒有記憶目錄設定。後果是新帳號對既有的記憶「又盲又不同步」：不自動載入索引、改了也不觸發同步；連擋 push main 那類安全 hook 也一起不見。搭配 `--dangerously-skip-permissions` 用的話，等於整層防護不在。

解法是把既有帳號那份 `settings.json` **整份**抄過去。hook 腳本本身不用複製 —— 用絕對路徑引用同一份就好，改一次全部帳號同步。注意設定是啟動時載入，改完要重開才生效。

---

## 最後長這樣

`~/.zshrc` 裡一個帳號一個 function：

```bash
# 帳號 1 用預設設定目錄，刻意不設 CLAUDE_CONFIG_DIR，理由見坑六
cc1() {
  ( export CLAUDE_CODE_OAUTH_TOKEN=$(security find-generic-password -a "$USER" -s cc_oauth_acct1 -w)
    command claude "$@" )
}

cc2() {
  ( export CLAUDE_CONFIG_DIR="$HOME/.claude-acc2"
    export CLAUDE_CODE_OAUTH_TOKEN=$(security find-generic-password -a "$USER" -s cc_oauth_acct2 -w)
    command claude "$@" )
}
```

第三個之後照抄換編號。要開幾個帳號就開幾個終端機分頁，各打各的 `ccN`。

三個細節：subshell 的括號讓環境變數不外洩到目前這個 shell、`export` 是坑二的解法、`command` 是繞過 alias。

> 我自己那份還掛了 `--dangerously-skip-permissions`，這裡刻意拿掉。要不要加是各自的風險判斷 —— 真要加，更該先確定坑四的 hook 已經補回來。

---

## 坑五：信任是逐目錄的，headless `-p` 會直接卡死

新設定目錄在某個工作目錄第一次跑 `-p`，會吐這種東西然後**完全不執行**：

```
Ignoring N permissions.allow entries ... this workspace has not been trusted
```

根因是信任旗標 `projects["<目錄>"].hasTrustDialogAccepted` 是**逐目錄**的，存在各帳號自己的 `.claude.json` 裡。互動模式會跳對話框讓你點一下，`-p` 沒地方點，就卡死在這。

修法是在該目錄先互動跑一次點信任，或直接改該帳號 `.claude.json` 裡對應目錄的旗標。**每個新工作目錄都要來一次。**

跟坑三的差別值得記一下：坑三是逐帳號、一次性；這個是逐目錄、每個新目錄都要。

## 坑六：主帳號別顯式指回預設目錄

主帳號的主設定檔可能落在家目錄根的 `~/.claude.json`，而不是 `~/.claude/.claude.json`（早期路徑一路沿用下來的）。一旦對它顯式 `export CLAUDE_CONFIG_DIR=$HOME/.claude`，它會改去找後者、找不到就**新生一個空殼** —— 只有機器碼跟幾個 migration 旗標，沒有 `projects`、也沒有信任旗標。那個空殼留著，等於埋了一顆坑五的地雷。

正確做法是：對主帳號下指令**完全不要設這個變數**（但仍要用 `command` 繞 alias）。已經誤生空殼的話，先確認 `~/.claude.json` 完好（`projects` 數量正常、`mcpServers` 還在），再把空殼刪掉。

附帶一個好消息：即使走了錯的路徑，`plugin install`、`mcp add` 這類註冊仍然寫進 `~/.claude/settings.json`（主帳號真的會讀），所以功能沒白裝，只需要清掉空殼。

---

## 新增第 N 個帳號

順序有意義，第 5 步一定要在第 1 步之後。

1. **互動登入產 token**：`CLAUDE_CONFIG_DIR=~/.claude-accN claude` → 走完 onboarding → `/login` 該帳號 → 在同一個 shell 跑 `claude setup-token` 拿一年效期的 token。
2. **存進鑰匙圈**：`security add-generic-password -a "$USER" -s cc_oauth_acctN -w '<token>'`。**不要加 `-U`**，避免被 iCloud 鑰匙圈同步出去。
3. **建設定目錄**：`mkdir -p ~/.claude-accN`，抄一份乾淨的 `settings.json` 進去，並確認 `.claude.json` 有坑三那兩個旗標。第 1 步的互動登入不保證會把目錄落地，所以 `mkdir` 這步不能省。
4. **加 shell function**：照上面的樣板換編號。
5. **清掉共用鑰匙圈項目**：`security delete-generic-password -s "Claude Code-credentials"`。第 1 步的互動登入會把該帳號寫進共用項目，不清就等於沒隔離。
6. **`source ~/.zshrc && ccN`**，開起來用 `/usage` 確認用量、並照下一節的方法確認吃的是訂閱額度。

第 3 步抄範本時**挑那份沒有偏移過的**。我踩過拿「已經用了一陣子的第二個帳號」當範本，把它累積的一堆指令白名單和改過的模型設定一起抄了過去。

---

## 怎麼確認吃的是訂閱額度、不是 API 計費

`setup-token` 產的 token 走訂閱額度。要親眼確認的話，最硬的一條證據在狀態列的資料來源裡。

Claude Code 會把一包 JSON 從 stdin 餵給你設定的狀態列指令，其中有這幾個欄位：

```
.rate_limits.five_hour.used_percentage    # 0–100
.rate_limits.five_hour.resets_at          # 該視窗重置的 Unix epoch 秒數
.rate_limits.seven_day.used_percentage
.rate_limits.seven_day.resets_at
```

[官方文件](https://code.claude.com/docs/en/statusline)明講這個物件**只對 Claude.ai 訂閱者（Pro/Max）出現**，而且要等該 session 第一次 API 回應之後才有。所以「讀不讀得到這個欄位」本身就是判準。

⚠️ 這兩個百分比**不是內建狀態列會顯示的東西** —— 要自己寫一支狀態列腳本才看得到。最小版本：

```bash
#!/usr/bin/env bash
input=$(cat)
five=$(echo  "$input" | jq -r '.rate_limits.five_hour.used_percentage // empty')
seven=$(echo "$input" | jq -r '.rate_limits.seven_day.used_percentage // empty')
[ -n "$five"  ] && printf '5h:%.0f%% ' "$five"
[ -n "$seven" ] && printf '7d:%.0f%%'  "$seven"
exit 0
```

設定寫進該帳號的 `settings.json`：

```json
{ "statusLine": { "type": "command", "command": "bash ~/.claude/statusline.sh" } }
```

`// empty` 那個兜底不要省 —— 欄位缺席是正常狀態（session 還沒發出第一次請求時就沒有）。

不想寫腳本的話，`/usage`（`/cost` 是它的別名）看這個 session 的用量、`/status` 看 session 狀態。另外 token 模式下 header 會標 "Claude API"，那只是認證機制的標籤，不是計費方式，不用被它嚇到。

**順便一個「隔離到底有沒有成功」的檢查**：兩個帳號的 5h / 7d 百分比應該各走各的。如果兩邊數字連動，代表坑一沒清乾淨，你其實只是同一個帳號開了兩個視窗。

## 續期

token 效期一年，到期就重做一輪：登入該帳號（會暫時寫回共用鑰匙圈）→ `claude setup-token` 拿新的 → 刪掉舊的鑰匙圈項目再存新的 → **最後再清一次共用項目**。最後那步跟新增帳號的第 5 步同理，不能省。

---

## 每個帳號各自要裝的東西

多帳號並存最容易踩的長期問題是「同一件事在某個帳號做得到、換一個就做不到」，通常出在這幾處：

- **plugin 是逐設定目錄的**（各自的 `plugins/` 目錄加各自 `settings.json` 的 `enabledPlugins`），不跨帳號共用。新增一個要每邊各裝一次。
- **MCP 伺服器設定**放在各帳號 `.claude.json` 頂層的 `mcpServers`（等同 user scope）。`projects["<目錄>"].mcpServers` 是 local scope、只對那個目錄生效，兩者別搞混。
- **同一個功能有 MCP 和 plugin 兩種裝法時挑一種、全部帳號對齊**，不然日後很難查。
- ⚠️ 盤點狀態去讀 `.claude.json` 時，可能撞到某個執行中 session 正在寫入、拿到殘缺 JSON。**別用 `2>/dev/null` 把錯誤吞掉** —— 會把解析失敗誤讀成「設定不見了」。

---

## 一條界線

這套做法是讓**自己持有的多個獨立訂閱**在同一台機器上並存，不是把一組憑證分給多人共用。鑰匙圈項目不加 `-U` 那條也是同一個意思：token 是本機憑證，不該同步出去。

## 相關

還有第四種意義的「多開」：介面維持 Claude Code、後端整個換成別家模型的設定目錄。那個不算多一個 Anthropic 帳號、也不吃 Anthropic 額度，設定方式與踩過的坑另見 [model-backends.md](model-backends.md)。
