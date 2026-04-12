# ADR-001: 混合模型路由策略

**狀態**：已採納  
**日期**：2026-04-12

---

## 決策

以百鍊 Qwen3.5-Plus 為預設模型執行所有例行掃描與初步判斷；遇到特定高複雜度或高風險情境，自動路由至 Claude Sonnet 4.6 進行二次審核。

---

## 背景

Anthropic 自 2026-04-04 起政策調整，OpenClaw 不能再使用 Claude Pro/Max 訂閱額度，所有 Claude 呼叫必須走獨立 API Key 按量計費。若全程使用 Claude，每月 API 費用將顯著超過預算。

---

## 路由規則

| 觸發條件 | 使用模型 |
|----------|---------|
| 例行市場掃描（每小時） | Qwen3.5-Plus |
| 初步套利評估 | Qwen3.5-Plus |
| 包含 `hedge` 操作 | Claude Sonnet 4.6 |
| 套利金額 > $10 | Claude Sonnet 4.6 |
| 市場異常狀況（`unusual market`） | Claude Sonnet 4.6 |
| 錯誤恢復（`error recovery`） | Claude Sonnet 4.6 |

---

## 被拒絕的替代方案

- **全程 Claude**：成本過高，每月估算超出 $200 預算，且多數掃描為例行工作無需頂級模型。
- **全程 Qwen**：無法在高風險決策時提供足夠的推理品質，且無法利用 Claude 的 Claude Code Skill 能力。

---

## 成本影響

Claude Sonnet 4.6 每月預估 $20-50（僅審核關鍵決策）；Qwen3.5-Plus 每月預估 $10-30（例行掃描）。合計 $30-80，優於全程 Claude 的預估成本。

---

## Revision Log

| 日期 | 做了什麼 | 為什麼 |
|------|---------|--------|
| 2026-04-12 | 初版建立 | Anthropic 政策變更後重新設計成本架構 |
