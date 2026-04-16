---
topic: polyclaw-hedge-scan-strategy
queried: 2026-04-16
expires: 2026-10-16
status: 有效
sources:
  - https://docs.chainstack.com/docs/polygon-creating-a-polymarket-trading-openclaw-skill
  - https://github.com/chainstacklabs/polyclaw
---

# Polyclaw hedge_scan 策略說明

## 核心機制：Split + CLOB

交易流程：
1. Split：將 USDC.e 透過 CTF 合約拆分為 YES + NO token
2. Sell unwanted：在 CLOB 訂單簿賣出不需要的那一側
3. 結果：持有目標側倉位，透過賣出另一側回收部分成本

範例：買入 YES at $0.70
- Split $100 USDC.e → 100 YES + 100 NO token
- 賣出 100 NO at ~$0.30 → 回收 ~$27 USDC.e
- 淨成本：~$73（有效進入價格 $0.73）

## hedge_scan 是語義套利，不是純數學套利

Hedge module 使用 **LLM 分析** 尋找「covering portfolio」：

- 透過反推邏輯（contrapositive）找邏輯相關的市場對
- 例：「X 贏得選舉」⟹「Y 輸掉選舉」→ YES(X) + NO(Y) 構成對沖組合
- Coverage % 衡量兩市場邏輯關聯強度（語義判斷），非價格總和計算

Coverage Tier（Polyclaw 官方定義）：
| Tier | Coverage | 說明 |
|------|----------|------|
| 1 (HIGH) | ≥95% | 接近套利機會 |
| 2 (GOOD) | 90-95% | 強對沖 |
| 3 (MODERATE) | 85-90% | 可接受風險 |
| 4 (LOW) | <85% | 投機性（預設過濾） |

預設 LLM：`nvidia/nemotron-nano-9b-v2:free`（via OpenRouter）

## 關鍵含義

- 套利窗口期：**數小時至數天**（語義關係不會在秒級消失）
- Coverage 計算：LLM 做語義判斷是**正確的工具選擇**
- 流動性風險：賣出 unwanted 側時依賴 CLOB 訂單簿深度，流動性不足會造成滑價
