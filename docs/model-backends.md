# Model backends — 讓 Claude Code 跑在 Claude 以外的模型上

**查這個的動機**：Codex CLI 用不慣，但想用手上的 ChatGPT 訂閱額度。目標是開一個獨立的 Claude Code 設定目錄（`CLAUDE_CONFIG_DIR`）指向別的後端，介面維持 Claude Code、模型換成 GPT。

**先講清楚一件事**：這樣開出來的**不是多一個 Anthropic 帳號**。它只是多一個設定目錄（`CLAUDE_CONFIG_DIR` 指過去），後端根本不是 Anthropic —— 不吃 Anthropic 額度、也不需要 Anthropic 登入，`ANTHROPIC_AUTH_TOKEN` 只是本機中介自己的金鑰。實際消耗的是 ChatGPT 訂閱額度。

所以它可以跟原本的設定並存，兩邊互不影響：原本那份照常連 Anthropic，這一份走訂閱端點。

Anthropic 官方文件明講不支援透過閘道跑非 Claude 模型，所以兩邊任一升版都可能要重調。

---

## 三條路

| 路線 | 認證 | 計費 | 評語 |
|---|---|---|---|
| LiteLLM ＋ 官方 API key | OpenAI Platform 金鑰 | 按量計費 | 最穩，但要另外付錢 |
| LiteLLM ＋ `chatgpt/` 供應商 | 訂閱制 OAuth（裝置碼流程） | 訂閱額度 | 通用閘道，Claude Code 專屬的細節要自己補 |
| 專用中介 | 自帶 OAuth 登入 | 訂閱額度 | 本來就是為這件事寫的，Claude Code 專屬處理都做掉了 |

專用中介指的是 `raine/claude-code-proxy`、CLIProxyAPI、`aryan877/claude-proxy` 這一類。它們比通用閘道多做兩件事（依各自專案自述，未獨立驗證）：**把加密推理區塊按對話快取、下一輪重播**，以及**剝掉 Claude Code 內建的網路搜尋工具**讓後端用自己的原生搜尋。

---

## Claude Code 這邊要設什麼

以下都出自 Anthropic 官方文件，逐項核對過。

**基本兩條**：

```bash
export ANTHROPIC_BASE_URL='http://127.0.0.1:4000'
export ANTHROPIC_AUTH_TOKEN="$GATEWAY_KEY"
```

`ANTHROPIC_AUTH_TOKEN` 走 `Authorization: Bearer`，`ANTHROPIC_API_KEY` 走 `x-api-key`。閘道讀哪個就設哪個，設錯會拿到 401。

**模型別名要全部改指**，否則背景工作還是會去找 Anthropic：

| 變數 | 管什麼 |
|---|---|
| `ANTHROPIC_MODEL` | 主對話模型 |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | `opus` 別名 |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | `sonnet` 別名 |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `haiku` 別名**與背景功能**（標題生成、額度查詢那些探針） |
| `ANTHROPIC_DEFAULT_FABLE_MODEL` | `fable` 別名 |
| `CLAUDE_CODE_SUBAGENT_MODEL` | 子代理 |

`ANTHROPIC_DEFAULT_HAIKU_MODEL` 是最容易漏的一個 —— 官方文件寫明它同時管背景功能。

**接閘道才會撞到的**：

| 變數 | 治什麼 |
|---|---|
| `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1` | 上游退 `context_management` 或 `Extra inputs are not permitted` 的 400 |
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1` | 上游不吃 adaptive 推理的 400 |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | 閘道回的超限錯誤字面跟 Anthropic 不同，自動壓縮不會觸發，得手動告知上限 |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | 設在閘道模型輸出上限之下 |
| `CLAUDE_CODE_SKIP_FAST_MODE_ORG_CHECK=1` | fast mode 的可用性檢查直接打 `api.anthropic.com`、不跟著 base URL 走 |

**模型選單**：`CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1` 只認 Anthropic 原生格式的 `/v1/models`，LiteLLM 回的是 OpenAI 格式（未解的功能請求 #27180），所以探索用不了。改用 `ANTHROPIC_CUSTOM_MODEL_OPTION` 手動加一筆進選單。

**首次啟動的坑**：全新設定目錄第一次跑，若憑證只放在專案層的設定檔，會在首次設定精靈之前讀不到、還是叫你登入。要放 shell 匯出或該設定目錄的 `settings.json` 的 `env` 區塊。

---

## 會失去的功能

擴展思考與推理強度對應、提示快取、fast mode、`/cost` 費用估算、Remote Control（2.1.196 起，只要 `ANTHROPIC_BASE_URL` 指向非 Anthropic 主機就停用）、語音聽寫。

`CLAUDE.md`、skills、MCP、hooks、子代理、檔案讀寫、指令執行、權限流程這些都在用戶端，不受影響。

---

## 那個 Cloudflare 403 是什麼（本頁重點）

LiteLLM 有一份還開著的回報（BerriAI/litellm#27175）：打 ChatGPT 後端拿到 Cloudflare 挑戰頁而不是 JSON，即使 OAuth 憑證合法。回報者的診斷是「LiteLLM 只送了 Authorization 跟 User-Agent、沒帶 Cookie」。

**那個診斷是錯的。** 我自己寫的供給層直打同一個後端是通的，而且**一個 Cookie 都沒送**；後來照下面的方法覆寫識別、把 LiteLLM 真的架起來，也一次 403 都沒有。

真因是**用戶端識別**。讀 LiteLLM 主線的 `litellm/llms/chatgpt/common_utils.py`：

```python
DEFAULT_ORIGINATOR = "codex_cli_rs"
...
version = _get_litellm_version()
candidate = f"{originator}/{version} ({os_type} {os_version}; {arch}) {terminal_ua}{suffix}"
```

它**本來就在宣稱自己是 Codex CLI**，只是宣稱得不一致 —— 版本號填的是 LiteLLM 自己的（真實的 Codex CLI 沒有那種版本號）、`platform.system()` 吐 `Darwin` 而真實用戶端送的是 `Mac OS`、後面還多黏一段終端機名稱。一個自稱某個用戶端、版本號卻不存在的請求，被擋下來很合理。

**還有路徑差別**：#27175 打的是 `/chat/completions`（Chat Completions 轉接那條），而 responses 供應商的 `get_complete_url` 回的是 `f"{api_base}/responses"` —— 那才是訂閱制真正的端點。設定裡指定 responses 模式，那份回報的路徑根本不會被走到。

**兩個覆寫開關**，同一支檔案裡：

```python
def get_chatgpt_originator():
    return os.getenv("CHATGPT_ORIGINATOR") or DEFAULT_ORIGINATOR

def get_chatgpt_user_agent(originator):
    override = os.getenv("CHATGPT_USER_AGENT")
    if override:
        return _safe_header_value(override) or DEFAULT_USER_AGENT
```

所以 `CHATGPT_ORIGINATOR` 與 `CHATGPT_USER_AGENT` 可以填成跟本機安裝的 Codex CLI 一致的值。**別寫死版本字串** —— 動態組（版本讀 `codex --version`，系統版本用 Darwin 的 release 號、不是行銷版號），否則升版當天就腐爛，而且會讓請求宣稱自己是一個不存在的舊版本。

另外 responses 供應商的 `validate_environment` 回的是 `{**default_headers, **headers}`，使用者給的 header 會蓋掉預設，所以走設定檔的 `extra_headers` 也是一條路。

### 憑證不能直接指向 Codex CLI 那份

`CHATGPT_TOKEN_DIR` 與 `CHATGPT_AUTH_FILE`（預設 `auth.json`）看起來可以指到 Codex CLI 已經登入好的目錄、省掉再跑一次裝置碼流程。**實際不行，兩邊的檔案結構不一樣。** 讀 `litellm/llms/chatgpt/authenticator.py`：

```python
access_token = auth_data.get("access_token")
refresh_token = auth_data.get("refresh_token")
account_id = auth_data.get("account_id")
```

它讀的是**頂層**欄位，而 Codex CLI 的 `auth.json` 把這些包在 `tokens` 底下（頂層只有 `auth_mode`、`OPENAI_API_KEY`、`tokens`、`last_refresh`）。直接指過去，LiteLLM 會讀到一個沒有 `access_token` 的檔，然後掉進裝置碼登入流程。

兩條路：

1. **讓 LiteLLM 跑自己的裝置碼登入**，拿一份獨立憑證。乾淨，而且沒有下面那個續期打架的問題。
2. **給它一份攤平過的複本**放在另一個目錄。可行，但要注意**別把 `refresh_token` 一起複製過去** —— 兩個程式拿同一個 refresh token 去續期會互相輪替，誰先續誰就可能把另一邊踢登出。只複製 `access_token`、`account_id`、`id_token`，LiteLLM 在 token 有效期內不會去碰續期；效期到了就重做一次複製，或改走第一條路。

無論哪條，**都不要讓 LiteLLM 寫到 `~/.codex/auth.json`**。那個檔是 Codex CLI 日常在用的。

---

## 實測：真的架起來會撞到的兩件事

上面那些查證做完之後實際架了一套（LiteLLM 1.95.0、摘要鎖版、Docker），**指紋覆寫確實有效，一次 403 都沒遇到**。但另外撞到兩個文件上沒寫的問題，兩個都會讓 Claude Code 直接不能用。

### 一、非串流會壞，串流正常

同一個請求，`stream: false` 回：

```
ChatgptException - Unknown items in responses API response: []
```

改成 `stream: true` 就一切正常。這應該就是 #27175 那位回報者提到的「有些流程回空陣列」—— 跟 Cloudflare 無關，是聚合非串流回應那段的問題。

**Claude Code 本來就走串流，所以這條不影響它**，但拿 curl 手動測的時候會先撞到這個，容易誤判成整條路不通。

### 二、系統提示會被退

```
{"detail":"System messages are not allowed"}
```

訂閱端點不收 system role —— 系統層指令要放在 `instructions` 欄位。而 LiteLLM 的 chatgpt 轉接層繼承 `OpenAIConfig`，原樣把 messages 送出去，**沒有把 system 折進 `instructions` 那一步**。

Claude Code 每一輪都送系統提示，所以這條是硬阻塞。繞法是掛一個 pre-call hook 在送出前改寫：

⚠️ **搬去哪很重要，這裡我第一次做錯了。** 把 system 折進第一則使用者訊息確實能讓請求通過，但那等於**把系統層指令降級成對話內容**——實際後果是專案的 `CLAUDE.md` 形同被無視（問它專案規範裡寫死的數字，它答不出來）。

正解是搬進 **`instructions`**。那是 Responses API 放系統層指令的正規欄位，而且在 chatgpt 轉接層的 `allowed_keys` 裡，會自動跟 Codex 的基礎指令合併：

```python
base_instructions = get_chatgpt_default_instructions()
existing_instructions = request.get("instructions")
if existing_instructions:
    if base_instructions not in existing_instructions:
        request["instructions"] = f"{base_instructions}\n\n{existing_instructions}"
```

改成塞 `instructions` 之後，同一題（問專案規範裡的數字）直接答對、而且沒去讀任何檔案。

```python
from litellm.integrations.custom_logger import CustomLogger

class FoldSystemIntoUser(CustomLogger):
    async def async_pre_call_hook(self, user_api_key_dict, cache, data, call_type):
        # Anthropic 格式（/v1/messages）：system 是頂層獨立欄位，不是 messages 裡的 role
        if data.get("system"):
            header = _text(data.pop("system"))
            # …接到第一則使用者訊息前面
        # chat 格式：system 在 messages 裡，另外處理
        return data

proxy_handler_instance = FoldSystemIntoUser()
```

設定檔掛 `litellm_settings: callbacks: custom_hooks.proxy_handler_instance`。

⚠️ **兩種形狀都要處理**。只處理 `messages` 裡的 system role 是不夠的 —— Claude Code 走 `/v1/messages`，那條路的 system 是**頂層獨立欄位**，補了才會生效。

代價是語意降級：系統提示變成對話內容，模型不會把它當系統層指令看待。要真正對等就得改 LiteLLM 原始碼、讓它折進 `instructions`。

### 三、工具結果裡的圖片會被靜默丟掉，然後模型開始編

這條最危險。同一張圖：

| 圖片放在哪 | 結果 |
|---|---|
| 使用者訊息裡 | 正確辨識 |
| **工具結果裡**（`tool_result` 包 `image` 區塊） | **圖被丟掉，模型捏造一個答案** |

實測拿一張寫著 `BANANA 7788` 的圖，經工具結果送過去，模型回「圖片裡的文字是：Hello, World!」。它不會說自己沒看到圖。

這正是檔案讀取工具回傳圖片走的路徑，所以「叫它看一張圖」會壞。表面症狀可能是它繞去呼叫 OCR 指令——那其實是比較好的結果，壞的情況是它一本正經地瞎掰。

**靜默捏造比報錯危險得多**，所以這條一定要處理。同一個 pre-call hook 裡把圖片從工具結果撈出來、改附成後面一則使用者訊息（那個位置的圖確定送得到模型眼前），原位置留一句文字說明。改完同一題正確答出 `BANANA7788`。

### 推理強度：綁在模型上，不要靠介面的強度選單

介面自己的強度選單是 Claude 專屬的，送到這個後端沒有意義。改成**在閘道設定裡把強度綁在模型條目上**，切換強度＝切換模型：

```yaml
  - model_name: gpt-sol-max
    model_info:
      mode: responses
    litellm_params:
      model: chatgpt/gpt-5.6-sol
      reasoning_effort: max
```

實測 `none` / `low` / `medium` / `high` / `xhigh` / `max` 六個值都通得過（`minimal` 會被閘道自己擋下）。再把模型別名對到強度階梯，介面上的模型選單就變得有意義——挑「最強那個別名」＝最高強度，挑「最快那個別名」＝低強度加快模型。

### 順帶：`/v1/messages` 打的是 `/responses`

1.95.0 上實測，錯誤訊息裡的上游網址是 `https://chatgpt.com/backend-api/codex/responses`。所以 Claude Code 那條路走的是原生端點，不是 #27175 回報的 chat 轉接路徑。

### 映像的 tag 不是你以為的那個

GitHub 發行版標 `v1.95.0`，但容器映像的 tag 是 **`main-v1.95.0`** 這種格式。用 `v1.95.0` 拉會拉不到。更穩的做法是**用摘要鎖版**（`ghcr.io/berriai/litellm@sha256:…`），tag 會被重推、摘要不會。arm64 壓縮後 363 MB、落地 1.16 GB。

---

## 脈絡上限：兩邊的算法不一樣，我在這裡算錯過

這條最值得記，因為它的失敗模式是「來不及壓縮、直接被伺服器拒絕」，而不是我以為的「早一點壓縮」。

**訂閱端點的輸入與輸出共用同一份總預算。** 官方 CLI 自己就是這樣切的——約 27 萬輸入加 12.8 萬輸出保留、合計 40 萬。而直打伺服器實測，輸入推到 39 萬就被拒。

Claude 那邊不是這樣：脈絡上限指的是輸入，輸出是**另外一份獨立預算**。所以從 Claude 的習慣直覺過來會算錯——我第一次就把輸入預算設成 35 萬，加上輸出保留 10 萬等於 45 萬，超過拒絕線。**輸入預算加輸出保留必須落在拒絕線以內**，這才是正確的算法。

順帶澄清一個常見誤解：**模型規格其實兩邊差不多**（都在百萬量級）。差別在用戶端與端點放你用多少——官方 CLI 客戶端自訂上限、訂閱端點另設遠低於規格的線，都不是模型本身的限制。

### 介面不知道這個上限，你得告訴它

介面靠模型探索或內建清單取得脈絡上限。走自訂閘道時它兩者都拿不到，只能用一個偏小的預設值當分母——後果是狀態列的用量比例失真，而且**自動壓縮會提早觸發**，脈絡還很空就被砍掉一輪。

用 `CLAUDE_CODE_AUTO_COMPACT_WINDOW` 明講輸入預算，並用 `CLAUDE_CODE_MAX_OUTPUT_TOKENS` 留住輸出那份。取值時保守比較安全：**低估只是早一點壓縮，高估會直接撞爆。**

⚠️ 官方文件寫這個值會被夾在十萬與「模型的脈絡上限」之間。如果介面心目中的上限就是那個偏小的預設值，設太大可能被夾回去，不完全生效。

### 模型選單只能改一個名字

`opus` / `sonnet` / `haiku` / `fable` 是介面**內建的四個固定格位**，只能改它們指向誰，改不了顯示的字。`ANTHROPIC_CUSTOM_MODEL_OPTION` 可以額外加一筆真名進選單，但**只能加一筆**。

正規解法是閘道模型探索。實測結果是**現在不行**：

```
[gatewayDiscovery] 0 usable models after filter
```

探索有跑、有抓到，但全被濾掉——介面只認 Anthropic 格式的 `/v1/models`（`type: "model"` 加 `display_name`），而通用閘道回的是 OpenAI 格式（`object: "model"`，沒有 display_name）。要按真名列出選單，得在閘道前面再加一層只改寫這個端點的代理。

實用的替代做法：把四個別名對到不同的推理強度，選單就變得有意義——挑「最強那個別名」等於最高強度，挑「最快那個」等於低強度加快模型。

### 提示快取那個警告可以關掉

停用提示快取的旗標會讓介面在啟動時印一行警告。實測**拿掉旗標請求照樣正常**——介面送的快取標記在轉譯過程被忽略，不會讓後端報錯，所以那個旗標本來只是在抑制一個嚇人的提示。

但這**不代表真的有快取了**。那套快取標記對這個後端沒有作用。後端自己有沒有做前綴快取，我沒有驗證，不替它保證。

---

## 加密推理區塊：請求端沒問題，跨輪回填還沒驗

多步工具任務靠 `reasoning.encrypted_content` 維持推理鏈。**請求端 LiteLLM 處理得比預期好** —— `litellm/llms/chatgpt/responses/transformation.py` 會主動把它塞進去，而且 `include` 本來就在白名單裡：

```python
include = list(request.get("include") or [])
if "reasoning.encrypted_content" not in include:
    include.append("reasoning.encrypted_content")
request["include"] = include

allowed_keys = { ..., "include", ... }
return {k: v for k, v in request.items() if k in allowed_keys}
```

**還沒驗的是回程**：Claude Code 走的是 Anthropic Messages 格式，而加密推理區塊不是那個格式裡的概念。它從回應拿到之後，下一輪轉回 Responses 的 input 陣列時有沒有被帶回去 —— 沒帶回去，多步任務每一輪都要重新想一次。

這是目前唯一還會左右「值不值得用」的未知。比較點已經不是 Cloudflare 過不過，而是哪一邊的推理區塊跨輪接得完整。專用中介把「按對話快取、下一輪重播」當賣點在講，通用閘道未必有這一層。

---

## 注意事項

- **LiteLLM 出過供應鏈事件**：PyPI 上的 1.82.7 與 1.82.8 兩版被植入憑證竊取程式與後門（2026-03-24，攻擊者從 Trivy 的 CI 偷走發佈密碼直接推包）。套件已下架，但那個時間窗內建的容器映像若沒鎖版也可能中鏢。**鎖明確版本、對 GitHub 正式發行版驗過再裝**，別抓 latest，也別 `uv tool install 'litellm[proxy]'` 這樣不鎖版就裝。
- **訂閱條款是灰帶**。以非官方用戶端使用訂閱憑證屬於各家條款的模糊地帶；Anthropic 從 2026 年初起封鎖訂閱制 token 被第三方工具使用，反向這條目前沒看到封鎖，但風險自負。
- **額度是共用的**。這條路吃的是同一份 ChatGPT 訂閱額度，跟自己日常用 Codex CLI 共用；Claude Code 每輪送的脈絡（系統提示加整套工具定義）比 Codex CLI 肥不少。
- **後端隨時可能調整**，別讓任何依賴穩定性的東西掛在這條路上。

---

## 參考

- [Claude Code：連接 LLM 閘道](https://code.claude.com/docs/en/llm-gateway-connect)
- [Claude Code：模型設定](https://code.claude.com/docs/en/model-config)
- [LiteLLM：ChatGPT 訂閱供應商](https://docs.litellm.ai/docs/providers/chatgpt)
- [LiteLLM：`/v1/messages` 端點](https://docs.litellm.ai/docs/anthropic_unified/)
- [BerriAI/litellm#27175](https://github.com/BerriAI/litellm/issues/27175) — 那份診斷有誤的 Cloudflare 回報
- [BerriAI/litellm#24518](https://github.com/BerriAI/litellm/issues/24518) — 供應鏈事件的完整時序
- [raine/claude-code-proxy](https://github.com/raine/claude-code-proxy)
