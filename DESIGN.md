# Food Guardian: 系統設計文件 (DESIGN.md)
> **領域智慧代理人 (Domain AI Agent) 系統架構、元件劃分、資料流與演算法規範**
> 依據「A-Z SOP 標準學術論文架構規範」與「期末專題提案作業規範」編撰

---

## 1. 系統工程架構與定位 (System Architecture & Positioning)

Food Guardian 定位為一款整合本機實體庫存管理、日期提醒與 AI 食材動態調配的 **Domain AI Agent（領域智慧代理人）**。
系統採用高內聚、低耦合的模組化單體式架構（Modular Monolithic Architecture），區分為四個主要層次：
1. **展示層 (UI Layer)**：Streamlit 互動介面，負責自然語言對話、庫存清單視覺化、即期預警儀表板與一鍵扣除操作。
2. **業務服務層 (Service Layer)**：包含 `inventory_service.py`、`reminder_service.py` 與 `recipe_service.py`，負責庫存狀態維護、時區邊界計算、多目標啟發式排序與食材缺料計算。
3. **資料庫管理層 (Database Manager Layer)**：`db_manager.py` 封裝 SQLite 資料庫操作，採用 `BEGIN IMMEDIATE` 獨佔寫入鎖機制，保證微觀層面的 ACID 交易一致性。
4. **Agent 工具封裝層 (Agent Adapter Layer)**：`agent_core.py` 實現白名單 Tool-calling 介面，將大語言模型（LLM）的語意解析能力與確定性資料庫服務嚴密連結，杜絕「幽靈食材」幻覺。

---

## 2. 8 步驟 Domain AI Agent 處理流程 (8-Step Agent Pipeline)

系統核心推理流程嚴格對齊專題提案規範之 8 步驟標準架構：

```text
① User/Application Scenario (使用者輸入需求與冰箱庫存查詢)
          │
          ▼
② Query/Input (解析自然語言 Prompt 與約束條件：如即期優先、免採買)
          │
          ▼
③ Planner & Reasoning Agent (FoodGuardianAgent 啟動，決定呼叫何種 Tools)
          │
          ▼
④ Data Acquisition / Retrieval (執行 query_expiring_items 與 query_active_inventory)
          │
          ▼
⑤ Analytics Engine (計算庫存覆蓋率 C_r、即期利用量 U_r 與缺料 M_r，進行評分 S_r)
          │
          ▼
⑥ Domain Knowledge Integration (對照 food_knowledge 規則，檢核 Use By / Best Before)
          │
          ▼
⑦ Decision / Recommendation (產生最優推薦食譜、即期消耗明細與缺少食材清單)
          │
          ▼
⑧ Output & Interaction (Streamlit 渲染結構化結果，提供使用者一鍵核銷庫存)
```

---

## 3. 模組職責劃分表 (Component Responsibility Matrix)

| 模組編號 | 模組名稱 | 原始碼路徑 | 核心職責 | 輸入 / 輸出 |
| :--- | :--- | :--- | :--- | :--- |
| **①** | UI Layer | `app.py`, `src/ui/` | 渲染對話介面、食材儀表板、手動盤點表格 | Input: 使用者點擊/自然語言<br>Output: 渲染畫面 |
| **②** | Service Layer | `src/services/` | 業務規則執行、即期計算、食譜候選多目標排程 | Input: 查詢條件<br>Output: 排序候選集 |
| **③** | DB Manager | `src/db/` | 封裝 SQLite 連線，執行 `BEGIN IMMEDIATE` 交易鎖 | Input: SQL 查詢/更新<br>Output: 資料集/交易狀態 |
| **④** | Agent Adapter | `src/agent/` | 白名單工具轉發、Prompt 範本注入、輸出格式化 | Input: 使用者意圖<br>Output: 結構化推薦物件 |
| **⑤** | Domain Rules | `src/domain/` | USDA FoodKeeper 食品保存期標準、日期標籤語意 | Input: 食材名稱/類別<br>Output: 安全邊界門檻 |

---

## 4. 資料庫綱要與狀態轉移模型 (Database Schema & State Transitions)

### 4.1 資料庫實體表
1. **`inventory_items`**：實體庫存項目表
   - `id`: 主鍵 (INTEGER)
   - `item_name`: 食材名稱 (TEXT)
   - `quantity`: 數量 (DECIMAL)
   - `unit`: 單位 (TEXT)
   - `purchase_date`: 購買日期 (DATE)
   - `expiry_date`: 有效日期 (DATE)
   - `storage_method`: 保存方式 (TEXT: fridge, freezer, pantry)
   - `status`: 狀態 (TEXT: `active`, `consumed`, `wasted`)
2. **`consumption_waste_logs`**：消耗與浪費稽核快照日誌表
   - `id`: 主鍵 (INTEGER)
   - `item_id`: 關聯食材 ID (可為 NULL，刪除不影響歷史快照)
   - `item_name_snapshot`: 處置當時名稱快照 (TEXT)
   - `quantity_snapshot`: 處置當時數量快照 (DECIMAL)
   - `unit_snapshot`: 處置當時單位快照 (TEXT)
   - `event_type`: 事件類別 (`consumed`, `wasted`)
   - `reason`: 棄置或消耗原因 (TEXT)
   - `timestamp`: 事件時間戳記 (DATETIME)
3. **`recipe_recommendations`**：食譜庫與知識索引表
   - `id`: 主鍵 (INTEGER)
   - `recipe_name`: 菜名 (TEXT)
   - `required_ingredients`: 必備食材與份量 JSON (TEXT)
   - `cooking_time`: 烹調時間 (INTEGER)
   - `difficulty`: 難易度 (TEXT)

### 4.2 狀態轉移與 ACID 交易鎖防護
```text
[新購入食材] ──(寫入盤點)──> [status = 'active']
                                 │
         ┌───────────────────────┴───────────────────────┐
         │ (烹飪消耗扣減 q <= Q)                         │ (過期腐壞處置)
         ▼                                               ▼
[status = 'consumed']                           [status = 'wasted']
+ 寫入 consumption_waste_logs                  + 寫入 consumption_waste_logs
(記錄 quantity_snapshot, unit_snapshot)         (記錄原因與浪費快照)
```
- **微觀 ACID 交易保證**：
  在執行庫存扣除時，一律使用 `BEGIN IMMEDIATE` 啟動獨佔交易，讀取最新數量 $Q$；若請求扣除量 $q \le Q$，則執行 `UPDATE` 並同步寫入 `consumption_waste_logs`；若 $q > Q$，立即觸發 `ROLLBACK` 並拋出例外，杜絕超額扣除。

---

## 5. 核心演算法虛擬碼 (Algorithm 1: Domain AI Agent)

```text
Algorithm 1: Domain AI Agent for Food Guardian
Input : User Query Q, Inventory Database State S_db, Knowledge Base K_domain
Output: Structured Recommendation R, Tool Execution Logs L

01: Receive user query Q from Streamlit interface
02: Parse query intent I and context constraints (e.g., expiry_priority = True)
03: Initialize agent planner and tool context container L
04: Call tool query_expiring_items(days_threshold = 3) to fetch expiring item set I_exp
05: Call tool query_active_inventory() to fetch full active inventory set I_active
06: Retrieve candidate recipes Recipe_Set from database or recipe service
07: foreach recipe r in Recipe_Set do
08:     Compute Inventory Coverage C_r = |Ingredients(r) ∩ I_active| / |Ingredients(r)|
09:     Compute Expiring Utilization U_r = |Ingredients(r) ∩ I_exp|
10:     Compute Missing Ingredients M_r = Ingredients(r) \ I_active
11:     Assign Score S_r = (w1 * U_r) + (w2 * C_r) - (w3 * |M_r|)
12: end for
13: Sort Recipe_Set by Score S_r descending
14: Select top-k candidate recipes R_candidates
15: Filter R_candidates using Domain Knowledge K_domain for food safety and label constraints
16: Construct LLM prompt with R_candidates, I_exp, and missing ingredients list M_r
17: Invoke LLM reasoning engine to produce natural language explanation and rationale
18: Formulate final response structure R = {Recommended_Recipes, Expiring_Used, Missing_Items, Rationale}
19: Append execution traces to log L
20: return R
```

- **計分公式參數定義**：
  $$S_r = (w_1 \cdot U_r) + (w_2 \cdot C_r) - (w_3 \cdot |M_r|)$$
  預設權重配置：$w_1 = 3.0$（即期食材消耗最高權重）、$w_2 = 2.0$（庫存食材匹配率）、$w_3 = 1.5$（缺少食材之採買懲罰項）。

---

## 6. 實驗評估與基準線對比規劃 (Evaluation & Baselines)

### 6.1 三大基準線設計
1. **Baseline 1 (Random Selection)**：隨機從現有食譜庫中抽取 3 道菜餚，不考慮即期與庫存覆蓋率。
2. **Baseline 2 (Pure LLM Generative)**：僅將使用者 Prompt 傳入純大語言模型生成食譜，不掛載實體資料庫 Tool-calling。
3. **Baseline 3 (Our Method - Food Guardian Agent)**：完整執行 8 步驟流程與 Algorithm 1 啟發式評分。

### 6.2 評估指標 (Evaluation Metrics)
- **庫存盤點正確率 (Inventory Accuracy %)**：$\frac{\text{系統記錄數量正確項目數}}{\text{實體盤點總項目數}} \times 100\%$
- **即期食材利用率 (Expiring Utilization Rate %)**：$\frac{\text{推薦食譜所消耗之即期食材數量}}{\text{目前處於即期狀態食材總數}} \times 100\%$
- **家庭食材廢棄降幅 (Waste Reduction %)**：對比引入系統前後 30 天滾動紀錄之廢棄總重量變化率。
- **幻覺食材比率 (Hallucination Rate %)**：推薦結果中要求使用家中既無庫存亦未標註於缺料清單的虛構食材比例（目標：0%）。
