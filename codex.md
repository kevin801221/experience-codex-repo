Codex.md — 旅遊 Agent 後端組合規範（給 coding agent 用）

目的：讓 coding agent 能讀取 ./example 下大量第三方的旅遊 Agent 範例專案，抽出可重用的後端元件（API、資料層、服務、middleware），並自動組合成符合現有 Next.js 前端（由 agent 已會產生）的後端實作或 microservice 清單。Codex 規範定義發現、抽取、轉換、整合與驗證的準則、介面與範本。

⸻

1. 高階原則
	1.	非破壞性優先：不直接改寫 ./example 裡原始程式碼，先以抽取（copy / wrap / adapter）為主。
	2.	最小耦合：產生的後端應以可獨立部署的 microservice 或 Next.js API route 形式呈現，避免緊耦合於前端。
	3.	可預測介面：所有產生的 endpoint 必須符合「契約模板」（見 §3），以便前端能預期欄位與錯誤代碼。
	4.	可配置：環境相關（DB、API keys、OAuth）全部透過 env variables 管理（見 §6）。
	5.	可測試：自動產生至少 1 組端對端 mock 與單元測試範例（見 §8）。

⸻

2. 檔案與代碼發現規則

agent 在 ./example 下應依序執行以下步驟來發現可用後端元件：
	1.	掃描目錄：列出 ./example/*（遞迴到 depth=3）尋找：
	•	package.json, pyproject.toml, requirements.txt（判斷 Node / Python 專案）
	•	Dockerfile（containerized service）
	•	典型後端入口：server.js, app.js, index.js, main.py, app.py, api/, routes/ 目錄
	•	OpenAPI / swagger 檔：openapi.yaml, swagger.json 或 api.yaml
	•	README 或 codex.md（若存在，優先讀取）
	2.	分類：將找到的項目標為：
	•	API provider（提供 REST/GraphQL 的現成服務）
	•	Auth handler（OAuth/Session/Token 示例）
	•	Data adapter（DB model、migrations、seed）
	•	Utils（地理/價格/行程演算法、parser）
	•	Integration（外部 OTA / payment / map / SMS）
	3.	抽取元資料：對每個 candidate 抽出：
	•	支援的 HTTP endpoints（路徑 + method）
	•	必要參數（path, query, body）
	•	回傳 schema（若無則用 example/response 推斷）
	•	Auth 類型（none / apiKey / oauth2 / bearer / session）
	•	相依性（DB、外部 API）
	•	licence（若有，報告是否可重用）

⸻

3. API 契約模板（所有自動產生 endpoint 必須遵守）

目標是讓 Next.js 前端可以直接呼叫，契約要簡單、穩定、易偵錯。

通用回應包裝（JSON）：

{
  "ok": true|false,
  "status": 200,
  "error": null | {
    "code": "ERR_CODE",
    "message": "human friendly"
  },
  "data": {} | []
}

成功範例：
	•	200 OK -> {"ok": true, "status": 200, "error": null, "data": {...}}
	•	201 Created -> 同上但 status=201

錯誤範例：
	•	400 Bad Request -> {"ok": false, "status": 400, "error": {"code":"INVALID_PARAMS","message":"..."},"data":null}

常用 endpoint pattern（命名慣例）：
	•	列表：GET /api/{resource}  (支援 ?page=, ?limit=, ?q=...)
	•	取單一：GET /api/{resource}/{id}
	•	建立：POST /api/{resource}
	•	更新（部分）：PATCH /api/{resource}/{id}
	•	刪除：DELETE /api/{resource}/{id}
	•	action：POST /api/{resource}/{id}/action

Content-Type：
	•	一律回應 application/json（除非為 file download）

驗證：
	•	所有 POST/PATCH 必須回傳欄位驗證錯誤陣列（若有）：error.details = [{field:"x.y", message:"..."}, ...]

⸻

4. Adapter / Transformation 規範

當 agent 找到第三方後端元件（例如 Express 路由或 FastAPI endpoint），必須產生一個 Adapter，使其輸出符合 §3 契約。Adapter 有兩種策略：

A. Wrapper adapter（快速）
	•	將原服務以 container 或 subprocess 啟動，透過 HTTP 反向代理（或 fetch）轉譯回應結構與錯誤碼。
	•	適用於：complex legacy service、無法輕易移植的外部 library。

B. Code-porting adapter（長期）
	•	將原 endpoint 的核心邏輯抽出，轉譯為 Node.js（或 Next.js API route）程式碼，直接整合到 monorepo 中，並加入契約包裝/驗證。
	•	適用於：較小、明確的 route 或 utils。

Adapter 必須提供：
	•	adapter-manifest.json（放在產生檔案夾），內容包含：name, source_path, endpoints_mapped, auth, env_required, strategy。

範例 adapter-manifest.json：

{
  "name": "example-hotel-search",
  "source_path": "./example/hotel-search",
  "endpoints_mapped": ["/search", "/details/:id"],
  "auth": "apiKey",
  "env_required": ["HOTEL_API_KEY"],
  "strategy": "wrapper"
}


⸻

5. 發現到 Next.js API route 的映射規則

目標：前端呼叫 /api/xxxx 時由 agent 產生或路由到相應後端。
	1.	自動路由生成
	•	按 resource 產生 ./apps/backend/api/{resource}/route.js（或 route.ts）檔案（Next.js 13+ App Router 格式），或 pages/api/{resource}.js（Legacy）。
	•	route 層只做：
	•	欄位驗證（schema）
	•	auth 檢查
	•	呼叫 adapter（內部 function / fetch to adapter service）
	•	回傳契約包裝（§3）
	2.	環境和 URL 決定
	•	若使用 wrapper strategy，route 會向 http://${ADAPTER_HOST}:${ADAPTER_PORT}{path} 轉發（ADAPTER_HOST/PORT 由 env 指定）。
	•	若 code-porting，route 直接 import adapter module 並執行。
	3.	名稱對應規則（簡單）
	•	源端 /hotel/search → /api/hotels/search
	•	源端 /booking/create → /api/bookings
	•	若衝突：採用 namespace（例如 providerName_resource）

⸻

6. 設定 / 機密 / 外部服務
	1.	環境變數命名慣例：PROVIDERNAME_SERVICE_KEY、PROVIDERNAME_BASE_URL、DB_URL。
	2.	Secrets 管理：在產生 adapter-manifest.json 時，列出必要 env，並在產出 README 中加入 .env.example。
	3.	多 provider 合併策略：
	•	若多個 provider 提供同類 resource（例如航班搜尋），建立 orchestrator 層：
	•	GET /api/flights/search -> orchestrator 向多個 provider 下查詢並合併結果（fallback、ranking、dedupe）
	•	合併策略在 manifest 中標示 priority 與 merge_strategy（first_success / best_price / combine）

⸻

7. Auth、Rate Limit、與安全
	1.	Auth 層：
	•	公開端點（public）與保護端點（protected）必須在 manifest 標註。
	•	支援：API Key（header x-api-key）、Bearer token、Session cookie（NextAuth 可選）。
	•	Agent 自動產生 middleware 檔案 ./apps/backend/middleware/auth.js 並在 route 中引入。
	2.	Rate limiting：
	•	為外部 provider 設置 client-side rate limit（token bucket），在 adapter wrapper 中實作（redis 或 in-memory for dev）。
	•	在 manifest 中記錄 provider rate limit 建議值。
	3.	安全檢查：
	•	禁止把 provider secrets 硬編進程式碼。
	•	Cross-site 請求必須有 CORS 規則（前端域名預設 allow）。
	•	輸入驗證以 JSON schema 驗證（AJV for Node / pydantic for Python）。

⸻

8. 測試與驗證

每個被採用/組合出的後端模組都必須自帶：
	1.	Contract test (必須)：對每個 endpoint 產生一個 contract test（範例 request + 期望 response shape）；可用 jest + supertest 或 pytest。
	2.	Mock server：若原 provider 不可用，產生一個 lightweight mock（回傳範例資料）。
	3.	E2E smoke test：前端最少一個頁面呼叫該 endpoint，檢查 ok: true。

測試檔案命名：{resource}.spec.js 或 {resource}_test.py，放在 ./tests.

⸻

9. CI/CD 與部署建議
	•	自動化 pipeline 步驟：
	1.	static analysis（lint）
	2.	unit & contract tests
	3.	build adapters（docker build 若 wrapper）
	4.	run smoke tests against staging
	•	部署目標：
	•	Next.js 前端：Vercel / Netlify
	•	Backend adapters：Kubernetes / Docker Compose / Cloud Run
	•	Deployment manifest（K8s / docker-compose）會由 agent 產生基本範本（含 env placeholders）。

⸻

10. 衝突處理與人工介入點
	1.	schema 衝突：若同一 resource schema 不一致，agent 應：
	•	優先採用 openapi.yaml 若存在。
	•	若仍衝突，產生 migration-help.md 並標註衝突欄位，將 resource 設為 manual-review。
	2.	認證無法自動化：若 provider 要手動 OAuth consent（例如某些 Google API），agent 必須生成步驟清單與 oauth_setup.md。
	3.	license/條款衝突：若 example 的 licence 不允許商用或重用，agent 必須標示 not-allowed，並排除該元件。

⸻

11. Output 結構（agent 最終要產出的檔案與檢查清單）

當 agent 完成一個 example 的抽取與組合，必須在 repo 中建立一個 ./generated/{providerName}-{resource}/ 目錄，包含以下檔案：
	•	adapter-manifest.json （必須）
	•	route.js/route.ts（Next.js API route）或 service/（dockerized）
	•	README.md（包含 usage, env, endpoints）
	•	.env.example
	•	tests/{...}
	•	openapi.generated.yaml（若可能）
	•	deploy/{docker-compose.yml | k8s.yaml}（如適用）

最後在 repo 根目錄產生 generated-index.json 列出所有已產生的 adapter 與其狀態（ready / manual-review / blocked）。

⸻

12. Agent 操作提示（Prompt template）

把這段作為 coding agent 的「任務 prompt」骨幹，agent 執行時把 {} 替換為實際路徑或 provider 名稱。

你是自動化整合 agent。目標：從 `{examples_path}` 裡抽取後端元件並為 Next.js 前端生成對應後端（route 或 microservice）。請按 Codex.md 規範執行。步驟：
1. 掃描並分類候選項目（依 §2）。
2. 為每個候選建立 adapter 並產生 `adapter-manifest.json`（見 §4）。
3. 依 §3 產生 Next.js API route 或 wrapper service。
4. 產生 `.env.example`、`README.md`、至少一個 contract test。
5. 若有衝突或需要人工處理，將狀態標註為 `manual-review` 並產生 `migration-help.md`。
6. 將所有產出放在 `./generated/{providerName}-{resource}/`，並更新 `generated-index.json`。

輸出格式：一次回報所有處理的 provider，並輸出 JSON summary（包含 manifest 路徑與狀態）。


⸻

13. 範例（實作小樣板）

Next.js API route 範例（route.js）

// ./apps/backend/api/hotels/route.js (Next.js App Router)
import { NextResponse } from 'next/server'
import { validateSearchBody } from '../../../lib/validators'
import { callHotelAdapter } from '../../../lib/adapters/hotel-adapter'

export async function POST(request) {
  const body = await request.json()
  const errors = validateSearchBody(body)
  if (errors.length) {
    return NextResponse.json({ ok:false, status:400, error:{code:'INVALID_PARAMS', message:'', details: errors }, data: null }, { status:400 })
  }
  try {
    const result = await callHotelAdapter('/search', { method:'POST', body })
    return NextResponse.json({ ok:true, status:200, error:null, data: result }, { status:200 })
  } catch (err) {
    return NextResponse.json({ ok:false, status:500, error:{code:'UPSTREAM_ERROR', message: err.message}, data:null }, { status:500 })
  }
}

adapter wrapper 範例（hotel-adapter.js）

export async function callHotelAdapter(path, opts) {
  const base = process.env.HOTEL_PROVIDER_BASE_URL
  const res = await fetch(`${base}${path}`, {
    method: opts.method || 'GET',
    headers: {
      'Content-Type':'application/json',
      'x-api-key': process.env.HOTEL_PROVIDER_KEY
    },
    body: opts.body ? JSON.stringify(opts.body) : undefined
  })
  const json = await res.json()
  // 把原 provider 的回應轉成 contract
  if (!res.ok) throw new Error(json.message || 'provider error')
  return json
}


⸻

14. 日誌與可觀察性
	•	所有 adapter 呼叫應產生結構化 log：{ts, provider, endpoint, duration_ms, statusCode, ok}。
	•	若部署支援，產生 basic metrics：requests_total, errors_total, latency_ms（Prometheus friendly names）。

⸻

15. 最後備註（Checklist）
	•	已掃描 ./example 並建立候選清單
	•	每個選中的 candidate 有 adapter-manifest.json
	•	每個 route 有 contract test + mock
	•	產出 generated-index.json 並註明 status
	•	產出 deploy 範本（或 wrapper Dockerfile）
	•	產出 .env.example 與 README.md（使用說明）
