# ADR-005：Composio 作為工具整合中樞

**日期**：2026-04-12  
**狀態**：已採用

---

## 決策

使用 **Composio** 作為外部工具整合中樞，首批整合 **Tavily** 搜尋，提供規劃階段 Claude Code 和 OpenClaw 生產系統的即時資訊查詢能力。

---

## 背景

規劃討論中頻繁遇到 Claude 知識截止日後的新技術或平台變更（OpenClaw 版本、Polymarket 政策、Qwen 模型更新等）。需要一個可靠的即時搜尋機制，且未來可能擴展其他工具整合（通知、資料源等）。

---

## 方案比較

| 方案 | 優點 | 缺點 |
|------|------|------|
| 內建 WebSearch | 零設定 | 無法用於 OpenClaw；品質不穩定 |
| Tavily MCP 直接 | 品質好、設定簡單 | 無法成為多工具整合中樞 |
| **Composio + Tavily** | 整合中樞、未來可擴展 | 多一個依賴、$0-29/月費用 |

---

## 實作細節

**規劃 Claude Code**：`.claude/settings.json` 加入 Composio Tavily MCP server

**OpenClaw 生產系統**：`composio-tavily` skill，安裝順序第三（ClawSec 之前）

**費用上限**：免費層 20,000 tool calls/月；超量升級至 $29/月方案

---

## Revision Log

| 日期 | 做了什麼 | 為什麼 |
|------|---------|--------|
| 2026-04-12 | 建立 ADR，決定採用 Composio + Tavily | 需覆蓋規劃 Claude Code 和 OpenClaw 兩個整合點；Composio 可作未來工具中樞 |
