# graphify

[English](README.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md) | [日本語](README.ja-JP.md)

[![CI](https://github.com/safishamsi/graphify/actions/workflows/ci.yml/badge.svg?branch=v3)](https://github.com/safishamsi/graphify/actions/workflows/ci.yml)
[![PyPI](https://img.shields.io/pypi/v/graphifyy)](https://pypi.org/project/graphifyy/)
[![Sponsor](https://img.shields.io/badge/sponsor-safishamsi-ea4aaa?logo=github-sponsors)](https://github.com/sponsors/safishamsi)

**一個面向 AI 編碼助手的技能。** 在 Claude Code、Codex、OpenCode、OpenClaw、Factory Droid 或 Trae 中輸入 `/graphify`，它會讀取你的檔案、建立知識圖譜，並把原本不明顯的結構關係還給你。更快理解程式碼庫，找到架構決策背後的「為什麼」。

完全多模態。你可以直接丟進去程式碼、PDF、Markdown、截圖、流程圖、白板照片，甚至其他語言的圖片——graphify 會用 Claude vision 從這些內容中提取概念和關係，並把它們連接到同一張圖裡。透過 tree-sitter AST 支援 20 種語言（Python、JS、TS、Go、Rust、Java、C、C++、Ruby、C#、Kotlin、Scala、PHP、Swift、Lua、Zig、PowerShell、Elixir、Objective-C、Julia）。

> Andrej Karpathy 會維護一個 `/raw` 資料夾，把論文、推文、截圖和筆記都丟進去。graphify 就是在解決這類問題——相比直接讀取原始檔案，每次查詢的 token 消耗可降低 **71.5 倍**，結果還能跨會話持久保存，並且會明確區分哪些內容是實際發現的，哪些只是合理推斷。

```
/graphify .                        # 可用於任意目錄：程式碼庫、筆記、論文都可以
```

```
graphify-out/
├── graph.html       可互動圖譜：可點節點、搜尋、按社群過濾
├── GRAPH_REPORT.md  God nodes、意外連接、建議提問
├── graph.json       持久化圖譜：數週後仍可查詢，無需重新讀原始檔案
└── cache/           SHA256 快取：重複執行時只處理變更過的檔案
```

新增 `.graphifyignore` 檔案，以排除你不想納入圖譜的資料夾：

```
# .graphifyignore
vendor/
node_modules/
dist/
*.generated.py
```

語法與 `.gitignore` 相同。樣式會比對相對於 graphify 執行目錄的檔案路徑。

## 運作原理

graphify 分兩輪執行。第一輪是確定性的 AST 提取，對程式碼檔案做結構分析（類別、函式、匯入、呼叫圖、docstring、解釋性注釋），這一輪不需要 LLM。第二輪會並行呼叫 Claude 子代理處理文件、論文和圖片，從中提取概念、關係和設計動機。最後把兩邊結果合併到一個 NetworkX 圖裡，用 Leiden 社群發現演算法做聚類，並匯出成可互動 HTML、可查詢 JSON，以及一份人類可讀的稽核報告。

**聚類是基於圖拓撲完成的，不依賴 embeddings。** Leiden 按邊密度發現社群。Claude 抽取出的語義相似邊（`semantically_similar_to`，標記為 `INFERRED`）本來就存在於圖中，所以會直接影響社群劃分。圖結構本身就是相似性訊號，不需要額外的 embedding 步驟，也不需要向量資料庫。

每條關係都會被標記為 `EXTRACTED`（直接在源材料中找到）、`INFERRED`（合理推斷，並附帶信心分數）或 `AMBIGUOUS`（有歧義，需要複核）。所以你始終知道哪些是實際發現的，哪些是模型猜出來的。

## 安裝

**需求：** Python 3.10+，並且使用以下平台之一：[Claude Code](https://claude.ai/code)、[Codex](https://openai.com/codex)、[OpenCode](https://opencode.ai)、[OpenClaw](https://openclaw.ai)、[Factory Droid](https://factory.ai) 或 [Trae](https://trae.com)

```bash
pip install graphifyy && graphify install
```

> PyPI 套件當前暫時叫 `graphifyy`，因為 `graphify` 這個名字還在回收中。CLI 命令和 skill 命令仍然都是 `graphify`。

### 平台支援

| 平台 | 安裝命令 |
|------|----------|
| Claude Code (Linux/Mac) | `graphify install` |
| Claude Code (Windows) | `graphify install`（自動偵測）或 `graphify install --platform windows` |
| Codex | `graphify install --platform codex` |
| OpenCode | `graphify install --platform opencode` |
| OpenClaw | `graphify install --platform claw` |
| Factory Droid | `graphify install --platform droid` |
| Trae | `graphify install --platform trae` |
| Trae CN | `graphify install --platform trae-cn` |

Codex 用戶還需要在 `~/.codex/config.toml` 的 `[features]` 下打開 `multi_agent = true`，這樣才能並行提取。Factory Droid 使用 `Task` 工具進行並行子代理調度。OpenClaw 目前的並行 agent 支援還比較早期，所以使用順序提取。Trae 使用 Agent 工具進行並行子代理調度，**不支援** PreToolUse hook，因此 AGENTS.md 是其常駐機制。

然後開啟你的 AI 編碼助手，輸入：

```
/graphify .
```

注意：Codex 使用 `$` 而非 `/` 來呼叫技能，所以請輸入 `$graphify .`。

### 讓助手始終優先使用圖譜（建議）

圖建立完成後，在專案裡執行一次：

| 平台 | 命令 |
|------|------|
| Claude Code | `graphify claude install` |
| Codex | `graphify codex install` |
| OpenCode | `graphify opencode install` |
| OpenClaw | `graphify claw install` |
| Factory Droid | `graphify droid install` |
| Trae | `graphify trae install` |
| Trae CN | `graphify trae-cn install` |

**Claude Code** 會做兩件事：
1. 在 `CLAUDE.md` 中寫入一段規則，告訴 Claude 在回答架構問題前先讀 `graphify-out/GRAPH_REPORT.md`
2. 安裝一個 **PreToolUse hook**（寫入 `settings.json`），在每次 `Glob` 和 `Grep` 前觸發

如果知識圖譜存在，Claude 會先看到：_"graphify: Knowledge graph exists. Read GRAPH_REPORT.md for god nodes and community structure before searching raw files."_——這樣 Claude 會優先按圖譜導覽，而不是一上來就 grep 整個專案。

**Codex、OpenCode、OpenClaw、Factory Droid、Trae** 會把同樣的規則寫進專案根目錄的 `AGENTS.md`。這些平台沒有 PreToolUse hook，所以 `AGENTS.md` 是它們的常駐機制。

卸載時使用對應平台的 uninstall 命令即可（例如 `graphify claude uninstall`）。

**常駐模式和顯式觸發有什麼區別？**

常駐 hook 會優先暴露 `GRAPH_REPORT.md`——這是一頁式總結，包含 god nodes、社群結構和意外連接。你的助手在搜尋檔案前會先讀它，因此會按結構導覽，而不是按關鍵字亂搜。這已經能涵蓋大部分日常問題。

`/graphify query`、`/graphify path` 和 `/graphify explain` 會更深入：它們會逐跳遍歷底層 `graph.json`，追蹤節點之間的精確路徑，並展示邊級別細節（關係類型、信心分數、源位置）。當你想從圖譜裡精確回答某個問題，而不僅僅是獲得整體感知時，就該用這些命令。

可以這樣理解：常駐 hook 是先給助手一張地圖，`/graphify` 這幾個命令則是讓它沿著地圖精確導覽。

<details>
<summary>手動安裝（curl）</summary>

```bash
mkdir -p ~/.claude/skills/graphify
curl -fsSL https://raw.githubusercontent.com/safishamsi/graphify/v3/graphify/skill.md \
  > ~/.claude/skills/graphify/SKILL.md
```

把下面內容加到 `~/.claude/CLAUDE.md`：

```
- **graphify** (`~/.claude/skills/graphify/SKILL.md`) - any input to knowledge graph. Trigger: `/graphify`
When the user types `/graphify`, invoke the Skill tool with `skill: "graphify"` before doing anything else.
```

</details>

## 用法

```
/graphify                          # 對目前目錄執行
/graphify ./raw                    # 對指定目錄執行
/graphify ./raw --mode deep        # 更積極地抽取 INFERRED 邊
/graphify ./raw --update           # 只重新提取變更檔案，並合併到已有圖譜
/graphify ./raw --cluster-only     # 只重新聚類已有圖譜，不重新提取
/graphify ./raw --no-viz           # 跳過 HTML，只產生 report + JSON
/graphify ./raw --obsidian                          # 額外產生 Obsidian vault（可選）
/graphify ./raw --obsidian --obsidian-dir ~/vaults/myproject  # 將 vault 寫入指定目錄

/graphify add https://arxiv.org/abs/1706.03762        # 抓取論文、儲存並更新圖譜
/graphify add https://x.com/karpathy/status/...       # 抓取推文
/graphify add https://... --author "Name"             # 標記原作者
/graphify add https://... --contributor "Name"        # 標記是誰把它加入語料庫的

/graphify query "attention 和 optimizer 之間有什麼連接？"
/graphify query "attention 和 optimizer 之間有什麼連接？" --dfs   # 追蹤一條具體路徑
/graphify query "attention 和 optimizer 之間有什麼連接？" --budget 1500  # 把預算限制在 N tokens
/graphify path "DigestAuth" "Response"
/graphify explain "SwinTransformer"

/graphify ./raw --watch            # 檔案變更時自動同步圖譜（程式碼：立即更新；文件：提醒你）
/graphify ./raw --wiki             # 建立可供 agent 抓取的 wiki（index.md + 每個 community 一篇文章）
/graphify ./raw --svg              # 匯出 graph.svg
/graphify ./raw --graphml          # 匯出 graph.graphml（Gephi、yEd）
/graphify ./raw --neo4j            # 產生給 Neo4j 用的 cypher.txt
/graphify ./raw --neo4j-push bolt://localhost:7687    # 直接推送到執行中的 Neo4j
/graphify ./raw --mcp              # 啟動 MCP stdio server

# git hooks - 跨平台，在 commit 和切分支後重建圖譜
graphify hook install
graphify hook uninstall
graphify hook status

# 常駐助手規則 - 按平台區分
graphify claude install            # CLAUDE.md + PreToolUse hook（Claude Code）
graphify claude uninstall
graphify codex install             # AGENTS.md（Codex）
graphify opencode install          # AGENTS.md（OpenCode）
graphify claw install              # AGENTS.md（OpenClaw）
graphify droid install             # AGENTS.md（Factory Droid）
graphify trae install              # AGENTS.md（Trae）
graphify trae uninstall
graphify trae-cn install           # AGENTS.md（Trae CN）
graphify trae-cn uninstall

# 直接從終端機查詢圖譜（不需要 AI 助手）
graphify query "attention 和 optimizer 之間有什麼連接？"
graphify query "顯示認證流程" --dfs
graphify query "CfgNode 是什麼？" --budget 500
graphify query "..." --graph path/to/graph.json
```

支援混合檔案類型：

| 類型 | 副檔名 | 提取方式 |
|------|--------|----------|
| 程式碼 | `.py .ts .js .jsx .tsx .go .rs .java .c .cpp .rb .cs .kt .scala .php .swift .lua .zig .ps1 .ex .exs .m .mm .jl` | tree-sitter AST + 呼叫圖 + docstring / 注釋中的 rationale |
| 文件 | `.md .txt .rst` | 透過 Claude 提取概念、關係和設計動機 |
| Office | `.docx .xlsx` | 轉換成 Markdown 後透過 Claude 提取（需要 `pip install graphifyy[office]`） |
| 論文 | `.pdf` | 引文挖掘 + 概念提取 |
| 圖片 | `.png .jpg .webp .gif` | Claude vision——截圖、圖表、任意語言都可以 |

## 你會得到什麼

**God nodes**——度最高的概念節點（整個系統最容易匯聚到的地方）

**意外連接**——按綜合得分排序。程式碼-論文之間的邊會比程式碼-程式碼邊權重更高。每條結果都會附帶一段人話解釋。

**建議提問**——圖譜特別擅長回答的 4 到 5 個問題。

**「為什麼」**——docstring、行內注釋（`# NOTE:`、`# IMPORTANT:`、`# HACK:`、`# WHY:`）以及文件裡的設計動機都會被抽取成 `rationale_for` 節點。不只是知道程式碼「做了什麼」，還能知道「為什麼要這麼寫」。

**信心分數**——每條 `INFERRED` 邊都有 `confidence_score`（0.0-1.0）。你不只知道哪些是猜出來的，還知道模型對這個猜測有多有把握。`EXTRACTED` 邊恒為 1.0。

**語義相似邊**——跨檔案的概念連接，即使結構上沒有直接依賴也能建立關聯。比如兩個函式做的是同一類問題但彼此沒有呼叫，或者某個程式碼類別和某篇論文裡的演算法概念本質相同。

**超邊（Hyperedges）**——用來表達 3 個以上節點的群組關係，這是普通兩兩邊表達不出來的。比如：一組類別共同實作一個協定、認證鏈路裡的一組函式、同一篇論文某一節裡的多個概念共同組成一個想法。

**Token 基準**——每次執行後都會自動印出。對混合語料（Karpathy 的儲存庫 + 論文 + 圖片），每次查詢的 token 消耗可以比直接讀原始檔案少 **71.5 倍**。第一次執行需要先提取並建圖，這一步會花 token；後續查詢直接讀取壓縮後的圖譜，節省會越來越明顯。SHA256 快取保證重複執行時只重新處理變更檔案。

**自動同步**（`--watch`）——在背景終端機裡跑著，程式碼庫一變化，圖譜就會跟著更新。程式碼檔案儲存會立刻觸發重建（只走 AST，不用 LLM）；文件/圖片變更則會提醒你跑 `--update` 進行 LLM 再提取。

**Git hooks**（`graphify hook install`）——安裝 `post-commit` 和 `post-checkout` hook。每次 commit 後、每次切分支後都會自動重建圖譜。如果重建失敗，hook 會以非零碼退出，讓 git 顯示錯誤而非靜默繼續。不需要額外開一個背景程序。

**Wiki**（`--wiki`）——為每個 community 和 god node 產生類似維基百科的 Markdown 文章，並提供 `index.md` 作為入口。任何 agent 只要讀 `index.md`，就能透過普通檔案導覽整個知識庫，而不必直接解析 JSON。

## 實作範例

| 語料 | 檔案數 | 壓縮比 | 輸出 |
|------|--------|--------|------|
| Karpathy 的儲存庫 + 5 篇論文 + 4 張圖片 | 52 | **71.5x** | [`worked/karpathy-repos/`](worked/karpathy-repos/) |
| graphify 原始碼 + Transformer 論文 | 4 | **5.4x** | [`worked/mixed-corpus/`](worked/mixed-corpus/) |
| httpx（合成 Python 函式庫） | 6 | ~1x | [`worked/httpx/`](worked/httpx/) |

Token 壓縮效果會隨著語料規模增大而更明顯。6 個檔案本來就塞得進上下文視窗，所以 graphify 在這種場景裡的價值更多是結構清晰度，而不是 token 壓縮。到了 52 個檔案（程式碼 + 論文 + 圖片）這種規模，就能做到 71x+。每個 `worked/` 目錄裡都帶了原始輸入和真實輸出（`GRAPH_REPORT.md`、`graph.json`），你可以自己跑一遍核對數字。

## 隱私

graphify 會把文件、論文和圖片的內容發送給你所用 AI 編碼助手背後的模型 API 來做語義提取——可能是 Anthropic（Claude Code）、OpenAI（Codex），或者你當前平台使用的其他提供方。程式碼檔案則完全在本地透過 tree-sitter AST 處理，不會把程式碼內容發出去。專案本身沒有任何遙測、使用追蹤或分析。唯一的網路請求就是語義提取階段呼叫你平台自己的模型 API，使用的也是你自己的 API key。

## 技術堆疊

NetworkX + Leiden（graspologic）+ tree-sitter + vis.js。語義提取由 Claude（Claude Code）、GPT-4（Codex）或你當前平台所執行的模型完成。不需要 Neo4j，不需要 server，整體是純本地執行。

## 下一步計劃

graphify 是圖層。我們正在它之上建立 [Penpax](https://safishamsi.github.io/penpax.ai)——一個在裝置上的數位孿生，將你的會議、瀏覽器歷史、檔案、電子郵件和程式碼連接成一個持續更新的知識圖譜。不上雲端，不用你的資料訓練。[加入等候名單。](https://safishamsi.github.io/penpax.ai)

## Star 歷史

[![Star History Chart](https://starchart.cc/safishamsi/graphify.svg)](https://starchart.cc/safishamsi/graphify)

<details>
<summary>貢獻</summary>

**實作範例**是最能建立信任的貢獻方式。對一個真實語料跑 `/graphify`，把輸出儲存到 `worked/{slug}/`，再寫一份誠實的 `review.md`，評價圖譜哪些地方做得對、哪些地方做得不對，然後提交 PR。

**提取 bug**——提 issue 時請附上輸入檔案、對應的快取項目（`graphify-out/cache/`）以及它漏提取或瞎編了什麼。

模組職責和新增語言的方法見 [ARCHITECTURE.md](ARCHITECTURE.md)。

</details>
