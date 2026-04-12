# ADR-003: 私鑰與敏感變數安全管理

**狀態**：已採納  
**日期**：2026-04-12

---

## 決策

所有敏感環境變數透過 systemd `EnvironmentFile` 載入，檔案設為 `chmod 600`，owner 為 openclaw service user。私鑰值不進入模型的 context window。

---

## 背景

系統需要管理以下敏感憑證：

| 變數 | 用途 |
|------|------|
| `ANTHROPIC_API_KEY` | Claude Sonnet 4.6 API |
| `BAILIAN_API_KEY` | 百鍊 Qwen3.5-Plus API |
| `POLYCLAW_PRIVATE_KEY` | Polygon 錢包私鑰（交易本金） |
| `POLYCLAW_RPC_URL` | Chainstack RPC 節點 |
| `CHAINSTACK_API_KEY` | Chainstack API |

---

## 安全設計

### 私鑰隔離

- `EnvironmentFile` 路徑：`/etc/openclaw/secrets.env`
- env var 存在於 process 層，不進入 LLM context window
- Polyclaw skill 直接讀取 env var 簽署交易，值不出現於 prompt

### 縱深防禦

| 防禦層 | 機制 |
|--------|------|
| 檔案系統 | `chmod 600`，限制讀取權限 |
| 網路隔離 | SSH 透過 Tailscale，不暴露公網 |
| LLM 防護 | ClawSec prompt injection 掃描，防止惡意市場資料誘導模型輸出私鑰 |
| 爆炸半徑 | 錢包僅存放交易本金（$100），損失上限可控 |

### systemd 配置

```ini
[Service]
EnvironmentFile=/etc/openclaw/secrets.env
```

---

## 被拒絕的替代方案

- **寫入 shell .env 文件**：容易被 shell history 或 ps 命令洩露。
- **HashiCorp Vault**：對個人專案過度複雜，維護負擔不符合規模可控原則。
- **雲端 Secret Manager**：需額外費用且引入外部依賴，$100 本金規模不合適。

---

## Revision Log

| 日期 | 做了什麼 | 為什麼 |
|------|---------|--------|
| 2026-04-12 | 初版建立 | 確立最小化私鑰洩露風險的管理方式 |
