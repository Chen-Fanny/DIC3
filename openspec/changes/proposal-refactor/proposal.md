# Proposal: Food Guardian Proposal Refactor using OpenSpec

## Why
家庭端食物浪費每年高達 6.31 億公噸（UNEP 2024），占全球食物浪費總量 60%，主要成因在於家庭缺乏對實體冰箱庫存與保存期限的精準掌控。現有生成式 AI 食譜助手缺乏實體庫存同步機制與 ACID 交易防護，容易推薦家中沒有的食材或導致即期食材過期丟棄。本提案旨在透過 OpenSpec 規格驅動開發（Spec-Driven Development, SDD），將 Food Guardian 的系統設計全面重構為具備白名單 Tool-calling、ACID 交易鎖與即期優先權演算法之領域智慧代理人（Domain AI Agent），建立可驗證、可追溯且高可靠的學術與工程基準。

## What Changes
- **新增庫存交易引擎規格 (`specs/inventory-engine/spec.md`)**：定義基於 SQLite BEGIN IMMEDIATE 獨佔鎖的庫存扣除與快照日誌機制，防止超額扣除（Over-deduction）。
- **新增 Agent 智慧調度規格 (`specs/agent-dispatch/spec.md`)**：規範大語言模型白名單工具呼叫（Tool-calling）、8 步驟推理循環與即期優先食譜評分演算法（$S_r = w_1 \cdot U_r + w_2 \cdot C_r - w_3 \cdot |M_r|$）。
- **重構系統設計文件 (`design.md`)**：依據期末專題規範建立包含 ①～⑧ 模組架構圖、Mermaid 決策流程圖與 Algorithm 1 帶行號虛擬碼。
- **任務拆解與驗證矩陣 (`tasks.md`)**：制定包含 30+ 極限測試劇本與三組基準線（Random / Pure LLM / Our Agent）的實驗驗證排程。

## Capabilities

### New Capabilities
- `inventory-engine`: 提供 ACID 保證的食材庫存狀態轉移（active/consumed/wasted）、日期區間劃分與快照稽核記錄。
- `agent-dispatch`: 提供白名單 Tool-calling 意圖解析、即期食材優先權排程演算法及自然語言可解釋推薦生成。

### Modified Capabilities
- *(無既有規格，此為專案初始化變更)*

## Impact
- **系統模組**：影響 `src/services/inventory_service.py`、`src/agent/agent_core.py`、`src/db/db_manager.py`。
- **文件規範**：全面對齊「期末專題提案作業規範」與 A-Z SOP 學術規範。
