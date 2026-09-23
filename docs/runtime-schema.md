# Runtime Schema: `spawn_agent`

> 本檔記錄**實際驗證到的** `spawn_agent` 行為，用來取代對文件與命名慣例的推測。
> 模型清單不在本檔，見 [`model-map.md`](model-map.md)。

## Verification metadata

| 項目 | 值 |
| --- | --- |
| Verified date | 2026-09-23 |
| Codex CLI | `codex-cli 0.152.0` |
| Codex binary build | `codex.exe` build `247581e40ee272fb` |
| Evidence | binary 內嵌的工具參數描述與 runtime 驗證錯誤字串 |

本檔所有引號內的英文字串都是 runtime 的原始訊息，不是改寫。

---

## Schema versions

`spawn_agent` 有兩個互不相容的參數集合。版本由 **主線（parent）模型的 `multi_agent_version`** 決定，
不是由目標 subagent 模型決定。對應關係見 [`model-map.md`](model-map.md)。

| | V1 | V2 |
| --- | --- | --- |
| Context isolation 欄位 | `fork_context`（boolean） | `fork_turns`（string） |
| `task_name` | 未宣告 | 宣告支援，且為必要 |
| 其他共同欄位 | `agent_type`、`model`、`reasoning_effort`、`message` / `items` | 同左 |

啟動前先確認目前 schema 宣告了哪些欄位，**不得傳入目前 schema 未宣告的欄位**。

---

## Context isolation

**V2：**

```
fork_turns = "none"
```

**V1：**

```
fork_context = false
```

**不得同時傳入兩者。** V2 收到 `fork_context` 會直接失敗：

> `fork_context is not supported in MultiAgentV2; use fork_turns instead`

V1 的 `fork_context` 語意（runtime 原文）：

> True forks the current thread history into the new agent; false or omitted starts with only the initial prompt.

因為 subagent 不繼承主線對話歷史，**subagent prompt 是唯一可信的任務背景**。
必要資訊、範圍、禁止事項與驗收條件都必須寫進 prompt。

---

## Partial context

`fork_turns` 接受三種值：

| 值 | 意義 |
| --- | --- |
| `"none"` | 不傳入任何周邊脈絡 |
| `"all"` | 傳入全部周邊脈絡（預設值） |
| 正整數字串，例如 `"3"` | 只 fork 最近 N 個 turn |

runtime 驗證訊息：

> `fork_turns must be 'none', 'all', or a positive integer string`

注意這是**字串**，不是整數。

**本專案政策：做跨模型 routing 時，預設使用完全隔離 `"none"`。**

- 不要預設使用 partial fork。
- partial fork 只在有明確理由時才使用，並且要在該次委派中說明理由。
- 使用 partial fork 時，prompt 的自足性要求不會因此放寬。

---

## Full-history inheritance

runtime 原文：

> Full-history forks (`fork_turns` omitted or `"all"`) inherit the parent model and reasoning effort
> and do not accept overrides. Only set `model` or `reasoning_effort` when explicitly requested by the
> user, applicable `AGENTS.md` instructions, or skill instructions; when doing so, set `fork_turns` to
> `"none"` or a positive integer string.

結論：

- full-history fork **會繼承 parent 的 model 與 reasoning effort**。
- 在 full-history fork 下**無法可靠 override** `model` / `reasoning_effort`。
- 因此 **full-history fork 不適合作為 model routing 的方法**。
- 跨模型 routing 必須搭配 context isolation。

同樣的耦合也適用於 `agent_type`：

> Full-history forked agents inherit the parent agent type; omit `agent_type`, or spawn without a full-history fork.

這代表 context isolation 在本專案中不是風格偏好，而是 **routing 的前提條件**。

---

## `task_name`

- **只有目前 schema 宣告支援時才使用。**
- V1 不假設存在，不要傳入。
- V2 為必要欄位，缺少時 runtime 會回報：

  > `spawned agent is missing a canonical task name`

V2 的命名語意（runtime 原文摘要）：spawn 時給定的 `task_name` 會接在 parent 的 canonical path 之後，
例如 parent 為 `/root/task1` 且 `task_name` 為 `task_3` 時，canonical task name 為 `/root/task1/task_3`。

若使用 `task_name` 作為可見標籤來標示實際路由，**名稱只是標籤，實際 `model` 與 `reasoning_effort` 才是權威**。
啟動前應確認標籤與實際參數一致，不一致就不要啟動。

---

## Capability detection

**允許**根據 `spawn_agent` 的 runtime 錯誤判斷能力：

| 情況 | runtime 訊息形式 |
| --- | --- |
| 模型不存在 | `Unknown model '<slug>' for spawn_agent. Available models: ...` |
| reasoning effort 不支援 | `Reasoning effort '<effort>' is not supported for model '<slug>'. Supported reasoning efforts: ...` |

兩則訊息都會附上實際可用清單，可直接用於 fallback 決策。

**不得**根據下列來源猜測能力：

- README 或任何文件的敘述
- 靜態的 model 名稱或命名慣例（例如假設某個世代必然存在對應的 slug）
- model picker 是否顯示某個選項（可作為提示，不作為判準）

判準只有一個：**實際 spawn 結果**。

---

## Other runtime-enforced constraints

以下限制由 runtime 自行強制，skill 不需要重複實作，但設計時應納入考量。

| 限制 | runtime 訊息 |
| --- | --- |
| Agent 巢狀深度上限 | `Agent depth limit reached. Solve the task yourself.` |
| 並行 agent 數量上限 | `There are N available concurrency slots, meaning that up to N agents can be active at once, including you.` |
| 共用檔案系統 | `All agents have access to the same container and filesystem as you.` / `All agents use the same current working directory.` |

並行度應讀取 runtime 宣告的 slot 數，不要臆測。

因為所有 agent 共用同一個 working directory，平行寫入必須指定互斥的檔案或模組 ownership。

### Platform default on delegation

runtime 對 `spawn_agent` 的自述指引：

> Only call this tool for a concrete, bounded subtask that can run independently alongside useful local
> work; otherwise continue locally.

平台預設立場即為「非必要不委派」。本專案的 Tier 0 優先原則與此一致。
