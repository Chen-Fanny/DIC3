# Spec Delta: inventory-engine

## Purpose
Provides transactional inventory state management with ACID write locks, expiration status calculation, and snapshot logs for food waste auditability.

## ADDED Requirements

### Requirement: Transactional inventory deduction
The system SHALL execute inventory deductions within an exclusive database write lock using BEGIN IMMEDIATE and Python Decimal arithmetic to prevent over-deduction and race conditions.

#### Scenario: Successful stock deduction
- **WHEN** user or agent requests deduction of quantity $q$ where $q \le Q$ (available stock)
- **THEN** system deducts quantity within transaction, updates item status to consumed if remaining is zero, and records an event snapshot to the consumption log

#### Scenario: Insufficient stock rejection
- **WHEN** deduction request exceeds available stock ($q > Q$)
- **THEN** system rolls back transaction immediately, raises InsufficientStockError, and leaves database state unchanged

### Requirement: Date classification and reminder boundaries
The system SHALL classify stored ingredients into date status buckets (passed_date, expiring_soon, normal, no_date) relative to current system time and threshold parameter $t$.

#### Scenario: Expiring item detection
- **WHEN** item remaining days $d = \text{expiry\_date} - \text{today}$ satisfies $0 \le d \le t$
- **THEN** system marks item as expiring_soon and includes it in expiring query results
