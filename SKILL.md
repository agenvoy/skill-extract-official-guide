---
name: extract-official-guide
description: 從 claude / openai / gemini 官網下載 prompting guide 到 skill 本地，再按統一規範萃取成每輪注入的模型指引檔。當使用者要求「更新 official guide」「抓官方 prompting guide」「萃取模型指引」「重寫 official_guides/<model>.md」或說「/extract-official-guide」時使用。
---

# Extract Official Guide

兩件事，順序固定：**下載**廠商公開的 prompting guide 到本地留底，再**萃取**成每輪注入模型 context 的反預設修正檔。

| | 路徑 |
|---|---|
| 來源留底 | `~/.claude/skills/extract-official-guide/offical_guide/<key>.md`（原始全文，16–98 KB） |
| 產出（模型檔） | `<DEST>/<key>.md`（55 行、6 區塊上限） |
| 產出（vendor） | `<DEST>/_vendor_<vendor>.md`（35 行、5 區塊上限） |
| 產出（base） | `<DEST>/_base.md`、`<DEST>/_base_unlisted.md`（各 20 行、4 區塊上下） |
| 預設 `<DEST>` | `<repo>/configs/prompts/system_prompt/official_guides/`（Agenvoy；其他 repo 必須明給） |

萃取規範對所有廠商、所有型號**完全一致**：同一份「一律不收」清單、同一份刪除理由表、同一個行數與區塊上限。不因廠商不同而放寬。

## Command Syntax

```
/extract-official-guide [<key>...] [--to <DEST>] [--download-only] [--extract-only]
```

| 參數 | 行為 |
|---|---|
| `<key>` 給了一個以上 | 只處理這些 key（`claude-opus-5`、`gpt-5.4`、`gemini`） |
| `<key>` 未給 | 對來源登錄表全部 key 走流程 |
| `--to` 未給 | 用預設 `<DEST>`；該目錄不存在則反問，不自建 |
| `--download-only` | 停在步驟 3，不萃取 |
| `--extract-only` | 跳過下載，直接用本地留底萃取 |
| `--base-only` | 只跑階段 B（重萃 `_base.md` / `_base_unlisted.md`），不動模型檔 |
| `--skip-base` | 跳過階段 B。給了 `<key>` 時的預設——單一 key 不該動一律注入的兩份 |

## 成功標準

| 項目 | 判準 |
|---|---|
| 留底完整 | 每份下載檔首行帶 `Retrieved <YYYY-MM-DD> from <url>`，內容為全文非摘要 |
| 規範一致 | 模型檔 ≤ 55 行／6 區塊，vendor 檔 ≤ 35 行／5 區塊，base 兩份 ≤ 25 行／4 區塊；標題全部取自固定字彙 |
| 刪除可追溯 | 每筆刪除都能指出對造原文（`_base.md`、共通層、本地契約、同檔另一條、工具 description）|
| 覆寫經同意 | 目標檔已存在時先出 diff 再問；未得同意不寫 |
| 實測 | 行數／區塊數／build／注入分岔皆實跑，不估算 |
| 破壞性變更對齊 | `CHANGELOG.md`「破壞性變更」逐項比對過既有產物，命中項已出 diff 並依同意處理 |

**停止條件：** 上述六項通過 + 輸出報告即停。不順手改 `_base.md` 以外的共通層、不補文件、不 commit。

---

## Workflow

三個階段，順序不可換：**先下載，再萃 base，最後萃模型檔**。模型檔要對 `_base.md` 去重，base 沒定稿就沒有去重的對照物。

開工前讀 `CHANGELOG.md` 定位差異：「破壞性變更」全部項目逐項比對既有產物（`offical_guide/<key>.md` 與 `<DEST>` 下各檔），命中即修改，但寫入照「留底既存」「目標既存」的契約走——出 diff → 問 → 同意才寫，命中不構成覆寫同意；回應中列出命中項與改動。

```
階段 A：下載（→ <SKILL_DIR>/offical_guide/）
  A1 Resolve    →  key → URL（來源登錄表）；未登錄的 key 反問 URL，不猜
  A2 Fetch      →  抓全文；抓失敗記下來繼續下一個，不用記憶內容代替
  A3 Reconcile  →  留底已存在 → diff → 問要不要更新（見「留底既存」）

階段 B：萃 base（→ <DEST>/_base.md、<DEST>/_base_unlisted.md）
  B1 Structural →  _base.md：harness 結構性且無模型檔在管的規則（不從廠商 guide 抓）
  B2 Common     →  _base_unlisted.md：跨廠商共通點（見「base 兩份的產生」）
  B2b Vendor    →  _vendor_<vendor>.md：該廠商總表的跨型號規則（有總表的廠商才有）
  B3 Overlap    →  重疊檢查：某類規則 >= 3 份模型檔在管 → 從 _base.md 移到 _base_unlisted.md
  B4 Land       →  目標已存在 → diff → 問要不要覆寫

階段 C：萃模型檔（→ <DEST>/<key>.md，逐 key）
  C1 Inventory  →  讀留底全文，列出所有 `##` 區塊
  C2 Sections   →  階段一刪區塊，留到 6 個上下
  C3 Items      →  階段二逐條篩，每筆刪除記下對造原文
  C4 Dedup      →  只扣 `_base.md` 的條目。`_base_unlisted.md` 的條目一律**保留**
  C5 Crosscheck →  對照共通層與本地契約（見「對照共通層」）
  C6 Land       →  目標已存在 → diff → 問要不要覆寫；同意才寫

階段 D：Verify + Report（見「Verify」「輸出格式」）
```

**C4 為何只扣 `_base.md`：** 命中模型檔時注入的是 `_base.md` + `<key>.md`，`_base_unlisted.md` 不注入。模型檔若對 `_base_unlisted.md` 去重，列名模型就完全拿不到那些規則。

---

## 來源登錄表

| Vendor | key | URL |
|---|---|---|
| Anthropic | `_vendor_claude`（總表） | `https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices.md` |
| Anthropic | `claude-<model>` | 同路徑下 `prompting-claude-<model>.md`（例：`prompting-claude-opus-5.md`、`prompting-claude-fable-5-1.md`）|
| OpenAI | `gpt-<ver>` | `https://developers.openai.com/api/docs/guides/latest-model/gpt-<ver>.md` |
| OpenAI | 當前家族 | `https://developers.openai.com/api/docs/guides/latest-model.md` |
| OpenAI | cookbook 補充 | `https://raw.githubusercontent.com/openai/openai-cookbook/main/examples/gpt-5/gpt-<ver 用 dash>_prompting_guide.ipynb` |
| Google | `gemini` | `https://ai.google.dev/gemini-api/docs/prompting-strategies`（HTML，需轉換） |

- OpenAI 與 Anthropic 的路徑加 `.md` 都直接回 markdown 全文（`content-type: text/markdown`），優先用它而不是 HTML 頁：不加後綴的 Anthropic 頁回 370 KB 的 Next.js HTML，`.md` 只有 12 KB
- **cookbook 抓 GitHub raw 的 `.ipynb`，不抓 cookbook 網頁**：網頁是 400 KB 的 Next.js app（內文埋在 JSON 裡），raw ipynb 是原始出處只有 30–54 KB。檔名的版本號用 dash（`gpt-5-2_prompting_guide.ipynb`，不是 `gpt-5.2`）。轉 markdown 的方式：`markdown` cell 原樣取出，`code` cell 包成 ```python 圍欄
- **Gemini 沒有 `.md` 變體**（加後綴回同一份 143 KB HTML）。抽 `<article>...</article>` 區段轉 markdown：`<pre>` → 圍欄、`<hN>` → `#`、`<li>` → `- `、其餘標籤剝除、`html.unescape`。留底首行要註明「Converted from the devsite HTML `<article>` region (no .md variant is served)」，讓下次比對時知道差異可能來自轉換器而非來源
- Anthropic 的總表頁有一張 model-specific guidance 表，列出每個型號自己的頁面 —— 新型號的 URL 從那張表取，不從型號名硬拼
- 舊型號的 live URL 會被新家族頁取代（實例：`gpt-6-astra` 的 URL 現在服務 GPT-6 家族頁）。這種情況走 Wayback（`https://web.archive.org/web/<timestamp>/<url>`），並把 snapshot timestamp 寫進留底首行
- 抓不到就在報告裡點名該 key 抓失敗，保留舊留底不動。**不用記憶中的內容補**——萃取的前提是原文在手

---

## 留底既存（下載階段）

`offical_guide/<key>.md` 已存在時：

```bash
SKILL_DIR=~/.claude/skills/extract-official-guide
diff -u "$SKILL_DIR/offical_guide/<key>.md" /tmp/fetched-<key>.md
```

| diff 結果 | 動作 |
|---|---|
| 無差異 | 不寫入，報告「無變動」，`--extract-only` 等價 |
| 只有 `Retrieved` 日期行不同 | 不寫入（重抓同一份內容不算更新） |
| 內容有差異 | 摘出差異要點（新增／刪除／改寫了哪些 `##` 區塊），**問要不要更新**；同意才覆寫 |

差異摘要要講「哪些區塊變了、變的是規則還是措辭」，不要貼整段 diff。廠商改寫措辭不影響產出；新增或刪除區塊才需要重跑萃取。

---

## 萃取規範：注入方式（疊加 + base 分岔）

`officialGuideSection()` 一次串三層，**每層各挑一份，同名區塊在組裝時合併**：

```
_base.md  +  _vendor_<vendor>.md  +  <key>.md
        ↓  mergeGuideSections()
一個 ## X 只出現一次，條目依 base → vendor → model 順序接續，逐字相同的條目去重
```

| 檔案 | 角色 | 何時注入 |
|---|---|---|
| `_base.md` | harness 結構性規則：指令衝突仲裁、工具呼叫紀律 | 一律 |
| `_vendor_<vendor>.md` | 該廠商總表的跨型號規則（例：`_vendor_claude.md` 來自 Anthropic 總表） | 模型名含該 vendor 名時，不管有沒有型號檔 |
| `<key>.md` | 該型號的反預設修正 | 命中時 |
| `_base_unlisted.md` | 通用 agentic 基線：Acting／Scope／Verification／Long context | **型號檔與 vendor 檔都沒命中**時 |

實測分岔（`.doc/test_offical_guide` 內容，2026-09-27）：

| 模型 | 拿到 |
|---|---|
| `claude-opus-5-5` / `claude-opus-5.5` | `_base` + `_vendor_claude` + `claude-opus-5.5`（兩者位元組相同） |
| `claude-opus-5` | `_base` + `_vendor_claude` + `claude-opus-5` |
| `claude-haiku-4-5` | `_base` + `_vendor_claude`（無型號檔） |
| `codex@gpt-5.4` | `_base` + `gpt-5.4`（OpenAI 無總表，故無 vendor 層） |
| `deepseek-chat` | `_base` + `_base_unlisted` |

前綴不影響比對（`copilot@`、`anthropic/`、`openrouter@google/` 都只是 `strings.Contains` 的雜訊）。

**型號檔只會命中一份**（長 key 優先，第一個命中即止），所以 `claude-opus-5-5` 與 `claude-opus-5` 是互斥的兩份，跨型號的通用規則**只能住 vendor 層**，不能靠某一份型號檔代管。

**跨層撞名由 code 處理，不由檔案迴避。** 固定字彙必然讓三層出現同名區塊（`_base` 的 `## Tools` 與 `gpt-5.4` 的 `## Tools`、`_vendor_claude` 的 `## Acting` 與 `claude-opus-5.5` 的 `## Acting`）。`mergeGuideSections()` 解析三層、按首次出現順序保留標題、把後面層的條目接到同一個標題下、丟掉逐字相同的條目。所以：

- 寫檔時**不必**為了避開撞名改標題或改用 `Other`——照字彙寫就好
- 「一個類別在一份檔案裡只出現一次」仍然成立（合併只跨層，不修一份檔案內部的重複）
- 合併後的標題順序是 `_base` → vendor → model 的首見順序，不是字彙表順序；字彙表順序只管單一檔案內部
- 實測：`copilot@claude-sonnet-5` 合併前 12 個標題（`Acting` `Scope` `Instructions` 各兩次），合併後 9 個

**這一節決定整份規範，改任何判準前先確認注入方式沒被改掉。**

兩段歷史：早期是互斥注入（命中模型檔就不注入 base），當時以「base 已經講過」為由刪模型檔的條目，等於兩邊都沒有——錯的。改成疊加後去重才安全。接著發現 base 本身沒被這份規範審過：它的 `## Acting` 有 8 份模型檔同管轄、`## Verification` 9 份、`## Scope` 10 份，前沿模型等於被規範兩次；而未列名的開源模型（`deepseek` / `qwen` / `gpt-oss`）真的需要那層基線。分岔同時解決這兩個相反的需求。

**什麼留在 `_base.md`：** 只有「harness 結構性、且沒有任何模型檔在管」的規則。

- `## Instruction conflicts` —— 疊了多層指令來源（system prompt、skill、MCP server instructions、工作目錄 `CLAUDE.md`、persona、rule），仲裁規則是本地需求。18 份模型檔只有 `gpt-5.4` 有 `## Instruction priority`，且講的是 recency 不是 specificity，兩者互補
- `## Tools` —— 18 份裡 10 份沒有工具區塊（6 份 claude-*、`gemini`、`gpt-5.5`、`gpt-6`、`gpt-6-astra`）

新增條目進 `_base.md` 前先跑重疊檢查（見「Verify」）。**計數是叫你去看，不是叫你直接搬**：固定字彙讓標題必然撞名，`## Tools` 在 8 份模型檔出現不代表那 8 份管的是同一條規則（模型檔的 `## Tools` 講觸發與批次，`_base.md` 的講「別猜參數」「沒讀過就別描述」）。逐條比對規則本身，真的有三份以上在講同一件事才搬。

---

## base 兩份的產生（階段 B）

兩份的來源與判準完全不同，不可用同一套方法產。預算各 **25 行、4 區塊**（比模型檔更緊：一律注入，成本乘以每一輪）。

| | `_base.md` | `_base_unlisted.md` |
|---|---|---|
| 來源 | harness 本地契約與工具層事實 | 所有留底的跨廠商共通點 |
| 從廠商 guide 抓？ | **不抓** —— 廠商不知道本地疊了幾層指令來源 | 抓，但只收多家共同講的 |
| 注入時機 | 一律 | 只在沒命中模型檔時 |
| 收錄判準 | 結構性 **且** 無模型檔在管（重疊 < 3） | 一條原則各家都講，或多數家講且其餘不反對 |
| 排除 | 任何單一廠商的型號專屬 steer | 只有一家講的、各家方向相反的 |

**`_base_unlisted.md` 的收錄門檻**：逐條回查是哪幾份留底講了同一件事，附得出依據才收。各家方向相反的一律不收——那是情境旋鈕不是共通點（實例：verbosity 的預設方向，同一廠商不同型號就相反，只能逐型號查）。

**階段 B 與階段 C 的相互依賴**：`_base.md` 的重疊檢查要數「有幾份模型檔在管這類規則」，而模型檔是階段 C 才產的。全檔重跑時用**上一版**的模型檔數目做 B3，階段 C 跑完後再重跑一次重疊檢查；數目跨過 3 就回頭改 base 並在報告點名。只跑單一 key 時不動 base。

**`_base_unlisted.md` 的服務對象是未列名模型**（`deepseek` / `qwen` / `gpt-oss` 等開源模型）。它們沒有官方 prompting guide，這份是它們唯一拿到的 agentic 基線，所以 Acting／Long inputs／Verification／Scope 四類缺一不可。

---

## 階段一：區塊選擇

**主要成本是區塊數，不是句子長度。** `gpt-5.6`（4 區塊）與 `gpt-6`（6 區塊）只挑該模型會做錯的地方；`gpt-5.2` / `gpt-5.4` 曾是 14–16 區塊的全生命週期 SOP，109 / 144 行全來自那些區塊。

超過預算時砍區塊，不要砍句子——把句子壓短只會讓每條都變模糊，區塊數不變。

預算 **55 行、6 區塊**（vendor 檔 35 行／5 區塊，base 兩份 25 行／4 區塊）。一個區塊只有在「這個模型在這件事上會偏離」時才收。覆蓋整個任務流程不是這份檔案的工作：流程模型自己會跑，這裡只放它跑偏的那幾處。

**一律不收：**

| 區塊 | 為何 |
|---|---|
| `## Plan` / `## Planning` / `## Before acting` | 流程 SOP，模型原生會做 |
| `## Progress updates` 的節奏規定 | 共通層 `Agentic runs narrate` 已管（該區塊若有模型專屬的反預設條目，只收那幾條） |
| `## Answer shape` / `## Final answers` / `## Delivery` 的長度與版面規定 | 共通層 `Output shape` 已管 |
| `## Design work` / `## Frontend` / `## Structured extraction` / `## One-shot applications` | 領域專章，`configs/prompts/guide/*.md` 按 topic 承接 |
| `## Without reasoning` | 注入時不看 reasoning level，reasoning 開著也會帶。要收回來得先讓 `officialGuideSection()` 收該參數 |
| API 參數操作（`effort`／`verbosity`／`budget_tokens`／`temperature`／prefill 遷移） | 那是 caller 的 request 設定，模型自己讀不到也改不了 |

刪領域專章前確認 `guide/` 真的有涵蓋。實例：`html_render.md` §123–142 的 typography / palette / motion / anti-AI-slop 比 `claude-opus-4.md` 的 `## Frontend` 更具體，刪掉不流失能力。沒有替代來源就先把內容搬進 `guide/`，再從模型檔刪。

---

## 階段二：條目篩選

留下來的區塊內，**條目預設全留**。廠商的 steer 是對該模型實測出來的，不能用「前沿模型本來就會」去判定要不要留。只有命中下表才刪。

| 刪除理由 | 判準與實例 |
|---|---|
| 與 `_base.md` 重複 | 逐字或近逐字同一條規則。加了 nuance 的保留——base 說「independent → one batch」，`gpt-5.4` 說「never parallelise dependent, ambiguous or irreversible steps」，後者更細，留 |
| 與共通層重複或相反 | `system_prompt.md` 的 `## Behavioral Constraints` 是本地輸出契約，優先。改模型檔，不改共通層。實例：`Complex work → bullets covering ... risks, next steps, open questions` 對造共通層 `Cut what you did not do ... closing summaries` |
| 與本地契約相反 | 本地契約優先。實例：`User instructions outrank a skill's guidance` 對造 `skill_execution.md` 原則 5（`follow the skill step but explicitly acknowledge the conflict`）與 `skillsHeader` 的 STRICT EXECUTION |
| 檔案內自相矛盾 | 保留較具體的一條。實例：`Summarise the intended action and its parameters before executing` vs 同檔 `Never narrate routine reads and test runs` |
| 思考指導 | 告訴模型「想多深、多用心」的句子，交給模型自己的推理能力。實例：`Set your own quality rubric first`、`Self-critique the current approach at intervals`、`Hold competing hypotheses`、`Do not stop at the first plausible answer`、`Certainty about correctness comes before handing back`、`Think very hard before answering` |
| 流程步驟 | 「先做 A 再做 B 最後 C」式的逐步 SOP。反預設條目要講「偏到哪」，不是排行程 |
| 怎麼用某個工具 | 寫進該工具的 `Description` 或參數 `describe`。實例：`Express an edit as the exact text before and after` → `edit_file`；`A purpose-built tool beats a raw shell command` → `run_command_readonly` |
| prompt 撰寫建議 | 廠商在教「你（開發者）該怎麼寫 prompt」，不是在 steer 模型。實例：few-shot 要 3–5 個、XML 包裹輸入、長 context 把查詢放最後、metaprompting |
| 具名案例 | 個案解不了下一個情境。實例：`Inter, Roboto, Arial`、`Space Grotesk`、具名公司、具名版本號 |

每一條刪除都要能指出對造原文。**「這條看起來多餘」不是理由。**

---

## 對照共通層

萃取到步驟 7 時，逐條拿產出去對下面這幾份。衝突一律**改模型檔**，不改共通層。

| 對照對象 | 路徑 | 管什麼 |
|---|---|---|
| 輸出契約 | `configs/prompts/system_prompt/system_prompt.md` §Behavioral Constraints | 語言、輸出深度、輸出形狀、long-form 落檔、narration、shortcut 揭露、路徑形狀 |
| 結構性規則 | `.../official_guides/_base.md` | 指令衝突仲裁、工具呼叫紀律 |
| agentic 基線 | `.../official_guides/_base_unlisted.md` | Acting／Long inputs／Verification／Scope |
| 領域專章 | `configs/prompts/guide/*.md` | 前端渲染、市場分析、subagent 派工、RAG、工具錯誤處理等 |
| 工具層 | 各工具的 `Description` 與參數 `describe` | 「哪個需求用哪個工具、怎麼用」 |

system prompt 只留跨工具的輸出規範；怎麼用哪個工具由工具自己的 description 說。這條同時是刪除理由表裡「怎麼用某個工具」的依據。

---

## 區塊類別（固定字彙）

標題只能取自下表，**照表中順序排列**，一個類別在一份檔案裡只出現一次。放不進任何類別的條目掛 `## Other`，永遠排在最後。

| # | 類別 | 收什麼 |
|---|---|---|
| 1 | `## Acting` | 自主程度、persistence、turn 何時結束、可逆性門檻、approval 時機、偏行動 vs 偏等待 |
| 2 | `## Scope` | 交付範圍、restraint、不加未被要求的東西、completeness 契約 |
| 3 | `## Instructions` | 字面遵循、指令衝突仲裁、優先序、skill 與 user 的先後、絕對化詞怎麼讀 |
| 4 | `## Tools` | 觸發時機、平行與批次、context gathering、retrieval budget、tool persistence |
| 5 | `## Delegation` | subagent 何時派、派幾個、派給誰做什麼 |
| 6 | `## Grounding` | 讀過才講、引用與 citation、不編造、progress claim 對照證據、該搜尋時搜尋、時間與 cutoff |
| 7 | `## Verification` | 跑什麼測、驗到什麼程度、過度驗證 |
| 8 | `## Review` | 審查別人或自己產出的程式碼時的回報行為 |
| 9 | `## Long context` | 長輸入的處理：先重述約束、錨回章節、引用決定答案的那一句 |
| 10 | `## Long horizon` | 長跑：compaction、memory、跨 context 接手、增量推進 |
| 11 | `## Progress` | 工具呼叫之間給使用者的更新：頻率、長度、內容 |
| 12 | `## Output` | verbosity、形狀、寫作風格、更正、結尾訊息、結構化輸出 |
| 13 | `## Code` | 實作與編輯慣例：錯誤處理、型別、重用、編輯粒度 |
| 14 | `## Other` | 以上皆非。**超過兩條就代表字彙不夠，回報要新增哪個類別，不要把 `Other` 養大** |

**為何固定字彙：** 自由命名會讓同一條規則在 18 份檔案裡掛在 18 個不同標題下（`Turn endings`／`Autonomy`／`Persistence`／`Outcome first` 其實是同一類），`_base.md` 的重疊檢查就數不出「有幾份模型檔在管這類規則」，去重判斷失效。固定字彙讓重疊檢查變成一行 grep。

**判斷邊界：** 類別由條目**管什麼行為**決定，不由來源的原標題決定。廠商的 `## Preambles` 講的是工具呼叫間給使用者的更新 → `## Progress`；`## Final answers` 刪掉版面規定後剩下的路徑引用 → `## Output`。

---

## 格式

- `##` 標題（模板已有 `## Model Guide` 作為外層，模型檔不再包一層）
- 一條一行 `- `，用 `→` 表達 when/then
- 不寫 motivation（每輪常駐，篇幅換不到遵循度；理由寫進 commit message 與本次報告）
- 省略號一律 ASCII `...`
- 全英文（注入模型 context，與 system prompt 其餘部分一致）

## 命名與打包

檔名去掉 `.md` 就是比對用的 key，要能被模型名包含。**版本號一律寫 dot 形**（`gpt-5.4`、`claude-opus-5.5`、`claude-fable-5.1`）。長 key 優先，`claude-opus-5.5` 贏過 `claude-opus-5`。

`claude` 開頭的 key 比對前把兩邊的 `.` 換成 `-` 再比（`guideKeyMatches`），所以 `claude-opus-5.5.md` 同時吃 `claude-opus-5.5`（router 與第三方寫法）與 `claude-opus-5-5`（Anthropic 官方 model id）。**只有 claude 走這條**：其他廠商不存在同一型號兩種寫法。

`_vendor_<vendor>.md` 的 `<vendor>` 是模型名裡的廠商字樣（`_vendor_claude` → 比對 `claude`），同樣走 claude 正規化。

`_` 前綴標示「不是型號 key」：`_base` 與 `_base_unlisted` 在比對迴圈裡跳過，`_vendor_` 開頭的另走 vendor 分支。`go:embed official_guides/*.md` 的 glob 會匹配 `_` 開頭的檔案（`_` 排除規則只作用於整個目錄形式的 `go:embed`），所以照常打包，新增檔案不需改 code。

---

## 目標既存（產出階段）

`<DEST>/<key>.md` 已存在時**只做比對並詢問**，不直接寫：

```bash
diff -u "<DEST>/<key>.md" /tmp/extracted-<key>.md
```

差異報告要分三類講，讓使用者知道覆寫會失去什麼：

| 類別 | 說明 |
|---|---|
| 新增區塊／條目 | 這次萃取多出來的（通常是廠商更新了 guide） |
| 移除區塊／條目 | 舊檔有、新檔沒有的 —— **逐條說明是命中哪條刪除理由，還是這次漏掉**。漏掉就補回去再問 |
| 措辭改寫 | 同一條規則換句話講；數量多但不改行為 |

同意才寫。未得同意時把萃取結果留在報告裡，不落檔、不寫 `/tmp` 以外的地方。

---

## Verify

寫入後、輸出報告前逐項執行：

```bash
cd <DEST>

# 1. 篇幅（模型檔 >55 行或 >6 區塊、vendor >35/5、base >25/4 → 回階段一砍區塊）
for f in *.md; do printf "%-24s %3s 行 %2s 區塊\n" "$f" "$(wc -l < $f)" "$(grep -c '^## ' $f)"; done

# 2. 階段一殘留（有輸出就是沒做乾淨）
grep -n '^## \(Plan\|Planning\|Before acting\|Answer shape\|Final answers\|Delivery\|Design work\|Frontend\|Structured extraction\|One-shot applications\|Without reasoning\)$' *.md

# 3. 空區塊（刪掉唯一條目會留下空標題）
awk '/^## /{if(h&&n==0)print FILENAME":"h": empty section"; h=FNR; n=0; next} /^- /{n++} END{if(h&&n==0)print FILENAME":"h": empty section"}' *.md

# 4. _base.md 重疊檢查：_base.md 的每個類別有幾份模型檔在管，>= 3 就搬 _base_unlisted.md
for c in $(grep -h '^## ' _base.md | sed 's/^## //'); do
  printf "%-16s %s 份模型檔\n" "$c" "$(grep -l "^## $c\$" *.md | grep -v '^_' | wc -l | tr -d ' ')"
done

# 5. 標題字彙（有輸出就是用了表外標題）
grep -h '^## ' *.md | sort -u | grep -v -x -E '## (Acting|Scope|Instructions|Tools|Delegation|Grounding|Verification|Review|Long context|Long horizon|Progress|Output|Code|Other)'

# 6. 標題順序遞增 + 同類別不重複 + Other <= 2 條
python3 - <<'PY'
import glob
ORDER=['Acting','Scope','Instructions','Tools','Delegation','Grounding','Verification','Review',
       'Long context','Long horizon','Progress','Output','Code','Other']
for f in sorted(glob.glob('*.md')):
    hs=[l[3:].strip() for l in open(f) if l.startswith('## ')]
    idx=[ORDER.index(h) for h in hs]
    if idx!=sorted(idx): print(f"ORDER {f}: {hs}")
    if len(set(hs))!=len(hs): print(f"DUP {f}: {hs}")
    n=0; inb=False
    for l in open(f):
        if l.startswith('## '): inb = l[3:].strip()=='Other'; continue
        if inb and l.startswith('- '): n+=1
    if n>2: print(f"OTHER {f}: {n} 條 — 回報要新增哪個類別")
PY

# 7. 省略號與全形字元
grep -n '…' *.md

# 8. 合併後無重複標題（跑實際注入，不是看檔案）
#    在 internal/agents/exec 寫臨時測試呼叫 officialGuideSection(model)，
#    對每個代表性模型檢查標題不重複、無連續空行；跑完即刪

# 9. build + 實際嵌入
cd <repo> && go build -tags fts5 ./... && grep -a -c '<新內容片段>' $(command -v agen)
```

禁用標題是對**內容類別**而非標題字串 —— 該區塊若仍有不屬於該類別的實質條目，改掛能描述它的標題（實例：`gpt-5.3` 的 `## Final answers` 刪掉版面規定後，剩下的路徑引用與 command output 轉述改掛 `## Reporting`），不要為了讓檢查過關而保留原標題。

分岔行為要實跑驗證，不能只靠 build：未列名模型（`deepseek-chat`、`qwen3-max`）應拿到 `_base.md` + `_base_unlisted.md`，列名模型（`codex@gpt-5.4`、`claude-opus-5`）應拿到 `_base.md` + 該模型檔，且**不含** `_base_unlisted.md` 的 `## Acting`。驗證用的測試檔跑完即刪，不留在 repo。

---

## 已知陷阱

- `grep` 對 binary 少了 `-a` 會靜默回 0，看起來像沒嵌入。驗證一律 `grep -a -c`
- 刪掉某區塊的唯一條目會留下空標題，Verify 第 3 項要掃
- 改 `officialGuideSection()` 的比對邏輯前，先確認 `_base.md` 與 `_base_unlisted.md` 都仍被跳過，否則會被當成模型 key
- 新增一份模型檔等於讓該模型**失去** `_base_unlisted.md`；新檔若沒涵蓋 Acting／Verification／Scope，那個模型就沒人管這幾件事
- 廠商總表頁的內容會漂移到子頁：Anthropic 把型號差異收進 `prompting-claude-<model>`，只抓總表會拿到指向子頁的一句話而非規則本文
- 一個 `claude-*` key 的產出通常由**兩份來源**餵：總表（agentic systems、long-horizon、general solutions 等跨型號段）＋型號子頁（該型號的反預設）。只抓子頁重萃會靜默丟掉總表來的區塊——覆寫前的 diff 若出現大量「移除區塊」，先確認是不是少抓了總表
- 型號檔一次只命中一份，所以跨型號的通用規則放在某一份型號檔等於其他型號拿不到；那些規則的位置是 `_vendor_<vendor>.md`
- 新增 vendor 檔會讓該廠商**所有**模型多一層注入，包含已有型號檔的。逐字相同的條目 `mergeGuideSections()` 會去掉，但**改寫過的同義條目不會**——同一條規則兩種措辭會並列在同一個區塊裡，看起來像兩條規則
- `_vendor_` 的比對不能靠「從命中的 key 推前綴」：`gpt-5` 是 `gpt-5.4` 的合法前綴，會被誤當成它的 vendor 層。用 `_vendor_` 檔名顯式宣告
- SKILL.md 裡的 shell 片段不要出現 `$0` 或 `$1`（skill loader 會把它替換成呼叫引數）；需要標示位置用 `FNR`／`FILENAME`
- 同一份 guide 同時談「開發者該怎麼寫 prompt」與「模型會怎麼偏」，前者整段不收（見刪除理由表「prompt 撰寫建議」）

---

## CHANGELOG 維護

修改本 skill 的 `SKILL.md` 時，同一次改動內：

1. 更新 `CHANGELOG.md` 的「最新改動」日期
2. 含移除行為或需既有產物端處理的變更 → 寫進「破壞性變更」（一項一行、新者在上，寫「既有產物中找什麼 → 改成什麼」）；新增與修正不記錄。不寫版號

**為何：** 讀完整規範（本檔）即得最新規範；CHANGELOG 只負責快速定位既有產物與最新規範的差異。只記破壞性變更，檔案不隨改動無限增長，落後多次的產物也能一次看完必須處理的項目；新增與修正在重跑時讀全規範自然取得。

---

## 禁止事項

| 禁止 | 為何 |
|---|---|
| 抓不到就用記憶內容補 | 萃取的前提是原文在手；記憶產出的規則無法回頭查證對造原文 |
| 未經同意覆寫留底或產出 | 留底是查證依據，產出是每輪注入的 prompt，兩者都不可靜默替換 |
| 為了塞進 50 行而把句子壓模糊 | 成本在區塊數；壓句子只會讓每條都失去觸發力 |
| 改共通層來遷就模型檔 | 共通層是本地契約，優先序高於廠商建議 |
| 在模型檔寫 motivation／API 參數／工具用法 | 每輪常駐的 token 只換觸發，不換說明 |
| 執行 `git commit` | 產出檔案 + 報告即停 |

---

## 輸出格式

```
## 下載

| key | URL | 結果 |
|---|---|---|
| ... | ... | 無變動 / 已更新 / 待確認 / 抓取失敗（原因） |

## 產出

| 檔案 | 行數 | 區塊 | 狀態 |
|---|---|---|---|
| ... | N | M | 新增 / 已覆寫 / 待確認 |

## 刪了什麼

<逐檔列出被刪的區塊與條目，每筆附對造原文（檔名＋那一條）>

## 類別覆蓋

<14 類各有幾份檔案在用；Other 用了幾條>

## 驗證

<篇幅、標題字彙、標題順序、空區塊、_base 重疊、build、grep -a、注入分岔 各一行>

## CHANGELOG 命中（有才寫）

<命中的破壞性變更項目、對應的既有產物與改動、是否已得同意寫入>

## 需要確認（有才寫）

<待覆寫的 diff 摘要、抓取失敗的 key、guide/ 沒有承接的領域專章>
```
