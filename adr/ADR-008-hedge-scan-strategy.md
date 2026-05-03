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

採用**漏斗式三原則**：

1. **Top Volume Event-based 掃描**：呼叫 `GammaClient.get_events(limit=N, order_by=volume24hr)`，以 24 小時成交量最高的 Events（MarketGroup）為掃描單位，而非個別市場。
2. **MarketGroup 原生分組**：同一 Group 內的所有市場（3-5 個）直接打包送入 `IMPLICATION_PROMPT` 進行 LLM 邏輯分析。無需 Embedding 或 Slug 比對。
3. **Fail-Fast 模式**：若 LLM 分析結果最高 Coverage < 90%（無 Tier 1 或 Tier 2），立即放棄此 Group 並切換下一個，不記錄、不消耗額外資源。

### 選取邏輯偽代碼

```python
top_events = gamma_client.get_events(limit=10, order_by="volume24hr")

for event in top_events:
    if len(event.markets) < 2:
        continue  # 單一市場無法對沖

    covers = extract_implications_for_market(event.markets, llm)

    if not covers or max(c["coverage"] for c in covers) < 0.90:
        continue  # Fail-Fast：無 Tier 1/2，切換下一個 Group

    execute_or_delegate(covers)  # 依 Tier 路由至 Qwen 或 Claude
```

## Options Considered

| 選項 | 優點 | 缺點 | 決策 |
|------|------|------|------|
| 全量市場列表送 LLM 做 N×N 比對 | 無需前置過濾 | Token 爆炸；LLM 注意力衰退（Lost in the Middle）；延遲 >15s；每月成本遠超 $200 預算 | ❌ 否決 |
| ChromaDB + 0.8 相似度門檻去重 | 能過濾重複市場 | 2C/4G SAS OOM 風險；互斥市場（YES/NO）相似度 ~0.95，會誤刪 Tier 1 機會；語義陷阱 | ❌ 否決 |
| 輕量 Embedding（百鍊 API + Numpy）主題聚類 | 比 ChromaDB 輕量；可找跨事件隱性套利 | 仍需 API 呼叫與本地快取維護；主線場景已由 MarketGroup 覆蓋；增加複雜度 | ❌ 延後（Optional Future） |
| MarketGroup + Top Volume + Fail-Fast（本方案） | 零額外 API 成本；確定性分組；天然流動性篩選；Token 成本可控 | 跨 Group 的隱性邏輯套利無法被發現 | ✅ 採用 |

## Consequences

### 正面

- 每次 LLM 調用僅處理 3-5 個市場，精準度高，杜絕 LLM 注意力衰退
- Top Volume 掃描天然滿足 ADR-004 的 Leg Risk 門檻（熱門市場 CLOB 深度充足）
- Fail-Fast 確保每輪掃描的 Token 成本與有效機會數量成正比
- 與 NegRisk 架構完全相容：NegRisk 互斥市場必定在同一 MarketGroup

### 負面／取捨

- 跨不同 Group 的「隱性邏輯套利」（例如美元通膨事件 × 聯準會降息事件）無法被發現
- 需確認 `GammaClient.get_events()` 的 `order_by=volume24hr` 參數在 Gamma API 中的實際支援狀況（待實作時驗證）

## Revision Log

| 日期 | 做了什麼 | 為什麼 |
|------|---------|--------|
| 2026-05-03 | 初版建立 | 確立 Top Volume Event-based + MarketGroup + Fail-Fast 三原則，取代個別市場抓取與 Embedding 方案 |
