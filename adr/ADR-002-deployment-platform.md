---
id: ADR-002
title: 部署平台選擇
status: Accepted
date: 2026-04-27
---

## Context

專案定位為個人維運的策略驗證系統，需要低成本、低維護負擔的部署方式。初始規劃使用阿里雲（China mainland）+ OpenClaw 一鍵鏡像，後因框架改為 Hermes Agent（見 [ADR-007](ADR-007-agent-framework.md)），部署平台隨之更新為 Alibaba Cloud（國際版）。

## Decision

使用 Alibaba Cloud（國際版）SAS，透過 Alibaba Cloud 官方 Hermes Agent 一鍵鏡像部署。systemd daemon 負責服務自動重啟，排程（30 分鐘 hedge scan 及每日結算）由 Hermes 內建排程管理。SSH 存取透過 Tailscale，不暴露公網 IP。Hermes 以 gateway 模式運行（Telegram adapter 啟用），透過 `send_message` 工具支援通知功能（見 [ADR-009](ADR-009-telegram-notifications.md)）。執行個體類型選為通用型 swas.s.c2m4s50b1.linux（2 vCPU / 4 GiB），部署地域為英國（倫敦）或德國（法蘭克福），以最小化與 Polymarket 匹配引擎的 RTT。Polymarket CLOB 引擎部署於 AWS 愛爾蘭（eu-west-1），英國/德國物理距離最近。通用型 BGP（中國優化）線路同時降低調用百鍊 Qwen API 的延遲，形成雙贏。

## Options Considered

### 部署平台選型

| 選項 | 優點 | 缺點 |
|------|------|------|
| 阿里雲（China mainland）+ OpenClaw 鏡像（原始計畫）| 一鍵鏡像簡化部署 | 因框架替換為 Hermes 而廢棄；China mainland 平台不符使用者需求 |
| Alibaba Cloud（國際版）+ Hermes Agent 鏡像（採用）| 官方鏡像；Hermes 內建排程取代 systemd cron；國際版適合非中國大陸使用情境 | 執行個體類型已確定為通用型 c2m4 |
| Docker Compose on VPS | 靈活性高 | 無一鍵鏡像，設置複雜，維護負擔較高 |
| 本地機器常駐 | 零成本 | 依賴本地網路，不適合 24/7 掃描 |
| AWS / GCP | 穩定、全球部署 | 成本較高，無 Hermes 官方優化鏡像 |

### SAS 規格族選型分析

| 規格族 | 優點 | 缺點 / 排除原因 |
|--------|------|----------------|
| 多公網IP型 | — | 記憶體上限 2 GiB，不足以同時支撐 Hermes Agent + Node.js guardian + Python Polyclaw；多 IP 對本專案無用 |
| CPU 優化型 | 獨佔超線程，計算性能有保障 | Hermes 為 I/O 密集（每 30 分鐘 API 呼叫），計算密集溢價無意義 |
| 容量型 | 大容量磁碟 | 60 GiB 起步磁碟設計針對大存儲需求；本專案 20 GiB 即足 |
| 國際型 | BGP 非中國優化，適合跨境場景 | 僅限 HK/新加坡/日本三地域，無法部署於歐洲 |
| 通用型（採用）| 全球地域覆蓋（含英國/德國）；BGP 中國優化有助連線 Bailian Qwen API | — |

### 地域選型分析

| 地域 | 優點 | 缺點 / 排除原因 |
|------|------|----------------|
| 新加坡地域（原計畫）| 地理位置居中 | 誤判 Polymarket 在美國；實際 CLOB 引擎在 AWS 愛爾蘭，新加坡往返愛爾蘭額外增加 150-200ms RTT，加劇 Leg Risk |
| 英國/德國地域（採用）| 與 Polymarket CLOB（AWS 愛爾蘭 eu-west-1）RTT 最低 | 距百鍊 Qwen API（中國大陸）較遠，但 BGP 中國優化線路已緩解 |

## 規格

| 項目 | 值 |
|------|-----|
| 主機類型 | Alibaba Cloud SAS |
| 執行個體類型 | 通用型（swas.s.c2m4s50b1.linux）|
| CPU / RAM | 2 vCPU / 4 GiB |
| 地域 | 英國（倫敦）或德國（法蘭克福）|
| 公網線路 | BGP（中國優化）|
| 作業系統 | Ubuntu 22.04 LTS |
| 儲存 | 50 GiB SSD（swas.s.c2m4s50b1 規格附帶）|
| 部署方式 | Alibaba Cloud 官方 Hermes Agent 鏡像 |
| 服務管理 | systemd（自動重啟）；以 **gateway 模式**運行（Telegram adapter 啟用）；排程由 Hermes 內建管理 |
| 網路存取 | 443 開放；SSH 透過 Tailscale |
| 每月成本 | $10-15（估算，需於 Alibaba Cloud 控制台確認最終定價）|

## Consequences

### 正面
- 一鍵鏡像大幅降低初始設置複雜度
- Hermes 內建排程取代 systemd cron，不需維護獨立排程腳本
- 國際版平台符合使用者需求；Hermes 輕量設計適合個人 VPS
- 英國/德國地域最小化 Polymarket CLOB RTT，降低 Leg Risk；通用型 BGP 中國優化同時改善 Bailian Qwen API 連線品質
- 4 GiB RAM 防止多進程（Hermes + guardian + Polyclaw）並存時 OOM

### 負面 / 取捨
- 平台從阿里雲切換為 Alibaba Cloud 國際版，需重新設置帳號與計費
- 每月成本仍需於 Alibaba Cloud 控制台確認最終定價
- Hermes 以 gateway 模式運行，systemd unit 配置較純 CLI 模式稍複雜；gateway 模式具體啟動指令待確認（見 ADR-009）

## Revision Log

| 日期 | 做了什麼 | 為什麼 |
|------|---------|--------|
| 2026-04-12 | 初版建立 | 確立個人可維運、低成本的部署方式（阿里雲 + OpenClaw） |
| 2026-04-27 | 更新平台至 Alibaba Cloud 國際版 + Hermes Agent 鏡像；補齊 ADR 標準結構 | 框架替換為 Hermes Agent（ADR-007），平台需同步更新 |
| 2026-04-28 | 確定執行個體類型為通用型 swas.s.c2m4s50b1.linux（2c/4G），部署地域英國或德國；補充各規格族排除原因與地域選型分析 | Polymarket CLOB 引擎在 AWS 愛爾蘭，歐洲地域可降低 RTT 150-200ms；4 GiB 防止 OOM |
| 2026-05-03 | 補充 Hermes 以 gateway 模式運行的說明 | 導入 Telegram 通知（ADR-009）需要 gateway 模式；更新服務管理描述 |
