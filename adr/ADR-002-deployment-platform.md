---
id: ADR-002
title: 部署平台選擇
status: Accepted
date: 2026-04-27
---

## Context

專案定位為個人維運的策略驗證系統，需要低成本、低維護負擔的部署方式。初始規劃使用阿里雲（China mainland）+ OpenClaw 一鍵鏡像，後因框架改為 Hermes Agent（見 [ADR-007](ADR-007-agent-framework.md)），部署平台隨之更新為 Alibaba Cloud（國際版）。

## Decision

使用 Alibaba Cloud（國際版）SAS，透過 Alibaba Cloud 官方 Hermes Agent 一鍵鏡像部署。systemd daemon 負責服務自動重啟，排程（30 分鐘 hedge scan 及每日結算）由 Hermes 內建排程管理。SSH 存取透過 Tailscale，不暴露公網 IP。

## Options Considered

| 選項 | 優點 | 缺點 |
|------|------|------|
| 阿里雲（China mainland）+ OpenClaw 鏡像（原始計畫）| 一鍵鏡像簡化部署 | 因框架替換為 Hermes 而廢棄；China mainland 平台不符使用者需求 |
| Alibaba Cloud（國際版）+ Hermes Agent 鏡像（採用）| 官方鏡像；Hermes 內建排程取代 systemd cron；國際版適合非中國大陸使用情境 | 執行個體類型尚待選定 |
| Docker Compose on VPS | 靈活性高 | 無一鍵鏡像，設置複雜，維護負擔較高 |
| 本地機器常駐 | 零成本 | 依賴本地網路，不適合 24/7 掃描 |
| AWS / GCP | 穩定、全球部署 | 成本較高，無 Hermes 官方優化鏡像 |

## 規格

| 項目 | 值 |
|------|-----|
| 主機類型 | Alibaba Cloud SAS |
| 執行個體類型 | 待定（詳見 CURRENT_SPEC.md 待解決事項）|
| CPU / RAM | 待定（原規劃 2核 4G，Hermes 官方稱可跑在 $5 VPS）|
| 作業系統 | Ubuntu 22.04 LTS |
| 儲存 | 20GB SSD |
| 部署方式 | Alibaba Cloud 官方 Hermes Agent 鏡像 |
| 服務管理 | systemd（自動重啟）；排程由 Hermes 內建管理 |
| 網路存取 | 443 開放；SSH 透過 Tailscale |
| 每月成本 | $10-15（估算，依執行個體類型調整）|

## Consequences

### 正面
- 一鍵鏡像大幅降低初始設置複雜度
- Hermes 內建排程取代 systemd cron，不需維護獨立排程腳本
- 國際版平台符合使用者需求；Hermes 輕量設計適合個人 VPS

### 負面 / 取捨
- 執行個體類型尚未確定，成本仍為估算值
- 平台從阿里雲切換為 Alibaba Cloud 國際版，需重新設置帳號與計費

## Revision Log

| 日期 | 做了什麼 | 為什麼 |
|------|---------|--------|
| 2026-04-12 | 初版建立 | 確立個人可維運、低成本的部署方式（阿里雲 + OpenClaw） |
| 2026-04-27 | 更新平台至 Alibaba Cloud 國際版 + Hermes Agent 鏡像；補齊 ADR 標準結構 | 框架替換為 Hermes Agent（ADR-007），平台需同步更新 |
