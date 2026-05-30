# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Wealth Management Platform · Created: 2026-05-20

## Philosophy

This model follows classical third-normal-form (3NF) relational design, where every concept in the wealth management domain gets its own table with explicit foreign key relationships. Securities, accounts, positions, transactions, clients, households, advisors, compliance records, and performance data all live in dedicated, strongly typed tables with referential integrity enforced at the database level.

The approach mirrors how enterprise wealth platforms like Orion and Addepar structure their data internally: entities are first-class objects with well-defined relationships. Every field has a single source of truth, every relationship is explicit, and every constraint is enforced by the schema. This model aligns closely with the OpenWealth API standard (custody services, customer management, order placement) and the FDX investment account data model, making it straightforward to implement standards-compliant API endpoints.

The trade-off is schema rigidity. Adding jurisdiction-specific fields, new asset classes, or novel compliance requirements means schema migrations. But for a platform where data integrity, regulatory auditability, and complex cross-entity queries are paramount, this is the strongest foundation.

**Best for:** Teams building a compliance-first platform where data integrity and standards alignment are more important than schema flexibility.

**Trade-offs:**
- (+) Strong referential integrity prevents orphaned or inconsistent data
- (+) Complex cross-entity queries (household aggregation, composite performance) are natural SQL joins
- (+) Standards alignment with OpenWealth, FDX, and GIPS is straightforward
- (+) Mature tooling for migrations, ORMs, and reporting
- (-) Schema migrations required for new fields or asset types
- (-) Higher table count increases initial complexity
- (-) Multi-jurisdiction variations require nullable columns or additional tables
- (-) Performance under extreme write loads requires careful indexing

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenWealth API | Custody services entities (positions, transactions, securities, accounts) map directly to tables |
| FDX v6.5 | Investment account, holding, and transaction schemas inform column definitions |
| ISO 20022 | Securities identifiers (ISIN, CUSIP) and transaction type codes used as reference data |
| GIPS 2020 | Composite and portfolio return tables structured for GIPS-compliant performance reporting |
| ISO 3166 | Country and subdivision codes for jurisdiction modeling |
| ISO 4217 | Currency codes for all monetary fields |
| MiFID II | Client categorisation, suitability assessment, and best-execution fields in dedicated tables |
| SEC Reg BI | Suitability documentation and conflict-of-interest tracking in compliance tables |
| OAuth 2.0 / OIDC | User authentication tables support federated identity |
| FIX Protocol | Trade order fields align with FIX tag definitions for execution integration |

---

## Core Identity & Multi-Tenancy

```sql
-- Firm / tenant isolation
CREATE TABLE firm (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    registration_no TEXT,                          -- e.g. SEC CRD number
    jurisdiction    CHAR(2) NOT NULL,              -- ISO 3166-1 alpha-2
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Row-level security policy pattern (applied to all tenant-scoped tables)
ALTER TABLE firm ENABLE ROW LEVEL SECURITY;

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    password_hash   TEXT,                          -- null if SSO-only
    oidc_subject    TEXT,                          -- OpenID Connect subject
    oidc_issuer     TEXT,                          -- identity provider
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, email)
);
CREATE INDEX idx_app_user_firm ON app_user(firm_id);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    name            TEXT NOT NULL,                 -- e.g. 'advisor', 'compliance_officer', 'client'
    permissions     TEXT[] NOT NULL DEFAULT '{}',   -- e.g. '{portfolio.read, trade.execute}'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, name)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id)
);
```

## Client & Household Management

```sql
CREATE TABLE household (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    name            TEXT NOT NULL,
    primary_advisor_id UUID REFERENCES app_user(id),
    risk_profile    TEXT CHECK (risk_profile IN ('conservative','moderate_conservative','moderate','moderate_aggressive','aggressive')),
    aum_total       NUMERIC(18,2),                 -- denormalised, updated by trigger
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_household_firm ON household(firm_id);
CREATE INDEX idx_household_advisor ON household(primary_advisor_id);

CREATE TABLE client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    household_id    UUID REFERENCES household(id),
    client_type     TEXT NOT NULL CHECK (client_type IN ('individual','joint','entity','trust')),
    first_name      TEXT,
    last_name       TEXT,
    entity_name     TEXT,                          -- for entity/trust clients
    date_of_birth   DATE,
    tax_id_encrypted BYTEA,                        -- encrypted SSN/TIN
    email           TEXT,
    phone           TEXT,
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state_province  TEXT,
    postal_code     TEXT,
    country         CHAR(2) REFERENCES country(code), -- ISO 3166-1
    -- MiFID II / Reg BI classification
    client_category TEXT CHECK (client_category IN ('retail','professional','eligible_counterparty')),
    kyc_status      TEXT NOT NULL DEFAULT 'pending' CHECK (kyc_status IN ('pending','verified','expired','rejected')),
    kyc_verified_at TIMESTAMPTZ,
    kyc_expiry_date DATE,
    user_id         UUID REFERENCES app_user(id),  -- portal login link
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_client_firm ON client(firm_id);
CREATE INDEX idx_client_household ON client(household_id);

CREATE TABLE client_advisor (
    client_id       UUID NOT NULL REFERENCES client(id),
    advisor_id      UUID NOT NULL REFERENCES app_user(id),
    relationship    TEXT NOT NULL DEFAULT 'primary' CHECK (relationship IN ('primary','secondary','team')),
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (client_id, advisor_id)
);
```

## Reference Data

```sql
CREATE TABLE country (
    code            CHAR(2) PRIMARY KEY,           -- ISO 3166-1 alpha-2
    name            TEXT NOT NULL,
    alpha3          CHAR(3) NOT NULL,              -- ISO 3166-1 alpha-3
    numeric_code    CHAR(3)
);

CREATE TABLE currency (
    code            CHAR(3) PRIMARY KEY,           -- ISO 4217
    name            TEXT NOT NULL,
    decimals        SMALLINT NOT NULL DEFAULT 2
);

CREATE TABLE asset_class (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            TEXT NOT NULL UNIQUE,           -- e.g. 'equity', 'fixed_income', 'real_estate'
    name            TEXT NOT NULL,
    parent_id       UUID REFERENCES asset_class(id) -- hierarchical classification
);

CREATE TABLE security (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticker          TEXT,
    isin            CHAR(12),                      -- ISO 6166
    cusip           CHAR(9),
    sedol           CHAR(7),
    figi            CHAR(12),                      -- Bloomberg FIGI
    name            TEXT NOT NULL,
    security_type   TEXT NOT NULL CHECK (security_type IN (
        'equity','bond','mutual_fund','etf','option','future',
        'alternative','real_estate','private_equity','cash_equivalent','other'
    )),
    asset_class_id  UUID REFERENCES asset_class(id),
    currency        CHAR(3) NOT NULL REFERENCES currency(code),
    exchange        TEXT,
    country         CHAR(2) REFERENCES country(code),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_security_ticker ON security(ticker);
CREATE INDEX idx_security_isin ON security(isin);
CREATE INDEX idx_security_cusip ON security(cusip);

CREATE TABLE security_price (
    security_id     UUID NOT NULL REFERENCES security(id),
    price_date      DATE NOT NULL,
    close_price     NUMERIC(18,6) NOT NULL,
    open_price      NUMERIC(18,6),
    high_price      NUMERIC(18,6),
    low_price       NUMERIC(18,6),
    volume          BIGINT,
    source          TEXT NOT NULL,                  -- e.g. 'morningstar', 'bloomberg', 'custodian'
    currency        CHAR(3) NOT NULL REFERENCES currency(code),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (security_id, price_date)
);

CREATE TABLE exchange_rate (
    from_currency   CHAR(3) NOT NULL REFERENCES currency(code),
    to_currency     CHAR(3) NOT NULL REFERENCES currency(code),
    rate_date       DATE NOT NULL,
    rate            NUMERIC(18,8) NOT NULL,
    source          TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (from_currency, to_currency, rate_date)
);
```

## Account & Portfolio Management

```sql
CREATE TABLE account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    custodian_id    UUID REFERENCES custodian(id),
    account_number  TEXT NOT NULL,
    account_name    TEXT NOT NULL,
    account_type    TEXT NOT NULL CHECK (account_type IN (
        'individual','joint','ira_traditional','ira_roth','401k','403b',
        'sep_ira','trust','corporate','partnership','custodial','education_529','other'
    )),
    tax_status      TEXT NOT NULL CHECK (tax_status IN ('taxable','tax_deferred','tax_exempt')),
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD' REFERENCES currency(code),
    inception_date  DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','closed','frozen','pending')),
    is_discretionary BOOLEAN NOT NULL DEFAULT true, -- GIPS: determines composite inclusion
    is_fee_paying    BOOLEAN NOT NULL DEFAULT true,  -- GIPS: composite eligibility
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, account_number)
);
CREATE INDEX idx_account_client ON account(client_id);
CREATE INDEX idx_account_firm ON account(firm_id);

CREATE TABLE custodian (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    code            TEXT NOT NULL UNIQUE,           -- e.g. 'schwab', 'fidelity', 'pershing'
    api_type        TEXT CHECK (api_type IN ('schwab_trader','plaid','fdx','openwealth','sftp','manual')),
    country         CHAR(2) REFERENCES country(code),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE model_portfolio (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    name            TEXT NOT NULL,
    description     TEXT,
    risk_level      TEXT CHECK (risk_level IN ('conservative','moderate_conservative','moderate','moderate_aggressive','aggressive')),
    benchmark_id    UUID REFERENCES security(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE model_portfolio_allocation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id        UUID NOT NULL REFERENCES model_portfolio(id),
    security_id     UUID REFERENCES security(id),
    asset_class_id  UUID REFERENCES asset_class(id),
    target_weight   NUMERIC(7,4) NOT NULL,         -- percentage e.g. 25.0000
    min_weight      NUMERIC(7,4),
    max_weight      NUMERIC(7,4),
    drift_threshold NUMERIC(7,4)                   -- rebalance trigger
);

CREATE TABLE account_model_assignment (
    account_id      UUID NOT NULL REFERENCES account(id),
    model_id        UUID NOT NULL REFERENCES model_portfolio(id),
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    assigned_by     UUID REFERENCES app_user(id),
    PRIMARY KEY (account_id, model_id)
);
```

## Positions & Transactions

```sql
CREATE TABLE position (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id      UUID NOT NULL REFERENCES account(id),
    security_id     UUID NOT NULL REFERENCES security(id),
    quantity        NUMERIC(18,6) NOT NULL,
    cost_basis      NUMERIC(18,2) NOT NULL,
    market_value    NUMERIC(18,2),                 -- updated daily
    unrealised_gain NUMERIC(18,2),                 -- market_value - cost_basis
    as_of_date      DATE NOT NULL,
    currency        CHAR(3) NOT NULL REFERENCES currency(code),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (account_id, security_id, as_of_date)
);
CREATE INDEX idx_position_account ON position(account_id);
CREATE INDEX idx_position_security ON position(security_id);
CREATE INDEX idx_position_date ON position(as_of_date);

CREATE TABLE transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id      UUID NOT NULL REFERENCES account(id),
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
    net_amount      NUMERIC(18,2) NOT NULL,        -- gross - commission - fees
    currency        CHAR(3) NOT NULL REFERENCES currency(code),
    custodian_ref   TEXT,                           -- external reference ID
    description     TEXT,
    is_cancelled    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_transaction_account ON transaction(account_id);
CREATE INDEX idx_transaction_security ON transaction(security_id);
CREATE INDEX idx_transaction_trade_date ON transaction(trade_date);
CREATE INDEX idx_transaction_type ON transaction(transaction_type);

CREATE TABLE lot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id      UUID NOT NULL REFERENCES account(id),
    security_id     UUID NOT NULL REFERENCES security(id),
    open_transaction_id UUID NOT NULL REFERENCES transaction(id),
    close_transaction_id UUID REFERENCES transaction(id),
    quantity        NUMERIC(18,6) NOT NULL,
    remaining_qty   NUMERIC(18,6) NOT NULL,
    cost_basis      NUMERIC(18,2) NOT NULL,
    acquisition_date DATE NOT NULL,
    disposition_date DATE,
    realised_gain   NUMERIC(18,2),
    is_short_term   BOOLEAN,                       -- based on holding period
    wash_sale_flag  BOOLEAN DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_lot_account_security ON lot(account_id, security_id);
```

## Trading & Rebalancing

```sql
CREATE TABLE trade_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    account_id      UUID NOT NULL REFERENCES account(id),
    security_id     UUID NOT NULL REFERENCES security(id),
    order_type      TEXT NOT NULL CHECK (order_type IN ('market','limit','stop','stop_limit')),
    side            TEXT NOT NULL CHECK (side IN ('buy','sell')),
    quantity        NUMERIC(18,6) NOT NULL,
    limit_price     NUMERIC(18,6),
    stop_price      NUMERIC(18,6),
    time_in_force   TEXT NOT NULL DEFAULT 'day' CHECK (time_in_force IN ('day','gtc','ioc','fok')),
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending','compliance_review','approved','submitted','partial_fill',
        'filled','cancelled','rejected'
    )),
    -- FIX Protocol alignment
    fix_cl_ord_id   TEXT,                          -- FIX tag 11
    fix_exec_id     TEXT,                          -- FIX tag 17
    submitted_at    TIMESTAMPTZ,
    filled_at       TIMESTAMPTZ,
    filled_quantity NUMERIC(18,6),
    filled_price    NUMERIC(18,6),
    created_by      UUID NOT NULL REFERENCES app_user(id),
    approved_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_trade_order_account ON trade_order(account_id);
CREATE INDEX idx_trade_order_status ON trade_order(status);

CREATE TABLE rebalance_proposal (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    name            TEXT,
    model_id        UUID REFERENCES model_portfolio(id),
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN ('draft','review','approved','executing','completed','cancelled')),
    tax_loss_harvest BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    approved_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rebalance_proposal_trade (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    proposal_id     UUID NOT NULL REFERENCES rebalance_proposal(id),
    account_id      UUID NOT NULL REFERENCES account(id),
    security_id     UUID NOT NULL REFERENCES security(id),
    side            TEXT NOT NULL CHECK (side IN ('buy','sell')),
    quantity        NUMERIC(18,6) NOT NULL,
    estimated_amount NUMERIC(18,2),
    reason          TEXT,                           -- e.g. 'drift_correction', 'tax_loss_harvest'
    trade_order_id  UUID REFERENCES trade_order(id) -- linked when executed
);
```

## Performance & Reporting (GIPS-Aligned)

```sql
CREATE TABLE account_return (
    account_id      UUID NOT NULL REFERENCES account(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    twr_return      NUMERIC(12,8) NOT NULL,        -- time-weighted return
    mwr_return      NUMERIC(12,8),                 -- money-weighted return (IRR)
    beginning_value NUMERIC(18,2) NOT NULL,
    ending_value    NUMERIC(18,2) NOT NULL,
    net_cash_flow   NUMERIC(18,2) NOT NULL,
    gross_return    NUMERIC(12,8),                 -- before fees
    net_return      NUMERIC(12,8),                 -- after fees
    benchmark_return NUMERIC(12,8),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (account_id, period_start, period_end)
);

-- GIPS composite structure
CREATE TABLE composite (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    name            TEXT NOT NULL,
    description     TEXT,
    inception_date  DATE NOT NULL,
    termination_date DATE,
    benchmark_id    UUID REFERENCES security(id),
    strategy        TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE composite_membership (
    composite_id    UUID NOT NULL REFERENCES composite(id),
    account_id      UUID NOT NULL REFERENCES account(id),
    start_date      DATE NOT NULL,
    end_date        DATE,
    reason_joined   TEXT,
    reason_left     TEXT,
    PRIMARY KEY (composite_id, account_id, start_date)
);

CREATE TABLE composite_return (
    composite_id    UUID NOT NULL REFERENCES composite(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    asset_weighted_return NUMERIC(12,8) NOT NULL,
    equal_weighted_return NUMERIC(12,8),
    composite_dispersion  NUMERIC(12,8),           -- GIPS required
    total_assets    NUMERIC(18,2),
    number_of_accounts INTEGER,
    benchmark_return NUMERIC(12,8),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (composite_id, period_start, period_end)
);
```

## Compliance & Suitability

```sql
CREATE TABLE suitability_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    assessed_by     UUID NOT NULL REFERENCES app_user(id),
    assessment_date DATE NOT NULL,
    -- Client profile
    investment_objective TEXT NOT NULL CHECK (investment_objective IN (
        'capital_preservation','income','growth_income','growth','aggressive_growth','speculation'
    )),
    risk_tolerance  TEXT NOT NULL CHECK (risk_tolerance IN ('low','moderate_low','moderate','moderate_high','high')),
    time_horizon    TEXT NOT NULL CHECK (time_horizon IN ('short','medium','long','very_long')),
    liquidity_needs TEXT CHECK (liquidity_needs IN ('high','moderate','low')),
    annual_income_range TEXT,
    net_worth_range TEXT,
    investment_experience TEXT CHECK (investment_experience IN ('none','limited','moderate','extensive')),
    -- Reg BI / MiFID II fields
    conflicts_disclosed BOOLEAN NOT NULL DEFAULT false,
    basis_for_recommendation TEXT,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','superseded','expired')),
    expires_at      DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_suitability_client ON suitability_assessment(client_id);

CREATE TABLE compliance_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    name            TEXT NOT NULL,
    rule_type       TEXT NOT NULL CHECK (rule_type IN (
        'restricted_security','concentration_limit','sector_limit',
        'trade_approval','holding_period','esg_exclusion'
    )),
    parameters      JSONB NOT NULL,
    /* Example parameters:
    {
        "security_ids": ["..."],
        "max_weight_pct": 10.0,
        "restricted_sectors": ["tobacco", "weapons"],
        "min_holding_days": 30
    }
    */
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE compliance_check (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    trade_order_id  UUID REFERENCES trade_order(id),
    rule_id         UUID NOT NULL REFERENCES compliance_rule(id),
    account_id      UUID NOT NULL REFERENCES account(id),
    result          TEXT NOT NULL CHECK (result IN ('pass','fail','warning','override')),
    details         TEXT,
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    overridden_by   UUID REFERENCES app_user(id),
    override_reason TEXT
);
CREATE INDEX idx_compliance_check_order ON compliance_check(trade_order_id);
```

## Billing & Fees

```sql
CREATE TABLE fee_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    name            TEXT NOT NULL,
    fee_type        TEXT NOT NULL CHECK (fee_type IN ('aum_tiered','flat','per_account','hourly','performance')),
    tiers           JSONB,
    /* Example AUM tiers:
    [
        {"min_aum": 0, "max_aum": 1000000, "rate_bps": 100},
        {"min_aum": 1000000, "max_aum": 5000000, "rate_bps": 75},
        {"min_aum": 5000000, "max_aum": null, "rate_bps": 50}
    ]
    */
    billing_frequency TEXT NOT NULL DEFAULT 'quarterly' CHECK (billing_frequency IN ('monthly','quarterly','semi_annual','annual')),
    billing_method  TEXT NOT NULL DEFAULT 'arrears' CHECK (billing_method IN ('advance','arrears')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE billing_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    account_id      UUID NOT NULL REFERENCES account(id),
    fee_schedule_id UUID NOT NULL REFERENCES fee_schedule(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    billable_aum    NUMERIC(18,2) NOT NULL,
    fee_amount      NUMERIC(18,2) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'calculated' CHECK (status IN ('calculated','approved','invoiced','paid','waived')),
    approved_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_billing_account ON billing_record(account_id);
```

## CRM & Activities

```sql
CREATE TABLE activity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    client_id       UUID REFERENCES client(id),
    household_id    UUID REFERENCES household(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    activity_type   TEXT NOT NULL CHECK (activity_type IN (
        'call','email','meeting','note','task','review','document','other'
    )),
    subject         TEXT NOT NULL,
    description     TEXT,
    scheduled_at    TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open','in_progress','completed','cancelled')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_activity_client ON activity(client_id);
CREATE INDEX idx_activity_household ON activity(household_id);
CREATE INDEX idx_activity_user ON activity(user_id);

CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    client_id       UUID REFERENCES client(id),
    account_id      UUID REFERENCES account(id),
    document_type   TEXT NOT NULL CHECK (document_type IN (
        'statement','tax_form','agreement','ips','suitability_report',
        'performance_report','correspondence','identity','other'
    )),
    title           TEXT NOT NULL,
    file_name       TEXT NOT NULL,
    mime_type       TEXT NOT NULL,
    file_size_bytes BIGINT,
    storage_key     TEXT NOT NULL,                  -- S3 / blob storage key
    uploaded_by     UUID REFERENCES app_user(id),
    retention_until DATE,                           -- MiFID II: 5-year minimum
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_document_client ON document(client_id);
CREATE INDEX idx_document_account ON document(account_id);
```

## Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    user_id         UUID REFERENCES app_user(id),
    entity_type     TEXT NOT NULL,                  -- e.g. 'client', 'trade_order', 'position'
    entity_id       UUID NOT NULL,
    action          TEXT NOT NULL CHECK (action IN ('create','update','delete','view','export','login','logout')),
    changes         JSONB,                          -- before/after field values
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_user ON audit_log(user_id);
CREATE INDEX idx_audit_log_created ON audit_log(created_at);
-- Partition by month for performance
-- CREATE TABLE audit_log PARTITION BY RANGE (created_at);
```

## Financial Planning

```sql
CREATE TABLE financial_goal (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    goal_type       TEXT NOT NULL CHECK (goal_type IN (
        'retirement','education','home_purchase','wealth_transfer',
        'emergency_fund','major_purchase','charitable','custom'
    )),
    name            TEXT NOT NULL,
    target_amount   NUMERIC(18,2),
    target_date     DATE,
    current_amount  NUMERIC(18,2),
    monthly_contribution NUMERIC(18,2),
    priority        SMALLINT NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','achieved','deferred','cancelled')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE planning_scenario (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    name            TEXT NOT NULL,
    parameters      JSONB NOT NULL,
    /* Example:
    {
        "retirement_age": 65,
        "life_expectancy": 90,
        "inflation_rate": 0.03,
        "return_assumption": 0.07,
        "social_security_start": 67,
        "monthly_retirement_spending": 8000
    }
    */
    result_summary  JSONB,
    /* Example:
    {
        "success_probability": 0.87,
        "median_ending_balance": 2450000,
        "shortfall_probability": 0.13,
        "monte_carlo_iterations": 10000
    }
    */
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 4 | firm, app_user, role, user_role |
| Client & Household | 3 | household, client, client_advisor |
| Reference Data | 5 | country, currency, asset_class, security, security_price, exchange_rate |
| Accounts & Portfolios | 5 | account, custodian, model_portfolio, model_portfolio_allocation, account_model_assignment |
| Positions & Transactions | 3 | position, transaction, lot |
| Trading & Rebalancing | 4 | trade_order, rebalance_proposal, rebalance_proposal_trade |
| Performance (GIPS) | 3 | account_return, composite, composite_membership, composite_return |
| Compliance | 3 | suitability_assessment, compliance_rule, compliance_check |
| Billing | 2 | fee_schedule, billing_record |
| CRM & Activities | 2 | activity, document |
| Audit | 1 | audit_log |
| Financial Planning | 2 | financial_goal, planning_scenario |
| **Total** | **~40** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation across microservices and avoids sequential enumeration attacks on financial data.

2. **firm_id on all tenant-scoped tables** — combined with PostgreSQL Row Level Security, this provides data isolation without schema-per-tenant overhead. Every query is scoped to the current firm.

3. **Separate security and security_price tables** — security master data changes rarely; prices change daily. Separating them enables efficient daily price loading without touching the master table.

4. **Tax lot tracking** — the `lot` table explicitly tracks cost basis per acquisition, enabling FIFO/LIFO/specific-identification tax lot methods required for US tax reporting and tax-loss harvesting.

5. **GIPS-compliant composite structure** — dedicated composite, composite_membership, and composite_return tables support the full GIPS 2020 standard including composite dispersion, asset-weighted returns, and portfolio count tracking.

6. **Encrypted PII** — tax IDs stored as BYTEA (application-level encryption) rather than plaintext, supporting GDPR and CCPA requirements. Column-level encryption keeps sensitive fields protected even if the database is compromised.

7. **FIX Protocol alignment in trade orders** — cl_ord_id and exec_id fields match FIX tags 11 and 17, enabling direct mapping to/from FIX messages for custodian trade execution.

8. **Compliance checks linked to trade orders** — every trade order can have multiple compliance checks, creating a documented trail of pre-trade compliance that satisfies Reg BI and MiFID II audit requirements.

9. **Audit log as append-only table** — designed for monthly partitioning, the audit log captures before/after snapshots for every data change, supporting the 5-year MiFID II record retention requirement.

10. **JSONB used sparingly** — only for genuinely variable structures (fee tier definitions, scenario parameters, compliance rule parameters) where the schema varies by configuration. Core financial data remains in typed columns.
