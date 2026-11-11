# Spec Delta: agent-dispatch

## Purpose
Governs the Domain AI Agent reasoning loop, enforcing strict tool-calling whitelisting, multi-objective expiring-first recipe ranking, and explainable recommendations.

## ADDED Requirements

### Requirement: Whitelisted tool execution
The Agent SHALL only interact with backend services through strictly defined, whitelisted tool adapters to prevent hallucinations and ungrounded statements.

#### Scenario: Tool calling with active inventory lookup
- **WHEN** user requests meal suggestions based on current fridge stock
- **THEN** agent executes query_expiring_items and query_active_inventory tools before formulating recipe candidates

### Requirement: Expiring-first multi-objective recipe scoring
The Agent analytics engine SHALL compute recipe priority score using weighted expiring utilization, inventory coverage, and missing ingredients count: $S_r = (w_1 \cdot U_r) + (w_2 \cdot C_r) - (w_3 \cdot |M_r|)$.

#### Scenario: Prioritizing expiring ingredients
- **WHEN** multiple candidate recipes match active inventory
- **THEN** recipe consuming the highest number of expiring items receives highest rank $S_r$ and is recommended first with explicit rationale
