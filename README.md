# Adaptive Agent Orchestrator

> 本 README 以中文為主，英文版請見文末。

本專案是 [irons163/three-tier-agent-orchestrator](https://github.com/irons163/three-tier-agent-orchestrator) 的個人 fork / 修改版，細節見 [`docs/upstream.md`](docs/upstream.md)。本 fork 保留 upstream 的 Git history 與作者資訊；upstream 於目前驗證時間點沒有明示的 LICENSE，本 fork 不宣稱重新授權 upstream-authored 內容。

一個 Codex skill，採 local-first、最低足夠能力（lowest sufficient capability）、risk-aware reasoning、選擇性委派（selective delegation）與 capability fallback，協調 coding、code review、debugging、測試與 repository workflow。

## 核心模式

Tier 描述**能力角色**，不是固定模型分工。實際可用模型與 reasoning 以 [`docs/model-map.md`](docs/model-map.md) 為準——該檔是特定環境的驗證 snapshot，不是永久產品規格。

| Tier | 能力角色 |
| --- | --- |
| Tier 0 | 目前主線直接完成，是預設路徑 |
| Tier A | 事實蒐集與機械式工作：repository 搜尋、找 symbol/call site、targeted tests、重現 bug、整理日誌、機械式修改 |
| Tier B | 一般開發：一般 coding、中小型 bug fix、規格明確的功能、一般 refactor、test 實作、一般 review |
| Tier C | 較困難的獨立子問題：跨模組實作、較困難 debugging、architecture analysis、concurrency、非平凡重構、深度 review |
| Tier D | 有明確邊界的困難專業子問題：security-sensitive 實作/review、資料完整性推理、破壞性遷移推理、財務正確性、高錯誤成本的根因分析 |
| Orchestrator / methodology 角色 | 跨系統判斷、methodology correctness、解決多個 subagent 結論衝突、高風險架構決策、多個高風險領域交互時的最終整合 |

Orchestrator / methodology 角色**不是** Tier D 的「更強 worker」——它承擔的是上述判斷責任，不是單純因為任務比較難。

本檔不寫任何實際 model slug；完整定義見 [`SKILL.md`](SKILL.md)。

## 委派前先判斷：Delegation Gate

不是每個任務都要啟動 subagent。只有以下至少一項有明確價值時才委派：

- **Parallelism**：這個子任務能否真正與其他工作獨立並行？
- **Independent verification**：是否需要一個不受主線既有推論影響的獨立判斷？
- **Capability advantage**：是否有明顯更適合這個子任務的能力層級？

三者皆不成立時，主線直接完成（continue locally）。context isolation（避免主線 context 被污染）可以是輔助理由，但**不能單獨成為 spawn subagent 的理由**。

## Difficulty × Risk

不是用單一「任務難度」決定要用多強的能力，而是分開判斷兩個軸：

- **Difficulty**：`routine` / `standard` / `complex` / `expert`
- **Risk**：`low` / `medium` / `high` / `critical`

工作量大不等於推理難度高；改動範圍小不等於風險低。例如：

- 大量 README 機械修改：工作量大，但 difficulty 低。
- authentication 的少量程式修改：範圍小，但 risk 高。

## Reasoning 策略

使用最低足夠的 reasoning effort，不預設 `max`。只有 correctness 風險、methodology 風險、較低 effort 多次無法解決，或明確需要更深推理時，才升級。實際可用的 reasoning effort 依 model 而異，見 [`docs/model-map.md`](docs/model-map.md)。

## Capability Fallback

必要能力暫時不可用時，不整體停止工作：

```
preferred capability unavailable
→ 找下一個仍足以滿足 acceptance criteria 的能力
→ 若主線本身足夠，continue locally
→ 只有高風險任務且最低安全能力不存在時，才 hard stop
```

禁止 silent downgrade。不再有「缺少 Sol / Luna / Astra 中任一固定模型 → 整個工作流全部停止」這種規則。

## Context Isolation

跨能力層級（跨模型）routing 時必須使用 isolated subagent context，避免 full-history fork 強制繼承主線的 model 與 reasoning effort。完整 schema 規格見 [`docs/runtime-schema.md`](docs/runtime-schema.md)；本 README 不重複列出 V1/V2 欄位細節。

## 驗收

subagent 回報「完成」不等於最終核准。主線依風險選擇驗收深度，可包含 diff、相關原始碼、targeted tests、lint、型別檢查、build、重現證據與 regression risk。低風險不過度驗證，高風險提高驗證深度。

## Secret Exposure Gate

Repository task 在 push / PR 前檢查 diff 與新增檔案是否包含 API key、token、password、private key、bearer token、`.env` secret 或 hard-coded credential。若疑似 secret 可能已進入 Git history，不自行 rewrite history 或 force push，先回報並等待明確授權。

## 必要條件

- Codex 支援 skills，且若要使用 delegation，runtime 需支援相應的 subagent capability。
- 可用的 model、reasoning effort 與 multi-agent schema 依 runtime 而異。
- 目前驗證 snapshot 見 [`docs/model-map.md`](docs/model-map.md)（**特定環境的快照，不是永久產品規格**）。
- Schema 行為見 [`docs/runtime-schema.md`](docs/runtime-schema.md)。

不需要建立額外的 custom agent 設定檔。

## 安裝

### Windows (PowerShell)

```powershell
$CodexHome = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $HOME ".codex" }
$SkillPath = Join-Path $CodexHome "skills\adaptive-agent-orchestrator"

if (Test-Path $SkillPath) {
    Write-Host "Skill path already exists: $SkillPath"
    Write-Host "Please check the existing skill before overwriting it."
} else {
    git clone https://github.com/nanachi1212/adaptive-agent-orchestrator.git $SkillPath
}
```

### macOS / Linux

```bash
git clone https://github.com/nanachi1212/adaptive-agent-orchestrator.git "${CODEX_HOME:-$HOME/.codex}/skills/adaptive-agent-orchestrator"
```

若目前 task 沒有重新載入 skill，請建立新 task；仍未出現時再重新啟動 Codex App。

## 使用

在 Codex prompt 明確啟用：

```text
$adaptive-agent-orchestrator
```

也可以直接描述工作，例如：

```text
使用 Adaptive Agent Orchestrator 處理這個 repository task：
小型工作留在主線完成，只有真正有價值時才委派，
使用最低足夠的能力，只有風險或複雜度足夠高時才升級 reasoning。
```

## Repository 結構

```text
.
├── README.md
├── SKILL.md
├── agents/
│   └── openai.yaml
└── docs/
    ├── model-map.md
    ├── runtime-schema.md
    └── upstream.md
```

不需要額外的 custom agent 設定檔。

---

## English Version

### Adaptive Agent Orchestrator

This project is a personal fork / modified version of [irons163/three-tier-agent-orchestrator](https://github.com/irons163/three-tier-agent-orchestrator); see [`docs/upstream.md`](docs/upstream.md) for details. This fork keeps upstream's Git history and authorship; upstream had no explicit LICENSE observed at verification time, and this fork does not claim to relicense upstream-authored content.

A Codex skill that coordinates coding, code review, debugging, testing, and repository workflow using local-first execution, lowest sufficient capability, risk-aware reasoning, selective delegation, and capability fallback.

### Core Model

Tiers describe **capability roles**, not fixed model assignments. The actual available models and reasoning efforts are defined in [`docs/model-map.md`](docs/model-map.md) — an environment-specific verification snapshot, not a permanent product spec.

| Tier | Capability role |
| --- | --- |
| Tier 0 | The current main thread completes the work directly; this is the default path |
| Tier A | Factual and mechanical work: repository search, symbol/call-site discovery, targeted tests, bug reproduction, log organization, mechanical edits |
| Tier B | Normal development: normal coding, medium-small bug fixes, clearly specified features, ordinary refactors, test implementation, normal review |
| Tier C | Difficult bounded subproblems: cross-module implementation, difficult debugging, architecture analysis, concurrency, non-trivial refactors, deep review |
| Tier D | Clearly bounded, difficult expert subproblems: security-sensitive implementation/review, data-integrity reasoning, destructive-migration reasoning, financial correctness, high-cost failure analysis |
| Orchestrator / methodology role | Cross-system judgment, methodology correctness, resolving conflicting subagent conclusions, high-risk architecture decisions, final integration when multiple high-risk domains interact |

The Orchestrator / methodology role is **not** "a stronger Tier D worker" — it exists for the judgment responsibilities above, not simply because a task is harder.

No actual model slug is written in this document; see [`SKILL.md`](SKILL.md) for the full definitions.

### Delegation Gate

Not every task should spawn a subagent. Delegate only when at least one of the following has clear value:

- **Parallelism** — can this subtask genuinely run independently alongside other work?
- **Independent verification** — is an unbiased second opinion needed, uninfluenced by the main thread's existing reasoning?
- **Capability advantage** — is there a capability tier clearly better suited to this subtask?

If none apply, continue locally. Context isolation (keeping the main thread's context clean) can be a supporting reason, but it **cannot alone justify** spawning a subagent.

### Difficulty × Risk

Capability selection is not driven by a single "task difficulty" score. Two independent axes are tracked:

- **Difficulty**: `routine` / `standard` / `complex` / `expert`
- **Risk**: `low` / `medium` / `high` / `critical`

A large amount of work does not mean high reasoning difficulty; a small change does not mean low risk. For example:

- Mechanically editing many lines in a README: large workload, but low difficulty.
- A small change to authentication logic: small scope, but high risk.

### Reasoning Policy

Use the lowest sufficient reasoning effort; do not default to `max`. Escalate only for correctness risk, methodology risk, repeated failure at lower effort, or a clear need for deeper reasoning. Actual reasoning-effort availability varies by model; see [`docs/model-map.md`](docs/model-map.md).

### Capability Fallback

When a preferred capability is temporarily unavailable, the workflow does not stop entirely:

```
preferred capability unavailable
→ find the next capability that still satisfies the acceptance criteria
→ if the main thread itself is sufficient, continue locally
→ hard stop only when the task is high-risk and no minimum safe capability is available
```

Silent downgrades are prohibited. There is no rule that says "if Sol, Luna, or Astra specifically is unavailable, stop the entire workflow."

### Context Isolation

Cross-tier (cross-model) routing requires an isolated subagent context, to avoid a full-history fork forcing inheritance of the main thread's model and reasoning effort. Full schema details are in [`docs/runtime-schema.md`](docs/runtime-schema.md); this README does not duplicate the V1/V2 field-level specification.

### Validation

A subagent reporting "done" is not final approval. The main thread chooses validation depth based on risk, which can include diffs, relevant source, targeted tests, lint, type checks, builds, reproduction evidence, and regression risk. Low-risk changes are not over-validated; high-risk changes get deeper validation.

### Secret Exposure Gate

Before pushing or opening a PR for a repository task, check the diff and any new files for API keys, tokens, passwords, private keys, bearer tokens, `.env` secrets, or hard-coded credentials. If a suspected secret may already be in Git history, do not rewrite history or force-push; report it and wait for explicit authorization.

### Requirements

- Codex supports skills, and the runtime must support the relevant subagent capability if delegation is used.
- Available models, reasoning efforts, and the multi-agent schema vary by runtime.
- The current verification snapshot is in [`docs/model-map.md`](docs/model-map.md) (**an environment-specific snapshot, not a permanent product spec**).
- Schema behavior is documented in [`docs/runtime-schema.md`](docs/runtime-schema.md).

No additional custom agent configuration files are required.

### Installation

#### Windows (PowerShell)

```powershell
$CodexHome = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $HOME ".codex" }
$SkillPath = Join-Path $CodexHome "skills\adaptive-agent-orchestrator"

if (Test-Path $SkillPath) {
    Write-Host "Skill path already exists: $SkillPath"
    Write-Host "Please check the existing skill before overwriting it."
} else {
    git clone https://github.com/nanachi1212/adaptive-agent-orchestrator.git $SkillPath
}
```

#### macOS / Linux

```bash
git clone https://github.com/nanachi1212/adaptive-agent-orchestrator.git "${CODEX_HOME:-$HOME/.codex}/skills/adaptive-agent-orchestrator"
```

If the current task does not reload the skill, create a new task. If it still does not appear, restart the Codex App.

### Usage

Explicitly enable the skill in a Codex prompt:

```text
$adaptive-agent-orchestrator
```

You can also describe the work directly, for example:

```text
Use Adaptive Agent Orchestrator for this repository task.
Keep simple work local, delegate only when useful,
use the lowest sufficient capability,
and escalate reasoning only when risk or complexity justifies it.
```

### Repository Structure

```text
.
├── README.md
├── SKILL.md
├── agents/
│   └── openai.yaml
└── docs/
    ├── model-map.md
    ├── runtime-schema.md
    └── upstream.md
```

No additional custom agent configuration files are required.
