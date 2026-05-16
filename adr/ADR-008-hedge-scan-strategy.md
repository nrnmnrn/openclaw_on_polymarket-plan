---
id: ADR-008
title: Hedge Scan 市場選取與分組策略
status: Accepted
date: 2026-05-03
---

## Context

Polyclaw Hedge Scan 原設計為「抓取前 N 個個別熱門市場」，但在高頻掃描（每 30 分鐘一次）下存在三個問題：

1. **重複分析**：同一事件的多個關聯市場可能被各自獨立抓取，導致 LLM 對同一邏輯空間重複推理。
2. **冷門市場 Leg Risk**：單純按市場熱度抓取可能包含 CLOB 深度不足的市場，即使找到套利機會也無法執行。
3. **N×N Token 浪費**：對無結構的市場列表做關聯分析，需要 LLM 進行 O(N²) 的比對，Token 成本難以控制。

需要一個兼顧成本控制、流動性保障、LLM 精準度的掃描架構。

## Decision

採用**漏斗式三原則 + 雙軌掃描**：

1. **Top Volume Event-based 掃描**：呼叫 `GammaClient.get_events(limit=N, order_by=volume24hr)`，以 24 小時成交量最高的 Events（MarketGroup）為掃描單位，而非個別市場。
2. **MarketGroup 原生分組**：同一 Group 內的所有市場（3-5 個）依 `negRisk` 欄位路由至對應掃描軌道。無需 Embedding 或 Slug 比對。
3. **Fail-Fast 模式**：Track 2（LLM 軌道）Coverage < 90% 時立即放棄此 Group，切換下一個，不記錄、不消耗額外資源。

### 雙軌掃描架構

```
Track 1（確定性，零 LLM）：
  negRisk=true → Σ asks < 1 - fees → Multi-Outcome Arbitrage
  
Track 2（LLM，跨事件語意）：
  negRisk=false → IMPLICATION_PROMPT → Coverage 分析 → Fail-Fast < 90%
```

### 選取邏輯偽代碼

```python
top_events = gamma_client.get_events(limit=10, order_by="volume24hr")

for event in top_events:
    if len(event.markets) < 2:
        continue

    # Track 1：確定性 NegRisk 掃描（零 LLM）
    if all(m.negRisk for m in event.markets):
        total_ask = sum(m.best_ask for m in event.markets)
        fees = estimate_fees(event.markets)
        if total_ask < 1.0 - fees:
            execute_multi_outcome_arb(event.markets)
        continue

    # Track 2：LLM 跨事件語意分析
    covers = extract_implications_for_market(event.markets, llm)
    if not covers or max(c["coverage"] for c in covers) < 0.90:
        continue  # Fail-Fast

    execute_or_delegate(covers)  # 依 Tier 路由至 Qwen 或 Claude
```

## Options Considered

| 選項 | 優點 | 缺點 | 決策 |
|------|------|------|------|
| 全量市場列表送 LLM 做 N×N 比對 | 無需前置過濾 | Token 爆炸；LLM 注意力衰退（Lost in the Middle）；延遲 >15s；每月成本遠超 $200 預算 | ❌ 否決 |
| ChromaDB + 0.8 相似度門檻去重 | 能過濾重複市場 | 2C/4G SAS OOM 風險；互斥市場（YES/NO）相似度 ~0.95，會誤刪 Tier 1 機會；語義陷阱 | ❌ 否決 |
| 輕量 Embedding（百鍊 API + Numpy）主題聚類 | 比 ChromaDB 輕量；可找跨事件隱性套利 | 仍需 API 呼叫與本地快取維護；主線場景已由 MarketGroup 覆蓋；增加複雜度 | ❌ 延後（Optional Future） |
| LLM 識別 NegRisk 互斥性（IMPLICATION_PROMPT 判定「A 與 B 互斥」） | 無需額外程式碼 | Gamma API 有 `negRisk` 欄位可直接確定性判定；讓 LLM 做確定性工作是 token 浪費；每 30 分鐘掃描累積成本不可忽視 | ❌ 否決 |
| MarketGroup + Top Volume + Fail-Fast + 雙軌掃描（本方案） | NegRisk Track 1 零 LLM 成本；確定性分組；天然流動性篩選；LLM 預算保留給跨事件語意分析 | Track 2 仍需 LLM；IMPLICATION_PROMPT 職責縮小為跨事件語意分析 | ✅ 採用 |

## Consequences

### 正面

- **NegRisk Track 1 零 LLM 成本**：`negRisk` 欄位直接判定互斥性，Multi-Outcome scan 純數學，不消耗 token
- 每次 LLM 調用（Track 2）僅處理 3-5 個非 NegRisk 市場，精準度高，杜絕 LLM 注意力衰退
- Top Volume 掃描天然滿足 ADR-004 的 Leg Risk 門檻（熱門市場 CLOB 深度充足）
- Fail-Fast 確保每輪掃描的 Token 成本與有效機會數量成正比
- LLM 預算集中用於「必須理解世界常識」的跨事件語意套利，ROI 更高

### 負面／取捨

- IMPLICATION_PROMPT 職責縮小為跨事件語意分析；NegRisk 類型的機會改由確定性邏輯處理
- 跨不同 Group 的「隱性邏輯套利」（例如美元通膨事件 × 聯準會降息事件）仍依賴 LLM，無法完全消除
- 需確認 `GammaClient.get_events()` 的 `order_by=volume24hr` 參數在 Gamma API 中的實際支援狀況（待實作時驗證）

## Revision Log

| 日期 | 做了什麼 | 為什麼 |
|------|---------|--------|
| 2026-05-03 | 初版建立 | 確立 Top Volume Event-based + MarketGroup + Fail-Fast 三原則，取代個別市場抓取與 Embedding 方案 |
| 2026-05-17 | 新增雙軌掃描架構（Track 1 確定性 NegRisk + Track 2 LLM 語意） | NegRisk 互斥性由 Gamma API `negRisk` 欄位直接判定，不再用 LLM；IMPLICATION_PROMPT 職責縮小為跨事件語意分析 |
