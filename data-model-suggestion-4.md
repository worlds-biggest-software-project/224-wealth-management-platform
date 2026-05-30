# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Wealth Management Platform · Created: 2026-05-20

## Philosophy

This model combines traditional relational tables for transactional data (trades, positions, billing, compliance) with a property graph layer for modelling the complex ownership structures, relationships, and hierarchies that are central to wealth management. The graph layer uses PostgreSQL tables (`graph_node` and `graph_edge`) rather than a separate graph database, keeping the entire system in one database while enabling graph traversal queries.

Wealth management is fundamentally a relationship-heavy domain. Addepar's entire data model is built around an ownership graph where positions are edges between entities. A family office might have an individual who is a beneficiary of two trusts, a managing member of an LLC, a joint account holder with a spouse, and a board member of a foundation — all with different ownership percentages, tax implications, and reporting requirements. Households aggregate across these structures. Compliance officers need to trace conflicts of interest across relationship chains. Model portfolios create strategy-to-account assignments. Advisor teams share clients.

A relational model can represent these relationships, but the queries become deeply nested joins. A graph model makes multi-hop traversals (e.g. "show me all entities controlled by this family, directly or indirectly") natural using recursive CTEs or, for more complex patterns, the Apache AGE PostgreSQL extension.

**Best for:** Platforms serving family offices, ultra-high-net-worth clients, and institutional wealth managers where complex ownership structures, multi-entity hierarchies, and relationship analysis are core to the value proposition.

**Trade-offs:**
- (+) Complex ownership queries (multi-hop traversals, indirect ownership) are natural
- (+) Household aggregation across any depth of entity hierarchy is a graph traversal
- (+) Conflict-of-interest analysis and related-party detection are graph pattern matches
- (+) Adding new relationship types requires only new edge types, not schema changes
- (+) Aligns with Addepar's entity-graph model — the market leader in complex portfolios
- (-) Graph query patterns have a learning curve for developers used to pure SQL
- (-) Graph traversal performance depends on relationship density and depth
- (-) Two conceptual models (relational + graph) increase cognitive load
- (-) Reporting tools often struggle with graph structures — need materialised views
- (-) Without a dedicated graph database, very large graphs (millions of nodes) may need optimisation

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenWealth API | Customer management domain maps to graph nodes; custody positions map to graph edges (entity-to-entity ownership) |
| FDX v6.5 | Investment holdings projected from graph edges into FDX-compliant flat structures |
| ISO 20022 | Securities and counterparties as graph nodes with ISO 20022 identifiers |
| GIPS 2020 | Composite membership modelled as edges; composite returns in relational tables |
| ISO 3166 / ISO 4217 | Jurisdiction and currency as node properties |
| MiFID II / Reg BI | Suitability and compliance as relational tables; related-party conflicts detected via graph queries |
| LEI (ISO 17442) | Legal Entity Identifier as a node property for institutional entities |

---

## Graph Layer

```sql
-- ================================================================
-- PROPERTY GRAPH: Nodes and Edges in PostgreSQL
-- ================================================================

-- Every entity in the system is a graph node
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    node_type       TEXT NOT NULL CHECK (node_type IN (
        'individual','joint','trust','corporation','partnership','llc',
        'foundation','estate','household','account','model_portfolio',
        'security','advisor','team','composite'
    )),
    label           TEXT NOT NULL,                  -- human-readable name
    properties      JSONB NOT NULL DEFAULT '{}',   -- type-specific attributes
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_gnode_firm ON graph_node(firm_id);
CREATE INDEX idx_gnode_type ON graph_node(node_type);
CREATE INDEX idx_gnode_props_gin ON graph_node USING gin(properties);

-- Relationships between entities
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    from_node_id    UUID NOT NULL REFERENCES graph_node(id),
    to_node_id      UUID NOT NULL REFERENCES graph_node(id),
    edge_type       TEXT NOT NULL CHECK (edge_type IN (
        -- Ownership and control
        'owns','beneficiary_of','trustee_of','manages','controls',
        'joint_owner','member_of','shareholder_of',
        -- Account relationships
        'holds_account','custodied_at','assigned_model',
        -- People relationships
        'advisor_for','spouse_of','parent_of','child_of',
        'related_to','employed_by',
        -- Portfolio structure
        'position_in','member_of_composite',
        -- Organisational
        'reports_to','team_member'
    )),
    properties      JSONB NOT NULL DEFAULT '{}',
    /* Example properties by edge type:
    
    owns: {"ownership_pct": 50.0, "voting_pct": 50.0, "effective_date": "2020-01-01"}
    beneficiary_of: {"benefit_type": "income", "distribution_pct": 33.33, "contingent": false}
    trustee_of: {"role": "co-trustee", "appointed_date": "2020-06-01"}
    holds_account: {"relationship": "primary", "authorized_trader": true}
    position_in: {"quantity": 100, "cost_basis": 15000, "market_value": 16500, "as_of_date": "2026-05-20"}
    advisor_for: {"relationship": "primary", "since": "2023-01-15"}
    assigned_model: {"assigned_date": "2026-01-01", "assigned_by": "uuid"}
    member_of_composite: {"start_date": "2026-01-01", "end_date": null}
    spouse_of: {"marriage_date": "2005-08-20"}
    */
    valid_from      DATE NOT NULL DEFAULT CURRENT_DATE,
    valid_to        DATE,                          -- null = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_gedge_firm ON graph_edge(firm_id);
CREATE INDEX idx_gedge_from ON graph_edge(from_node_id);
CREATE INDEX idx_gedge_to ON graph_edge(to_node_id);
CREATE INDEX idx_gedge_type ON graph_edge(edge_type);
CREATE INDEX idx_gedge_valid ON graph_edge(valid_from, valid_to);
CREATE INDEX idx_gedge_props_gin ON graph_edge USING gin(properties);

-- Composite index for common traversal pattern
CREATE INDEX idx_gedge_from_type ON graph_edge(from_node_id, edge_type) WHERE valid_to IS NULL;
CREATE INDEX idx_gedge_to_type ON graph_edge(to_node_id, edge_type) WHERE valid_to IS NULL;
```

### Example Node Properties by Type

```
-- Individual node:
{
    "first_name": "John",
    "last_name": "Smith",
    "date_of_birth": "1965-03-15",
    "email": "john@example.com",
    "country": "US",
    "tax_id_encrypted": "...",
    "client_category": "retail",
    "kyc_status": "verified",
    "kyc_expiry": "2027-03-15",
    "risk_tolerance": "moderate_high",
    "investment_objective": "growth",
    "lei": null
}

-- Trust node:
{
    "trust_name": "Smith Family Irrevocable Trust",
    "trust_type": "irrevocable",
    "formation_date": "2018-06-01",
    "formation_jurisdiction": "DE",
    "tax_id_encrypted": "...",
    "lei": null,
    "distribution_standard": "HEMS"
}

-- Corporation node:
{
    "entity_name": "Smith Holdings LLC",
    "formation_date": "2015-01-15",
    "formation_jurisdiction": "DE",
    "tax_id_encrypted": "...",
    "lei": "549300EXAMPLE000001",
    "entity_type": "limited_liability_company"
}

-- Account node:
{
    "account_number": "SCHW-12345",
    "account_type": "individual",
    "tax_status": "taxable",
    "base_currency": "USD",
    "inception_date": "2020-01-15",
    "custodian_id": "uuid",
    "is_discretionary": true,
    "is_fee_paying": true,
    "status": "active"
}

-- Household node:
{
    "service_tier": "platinum",
    "primary_advisor_id": "uuid",
    "total_aum": 12500000,
    "annual_review_month": 3
}

-- Security node:
{
    "ticker": "AAPL",
    "isin": "US0378331005",
    "cusip": "037833100",
    "security_type": "equity",
    "currency": "USD",
    "asset_class": "us_large_cap_equity",
    "exchange": "NASDAQ"
}
```

## User & Authentication (Relational)

```sql
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    graph_node_id   UUID REFERENCES graph_node(id), -- links to advisor/team node
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL,
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    auth_provider   TEXT DEFAULT 'local',
    auth_subject    TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, email)
);
CREATE INDEX idx_user_firm ON app_user(firm_id);
CREATE INDEX idx_user_node ON app_user(graph_node_id);
```

## Transactional Data (Relational)

```sql
-- Securities reference data (relational for joins with transactions)
CREATE TABLE security (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    graph_node_id   UUID REFERENCES graph_node(id), -- optional link to graph
    ticker          TEXT,
    isin            CHAR(12),
    cusip           CHAR(9),
    name            TEXT NOT NULL,
    security_type   TEXT NOT NULL,
    currency        CHAR(3) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_security_ticker ON security(ticker);
CREATE INDEX idx_security_isin ON security(isin);
CREATE INDEX idx_security_node ON security(graph_node_id);

CREATE TABLE security_price (
    security_id     UUID NOT NULL REFERENCES security(id),
    price_date      DATE NOT NULL,
    close_price     NUMERIC(18,6) NOT NULL,
    source          TEXT NOT NULL,
    currency        CHAR(3) NOT NULL,
    PRIMARY KEY (security_id, price_date)
);

-- Transactions are relational: they need strong typing, constraints, and fast range queries
CREATE TABLE transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    account_node_id UUID NOT NULL REFERENCES graph_node(id),
    security_id     UUID REFERENCES security(id),
    transaction_type TEXT NOT NULL CHECK (transaction_type IN (
        'buy','sell','dividend','interest','contribution','withdrawal',
        'transfer_in','transfer_out','fee','tax','split','merger',
        'reinvestment','return_of_capital','other'
    )),
    trade_date      DATE NOT NULL,
    settlement_date DATE,
    quantity        NUMERIC(18,6),
    price           NUMERIC(18,6),
    gross_amount    NUMERIC(18,2) NOT NULL,
    commission      NUMERIC(18,2) DEFAULT 0,
    fees            NUMERIC(18,2) DEFAULT 0,
    net_amount      NUMERIC(18,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    custodian_ref   TEXT,
    details         JSONB DEFAULT '{}',
    is_cancelled    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_txn_account ON transaction(account_node_id);
CREATE INDEX idx_txn_firm ON transaction(firm_id);
CREATE INDEX idx_txn_trade_date ON transaction(trade_date);

-- Tax lots
CREATE TABLE lot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_node_id UUID NOT NULL REFERENCES graph_node(id),
    security_id     UUID NOT NULL REFERENCES security(id),
    open_txn_id     UUID NOT NULL REFERENCES transaction(id),
    close_txn_id    UUID REFERENCES transaction(id),
    quantity        NUMERIC(18,6) NOT NULL,
    remaining_qty   NUMERIC(18,6) NOT NULL,
    cost_basis      NUMERIC(18,2) NOT NULL,
    acquisition_date DATE NOT NULL,
    disposition_date DATE,
    realised_gain   NUMERIC(18,2),
    is_short_term   BOOLEAN,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_lot_account ON lot(account_node_id);

-- Trade orders
CREATE TABLE trade_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    account_node_id UUID NOT NULL REFERENCES graph_node(id),
    security_id     UUID NOT NULL REFERENCES security(id),
    side            TEXT NOT NULL CHECK (side IN ('buy','sell')),
    order_type      TEXT NOT NULL CHECK (order_type IN ('market','limit','stop','stop_limit')),
    quantity        NUMERIC(18,6) NOT NULL,
    limit_price     NUMERIC(18,6),
    status          TEXT NOT NULL DEFAULT 'pending',
    execution       JSONB DEFAULT '{}',
    compliance      JSONB DEFAULT '{}',
    created_by      UUID NOT NULL REFERENCES app_user(id),
    approved_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_order_firm ON trade_order(firm_id);
CREATE INDEX idx_order_account ON trade_order(account_node_id);
CREATE INDEX idx_order_status ON trade_order(status);
```

## Compliance (Relational)

```sql
CREATE TABLE suitability_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    client_node_id  UUID NOT NULL REFERENCES graph_node(id),
    assessed_by     UUID NOT NULL REFERENCES app_user(id),
    assessment_date DATE NOT NULL,
    profile         JSONB NOT NULL,
    /* Example:
    {
        "investment_objective": "growth",
        "risk_tolerance": "moderate_high",
        "time_horizon_years": 20,
        "liquidity_needs": "low",
        "investment_experience": "extensive",
        "regime": "reg_bi",
        "conflicts_disclosed": true,
        "basis_for_recommendation": "..."
    }
    */
    status          TEXT NOT NULL DEFAULT 'active',
    expires_at      DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_suitability_client ON suitability_assessment(client_node_id);

CREATE TABLE compliance_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    name            TEXT NOT NULL,
    rule_type       TEXT NOT NULL,
    parameters      JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE compliance_check (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    trade_order_id  UUID REFERENCES trade_order(id),
    rule_id         UUID NOT NULL REFERENCES compliance_rule(id),
    account_node_id UUID NOT NULL REFERENCES graph_node(id),
    result          TEXT NOT NULL CHECK (result IN ('pass','fail','warning','override')),
    details         TEXT,
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    overridden_by   UUID REFERENCES app_user(id),
    override_reason TEXT
);
CREATE INDEX idx_cc_order ON compliance_check(trade_order_id);
```

## Performance Reporting (Relational, GIPS-Aligned)

```sql
CREATE TABLE account_return (
    account_node_id UUID NOT NULL REFERENCES graph_node(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    twr_return      NUMERIC(12,8) NOT NULL,
    mwr_return      NUMERIC(12,8),
    beginning_value NUMERIC(18,2) NOT NULL,
    ending_value    NUMERIC(18,2) NOT NULL,
    net_cash_flow   NUMERIC(18,2) NOT NULL,
    gross_return    NUMERIC(12,8),
    net_return      NUMERIC(12,8),
    benchmark_return NUMERIC(12,8),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (account_node_id, period_start, period_end)
);

-- Composites are graph nodes; membership is graph edges.
-- But returns are relational for calculation integrity.
CREATE TABLE composite_return (
    composite_node_id UUID NOT NULL REFERENCES graph_node(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    asset_weighted_return NUMERIC(12,8) NOT NULL,
    composite_dispersion NUMERIC(12,8),
    total_assets    NUMERIC(18,2),
    number_of_accounts INTEGER,
    benchmark_return NUMERIC(12,8),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (composite_node_id, period_start, period_end)
);
```

## Billing (Relational)

```sql
CREATE TABLE fee_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    name            TEXT NOT NULL,
    fee_config      JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE billing_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    account_node_id UUID NOT NULL REFERENCES graph_node(id),
    fee_schedule_id UUID NOT NULL REFERENCES fee_schedule(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    billable_aum    NUMERIC(18,2) NOT NULL,
    fee_amount      NUMERIC(18,2) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'calculated',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## CRM & Activities (Relational)

```sql
CREATE TABLE activity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    client_node_id  UUID REFERENCES graph_node(id),
    household_node_id UUID REFERENCES graph_node(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    activity_type   TEXT NOT NULL,
    subject         TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'open',
    details         JSONB DEFAULT '{}',
    scheduled_at    TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_activity_client ON activity(client_node_id);

CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    owner_node_id   UUID REFERENCES graph_node(id), -- can be client, account, household
    document_type   TEXT NOT NULL,
    title           TEXT NOT NULL,
    storage_key     TEXT NOT NULL,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_doc_owner ON document(owner_node_id);
```

## Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    user_id         UUID,
    entity_type     TEXT NOT NULL,                  -- 'graph_node', 'graph_edge', 'transaction', etc.
    entity_id       UUID NOT NULL,
    action          TEXT NOT NULL,
    changes         JSONB,
    context         JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

---

## Graph Query Examples

```sql
-- ================================================================
-- 1. "Show me all entities in this household and their total AUM"
-- ================================================================
WITH RECURSIVE household_entities AS (
    -- Start from the household node
    SELECT
        gn.id,
        gn.node_type,
        gn.label,
        ge.edge_type,
        (ge.properties->>'ownership_pct')::NUMERIC AS ownership_pct,
        1 AS depth
    FROM graph_node gn
    JOIN graph_edge ge ON ge.to_node_id = gn.id
    WHERE ge.from_node_id = '<household_node_id>'
      AND ge.valid_to IS NULL

    UNION ALL

    -- Traverse deeper: entities owned by household members
    SELECT
        gn2.id,
        gn2.node_type,
        gn2.label,
        ge2.edge_type,
        (ge2.properties->>'ownership_pct')::NUMERIC,
        he.depth + 1
    FROM household_entities he
    JOIN graph_edge ge2 ON ge2.from_node_id = he.id
    JOIN graph_node gn2 ON gn2.id = ge2.to_node_id
    WHERE ge2.valid_to IS NULL
      AND ge2.edge_type IN ('owns','controls','member_of')
      AND he.depth < 5  -- prevent infinite recursion
)
SELECT * FROM household_entities;

-- ================================================================
-- 2. "What is the total AUM across all accounts controlled by John Smith?"
--    (directly or indirectly through trusts, LLCs, etc.)
-- ================================================================
WITH RECURSIVE controlled_entities AS (
    SELECT '<john_smith_node_id>'::UUID AS node_id, 1.0::NUMERIC AS effective_ownership
    UNION ALL
    SELECT
        ge.to_node_id,
        ce.effective_ownership * COALESCE((ge.properties->>'ownership_pct')::NUMERIC / 100, 1.0)
    FROM controlled_entities ce
    JOIN graph_edge ge ON ge.from_node_id = ce.node_id
    WHERE ge.edge_type IN ('owns','controls','manages','trustee_of')
      AND ge.valid_to IS NULL
),
account_values AS (
    SELECT
        gn.id AS account_id,
        gn.label AS account_name,
        ce.effective_ownership,
        -- Sum position market values for this account
        COALESCE(SUM((pos.properties->>'market_value')::NUMERIC), 0) AS total_value
    FROM controlled_entities ce
    JOIN graph_edge acct_edge ON acct_edge.from_node_id = ce.node_id
        AND acct_edge.edge_type = 'holds_account'
        AND acct_edge.valid_to IS NULL
    JOIN graph_node gn ON gn.id = acct_edge.to_node_id
        AND gn.node_type = 'account'
    LEFT JOIN graph_edge pos ON pos.from_node_id = gn.id
        AND pos.edge_type = 'position_in'
        AND pos.valid_to IS NULL
    GROUP BY gn.id, gn.label, ce.effective_ownership
)
SELECT
    account_name,
    total_value,
    effective_ownership,
    total_value * effective_ownership AS attributed_value
FROM account_values
ORDER BY attributed_value DESC;

-- ================================================================
-- 3. "Detect potential conflicts of interest for a trade"
--    Find all entities related to the security issuer within 3 hops
-- ================================================================
WITH RECURSIVE related_entities AS (
    SELECT
        gn.id,
        gn.node_type,
        gn.label,
        1 AS hops,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.id = '<security_issuer_node_id>'

    UNION ALL

    SELECT
        gn2.id,
        gn2.node_type,
        gn2.label,
        re.hops + 1,
        re.path || gn2.id
    FROM related_entities re
    JOIN graph_edge ge ON (ge.from_node_id = re.id OR ge.to_node_id = re.id)
    JOIN graph_node gn2 ON gn2.id = CASE
        WHEN ge.from_node_id = re.id THEN ge.to_node_id
        ELSE ge.from_node_id
    END
    WHERE re.hops < 3
      AND NOT gn2.id = ANY(re.path)  -- prevent cycles
      AND ge.valid_to IS NULL
)
SELECT DISTINCT re.id, re.node_type, re.label, re.hops
FROM related_entities re
WHERE re.node_type IN ('individual','trust','corporation')
  AND re.id IN (
      -- Check if any of these entities are firm clients
      SELECT id FROM graph_node WHERE firm_id = '<firm_id>'
  );

-- ================================================================
-- 4. "Show the complete ownership chain from individual to security"
-- ================================================================
WITH RECURSIVE ownership_chain AS (
    SELECT
        ge.from_node_id,
        ge.to_node_id,
        ge.edge_type,
        gn_from.label AS from_label,
        gn_to.label AS to_label,
        (ge.properties->>'ownership_pct')::NUMERIC AS ownership_pct,
        1 AS depth,
        ARRAY[ge.from_node_id] AS path
    FROM graph_edge ge
    JOIN graph_node gn_from ON gn_from.id = ge.from_node_id
    JOIN graph_node gn_to ON gn_to.id = ge.to_node_id
    WHERE ge.from_node_id = '<individual_node_id>'
      AND ge.valid_to IS NULL

    UNION ALL

    SELECT
        ge2.from_node_id,
        ge2.to_node_id,
        ge2.edge_type,
        gn_from2.label,
        gn_to2.label,
        (ge2.properties->>'ownership_pct')::NUMERIC,
        oc.depth + 1,
        oc.path || ge2.from_node_id
    FROM ownership_chain oc
    JOIN graph_edge ge2 ON ge2.from_node_id = oc.to_node_id
    JOIN graph_node gn_from2 ON gn_from2.id = ge2.from_node_id
    JOIN graph_node gn_to2 ON gn_to2.id = ge2.to_node_id
    WHERE ge2.valid_to IS NULL
      AND NOT ge2.from_node_id = ANY(oc.path)
      AND oc.depth < 10
)
SELECT * FROM ownership_chain ORDER BY depth;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_node, graph_edge (replaces ~8 entity tables) |
| User & Auth | 1 | app_user (linked to graph via node_id) |
| Securities & Prices | 2 | security, security_price (also linkable to graph) |
| Transactions | 3 | transaction, lot, trade_order |
| Compliance | 3 | suitability_assessment, compliance_rule, compliance_check |
| Performance | 2 | account_return, composite_return |
| Billing | 2 | fee_schedule, billing_record |
| CRM | 2 | activity, document |
| Audit | 1 | audit_log |
| **Total** | **~18** | Graph layer absorbs client, household, account, advisor, model, composite tables |

---

## Key Design Decisions

1. **Graph for entities, relational for transactions** — entities (clients, trusts, accounts, securities, advisors) are graph nodes because their relationships are the core of wealth management. Transactions, prices, and returns are relational because they need strong typing, constraints, and efficient range queries.

2. **Positions as graph edges** — following Addepar's model, a position is a relationship between an account node and a security node, with quantity, cost basis, and market value as edge properties. This makes portfolio traversal queries natural: "What securities does this household own, across all entities and accounts?"

3. **Temporal edges with valid_from/valid_to** — relationships change over time (clients join and leave households, accounts are reassigned to advisors, composite memberships change). Temporal edges preserve the full history without deleting data, supporting "as-of" queries for compliance.

4. **Unified entity model** — individuals, trusts, corporations, partnerships, LLCs, foundations, estates, and households are all graph_node rows with different node_types. This eliminates the need for separate tables per entity type and makes it trivial to add new entity types.

5. **Effective ownership calculations via graph traversal** — the recursive CTE pattern computes effective ownership through any depth of intermediary entities (individual -> LLC -> trust -> account). This is essential for family office reporting, tax allocation, and regulatory disclosure.

6. **Conflict-of-interest detection as graph pattern matching** — compliance officers can query "are any of our clients related to the issuer of this security within N hops?" This is a natural graph query that would be extremely complex in pure relational SQL.

7. **Graph node IDs as foreign keys in relational tables** — transactions, compliance checks, and returns reference graph_node(id) rather than separate client_id or account_id columns. This keeps the relational layer consistent with the graph layer.

8. **Securities dual-registered** — securities exist as both relational rows (for price joins and transaction constraints) and optionally as graph nodes (for relationship queries like "which accounts hold AAPL?"). The security table has a graph_node_id FK for the link.

9. **JSONB properties on nodes and edges** — each entity type has different attributes (an individual has date_of_birth; a trust has formation_date and distribution_standard). JSONB properties absorb this variability without type-specific tables.

10. **Materialised views for reporting** — graph queries are powerful but can be slow for large-scale aggregation reporting. Critical reports (household AUM, advisor book of business, composite membership) should be materialised on a schedule and served from flat reporting tables.
