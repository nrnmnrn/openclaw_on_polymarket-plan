---
id: ADR-009
title: Telegram 通知整合策略
status: Accepted
date: 2026-05-03
---

## Context

Paper Trading 階段需要在不持續盯著終端機的情況下，即時掌握：
- 模擬交易是否被觸發（驗證 Tier 判定與 Kelly 計算邏輯）
- 每日 P&L 是否符合進入實盤的量化條件（ADR-004）
- hermes-attestation-guardian 偵測到環境異常時立即收到警報

Hermes Agent 原生支援 Telegram 平台（`gateway/platforms/telegram.py`），提供兩種通知機制：
1. **Auto-delivery**（`--deliver telegram`）：Hermes scheduler 自動將 agent 的 `send_message` 工具呼叫推送到 Telegram，含重複目標偵測（相同 target 的 cron 重複發送會自動跳過）
2. **獨立腳本**（`--no-agent` script-only cron）：零 LLM 成本的確定性腳本，直接呼叫 Telegram Bot API

## Decision

Hermes 以 **gateway 模式**運行，透過 `sudo hermes gateway install --system` 安裝為 Linux 開機自啟的 system service（非 user service），啟用 Telegram adapter。

配置方式：在 `~/.hermes/config.yaml` 啟用 Telegram 平台，憑證透過 `secrets.env` 的 `TELEGRAM_BOT_TOKEN` 與 `TELEGRAM_CHAT_ID` 注入（符合 ADR-003 安全管理規範）。

三類通知觸發事件：

| 觸發事件 | 通知內容 | 實作方式 |
|---------|---------|---------|
| 虛擬交易成交（Paper Trading） | 市場名稱、倉位（YES/NO）、模擬金額、Quarter-Kelly 值 | Hermes agent 呼叫 `send_message` 工具 → auto-delivery（`--deliver telegram`）推送，含重複目標偵測 |
| 每日結算摘要（UTC 00:00） | 當日 P&L、勝率統計、bankroll 餘額 | 確定性 Python 腳本（`--no-agent` script-only cron，零 LLM 成本）→ 直接呼叫 Telegram Bot API |
| 風控警報（即時） | drift 偵測異常 / 單日虧損上限觸發 / API 調用失敗 | hermes-attestation-guardian 以 `--no-agent` 模式執行 script-only cron，偵測異常時直接呼叫 Telegram Bot API；agent 觸發的風控警報（如 `unusual market`）走 auto-delivery |

純 LLM 任務若不需通知（如內部 decision logging），使用 `[SILENT]` tag 抑制 delivery。

每日結算摘要與 Guardian 警報採 `--no-agent` 腳本而非 LLM 生成，節省 token 成本並確保格式穩定。

## Options Considered

| 選項 | 優點 | 缺點 |
|------|------|------|
| Auto-delivery + no-agent 混用（採用）| Trade 通知走 agent auto-delivery；結算/Guardian 走 `--no-agent` 零 LLM 成本；重複目標自動去重 | 需理解兩種 delivery 機制的分工 |
| 全部走 `send_message` + gateway（排除）| 單一機制，概念簡單 | 每次 `send_message` 都消耗 LLM token；純腳本任務不適合用 agent |
| 獨立 Python notify.py 腳本（排除）| 輕量，10 行實作，不依賴 Hermes gateway | 與 Hermes agentic 工作流解耦；需另外維護腳本；不符合框架原生設計 |
| 維持驗證期不通知（原 YAGNI，排除）| 降低初期複雜度 | 無法在不盯終端機的情況下驗證邏輯正確性；難以追蹤進入實盤的量化條件進度 |

## Consequences

### 正面
- Paper Trading 期間可即時收到模擬交易通知，驗證 Tier 判定與 Kelly 計算是否正確
- Auto-delivery 內建重複偵測，同一 target 的重复發送自動跳過（`cron_auto_delivery_duplicate_target`）
- 每日結算摘要與 Guardian 警報採 `--no-agent` 模式，零 LLM 成本
- `[SILENT]` tag 可精確控制哪些 LLM 任務不需通知
- 每日結算摘要便於追蹤是否達到 ADR-004 進入實盤條件（勝率 > 55%、每筆利潤 > 手續費 2 倍）
- Gateway 模式同時允許用戶透過 Telegram 對 bot 下指令（雙向互動）

### 負面 / 取捨
- `sudo hermes gateway install --system` 需 sudo 權限，system service 管理比 user service 稍複雜
- 需理解 auto-delivery 與 `--no-agent` 腳本的分工，避免 `send_message` 重複觸發被去重機制過濾

## Revision Log

| 日期 | 做了什麼 | 為什麼 |
|------|---------|--------|
| 2026-05-03 | 初版建立 | Paper Trading 驗證期導入 Telegram 通知；推翻原 YAGNI（驗證期不需通知）決定 |
| 2026-05-10 | 釐清通知機制分工 | Trade 通知改用 auto-delivery（`--deliver telegram`）；Guardian 改用 `--no-agent` 模式；gateway 安裝確認用 `sudo hermes gateway install --system`；新增 `[SILENT]` tag 用法 |
