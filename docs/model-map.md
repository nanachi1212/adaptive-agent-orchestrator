# Model Capability Map

> **這是一份 runtime factual mapping，不是產品承諾。**
> 本檔只記錄「在特定環境、特定時間點實際觀察到的可用模型」。
> 它不保證任何模型在其他帳號、方案、工作區政策或未來版本下仍然存在。

## Verification metadata

| 項目 | 值 |
| --- | --- |
| Verified date | 2026-09-23 |
| Platform | Windows 11 Pro (10.0.26200) |
| Codex CLI | `codex-cli 0.152.0` |
| Codex binary build | `codex.exe` build `247581e40ee272fb` |
| Model catalogue `client_version` | `0.155.0` |
| Model catalogue `fetched_at` | `2026-09-22T16:08:17Z` |

### Evidence source

1. `%USERPROFILE%\.codex\models_cache.json` — 欄位 `models[].slug`、`supported_reasoning_levels[].effort`、`multi_agent_version`、`default_reasoning_level`、`multi_agent_reasoning_effort`、`visibility`、`upgrade`。
2. `codex.exe` 內嵌的 `spawn_agent` 參數描述與 runtime 驗證錯誤字串（詳見 [`runtime-schema.md`](runtime-schema.md)）。

**未採用的來源：** 上游 README、skill 文件、模型命名慣例、model picker 畫面。
這些都不是能力來源，只能當作提示。

### Environment-specific warning

以下清單**與帳號、方案與工作區模型政策綁定**。

- 不同帳號可能看到不同的 model slug 集合。
- 工作區管理員的模型政策可能移除其中任何一項。
- Codex 版本更新可能新增、改名或移除 slug。
- 因此：**本檔是 snapshot，不是常數表。** 使用前請以實際 runtime 行為驗證。

---

## Currently observed models

| slug | visibility | supported reasoning efforts | `multi_agent_version` | default reasoning | known restrictions |
| --- | --- | --- | --- | --- | --- |
| `gpt-6-astra` | list | `low` `medium` `high` `xhigh` `max` | `v2` | `low` | `multi_agent_reasoning_effort` 宣告為 `xhigh`（非 `max`） |
| `gpt-5.6-sol` | list | `low` `medium` `high` `xhigh` `max` | `v2` | `low` | 無 |
| `gpt-5.6-terra` | list | `low` `medium` `high` `xhigh` `max` | `v2` | `medium` | 無 |
| `gpt-5.6-luna` | list | `low` `medium` `high` `xhigh` `max` | **`v1`** | `medium` | multi-agent schema 為 V1，與其他 list 模型不同 |
| `gpt-5.5` | list | `low` `medium` `high` `xhigh` | 未宣告 | `medium` | **不支援 `max`**；宣告於 2026-10-14 退役，`upgrade` 指向 `gpt-5.6-sol` |

另有兩個 `visibility = "hide"` 的內部模型（`gpt-reserve`、`codex-auto-review`）。
它們不是使用者可選路由目標，本專案不將其納入任何 tier。

### Not available on this account

| slug | 狀態 |
| --- | --- |
| `gpt-6-sol` | **不存在於本帳號的模型清單中** |
| `gpt-6-luna` | **不存在於本帳號的模型清單中** |

這兩個 slug 出現在 upstream `three-tier-agent-orchestrator` 於 commit `ebd723e`
（"feat: move Sol and Luna tiers to GPT-6"）之後的文件中。

在本環境傳入任一者，`spawn_agent` 會以 `Unknown model ... for spawn_agent` 失敗。

**因此：不得把任何 model slug hard-code 成 skill 的必要條件。**

---

## Tier 與 slug 的關係

本專案的 orchestration tier 是 **capability role**，不是模型名稱。

```
Tier  (capability role)        ──resolved at runtime──▶   model slug + reasoning effort
```

- **Tier 定義**寫在 `SKILL.md`，用能力與任務性質描述，不提 slug。
- **slug 對應**只寫在本檔。
- 兩者刻意分離，讓 runtime 變動時**只需要更新 mapping，不需要重新設計 orchestration architecture**。

### Current mapping snapshot

| Tier | capability role | 目前解析結果 |
| --- | --- | --- |
| **0** | 主線自行完成，不 spawn | 當下 session 的主線模型與 reasoning |
| **A** | fast-agentic / 低成本 | `gpt-5.6-luna` + `medium` |
| **B** | fast-agentic / 標準 coding | `gpt-5.6-luna` + `high` |
| **C** | reasoning / 跨模組與深度分析 | `gpt-5.6-sol` + `medium` 或 `high` |
| **D** | frontier / 高 correctness 成本 | `gpt-5.6-sol` + `max`，或 `gpt-6-astra` + `high` / `xhigh` |

備註：

- `gpt-5.6-terra` 目前未納入預設路由，保留為 Tier B 與 Tier C 之間的可選項。
- Tier D 預設不使用 `max`。`gpt-6-astra` 自己宣告的 `multi_agent_reasoning_effort` 是 `xhigh`；`max` 只保留給錯誤成本最高的任務。
- 主線模型決定本次 session 的 multi-agent schema 版本，詳見 [`runtime-schema.md`](runtime-schema.md)。

---

## Maintenance policy

runtime 改變時的處理順序：

1. 重新讀取 `models_cache.json`，更新本檔的 verification metadata 與模型表。
2. 更新上方的 tier mapping snapshot。
3. **不要**因此改寫 `SKILL.md` 的 tier 定義或 orchestration 架構。
4. 只有當「某個 capability role 在任何可用模型上都無法被滿足」時，才需要回頭檢討架構。

若發現本檔與實際 runtime 行為不一致，**以 runtime 行為為準**，並更新本檔。
