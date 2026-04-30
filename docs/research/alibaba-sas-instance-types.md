---
topic: Alibaba Cloud 輕量應用服務器（SAS）執行個體類型
queried: 2026-04-27
expires: 2026-10-27
sources:
  - https://help.aliyun.com/zh/simple-application-server/product-overview/instance-families
  - https://developer.aliyun.com/article/1678983
---

# Alibaba Cloud SAS 執行個體規格族

## 五大規格族對比

| 規格族 | 計算規模 | 儲存規模 | 網路線路 | 固定 IP | 適用場景 |
|--------|---------|---------|---------|---------|---------|
| 通用型 | 2c0.5G～4c16G | 20～80 GiB | BGP（中國優化） | 1 個 IPv4 | 網站、Web 應用、開發測試、APP 後端 |
| CPU 優化型 | 2c4G～16c64G | 60～480 GiB | BGP（中國優化） | 1 個 IPv4 | 企業級應用、資料庫、快取、遊戲服務器 |
| 多公網IP型 | 2c0.5G～2c2G | 20～40 GiB | BGP（中國優化） | 2～3 個 IPv4 | 電商帳號安全管理、遊戲加速 |
| 國際型 | 2c0.5G～4c16G | 20～80 GiB | BGP（**非**中國優化） | 1 個 IPv4 | 跨境應用、不服務中國大陸用戶場景 |
| 容量型 | 2c2G～4c16G | 60～300 GiB | BGP（中國優化） | 1 個 IPv4 | 私有網盤、實訓環境 |

**重要提醒（官方）**：
- **國際型** BGP（非中國優化）線路：服務中國大陸用戶有較高延遲與丟包率，僅適用於不需服務中國大陸用戶的場景。
- **CPU 優化型**：始終保證對整個超線程的佔用，適合對計算性能有強保障需求的企業級場景。

## 通用型具體規格

| 實例規格 | vCPU | 記憶體 | 系統盤 | 頻寬上限 |
|---------|------|--------|--------|---------|
| swas.s.c2m05s20b1.linux | 2 | 0.5 GiB | 20 GiB | 200 Mbps |
| swas.s.c2m1s30b1.linux | 2 | 1 GiB | 30 GiB | 200 Mbps |
| swas.s.c2m2s40b1.linux/win | 2 | 2 GiB | 40 GiB | 200 Mbps |
| swas.s.c2m4s50b1.linux/win | 2 | 4 GiB | 50 GiB | 200 Mbps |
| swas.s.c4m8s70b1.linux/win | 4 | 8 GiB | 70 GiB | 200 Mbps |
| swas.s.c4m16s80b1.linux/win | 4 | 16 GiB | 80 GiB | 200 Mbps |

## 國際型具體規格（僅限 HK / 新加坡 / 日本）

| 實例規格 | vCPU | 記憶體 | 系統盤 | 頻寬上限 |
|---------|------|--------|--------|---------|
| swas.s.c2m05s20i1.linux | 2 | 0.5 GiB | 20 GiB | 200 Mbps |
| swas.s.c2m1s30i1.linux | 2 | 1 GiB | 30 GiB | 200 Mbps |
| swas.s.c2m2s40i1.linux/win | 2 | 2 GiB | 40 GiB | 200 Mbps |
| swas.s.c2m4s50i1.linux/win | 2 | 4 GiB | 50 GiB | 200 Mbps |
| swas.s.c4m8s70i1.linux/win | 4 | 8 GiB | 70 GiB | 200 Mbps |
| swas.s.c4m16s80i1.linux/win | 4 | 16 GiB | 80 GiB | 200 Mbps |

## CPU 優化型具體規格

| 實例規格 | vCPU | 記憶體 | 系統盤 | 頻寬上限 |
|---------|------|--------|--------|---------|
| swas.d.c2m4s60b1 | 2 | 4 GiB | 60 GiB | 200 Mbps |
| swas.d.c2m8s120b1 | 2 | 8 GiB | 120 GiB | 200 Mbps |
| swas.d.c4m8s180b1 | 4 | 8 GiB | 180 GiB | 200 Mbps |
| swas.d.c4m16s240b1 | 4 | 16 GiB | 240 GiB | 200 Mbps |
| ... | ... | ... | ... | ... |

## 地域可用性

- **通用型**：中國大陸（北京/上海/杭州等）+ 國際（HK、新加坡、馬來西亞、印尼、菲律賓、泰國、日本、韓國、英國、德國、美國）
- **國際型**：僅 **中國香港、新加坡、日本（東京）**
- **CPU 優化型**：同通用型（全球可用）
- **容量型**：同通用型（全球可用）
- **多公網IP型**：同通用型（全球可用）

## 計費方式

- 所有現代規格族：**無固定流量費**（不另收流量費用），按頻寬 200 Mbps 峰值使用
- 上一代：有固定月流量包限制（超量計費）
