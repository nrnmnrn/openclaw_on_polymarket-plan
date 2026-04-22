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

### Kelly 公式

```
f* = (b·p - q) / b

b = m / C                    # 淨報酬率
m = 1.0 - C - fees           # 預期利潤（hedge 成功時回收 ≈ $1）
C = portfolio["total_cost"]  # 組合入場成本
p = portfolio["coverage"]    # 直接使用 coverage 作為成功機率
q = 1 - p
fees = C × 1%                # Polymarket taker fee 估算

position_usd = f* × 0.25 × bankroll
```

### 參數來源

| 參數 | 來源 | 說明 |
|------|------|------|
| `cost` | `hedge scan` 輸出 | 每筆 portfolio 不同 |
| `coverage` | `hedge scan` 輸出 | 直接作為 p，無需 Tier→p 映射 |
| `bankroll` | `state/bankroll.json`（paper）<br>`wallet_manager`（live）| 動態讀取，隨資金成長調整 |

Paper trading 的 bankroll 狀態管理（入場扣款、結算更新、持倉追蹤）詳見 [Bankroll Update Mechanism Design](../docs/specs/2026-04-21-bankroll-update-mechanism-design.md)。

### 靜態常數（`lib/kelly.py`）

```python
KELLY_FRACTION = 0.25      # Quarter-Kelly
MIN_EDGE_PCT = 5.0         # 低於此不執行
MAX_POSITION_USD = 10.0    # 單筆上限（固定金額）
FEES_PCT = 1.0             # Polymarket taker fee
```

### 決策邏輯

| 條件 | 決策 | 原因 |
|------|------|------|
| `f* ≤ 0` | SKIP | 無正期望值 |
| `edge < 5%` | SKIP | 利潤太薄 |
| `position < $1` | SKIP | 不值得 gas |
| `position > $10` | CAP at $10 | 單筆上限 |
| 其他 | EXECUTE | — |

### 整合方式

Kelly 計算整合於 `hedge scan` 輸出，不另設獨立 CLI。LLM 執行一次 `hedge scan` 即可取得完整決策建議（含 `f*`、`position_usd`、`decision`）。

Audit log 寫入 `logs/kelly_audit.jsonl`，不定期清理。

## Options Considered

### 倉位計算方法（2026-04-20）

| 選項 | 優點 | 缺點 | 決策 |
|------|------|------|------|
| 固定倉位（$10 上限）| 簡單、無需計算 | 忽略機會品質差異，Tier 2 機會也全倉；Kelly 值低時過度暴險 | ❌ 棄用 |
| LLM 計算 Kelly | 無需額外程式碼 | LLM 數值計算不可靠，可能輸出錯誤公式結果；確定性財務計算不應依賴 LLM | ❌ 棄用 |
| Full Kelly | 長期期望值最大化 | 波動極大，連續虧損時本金快速侵蝕；個人小本金不適合 | ❌ 棄用 |
| Quarter-Kelly Python | 數學嚴謹；確定性；波動比 Full Kelly 低 75% | 需在 polyclaw 中實作計算函式 | ✅ 採用 |

### 介面與整合設計（2026-04-21）

| 選項 | 優點 | 缺點 | 決策 |
|------|------|------|------|
| 獨立 `kelly-calc` CLI | 職責分離，可單獨測試 | 增加元件數量；LLM 需多一輪工具調用；hedge scan 已有所需資料 | ❌ 棄用 |
| `kelly_config.yaml` 設定檔 | 參數集中管理 | 僅 ~5 個參數，YAML 過度工程；修改頻率極低，Python 常數即可 | ❌ 棄用 |
| `--override-p` flag | LLM 可覆寫成功機率 | Paper trading 應驗證 coverage 準確度，而非繞過；增加 code path 複雜度 | ❌ 棄用 |
| `--override-k` flag | 可動態調整 kelly_fraction | 驗證期應固定參數；若需調整，改常數重啟即可 | ❌ 棄用 |
| Tier → p 映射表 | Tier 標籤轉換為概率 | coverage 已是計算後的概率，無需反向映射；增加維護負擔 | ❌ 棄用 |
| `MAX_POSITION` 為本金百分比 | 資金成長後自動放寬單筆上限 | Paper trading 階段風控優先；驗證策略後可再考慮 | ❌ 延後（未來選項）|
| 整合於 `hedge scan` 輸出 | 減少元件；一次調用取得完整決策 | Kelly 邏輯耦合於 hedge scan | ✅ 採用 |

## Consequences

### 正面

- 倉位與機會品質正相關（高 coverage → 高倉位）
- Python 確定性計算，結果可重現、可測試
- Quarter-Kelly 在小本金下提供合理的風險/報酬平衡
- 整合於 hedge scan 減少元件數量，符合個人維運原則
- bankroll 動態讀取，資金成長後自動調整倉位

### 負面／取捨

- 需在 polyclaw 中實作 Kelly 計算函式（實作成本約 30 行 Python）
- coverage 作為 p 值的準確度需 paper trading 驗證後校正
- MIN_EDGE = 5% 可能錯失部分有效機會；需根據實際結果調整

## Revision Log

| 日期 | 做了什麼 | 為什麼 |
|------|---------|--------|
| 2026-04-20 | 初版建立 | 確立 Quarter-Kelly Python 為倉位計算標準，取代固定倉位與 LLM 計算方案 |
| 2026-04-21 | 精簡設計：移除獨立 CLI、YAML config、Tier→p 映射、override flags；改為整合於 hedge scan 輸出；bankroll 動態讀取 | 減少元件數量，符合個人維運原則；bankroll 動態化確保資金成長後自動調整倉位 |
| 2026-04-21 | 新增 bankroll 狀態管理機制交叉引用 | 連結至詳細 spec，避免文件孤立 |
