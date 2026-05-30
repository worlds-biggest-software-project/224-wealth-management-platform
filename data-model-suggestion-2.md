# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Wealth Management Platform · Created: 2026-05-20

## Philosophy

This model treats every state change as an immutable event in a single, append-only event store. The current state of any entity (portfolio, position, client profile, compliance record) is derived by replaying its event stream. Read-optimised materialised views (projections) serve queries, while the event store is the single source of truth. This is the CQRS (Command Query Responsibility Segregation) pattern applied to wealth management.

Event sourcing is already the dominant architecture at major investment banks for regulatory compliance and risk management. Every trade, modification, settlement, rebalance, and client interaction is captured as an immutable event, providing the complete audit trail that MiFID II (5-year WORM retention), Reg BI, and GIPS demand by design rather than as an afterthought. The system can answer temporal queries like "what was this client's portfolio worth on March 15th?" or "what was the compliance status of this trade at the time it was approved?" by replaying events to any point in time.

The trade-off is implementation complexity. Event replay, projection management, eventual consistency between write and read models, and the learning curve for developers unfamiliar with event sourcing are real costs. But for a platform where regulators may demand full historical reconstruction and non-repudiation, event sourcing provides guarantees that no other architecture can match.

**Best for:** Platforms operating under strict regulatory scrutiny (MiFID II, Reg BI, DORA) where full historical reconstruction, temporal queries, and tamper-evident audit trails are non-negotiable requirements.

**Trade-offs:**
- (+) Complete audit trail is intrinsic — not bolted on after the fact
- (+) Temporal queries ("state as of date X") are natural event replays
- (+) Events can feed multiple read models: operational views, analytics, compliance reports
- (+) New projections can be built retroactively from the full event history
- (+) Naturally supports regulatory WORM (Write Once Read Many) requirements
- (-) Higher implementation complexity — developers must understand event sourcing patterns
- (-) Eventual consistency between event store and read models requires careful handling
- (-) Event schema evolution over years requires versioning strategy
- (-) Debugging requires event replay rather than direct row inspection
- (-) Storage grows continuously (though append-only is cheap on modern infrastructure)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| MiFID II | 5-year WORM record retention satisfied by immutable event store; full transaction reconstruction |
| SEC Reg BI | Every suitability decision and conflict disclosure is a recorded event with causal chain |
| GIPS 2020 | Performance returns calculated by replaying portfolio events; composite membership is event-driven |
| DORA | Incident events and vendor risk events captured in the same event infrastructure |
| ISO 20022 | Custodian message events stored in ISO 20022-aligned structure |
| FIX Protocol | Trade execution events map to FIX message lifecycle (NewOrderSingle, ExecutionReport) |
| OpenWealth API | Read models (projections) shaped to serve OpenWealth-compliant API responses |
| FDX v6.5 | Investment holding and transaction projections align with FDX data clusters |

---

## Event Store (Write Side)

```sql
-- The single source of truth: an append-only event log
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                 -- aggregate root ID (e.g. account_id, client_id)
    stream_type     TEXT NOT NULL,                  -- e.g. 'Account', 'Client', 'TradeOrder', 'Portfolio'
    event_type      TEXT NOT NULL,                  -- e.g. 'AccountOpened', 'TradeExecuted', 'PositionUpdated'
    event_version   INTEGER NOT NULL,              -- per-stream sequence number for ordering
    firm_id         UUID NOT NULL,                 -- tenant isolation
    caused_by       UUID,                          -- user or system that triggered the event
    correlation_id  UUID,                          -- links related events across aggregates
    causation_id    UUID,                          -- the event that caused this event
    event_data      JSONB NOT NULL,                -- the event payload
    metadata        JSONB DEFAULT '{}',            -- context: IP, user-agent, source system
    occurred_at     TIMESTAMPTZ NOT NULL,          -- when the real-world event happened
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(), -- when stored (system time)
    schema_version  SMALLINT NOT NULL DEFAULT 1,   -- for event schema evolution
    UNIQUE (stream_id, event_version)              -- optimistic concurrency control
);

-- Primary query patterns
CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_firm ON event_store(firm_id);
CREATE INDEX idx_event_type ON event_store(event_type);
CREATE INDEX idx_event_occurred ON event_store(occurred_at);
CREATE INDEX idx_event_correlation ON event_store(correlation_id);

-- Partition by month for manageability (events are never deleted)
-- CREATE TABLE event_store PARTITION BY RANGE (recorded_at);
```

## Event Type Catalogue

```sql
-- Registry of all known event types with their JSON schemas
CREATE TABLE event_type_registry (
    event_type      TEXT PRIMARY KEY,
    stream_type     TEXT NOT NULL,
    description     TEXT NOT NULL,
    json_schema     JSONB NOT NULL,                -- JSON Schema for validation
    current_version SMALLINT NOT NULL DEFAULT 1,
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Example Event Types and Payloads

```
-- Client Lifecycle Events
ClientOnboarded:
{
    "client_id": "uuid",
    "firm_id": "uuid",
    "client_type": "individual",
    "first_name": "John",
    "last_name": "Smith",
    "email": "john@example.com",
    "country": "US",
    "client_category": "retail",
    "kyc_status": "pending"
}

ClientKYCVerified:
{
    "client_id": "uuid",
    "verified_by": "uuid",
    "verification_method": "document_review",
    "expiry_date": "2027-05-20"
}

SuitabilityAssessed:
{
    "client_id": "uuid",
    "assessed_by": "uuid",
    "investment_objective": "growth",
    "risk_tolerance": "moderate_high",
    "time_horizon": "long",
    "annual_income_range": "150000-250000",
    "conflicts_disclosed": true,
    "basis_for_recommendation": "Client has 20+ year horizon..."
}

-- Account Lifecycle Events
AccountOpened:
{
    "account_id": "uuid",
    "client_id": "uuid",
    "custodian_id": "uuid",
    "account_number": "SCHW-12345",
    "account_type": "individual",
    "tax_status": "taxable",
    "base_currency": "USD",
    "is_discretionary": true,
    "is_fee_paying": true
}

AccountAssignedToModel:
{
    "account_id": "uuid",
    "model_id": "uuid",
    "assigned_by": "uuid"
}

-- Transaction Events
TradeOrderCreated:
{
    "order_id": "uuid",
    "account_id": "uuid",
    "security_id": "uuid",
    "side": "buy",
    "order_type": "market",
    "quantity": 100,
    "created_by": "uuid"
}

ComplianceCheckPerformed:
{
    "order_id": "uuid",
    "rule_id": "uuid",
    "result": "pass",
    "details": "No restricted securities; concentration within limits"
}

TradeOrderApproved:
{
    "order_id": "uuid",
    "approved_by": "uuid"
}

TradeExecuted:
{
    "order_id": "uuid",
    "account_id": "uuid",
    "security_id": "uuid",
    "side": "buy",
    "quantity": 100,
    "price": 152.35,
    "gross_amount": 15235.00,
    "commission": 4.95,
    "net_amount": 15239.95,
    "trade_date": "2026-05-20",
    "settlement_date": "2026-05-22",
    "fix_exec_id": "EX-20260520-001",
    "custodian_ref": "SCHW-REF-789"
}

-- Position Events (derived from trade executions and market data)
PositionOpened:
{
    "account_id": "uuid",
    "security_id": "uuid",
    "quantity": 100,
    "cost_basis": 15239.95,
    "acquisition_date": "2026-05-20"
}

PositionMarkedToMarket:
{
    "account_id": "uuid",
    "security_id": "uuid",
    "quantity": 100,
    "market_value": 15500.00,
    "price": 155.00,
    "as_of_date": "2026-05-20"
}

-- Rebalancing Events
RebalanceProposed:
{
    "proposal_id": "uuid",
    "model_id": "uuid",
    "accounts": ["uuid1", "uuid2"],
    "trades": [
        {"account_id": "uuid1", "security_id": "uuid", "side": "sell", "quantity": 50, "reason": "drift_correction"},
        {"account_id": "uuid1", "security_id": "uuid", "side": "buy", "quantity": 30, "reason": "target_allocation"}
    ],
    "tax_loss_harvest": true
}

-- Billing Events
FeeCalculated:
{
    "account_id": "uuid",
    "fee_schedule_id": "uuid",
    "period_start": "2026-04-01",
    "period_end": "2026-06-30",
    "billable_aum": 1250000.00,
    "fee_amount": 3125.00
}

-- Performance Events
PerformanceCalculated:
{
    "account_id": "uuid",
    "period_start": "2026-04-01",
    "period_end": "2026-06-30",
    "twr_return": 0.0342,
    "beginning_value": 1200000.00,
    "ending_value": 1241040.00,
    "net_cash_flow": 0.00
}
```

## Snapshot Store (Optimisation)

```sql
-- Periodic snapshots to avoid replaying entire event history
CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    snapshot_version INTEGER NOT NULL,             -- corresponds to event_version
    firm_id         UUID NOT NULL,
    state_data      JSONB NOT NULL,                -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- To rebuild state: load latest snapshot, then replay events from snapshot_version + 1
```

## Read Models (Projections / Query Side)

Projections are materialised views built by processing events. They are disposable and rebuildable. Each projection is optimised for a specific query pattern.

```sql
-- ================================================================
-- PROJECTION: Current client state (for CRM and client portal)
-- ================================================================
CREATE TABLE proj_client (
    id              UUID PRIMARY KEY,
    firm_id         UUID NOT NULL,
    household_id    UUID,
    client_type     TEXT NOT NULL,
    first_name      TEXT,
    last_name       TEXT,
    entity_name     TEXT,
    email           TEXT,
    country         CHAR(2),
    client_category TEXT,
    kyc_status      TEXT NOT NULL,
    kyc_expiry_date DATE,
    risk_tolerance  TEXT,
    investment_objective TEXT,
    total_aum       NUMERIC(18,2),
    last_activity_at TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL,           -- tracks projection currency
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_client_firm ON proj_client(firm_id);

-- ================================================================
-- PROJECTION: Current positions (for portfolio dashboard)
-- ================================================================
CREATE TABLE proj_position (
    account_id      UUID NOT NULL,
    security_id     UUID NOT NULL,
    firm_id         UUID NOT NULL,
    ticker          TEXT,
    security_name   TEXT,
    quantity        NUMERIC(18,6) NOT NULL,
    cost_basis      NUMERIC(18,2) NOT NULL,
    market_value    NUMERIC(18,2),
    unrealised_gain NUMERIC(18,2),
    weight_pct      NUMERIC(7,4),
    as_of_date      DATE NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (account_id, security_id)
);
CREATE INDEX idx_proj_position_firm ON proj_position(firm_id);

-- ================================================================
-- PROJECTION: Account summary (for advisor dashboard)
-- ================================================================
CREATE TABLE proj_account (
    id              UUID PRIMARY KEY,
    firm_id         UUID NOT NULL,
    client_id       UUID NOT NULL,
    client_name     TEXT,
    household_id    UUID,
    account_number  TEXT NOT NULL,
    account_type    TEXT NOT NULL,
    custodian_name  TEXT,
    total_value     NUMERIC(18,2),
    total_cost_basis NUMERIC(18,2),
    total_gain_loss NUMERIC(18,2),
    model_name      TEXT,
    max_drift_pct   NUMERIC(7,4),
    status          TEXT NOT NULL,
    inception_date  DATE NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_account_firm ON proj_account(firm_id);
CREATE INDEX idx_proj_account_client ON proj_account(client_id);

-- ================================================================
-- PROJECTION: Trade order status (for trading desk)
-- ================================================================
CREATE TABLE proj_trade_order (
    id              UUID PRIMARY KEY,
    firm_id         UUID NOT NULL,
    account_id      UUID NOT NULL,
    account_number  TEXT,
    security_id     UUID NOT NULL,
    ticker          TEXT,
    side            TEXT NOT NULL,
    order_type      TEXT NOT NULL,
    quantity        NUMERIC(18,6) NOT NULL,
    limit_price     NUMERIC(18,6),
    status          TEXT NOT NULL,
    compliance_status TEXT,                         -- aggregated from compliance events
    created_by_name TEXT,
    approved_by_name TEXT,
    filled_quantity NUMERIC(18,6),
    filled_price    NUMERIC(18,6),
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_trade_firm ON proj_trade_order(firm_id);
CREATE INDEX idx_proj_trade_status ON proj_trade_order(status);

-- ================================================================
-- PROJECTION: Compliance timeline (for compliance officer dashboard)
-- ================================================================
CREATE TABLE proj_compliance_timeline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    entity_type     TEXT NOT NULL,                  -- 'client', 'trade_order', 'account'
    entity_id       UUID NOT NULL,
    event_type      TEXT NOT NULL,
    summary         TEXT NOT NULL,
    details         JSONB,
    actor_name      TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_compliance_entity ON proj_compliance_timeline(entity_type, entity_id);
CREATE INDEX idx_proj_compliance_firm ON proj_compliance_timeline(firm_id);

-- ================================================================
-- PROJECTION: GIPS composites (for performance reporting)
-- ================================================================
CREATE TABLE proj_composite_performance (
    composite_id    UUID NOT NULL,
    firm_id         UUID NOT NULL,
    composite_name  TEXT NOT NULL,
    period_end      DATE NOT NULL,
    period_type     TEXT NOT NULL,                  -- 'monthly', 'quarterly', 'annual'
    asset_weighted_return NUMERIC(12,8),
    composite_dispersion  NUMERIC(12,8),
    total_assets    NUMERIC(18,2),
    number_of_accounts INTEGER,
    benchmark_return NUMERIC(12,8),
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (composite_id, period_end, period_type)
);

-- ================================================================
-- PROJECTION: Household aggregation (for client portal)
-- ================================================================
CREATE TABLE proj_household (
    id              UUID PRIMARY KEY,
    firm_id         UUID NOT NULL,
    name            TEXT NOT NULL,
    primary_advisor TEXT,
    total_aum       NUMERIC(18,2),
    total_accounts  INTEGER,
    total_clients   INTEGER,
    risk_profile    TEXT,
    ytd_return      NUMERIC(12,8),
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_household_firm ON proj_household(firm_id);
```

## Reference Data (Shared Between Read and Write)

```sql
-- These are traditional relational tables, not event-sourced,
-- because they are reference data that doesn't change per-tenant.

CREATE TABLE ref_security (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticker          TEXT,
    isin            CHAR(12),
    cusip           CHAR(9),
    sedol           CHAR(7),
    name            TEXT NOT NULL,
    security_type   TEXT NOT NULL,
    asset_class     TEXT,
    currency        CHAR(3) NOT NULL,
    exchange        TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ref_security_ticker ON ref_security(ticker);
CREATE INDEX idx_ref_security_isin ON ref_security(isin);

CREATE TABLE ref_security_price (
    security_id     UUID NOT NULL REFERENCES ref_security(id),
    price_date      DATE NOT NULL,
    close_price     NUMERIC(18,6) NOT NULL,
    source          TEXT NOT NULL,
    currency        CHAR(3) NOT NULL,
    PRIMARY KEY (security_id, price_date)
);

CREATE TABLE ref_custodian (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    code            TEXT NOT NULL UNIQUE,
    api_type        TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE ref_currency (
    code            CHAR(3) PRIMARY KEY,
    name            TEXT NOT NULL,
    decimals        SMALLINT NOT NULL DEFAULT 2
);

CREATE TABLE ref_country (
    code            CHAR(2) PRIMARY KEY,
    name            TEXT NOT NULL
);
```

## Projection Management

```sql
-- Tracks the last processed event for each projection (idempotency)
CREATE TABLE projection_checkpoint (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_recorded_at TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    status          TEXT NOT NULL DEFAULT 'running' CHECK (status IN ('running','paused','rebuilding','error')),
    error_message   TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Temporal Query Examples

```sql
-- "What was this account's portfolio worth on March 15, 2026?"
-- Replay all position events up to that date:
SELECT
    e.event_data->>'security_id' AS security_id,
    e.event_data->>'quantity' AS quantity,
    e.event_data->>'market_value' AS market_value
FROM event_store e
WHERE e.stream_id = '<account_id>'
  AND e.stream_type = 'Account'
  AND e.event_type = 'PositionMarkedToMarket'
  AND e.occurred_at <= '2026-03-15T23:59:59Z'
  AND e.event_version = (
      SELECT MAX(e2.event_version)
      FROM event_store e2
      WHERE e2.stream_id = e.stream_id
        AND e2.event_type = 'PositionMarkedToMarket'
        AND e2.event_data->>'security_id' = e.event_data->>'security_id'
        AND e2.occurred_at <= '2026-03-15T23:59:59Z'
  );

-- "Show me the full compliance trail for trade order X"
SELECT
    event_type,
    event_data,
    occurred_at,
    caused_by
FROM event_store
WHERE correlation_id = '<trade_order_correlation_id>'
ORDER BY occurred_at;

-- "What suitability profile did this client have when trade Y was approved?"
WITH trade_approval AS (
    SELECT occurred_at
    FROM event_store
    WHERE stream_id = '<trade_order_id>'
      AND event_type = 'TradeOrderApproved'
    LIMIT 1
)
SELECT event_data
FROM event_store, trade_approval
WHERE stream_id = '<client_id>'
  AND event_type = 'SuitabilityAssessed'
  AND occurred_at <= trade_approval.occurred_at
ORDER BY occurred_at DESC
LIMIT 1;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | event_store, event_type_registry, event_snapshot |
| Reference Data | 5 | ref_security, ref_security_price, ref_custodian, ref_currency, ref_country |
| Projections (Read Models) | 7 | proj_client, proj_position, proj_account, proj_trade_order, proj_compliance_timeline, proj_composite_performance, proj_household |
| Infrastructure | 1 | projection_checkpoint |
| **Total** | **16** | Far fewer tables than normalised; complexity lives in event processing logic |

---

## Key Design Decisions

1. **Single event_store table as the source of truth** — all domain events (client, account, trade, compliance, billing, performance) flow into one table. This simplifies backup, replication, retention, and regulatory archival. Partitioning by `recorded_at` manages growth.

2. **Optimistic concurrency via (stream_id, event_version) uniqueness** — prevents concurrent writes to the same aggregate from producing inconsistent state. If two commands try to write event version 5 for the same stream, one fails and retries.

3. **Correlation and causation IDs** — `correlation_id` links all events that originated from the same user action (e.g. a rebalance proposal generates multiple trade orders, each generating compliance checks). `causation_id` tracks direct parent-child event relationships. Together they enable full causal chain reconstruction for regulators.

4. **Snapshots for performance** — replaying thousands of events for a long-lived account is expensive. Periodic snapshots (e.g. monthly) store the aggregate state at that point, so replay only needs events since the last snapshot.

5. **Projections are disposable and rebuildable** — if a projection schema needs to change (e.g. adding a new column to proj_account), you drop it, create the new table, and replay all events from the beginning. No data migration needed.

6. **Reference data is not event-sourced** — securities, currencies, and countries are shared, slowly-changing reference data. Event sourcing them adds complexity without benefit. They live in traditional relational tables.

7. **Bi-temporal by design** — every event has both `occurred_at` (when the real-world event happened, valid time) and `recorded_at` (when the system stored it, transaction time). This naturally supports bi-temporal queries required for backdated corrections and regulatory reporting.

8. **Schema versioning on events** — the `schema_version` field on each event supports event evolution over the platform's lifetime. Upcasters transform old event versions to current schema during replay.

9. **Firm-level tenant isolation** — `firm_id` on every event enables RLS policies on the event store itself. A firm can only see its own events.

10. **Natural compliance trail** — MiFID II requires 5 years of WORM records. The event store is append-only by design — events are never updated or deleted. Compliance officers query the event store directly rather than relying on a separate audit log that might drift from truth.
