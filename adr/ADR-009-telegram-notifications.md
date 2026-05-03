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

Hermes Agent 原生支援 Telegram 平台（`gateway/platforms/telegram.py`），透過 `send_message` 工具跨平台發送訊息。但 `send_message` 工具有 gate 限制：在純 CLI/cron 模式（gateway 未啟動）下不可用。

## Decision

Hermes 以 **gateway 模式**運行，啟用 Telegram adapter，使 `send_message` 工具在 cron 任務與 agent 執行期間原生可用。

配置方式：在 `~/.hermes/config.yaml` 啟用 Telegram 平台，憑證透過 `secrets.env` 的 `TELEGRAM_BOT_TOKEN` 與 `TELEGRAM_CHAT_ID` 注入（符合 ADR-003 安全管理規範）。

三類通知觸發事件：

| 觸發事件 | 通知內容 | 實作方式 |
|---------|---------|---------|
| 虛擬交易成交（Paper Trading） | 市場名稱、倉位（YES/NO）、模擬金額、Quarter-Kelly 值 | Hermes agent 呼叫 `send_message` 工具 |
| 每日結算摘要（UTC 00:00） | 當日 P&L、勝率統計、bankroll 餘額 | 確定性 Python 腳本讀 `logs/settle_check.jsonl` → Telegram Bot API |
| 風控警報（即時） | drift 偵測異常 / 單日虧損上限觸發 / API 調用失敗 | Hermes agent 呼叫 `send_message` 工具 |

每日結算摘要採確定性 Python 腳本而非 LLM 生成，節省 token 成本並確保格式穩定。

## Options Considered

| 選項 | 優點 | 缺點 |
|------|------|------|
| Hermes gateway 模式（採用）| `send_message` 工具原生可用；用戶可透過 Telegram 對 bot 下指令；符合 Hermes agentic 工作流設計 | 需要 gateway 常駐運行；systemd 配置需更新；啟動指令待確認 |
| 獨立 Python notify.py 腳本（排除）| 輕量，10 行實作，不依賴 Hermes gateway | 與 Hermes agentic 工作流解耦；需另外維護腳本；不符合框架原生設計 |
| 維持驗證期不通知（原 YAGNI，排除）| 降低初期複雜度 | 無法在不盯終端機的情況下驗證邏輯正確性；難以追蹤進入實盤的量化條件進度 |

## Consequences

### 正面
- Paper Trading 期間可即時收到模擬交易通知，驗證 Tier 判定與 Kelly 計算是否正確
- 每日結算摘要便於追蹤是否達到 ADR-004 進入實盤條件（勝率 > 55%、每筆利潤 > 手續費 2 倍）
- 風控警報確保環境異常（drift、虧損上限）第一時間被察覺
- Gateway 模式同時允許用戶透過 Telegram 對 bot 下指令（雙向互動）

### 負面 / 取捨
- Hermes 需以 gateway 模式運行，systemd unit 配置較純 CLI 模式複雜
- gateway 模式具體啟動指令尚待確認（見 CURRENT_SPEC.md 待解決事項）
- Telegram API 免費，但 agent 呼叫 `send_message` 工具會略微增加 LLM 輸出 token 消耗；已透過確定性 Python 腳本處理每日結算摘要來緩解

## Revision Log

| 日期 | 做了什麼 | 為什麼 |
|------|---------|--------|
| 2026-05-03 | 初版建立 | Paper Trading 驗證期導入 Telegram 通知；推翻原 YAGNI（驗證期不需通知）決定 |
