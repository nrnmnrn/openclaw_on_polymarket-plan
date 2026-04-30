# ADR-001: 混合模型路由策略

**狀態**：已採納  
**日期**：2026-04-12

---

## 決策

以百鍊 Qwen3.5-Plus 為預設模型執行所有例行掃描與初步判斷；遇到特定高複雜度或高風險情境，自動路由至 Claude Sonnet 4.6 進行二次審核。

---

## 背景

Anthropic 自 2026-04-04 起政策調整，所有 Claude 呼叫必須走獨立 API Key 按量計費，不能使用 Claude Pro/Max 訂閱額度。若全程使用 Claude，每月 API 費用將顯著超過預算。本路由策略適用於 Hermes Agent 框架（見 [ADR-007](ADR-007-agent-framework.md)），邏輯不因框架替換而變動。

---

## 路由規則

| 觸發條件 | 使用模型 |
|----------|---------|
| 例行市場掃描（每 30 分鐘） | Qwen3.5-Plus |
| Hedge Scan Tier 1（≥95% coverage） | Qwen3.5-Plus 直接執行 |
| Hedge Scan Tier 2（90–95% coverage） | Claude Sonnet 4.6 審核後執行 |
| Hedge Scan Tier 3（< 90% coverage） | 略過，不使用任何模型執行 |
| 市場異常狀況（`unusual market`） | Claude Sonnet 4.6 |
| 錯誤恢復（`error recovery`） | Claude Sonnet 4.6 |

**Tier 2 實作**：透過 Hermes 原生 `delegate_task` 工具，設定 `~/.hermes/config.yaml`：

```yaml
delegation:
  model: "anthropic/claude-sonnet-4-6"
  provider: "anthropic"
```

子代理以獨立 context 執行推理，僅回傳結論給主代理（Qwen）。

---

## 被拒絕的替代方案

- **全程 Claude**：成本過高，每月估算超出 $200 預算，且多數掃描為例行工作無需頂級模型。
- **全程 Qwen**：無法在高風險決策時提供足夠的推理品質。

---

## 成本影響

Claude Sonnet 4.6 每月預估 $20-50（僅審核關鍵決策）；Qwen3.5-Plus 每月預估 $10-30（例行掃描）。合計 $30-80，優於全程 Claude 的預估成本。

---

## Revision Log

| 日期 | 做了什麼 | 為什麼 |
|------|---------|--------|
| 2026-04-12 | 初版建立 | Anthropic 政策變更後重新設計成本架構 |
| 2026-04-14 | 路由條件改為 Hedge Scan Tier 1/2/3 | 策略從 yes_no_spread 升級為 hedge_scan，Tier 決定執行路徑 |
| 2026-04-30 | 補充 Tier 2 的 delegate_task 實作細節；移除過時的 OpenClaw claude-code-skill 參照 | 確認 Hermes 等效方案後同步更新 |
