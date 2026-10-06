# Vane MCP Server — 研究筆記

> 日期: 2026-07-22
> 目標: 將 Vane (Perplexica) 實作為 homelab MCP 工具

## 1. 上游 MCP 狀態

| 類型 | 編號 | 內容 |
|------|------|------|
| PR | #1044 | MCP server for Perplexica integration（進行中） |
| Issue | #758 | MCP integration 請求，社群要求 SSE + stdio 雙 transport |
| Issue | #1103 | `summary-only` response format，減少 MCP token 消耗 |
| Issue | #1094, #1118 | hanging searches / empty tool-call messages |
| Issue | #1032, #1042 | LLM JSON 回應中的 markdown fence 清理 |

**結論**: 上游有 PR #1044 在做 MCP server，但尚未 merge。社群有多個 MCP 相關 bug 報告，說明已有早期採用者。

## 2. Vane API 可暴露為 MCP Tool 的端點

| Tool 建議名 | 端點 | 功能 | 關鍵參數 |
|-------------|------|------|----------|
| `vane_search` | `POST /api/search` | 主搜尋 | `query`, `sources` (web/academic/discussions), `optimizationMode` (speed/balanced/quality), `stream` |
| `vane_chat` | `POST /api/search` (帶 history) | 對話式搜尋 | `history`, `query`, `systemInstructions` |
| `vane_providers` | `GET /api/providers` | 列出可用 LLM + embedding 模型 | 無 |
| `vane_images` | `POST /api/images` | 圖片搜尋 | `query`, `sources` |
| `vane_videos` | `POST /api/videos` | 影片搜尋 | `query` |
| `vane_suggest` | `POST /api/suggestions` | 搜尋建議 | `query` |

### /api/search 完整參數

```json
{
  "chatModel": { "providerId": "<uuid>", "key": "model-name" },
  "embeddingModel": { "providerId": "<uuid>", "key": "embedding-name" },
  "optimizationMode": "balanced",
  "sources": ["web"],
  "query": "What is ...",
  "history": [["human", "..."], ["assistant", "..."]],
  "systemInstructions": "...",
  "stream": false
}
```

回傳: `{ "message": "...", "sources": [{ "content": "...", "metadata": { "title": "...", "url": "..." } }] }`

### Streaming 格式 (SSE)

```
{"type":"init","data":"Stream connected"}
{"type":"sources","data":[...]}
{"type":"response","data":"chunk..."}
{"type":"done"}
```

## 3. 架構筆記

- **搜尋後端**: SearxNG（提供原始搜尋結果）
- **LLM 整合**: OpenAI, Anthropic, Groq, Ollama 等多 provider
- **結構化輸出**: 使用 Zod schema 確保 JSON 格式
- **結果合併**: Web + Academic + Discussions 結果去重後合併
- **Embedding**: 用於結果相似度排序

## 4. Homelab 整合方案

### 環境對應

| Vane 功能 | 架構資源建議 |
|-----------|-------------|
| SearxNG | 外部自架 SearxNG 或使用 Vane 內建 |
| Chat LLM | LiteLLM Proxy / OpenAI compatible gateway |
| Embedding | Ollama bge-m3 / local embedding |
| Ollama | 本地或 LAN 內 Ollama 實例 |

### MCP 實作方向

**不做**: 自己從頭寫 MCP server（上游 PR #1044 會做）

**改做**: 
1. 部署 Vane 到 homelab（Docker slim 版 + 外接 SearxNG）
2. 等上游 PR #1044 merge 後直接用
3. 若上游太慢，fork PR #1044 的實作，適配內部環境

### 建議部署架構

```
agent (CLI / MCP client)
  ↓ MCP protocol (stdio or SSE)
Vane MCP Server (port TBD)
  ↓ HTTP API
Vane Next.js app (port 3000, internal)
  ↓
SearxNG (searxng.local)
LiteLLM (litellm.local)
Ollama (ollama.local:11434)
```

## 5. 搜尋結果持久化 — 評估與 Use Case

### 5.1 核心問題

Vane 搜尋 API 回傳的結果是一次性的：查完即丟。agent 每次遇到同樣的問題都要重新搜尋、重新消耗 API 額度、重新等待回應。

**持久化 = 讓搜尋結果成為可重用的知識資產。**

### 5.2 評估

#### 價值

| 維度 | 說明 | 量化估算 |
|------|------|----------|
| **API 成本** | 減少重複搜尋對 SearxNG + litellm 的呼叫 | litellm cloud provider 搜尋一次約消耗 2-5k tokens，每日 50 次重複查詢 = 100-250k tokens/天 |
| **時間** | 緩存命中時省去搜尋 + LLM 整理時間 | 首次搜尋 5-15 秒 → 緩存命中 <1 秒 |
| **知識累積** | 搜尋結果隨時間增長，形成領域知識庫 | 每次搜尋約 2-5KB JSON，1000 次搜尋 = 2-5MB |
| **變化追蹤** | 同一主題不同時間的搜尋結果對比 | 支援「這個專案有什麼新進展？」類型查詢 |
| **RAG 來源** | 格式化搜尋結果適合向量化，支援語義檢索 | 與 DeepAnalyze MCP 的 `upload_file` 整合 |

#### 風險

| 風險 | 等級 | 緩解 |
|------|------|------|
| 搜尋結果過期 | 中 | TTL 機制：web 結果 7 天、academic 30 天、discussions 14 天 |
| 磁碟空間 | 低 | 單次搜尋 ~3KB，10K 次搜尋才 ~30MB |
| 隱私 | 中 | 存檔僅限本機 MariaDB 或 local filesystem，不上雲 |
| 重複資料 | 低 | 以 `query + sources + optimizationMode` 的 hash 做 dedup |

#### 成本效益結論

**正向**。開發成本低（1-2 天），邊際成本近零（JSON 檔案或 MariaDB row），收益隨使用量線性增長。

### 5.3 Use Cases

#### Use Case 1: Agent 工作流程中的搜尋重用

```
agent 開始新 task → 發現需要某領域知識
  → vane_search(query="...", cache=true)
  → 有緩存？直接回傳 file_id
  → 無緩存？搜尋 → 存檔 → 回傳 file_id
  → agent 用 ask(file_id=xxx) 分析結果
```

**場景**:
- agent 在修 Vane 的 bug，需要查 SearxNG API 文檔 → 搜過的結果直接用
- agent 在評估 litellm 的 pricing → 之前搜過的價格資料直接引用
- agent 在 debug DeepAnalyze 的 `<Code>` tag 問題 → 歷史搜尋結果對比

#### Use Case 2: 知識庫 RAG

```
搜尋結果存入 MariaDB
  → 定期 batch embedding (bge-m3)
  → 支援語義檢索：「找找看之前有沒有搜過類似的東西」
```

**場景**:
- 「上次我們查 litellm pricing 是什麼時候？結果是什麼？」
- 「有沒有之前搜過的 homelab 部署 best practices？」
- 「比較三次搜尋 Vane MCP 進度的結果」

#### Use Case 3: 監控與警報

```
定期搜尋特定關鍵字（cron / MCP schedule）
  → 對比歷史結果
  → 發現變化時通知（Telegram hook）
```

**場景**:
- 每天搜「Vane MCP PR #1044」，偵測是否 merge
- 每週搜「litellm pricing change」，偵測價格變動
- 每天搜「DeepAnalyze GitHub issue」，偵測新 bug 報告

#### Use Case 4: 跨 session 的搜尋接力

```
Session A: 搜尋「Docker best practices」→ 存檔
Session B: agent 需要 Docker 知識 → 查歷史搜尋 → 找到 Session A 的結果 → 引用
```

**場景**:
- 不同 session 的 agent 共享搜尋知識
- 避免同一團隊重複搜尋

### 5.4 資料格式設計

#### 搜尋結果存檔 Schema

```typescript
interface StoredSearch {
  id: string;                    // ULID or UUID
  query: string;                 // 原始搜尋查詢
  sources: string[];             // ["web", "academic", "discussions"]
  optimizationMode: string;      // "speed" | "balanced" | "quality"
  
  // 結果
  message: string;               // LLM 整理後的回答
  rawResults: Source[];          // 原始搜尋結果
  
  // 元資料
  timestamp: string;             // ISO 8601
  provider: string;              // 使用的 LLM provider
  model: string;                 // 使用的模型
  queryHash: string;             // sha256(query + sources + mode) 用於 dedup
  
  // TTL
  expiresAt: string;             // 根據 sources 決定過期時間
  
  // RAG（可選）
  embedding?: number[];          // bge-m3 向量
}

interface Source {
  content: string;
  title: string;
  url: string;
}
```

#### 存儲選擇

| 方案 | 優點 | 缺點 | 適用 |
|------|------|------|------|
| **MariaDB** | SQL 查詢、跨 host 共享、已有 galera cluster | 需 schema migration | ✅ 推薦 |
| **JSON 檔案** | 最簡單、無依賴 | 查詢能力弱、不易 dedup | 適合 MVP |
| **LanceDB** | 原生向量支援 | 新增依賴 | 適合 RAG 進階需求 |

### 5.5 MCP Tool 設計

#### 新增工具

| Tool | 功能 | 參數 |
|------|------|------|
| `vane_search_cached` | 搜尋 + 自動存檔 + 命中時直接回傳 | `query`, `sources`, `optimizationMode`, `forceRefresh` |
| `vane_find_similar` | 用語義找相似搜尋結果 | `query`, `topK` |
| `vane_list_searches` | 列出歷史搜尋 | `limit`, `offset`, `source` |
| `vane_get_search` | 取得特定搜尋結果 | `id` |

### 5.6 實作優先順序

| 階段 | 工作 | 天數 |
|------|------|------|
| Phase 1 | JSON 檔案存檔 + `vane_search_cached` tool | 0.5 天 |
| Phase 2 | MariaDB schema + dedup + TTL | 1 天 |
| Phase 3 | Batch embedding + `vane_find_similar` | 1-2 天 |
| Phase 4 | Cron 監控 + Telegram 通知 | 0.5 天 |

### 5.7 與 DeepAnalyze MCP 的整合

```
vane_search_cached("Vane MCP PR #1044 status")
  → 存檔為 JSON → file_id: search-abc123
  → ask(file_id="search-abc123", query="總結進度和下一步")
  → DeepAnalyze 用 code execution 分析結果
  → 回傳結構化摘要
```

## 6. 風險與待確認

- [ ] PR #1044 的 transport 類型（stdio vs SSE vs 雙支援）
- [ ] Vane 的 provider 認證方式如何傳入 MCP
- [ ] 本機 SearxNG vs Vane 內建 SearxNG 的選擇
- [ ] 檔案上傳功能（Vane 支援 PDF/圖片問答）是否需要暴露為 MCP tool
- [ ] MariaDB schema 設計確認（與現有 galera cluster 整合方式）
- [ ] embedding 模型選擇（bge-m3 已有，或用 Vane 內建）
