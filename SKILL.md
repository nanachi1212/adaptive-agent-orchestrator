---
name: adaptive-agent-orchestrator
description: "讓目前主線優先直接完成工作，只有委派確實有平行、獨立驗證或能力落差價值時才建立 subagent；依任務的 difficulty 與 risk 兩軸選擇最低足夠的 capability tier，而非依工作量或檔案數量升級。必要 capability 暫時不可用時採 capability-based fallback，不整體停止工作；只有高風險任務缺少最低安全能力時才 hard stop。驗證深度依風險調整，push/PR 前執行 secret exposure 檢查。適用於 multi-agent coding、code review、debugging、測試、跨模組實作、架構分析等情境。不假設特定 model 名稱為必要條件，實際 model 與 reasoning mapping 一律以 docs/model-map.md 為準，schema 細節以 docs/runtime-schema.md 為準。"
---

# Adaptive Agent Orchestrator

以「最低足夠能力優先」取代固定三層強制委派。目前主線負責理解目標、判斷 difficulty 與 risk、決定是否委派、選擇最低足夠的 capability tier、驗收整合結果；只有委派確實有價值時才建立 subagent，且優先使用最低足夠的 reasoning effort。

## 1. Tier 0：主線優先

預設由目前主線直接完成工作。

- 小型、局部、容易驗證、主線能力足以安全完成的任務：不要 spawn subagent。
- 只有第 2 節的 Delegation Gate 至少有一項成立時，才考慮委派。
- 不得因為任務「工作量大」或「涉及檔案多」就自動升級模型或自動 spawn subagent。工作量、推理難度（difficulty）、錯誤成本（risk）是三個不同維度，見第 3 節。

## 2. Delegation Gate（委派前三道閘）

Spawn 前至少判斷以下三類價值是否成立：

- **A. Parallelism**：這個子任務能否與其他工作真正獨立並行，且有實際效益？
- **B. Independent verification**：是否需要一個不被主線既有推論污染的獨立判斷，例如 second opinion、review、根因交叉驗證？
- **C. Capability advantage**：是否存在明顯更適合這個子任務的 capability tier，落差會實質改變結果品質或成本？

三者皆不成立時，continue locally，不要 spawn。

Context isolation（避免主線 context 被子任務細節污染）可以作為輔助理由之一，但**不能單獨成為濫用 subagent 的藉口**——仍需搭配 A、B、C 至少一項成立。

## 3. Difficulty × Risk：兩個獨立維度

不要用單一「任務難度」或「工作量」決定 capability tier。分開判斷：

**Difficulty**（完成這個任務所需的推理深度）
- `routine`：機械式、規格明確
- `standard`：一般開發工作，做法清楚
- `complex`：需要跨模組推理、架構判斷或非顯而易見的除錯
- `expert`：需要深度專業判斷，例如複雜併發、安全推理、大型架構決策

**Risk**（做錯的代價）
- `low`：容易復原，影響範圍小
- `medium`：需要正常程度的驗證
- `high`：影響使用者資料、對外行為或系統穩定性
- `critical`：security、authn/authz、財務正確性、資料完整性、破壞性遷移，錯誤代價高且可能不可逆

兩軸各自獨立影響決策：

- 難度低但工作量大（例如大量機械式 README 修改）：difficulty 低 → 不需要高 tier，即使檔案很多。
- 範圍小但風險高（例如 authentication 邏輯的 20 行修改）：difficulty 可能不高，但 risk 高 → 需要更強的驗證深度與必要時更高 reasoning，而不是單純因為「檔案小」就降規格。

Capability tier 的選擇同時參考 difficulty（決定 reasoning 深度）與 risk（決定驗證深度與是否允許 fallback，見第 8、9、14 節）。

## 4. Capability Tiers

Tier 描述**能力角色**（capability role），不是模型名稱。實際 model slug 與其支援的 reasoning effort 一律以 [`docs/model-map.md`](docs/model-map.md) 為準，本檔不得出現任何 model slug。

**Tier A** — 事實蒐集與機械式工作
- repository 搜尋
- 找 symbol / call site
- 事實性盤點（factual inventory）
- 執行 targeted tests
- 重現 bug
- 整理日誌與錯誤訊息
- 機械式修改
- 規格明確的小型變更

**Tier B** — 一般開發
- 一般 coding
- 中小型 bug fix
- 規格明確的功能實作
- 一般 refactor
- test 實作
- 一般 code review

**Tier C** — 較困難的獨立子問題
- 跨模組實作
- 較困難的 debugging
- architecture analysis
- concurrency
- 複雜資料流
- 非平凡重構
- 深度 code review

**Tier D** — 有明確邊界的困難專業子問題
- security-sensitive 的實作或 review
- 資料完整性（data integrity）推理
- 破壞性遷移（destructive migration）推理
- 財務正確性（financial correctness）
- 高錯誤成本的根因分析

**Orchestrator / methodology 角色**：不應被視為「Tier D 的更強 worker」。它的責任與 Tier D 不同：
- 跨系統判斷（cross-system judgment）
- methodology correctness
- 解決多個 subagent 之間互相衝突的結論
- 高風險架構決策
- 當多個高風險領域同時交互時的最終整合

需要這個角色時，是因為需要上述責任，而不是單純因為任務「很難」。

## 5. Reasoning Effort：最低足夠原則

使用最低足夠的 reasoning effort，不要預設 max。粗略參考：

- **Tier A**：`medium` 為常見起點
- **Tier B**：`medium` / `high`
- **Tier C**：`medium` / `high`，必要時升級
- **Tier D**：`high` / `xhigh` / `max`，依錯誤成本與 runtime 實際支援情況決定

只有下列情況才升到最高可用 effort：
- correctness 高風險
- methodology 高風險
- 多次使用較低 effort 仍無法解決
- 明確需要更深推理才能滿足 acceptance criteria

不得因為某個 tier「支援」`max` 就自動使用 `max`。

實際可用的 reasoning effort enum 因 model 而異，以 [`docs/model-map.md`](docs/model-map.md) 為準。

## 6. Context Isolation

跨模型（跨 capability tier）routing 時必須使用 context isolation，因為 full-history fork 會繼承主線的 model 與 reasoning effort，無法可靠地在委派時 override 它們——因此 isolation 是 routing 能成立的前提條件，不只是風格偏好。

- V2：`fork_turns = "none"`
- V1：`fork_context = false`
- 不得同時傳入兩者。
- 不要預設使用 partial fork（`fork_turns` 傳正整數字串）；只有在有明確理由時才使用，且該次委派需說明理由。

因為 subagent 不繼承主線完整對話歷史，**subagent prompt 是唯一可信的任務背景**。

Schema 版本、欄位差異與驗證細節以 [`docs/runtime-schema.md`](docs/runtime-schema.md) 為準。

## 7. Subagent Contract

每個 subagent prompt 必須包含以下七部分：

1. **objective**：只描述一個可獨立完成的成果
2. **necessary context**：必要檔案、符號、錯誤訊息或規格
3. **allowed scope**：允許讀寫的模組或檔案
4. **forbidden changes**：不可變更的介面、行為與相鄰工作
5. **acceptance criteria**：可檢查的完成條件
6. **validation**：指定測試、型別檢查、lint、重現步驟或證據
7. **expected report format**：要求摘要、變更檔案、驗證結果、風險與未解問題

不要把整個 conversation history 當作背景餵給 subagent。不要要求 subagent 自行猜測 adjacent scope——範圍必須在 prompt 中明確寫出。

## 8. Capability Fallback

必要 capability 暫時不可用時，不整體停止工作，改用 fallback：

```
preferred capability unavailable
→ 尋找下一個仍足以滿足 acceptance criteria 的 capability
→ 若主線本身足以安全完成，允許 continue locally
→ 若仍在安全範圍內，繼續執行
→ 只有連最低安全 capability 都不存在時，才 hard stop（見第 9 節）
```

能力是否可用，以實際 spawn 結果為準（runtime 會回報 `Unknown model` 或 `Reasoning effort ... not supported`，並列出可用選項），不得依 README、model 命名慣例或猜測判斷。

規則：
- **禁止 silent downgrade**。
- 每次有意義的 fallback 都要判斷：原目標 tier、實際使用的 tier、acceptance criteria 是否仍可被滿足、risk 是否因降級而變得不可接受。
- 不需要每次都輸出冗長的模型 routing log；但若最終結果發生了有意義的 fallback，需在回報中簡潔說明。

## 9. Hard Stop

Hard stop **不**由「某個固定模型不存在」觸發。

唯一觸發條件：

```
任務屬於高風險（risk = high 或 critical）
AND
目前可用 capability 不足以安全滿足 acceptance criteria
→ hard stop
```

高風險領域包含（僅供判斷 risk，不是「看到關鍵字就停」的觸發清單）：
- security
- authentication / authorization
- destructive migration
- financial correctness
- data integrity
- methodology correctness
- 對使用者資料造成不可逆影響

只要目前可用 capability（含 fallback 後的結果）足以安全滿足該任務的 acceptance criteria，就正常執行，不因為任務屬於上述領域就無條件停止。

## 10. Parallelism 與 Ownership

保留 star topology：

```
main orchestrator
├─ subagent A
├─ subagent B
└─ subagent C
```

不要預設允許 subagent 再無限制地 spawn 自己的 subagent。只把互不依賴的工作平行化。

共用 working tree 時：
- 不允許兩個 agent 同時修改同一檔案或相同 ownership scope。
- scope 有重疊時改為 sequential。
- 確實有價值時才使用 isolated worktree。

並行數以 runtime 實際暴露的 concurrency capability 為準，不要 hard-code 併發上限。

## 11. task_name

只有目前 schema 宣告支援 `task_name` 時才使用（V1 不要傳）。

命名格式：

```
<task>_tier_<tier>_<effort>
```

例如：

- `security_review_tier_d_high`
- `run_tests_tier_a_medium`
- `architecture_analysis_tier_c_high`

只使用小寫字母、數字與底線。**不使用 model slug 命名**（不採 `_sol_max` / `_luna_max` 或 `_<model>_<effort>` 這類寫法），因為那會讓本檔再度依賴具體 model 名稱。實際 model 由 runtime mapping（[`docs/model-map.md`](docs/model-map.md)）決定，`task_name` 只是描述 tier 與 effort 的可見標籤。

## 12. Validation 與整合

任何 subagent 回報「完成」都不是最終驗收。主線依 risk 選擇驗收深度，可包含：

- actual diff
- relevant source
- targeted tests
- type check
- lint
- build
- reproduction evidence
- regression risk

低風險小修改不要過度驗證；高風險修改提高驗證深度。

若多個 agent 結論衝突，以 source、diff、tests、runtime evidence 解決，不以「哪個模型更強」判斷誰對。

## 13. Secret Exposure Gate

Repository task 在準備 push 或建立 PR 前，檢查本次 diff 與新增檔案是否包含：

- API key
- token
- password
- private key
- bearer token
- `.env` secret
- hard-coded credential

不得在輸出中完整顯示疑似 secret。

- 若疑似 secret 尚未 push：停止 push，修正或回報。
- 若疑似 secret 可能已進入 Git history：不得自行 rewrite history 或 force push，回報並等待明確授權。

## 14. Repository Workflow

對 implementation task，若適用且使用者沒有另行限制：

```
analysis
→ implementation
→ targeted validation
→ secret scan
→ commit
→ push
→ PR
→ CI
→ applicable review
→ fixes
→ squash merge
→ sync main
→ branch cleanup
```

這是 completion path，不是機械 checklist；不適用的步驟略過。

使用者明確要求不 commit、不 push、不 PR、不 merge，或 read-only 時，以使用者要求優先。

## 15. Attribution

本 skill 是 fork 自 [irons163/three-tier-agent-orchestrator](https://github.com/irons163/three-tier-agent-orchestrator) 的個人修改版，詳見 [`docs/upstream.md`](docs/upstream.md)。保留 upstream 的 Git history 與作者資訊；不新增 LICENSE；不將 upstream-authored 內容重新宣稱授權。
