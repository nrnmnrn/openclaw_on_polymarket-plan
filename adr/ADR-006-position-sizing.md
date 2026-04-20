---
id: ADR-006
title: 倉位大小策略（Quarter-Kelly Python 計算）
status: Accepted
date: 2026-04-20
---

## Context

Polyclaw hedge_scan 的 Tier 機制提供了市場邏輯確定性評分（Tier 1 ≥95%，Tier 2 90-95%），但沒有配套的倉位大小計算。若每筆固定押注 $10 上限，在低勝率市場仍會過度暴險；若依 Tier 線性縮放，缺乏理論依據。需要一個有數學基礎的動態倉位方法，同時保持個人維運可控。

## Decision

採用 Quarter-Kelly（Kelly Criterion × 0.25），由 polyclaw 內建 Python 確定性計算，不透過 LLM 計算。paper trading 起即生效。

Kelly 公式：`f* = (b·p - q) / b`，其中：

- `b = (1 - C) / C`（C 為入場成本比例）
- `p` 依 Tier 映射：Tier 1 → 0.95-0.98，Tier 2 → 0.90-0.95
- `q = 1 - p`

實際下注比例：`position = f* × 0.25`（Quarter-Kelly 降低波動）

邊界條件：

- `MIN_EDGE = 2-5%`：Kelly 值低於此閾值時不執行交易
- `MAX_FRACTION = 0.25`：Quarter-Kelly 上限，防止高 Tier 市場過度集中

## Options Considered

| 選項 | 優點 | 缺點 |
|------|------|------|
| 固定倉位（$10 上限）| 簡單、無需計算 | 忽略機會品質差異，Tier 2 機會也全倉；Kelly 值低時過度暴險 |
| LLM 計算 Kelly | 無需額外程式碼 | LLM 數值計算不可靠，可能輸出錯誤公式結果；確定性財務計算不應依賴 LLM |
| Full Kelly | 長期期望值最大化 | 波動極大，連續虧損時本金快速侵蝕；個人小本金不適合 |
| Quarter-Kelly Python（本決策）| 數學嚴謹；確定性；波動比 Full Kelly 低 75% | 需在 polyclaw 中實作計算函式 |

## Consequences

### 正面

- 倉位與機會品質正相關（Tier 1 > Tier 2）
- Python 確定性計算，結果可重現、可測試
- Quarter-Kelly 在小本金下提供合理的風險/報酬平衡

### 負面／取捨

- 需在 polyclaw 中實作 Kelly 計算函式（實作成本約 50-100 行 Python）
- Tier 到 p 值的映射（0.95-0.98）為估算值，實際勝率需 paper trading 驗證後校正
- MIN_EDGE 設定過高（>5%）可能錯失有效機會；設定過低（<2%）可能追逐雜訊

## Revision Log

| 日期 | 做了什麼 | 為什麼 |
|------|---------|--------|
| 2026-04-20 | 初版建立 | 確立 Quarter-Kelly Python 為倉位計算標準，取代固定倉位與 LLM 計算方案 |
