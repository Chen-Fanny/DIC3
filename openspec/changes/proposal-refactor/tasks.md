# Tasks

## 1. Documentation & Specification Alignment

- [x] 1.1 Complete OpenSpec proposal refactor artifacts (`proposal.md`, `specs/`, `design.md`, `tasks.md`) and verify via `openspec validate`
- [x] 1.2 Formulate comprehensive `README.md` adhering to Final Project Proposal requirements (Parts 1 to 5)
- [x] 1.3 Maintain `DESIGN.md` in root directory mapping A-Z SOP requirements to system implementation

## 2. Core Implementation Readiness

- [ ] 2.1 Verify SQLite transactional write lock (`BEGIN IMMEDIATE`) in inventory deduction and run ACID concurrency unit tests
- [ ] 2.2 Wire Agent Adapter tool-calling loop in `src/agent/agent_core.py` to replace NotImplementedError
- [ ] 2.3 Implement multi-objective scoring formula $S_r = (w_1 \cdot U_r) + (w_2 \cdot C_r) - (w_3 \cdot |M_r|)$ in recipe service

## 3. Experimental Validation

- [ ] 3.1 Execute 30+ synthetic boundary test scenarios (out-of-stock, expired, corrupted date, concurrent deduction)
- [ ] 3.2 Benchmark Our Agent against Baseline 1 (Random) and Baseline 2 (Pure LLM) on simulated pantry traces
