---
id: ADR-007
title: Agent 框架選擇（Hermes Agent 取代 OpenClaw）
status: Accepted
date: 2026-04-27
---

## Context

專案初始規劃以 OpenClaw（Claude Code CLI）作為主 agent 框架，搭配 clawsec-suite 作為 runtime 層安全防護。評估後，決定改用 Hermes Agent（Nous Research）：Alibaba Cloud 官方提供 Hermes Agent 一鍵鏡像、框架內建排程機制可取代 systemd cron、子代理委派架構與混合模型路由設計吻合，且 hermes-attestation-guardian（外部 Node.js CLI）可提供環境完整性監控而無需與 runtime 深度整合。

此決策為規劃階段決策，無現有 OpenClaw 實例需要遷移。

## Decision

採用 Hermes Agent（Nous Research）作為主 agent 框架，部署於 Alibaba Cloud SAS 官方鏡像。原計畫的 systemd 30 分鐘 cron job 改由 Hermes 內建排程管理；安全監控由 hermes-attestation-guardian（外部 Node.js CLI）負責，以外部文件系統監控取代 clawsec-suite 的 runtime 深度整合方式。

## Options Considered

| 選項 | 優點 | 缺點 |
|------|------|------|
| OpenClaw（Claude Code CLI，原始計畫）| Anthropic 官方維護；polyclaw 原為 OpenClaw skill；clawsec-suite 與 runtime 深度整合（auto-restore、prompt injection 掃描）| 阿里雲（China mainland）平台；clawsec-suite 為 Node.js，遷移至其他環境需重建；無內建排程，需維護 systemd cron |
| Hermes Agent（Nous Research，採用）| Alibaba Cloud 官方一鍵鏡像；內建排程；子代理委派支援混合模型路由；hermes-attestation-guardian 架構清晰（外部 CLI，不與 runtime 耦合）| clawsec-suite auto-restore 無等效替代（改由 git 手動還原）；claude-code-skill 等效方案已確認為原生 delegate_task（官方 claude-code skill 用於編程任務，非本專案路由需求，排除）|

## Consequences

### 正面
- 部署流程簡化：官方一鍵鏡像，無需手動配置 agent 環境
- 排程內建：Hermes 管理定時任務，不需維護 systemd cron 腳本
- 安全層解耦：hermes-attestation-guardian 作為獨立外部 CLI，不與 Hermes runtime 耦合，架構更清晰
- 支援混合模型路由：Hermes 子代理委派與 Qwen+Claude 路由設計（ADR-001）天然吻合

### 負面 / 取捨
- clawsec-suite 的 auto-restore（soul-guardian）功能無對應替代，改由 git 手動還原流程處理（detect → 人工調查 → `git reset --hard`）
- `claude-code-skill`：已解決。Tier 2 Claude Sonnet 路由由 Hermes 原生 `delegate_task` 實現（`delegation.model: anthropic/claude-sonnet-4-6`），無需安裝額外 skill；官方 `claude-code` skill（編程任務用）不符本專案需求，排除
- Alibaba Cloud SAS 執行個體類型尚未確定（詳見 ADR-002）

## Revision Log

| Date | Change |
|------|--------|
| 2026-04-27 | 初版建立，記錄 OpenClaw → Hermes Agent 框架替換決策 |
| 2026-04-30 | 確認 claude-code-skill 等效方案（delegate_task），更新 Consequences 與 Options Considered |
