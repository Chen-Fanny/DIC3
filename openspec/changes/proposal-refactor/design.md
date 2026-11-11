# Design: Food Guardian System Architecture & OpenSpec SDD Refactor

## Context
See `proposal.md` for project background and motivation. The system transitions from a conceptual prototype to an academically validated and strictly testable Domain AI Agent built with Python, Streamlit, SQLite, and Tool-calling Agent architecture.

## Goals / Non-Goals

**Goals:**
- Implement an 8-step Domain AI Agent execution pipeline connecting Streamlit UI, Service Layer, SQLite DB, and Domain Knowledge.
- Formulate Algorithm 1 with explicit multi-objective scoring formula $S_r = (w_1 \cdot U_r) + (w_2 \cdot C_r) - (w_3 \cdot |M_r|)$.
- Enforce ACID transactional consistency using SQLite `BEGIN IMMEDIATE` locks to eliminate race conditions.
- Structure verifiable test matrices conforming to Final Project Proposal requirements.

**Non-Goals:**
- Full computer vision integration for automated pantry scanning (reserved for phase 2).
- Automatic medical/dietary prescription (the agent strictly defers to official food safety rules).

## System Architecture

The system follows an 8-stage architectural model:
1. `① User/Application Scenario`: Household inventory check and daily cooking query.
2. `② Query/Input`: Natural language intent combined with strict constraints (e.g. expiring first).
3. `③ Planner & Reasoning Agent`: Intent parser and tool execution coordinator.
4. `④ Data Acquisition/Retrieval`: Tool adapters fetching active and expiring items from SQLite.
5. `⑤ Analytics Engine`: Multi-objective recipe matching and priority calculation.
6. `⑥ Domain Knowledge Integration`: Food safety rules (Use By vs Best Before) and shelf-life guidelines.
7. `⑦ Decision/Recommendation`: Ranked meal suggestion with missing ingredients list and audit reasoning.
8. `⑧ Output & Interaction`: Structured Streamlit response with one-click deduction trigger.

## Decisions

### Decision 1: SQLite with `BEGIN IMMEDIATE` over Client-Side Locks
- **Rationale**: Guarantees ACID serialization at the storage engine level. Prevents multi-tab or concurrent request race conditions.
- **Alternatives Considered**: In-memory Python threading locks (fail across process restarts), external PostgreSQL server (unnecessary deployment complexity for local desktop prototype).

### Decision 2: Rule-Based Recipe Pre-Filtering + LLM Rationale Generation
- **Rationale**: Eliminates phantom hallucinated ingredients by filtering valid recipes deterministically before prompting the LLM for explanations.
- **Alternatives Considered**: Pure generative prompting (high hallucination rate, frequently invents ingredients not in the fridge).

## Risks / Trade-offs

- `[Risk] Long LLM inference latency during dinner preparation` → **Mitigation**: Cache deterministic candidate sets locally and invoke LLM with streaming responses.
- `[Risk] Date ambiguity in user inputs` → **Mitigation**: Standardize on ISO-8601 timestamps and default to conservative shelf-life thresholds from USDA FoodKeeper.
