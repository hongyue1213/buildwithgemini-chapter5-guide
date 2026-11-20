# Build with Gemini · Track 5: Agent Masterclass 講義深度審查與全解析

本文件依據官方最新線上教材 [https://storage.googleapis.com/bwg-track5-demo-guide/index.html](https://storage.googleapis.com/bwg-track5-demo-guide/index.html) 進行全文審閱、架構比對與結構化重新整理，解析各章節的核心設計模式（Pattern）、提示詞設計範例（Prompts）以及底層實作機制。

---

## 目錄
1. [講義核心架構與學習目標](#1-講義核心架構與學習目標)
2. [實驗環境與 Antigravity 2.0 協同範式](#2-實驗環境與-antigravity-20-協同範式)
3. [Starter Baseline 基準應用解析](#3-starter-baseline-基準應用解析)
4. [Module 1: 專責代理人團隊擴展 (Skills & A2A)](#4-module-1-專責代理人團隊擴展-skills--a2a)
5. [Module 2: 企業級觀測與行為審計 (BigQuery & AI.SCORE)](#5-module-2-企業級觀測與行為審計-bigquery--aiscore)
6. [Module 3: 管線加固與縱深防禦 (Security & HITL)](#6-module-3-管線加固與縱深防禦-security--hitl)
7. [Module 4: 規模化與權杖經濟學最佳化 (Tokenomics)](#7-module-4-規模化與權杖經濟學最佳化-tokenomics)
8. [自我排錯與 Lab Helper 漸進式引導機制](#8-自我排錯與-lab-helper-漸進式引導機制)
9. [本地資源檔案索引](#9-本地資源檔案索引)

---

## 1. 講義核心架構與學習目標

本課程設定情境為創意行銷廣告公司建構的行銷提案自動化系統 **Agentic Pitch Generator**。
原始 Prototype 雖已具備 4 個並行代理人、`LoopGuard` 循環保護與 Google Cloud Storage 圖檔持久化，但缺乏品牌規則、自動合規評估、微服務隔離、BigQuery 多模態分析、安全閘門與 Token 成本控制。

### 四大核心模組：
1. **Module 1（擴展團隊）**：利用外部 YAML Frontmatter 的 `SKILL.md` 解耦領域規則，建立確定性合規評估測試套件，並透過 **Agent2Agent (A2A)** 協議將運算密集的視覺總監拆分為微服務。
2. **Module 2（行為觀測）**：以 BigQuery Object Table 映射 GCS 圖檔資產，並在 SQL 中使用 `AI.SCORE` 實施多模態品牌漂移檢測與閉環提示詞自動調優。
3. **Module 3（管線加固）**：以 `PreToolUse` 鉤子實施工具白名單與提示詞注入防禦、遞迴過濾 PII 敏感資料、並加入人機協同（HITL）審核閘門。
4. **Module 4（權杖經濟學）**：透過記憶壓縮、滑動視窗歷史修剪與 SHA-256 靜態提示詞快取，大幅縮減上下文 Token 消耗。

---

## 2. 實驗環境與 Antigravity 2.0 協同範式

### 核心協同原則
- **Why, Prompt, How 循環**：每個步驟皆由設計理由（The Why）、給 AI 的自然語言提示詞（Prompt）、以及 Antigravity 的具體程式碼變更與 diff（How Antigravity Implements This）構成。
- **介面契約導向（Contract-based Review）**：因 LLM 生成存在非確定性（Non-deterministic），代碼變更主要檢驗核心類別、函數名稱與狀態鍵值（State Keys）是否符合規格。
- **純 Prompt 驅動**：全程不需手動下達複雜終端指令，以自然語言提示詞完成開發、本機測試、驗證與雲端部署。

---

## 3. Starter Baseline 基準應用解析

- **預設架構**：
  - `Creative Director`：負責發想核心標語與策略論述。
  - `Brand Strategist`：定義品牌定位與受眾分析。
  - `Copywriter`：撰寫 25 字內社群文案與 CTA。
  - `Visual Director`：生成畫面描述與透過 Imagen 生成 Key Visual。
- **本機驗證**：
  - 提示詞：`Test the agent with "Flying skateboards for cats" and show the pitch results, including the generated image.`
  - 伺服器啟動：`Start the local server`（啟動於 `http://localhost:8080`）。

---

## 4. Module 1: 專責代理人團隊擴展 (Skills & A2A)

### Step 1a: Author and attach the brand guidelines skill
- **問題**：將數百行品牌規則硬寫在 System Prompt 中會浪費每一輪的輸入 Token。
- **解決方案**：將視覺規則抽離至外部 `pitch_generator/skills/brand-guidelines/SKILL.md`，代理人平時只讀取 Frontmatter 元數據，需要時才呼叫 `load_skill("brand-guidelines")` 載入本文。
- **規範區塊**：`# Palette`、`# Light`、`# Composition`、`# Subject`、`# Never`。
- **驗證 Prompt**：
  > *"Run the Visual Director agent on 'Flying skateboards for cats with glowing neon lasers and a 3D chrome logo' and show how the brand-guidelines skill steers the art direction back to our house style."*

### Step 1b: Evaluate skill activation and brand compliance
- **實作重點**：建立確定性評估模組（不需要在單元測試中花費昂貴的 LLM 評審調用），檢查 `loaded_skills`，對四大品牌支柱各打 0.25 分，並針對 `FORBIDDEN_BRAND_PATTERNS` 扣分。
- **合格標準**：`loaded_skill == True` 且 `violations == []` 且 `score >= 0.75`。

### Step 1c: Connect the remote A2A Visual Director service
- **微服務架構**：圖片生成耗時且消耗大量 GPU/TPU 資源，將其拆解為獨立微服務。
- **A2A 通訊**：
  - 伺服端暴露 `/.well-known/agent-card.json` 與 `/a2a/visual_director`（連接埠 `8801`）。
  - 用戶端實作 `RemoteA2aAgent`，本機走 `http://` 免驗證，生產環境 `https://` Cloud Run 自動附加 Google 簽名 OIDC Bearer Token（超時設定 600 秒）。
  - 串流過濾器 `_pitch_parts_only`：過濾中途的 `function_call` 與 `function_response` JSON 片段，只保留最終生成的純文字與 PNG 圖檔。

---

## 5. Module 2: 企業級觀測與行為審計 (BigQuery & AI.SCORE)

### Step 2a: Enable agent analytics in BigQuery
- **Object Tables 設計**：直接在資料庫儲存大圖檔會造成膨脹與查詢低效；BigQuery Object Table 透過 `OBJ.MAKE_REF(uri, '${REGION}.pitch-connection')` 映射 GCS 檔案，並以 `OBJ.FETCH_METADATA` 擷取元數據。
- **遙測服務**：`BigQueryAnalyticsService` 實作 `record_telemetry` 與 `register_key_visuals`，強制校驗圖片 URI 必須以 `gs://` 開頭。

### Step 2b: Identify brand drift in BigQuery and tune prompts
- **SQL 多模態審查**：
  ```sql
  SELECT *, IF(brand_fit >= 7, 'on brand', 'needs another pass') AS verdict
  FROM AI.SCORE(TABLE bwg.key_visuals, ...)
  ```
- **閉環自動微調 (Closed-Loop Tuning)**：當檢測到漂移評分 $< 7.0$（`needs another pass`），Python 自動執行 `tune_prompt_and_skill` 移除霓虹、浮水印等違規字眼，並注入品牌正向準則，使重新評分達 $\ge 7.0$。

---

## 6. Module 3: 管線加固與縱深防禦 (Security & HITL)

### Step 3a: Enforce tool policies with PreToolUse lifecycle hooks
- **防禦目標**：防止對抗性提示詞調用惡意指令（如 `exec_shell`、`bash`）或傳入危險參數（如 `../../etc/passwd`）。
- **實作原則**：Fail-closed 阻斷，工具白名單僅允許 `load_skill` 與 `generate_key_visual`；拒絕覆蓋參數（如 `bypass_safety`）、參數長度大於 4096 字元及提示詞注入語句。

### Step 3b: Scrub sensitive PII data from inputs and outputs
- **隱私規範**：行銷簡報常包含聯絡電話、郵箱或機密金鑰。在進入模型或日誌前，透過遞迴正則表達式取代敏感資訊（`[REDACTED_SECRET]`、`[REDACTED_CC]`、`[REDACTED_EMAIL]` 等）。
- **不變性保證**：以深拷貝（Deep Copy）處理嵌套資料結構，確保呼叫端傳入物件不被原地修改，且保留合法日期（`2026-10-15`）與轉換率數據。

### Step 3c: Add human-in-the-loop approval gates
- **ADK 兩階段節點模式**：
  - `approve_concept`：標註 `@node(rerun_on_resume=False)`，在暫停等待時發送 `RequestInput`，恢復執行後不重複發送。
  - `user_approval`：標註 `@node(rerun_on_resume=True)`，在恢復時重新評估審核意見，若使用者輸入不是肯定意圖（"yes" 或 "y"），拋出 `ValueError` 中斷下游文案與視覺生成，落實 Fail-closed。

---

## 7. Module 4: 規模化與權杖經濟學最佳化 (Tokenomics)

### Step 4a: Implement tokenomics with compression, pruning, and caching
- **三項核心優化手法**：
  1. **Memory Compression（記憶壓縮）**：`compress_memory(turns, keep_recent=2)` 將第 3 輪以前的舊對話壓縮為單一 System Summary，最近 2 輪完整保留。
  2. **History Pruning（歷史修剪）**：`prune_history(turns, max_turns=6, max_tokens=512)` 依據滑動視窗維護最新輪次，若單一訊息超過預算則截斷並標記 `truncated=True`。
  3. **Prompt Caching（提示詞快取）**：以 SHA-256 對靜態品牌指引計算 64 碼 Hash，於有效 TTL 內重複利用快取前綴，避免重複解析。
  4. **Model Tier Routing（模型分級路由）**：提供 `flash`、`pro`、`low`、`high` 與 `local` 5 種模型策略適應不同成本場景。

---

## 8. 自我排錯與 Lab Helper 漸進式引導機制

實驗室內建了由自然語言提示驅動的排錯引擎（`.agents/skills/lab-helper/`）：
- **Tier 1 (Conceptual Nudge)**：提供高層架構設計理念提示。
- **Tier 2 (Targeted File and Symbol Hint)**：指出特定檔案路徑、函式簽名或缺少的變數名稱。
- **Tier 3 (Implementation Blueprint)**：提供具體的程式碼修復範例。
- **Remediation**：若完全卡住，自動進行在位外科手術式修復（保留使用者先前的其他自訂程式碼與後續 guidepost）。

---

## 9. 本地資源檔案索引

- **原始線上教材**：[https://storage.googleapis.com/bwg-track5-demo-guide/index.html](https://storage.googleapis.com/bwg-track5-demo-guide/index.html)
- **本地完整 HTML 離線講義**：[`docs/guide_local.html`](file:///config/Desktop/BuildWithGemini/docs/guide_local.html)（支援深色/淺色主題切換、排版與外部圖片映射）
- **本地提取之原始 Markdown**：[`docs/GUIDE_ORIGINAL.md`](file:///config/Desktop/BuildWithGemini/docs/GUIDE_ORIGINAL.md)
- **本地深度解析與架構綜整**：[`docs/GUIDE_DETAILED_SUMMARY.md`](file:///config/Desktop/BuildWithGemini/docs/GUIDE_DETAILED_SUMMARY.md)
