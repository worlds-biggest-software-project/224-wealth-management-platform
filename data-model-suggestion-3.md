# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Wealth Management Platform · Created: 2026-05-20

## Philosophy

This model uses traditional relational tables for core financial data (accounts, positions, transactions, securities) where data integrity and query performance matter most, but employs PostgreSQL JSONB columns extensively for domain areas where requirements vary by jurisdiction, client type, firm configuration, or asset class. The philosophy is: "relational where it must be, flexible where it should be."

Wealth management is a domain with enormous variability. A US RIA operating under Reg BI has different compliance fields than a UK IFA operating under FCA rules (the onshored MiFID II equivalent). An account holding publicly traded equities has different position attributes than one holding private equity or real estate. A family office needs different reporting fields than a solo advisor practice. Rather than creating dozens of nullable columns or jurisdiction-specific tables, JSONB columns absorb this variability while keeping the core schema stable.

This approach mirrors how products like Salesforce Financial Services Cloud and Addepar handle extensibility: a fixed relational backbone with flexible attribute layers on top. It enables rapid iteration on jurisdiction-specific features without schema migrations, making it well suited for a platform that needs to ship fast and expand globally.

**Best for:** Teams building an MVP that needs to support multiple jurisdictions, asset types, or firm configurations without extensive schema migrations, while still maintaining relational integrity for core financial data.

**Trade-offs:**
- (+) Core financial data has full relational integrity (FKs, constraints, typed columns)
- (+) Jurisdiction-specific and firm-specific fields live in JSONB without schema changes
- (+) New asset types or compliance requirements can be added without migrations
- (+) GIN indexes on JSONB provide good query performance for structured JSON
- (+) Faster time-to-market than fully normalised approach
- (-) JSONB fields lack database-level type constraints — validation must happen in application code
- (-) Complex queries spanning relational and JSONB fields require careful indexing
- (-) JSONB data can drift if application validation is inconsistent
- (-) Reporting tools may struggle with nested JSON structures
- (-) Schema documentation is split between DDL and JSON shape definitions

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenWealth API | Core relational tables align with OpenWealth custody services; JSONB extensions handle bank-specific fields |
| FDX v6.5 | Investment account and holding schemas inform relational columns; FDX-specific metadata in JSONB |
| ISO 20022 | Security identifiers in relational columns; ISO 20022 message metadata in JSONB for custodian integrations |
| GIPS 2020 | Performance reporting tables are fully relational for calculation reliability |
| ISO 3166 / ISO 4217 | Country and currency codes as relational foreign keys |
| MiFID II | EU-specific suitability and disclosure fields in JSONB compliance_data column |
| SEC Reg BI | US-specific best-interest documentation in JSONB compliance_data column |
| FCA (UK) | UK-specific consumer duty fields in JSONB compliance_data column |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE firm (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    jurisdiction    CHAR(2) NOT NULL,              -- ISO 3166-1 alpha-2
    regulatory_regime TEXT NOT NULL CHECK (regulatory_regime IN ('sec_ria','mifid2','fca','multi')),
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    -- Firm-specific configuration in JSONB
    config          JSONB NOT NULL DEFAULT '{}',
    /* Example config:
    {
        "branding": {"logo_url": "...", "primary_color": "#1a5276"},
        "compliance": {
            "require_pre_trade_approval": true,
            "suitability_review_months": 12,
            "document_retention_years": 7
        },
        "billing": {
            "default_fee_type": "aum_tiered",
            "billing_frequency": "quarterly"
        },
        "integrations": {
            "custodians": ["schwab", "fidelity"],
            "data_feeds": ["morningstar"]
        }
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
ALTER TABLE firm ENABLE ROW LEVEL SECURITY;

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN ('admin','advisor','compliance_officer','operations','client','readonly')),
    permissions     JSONB NOT NULL DEFAULT '[]',
    /* Example permissions:
    ["portfolio.read", "portfolio.write", "trade.execute", "trade.approve",
     "client.read", "client.write", "billing.read", "compliance.read"]
    */
    auth_provider   TEXT DEFAULT 'local' CHECK (auth_provider IN ('local','oidc','saml')),
    auth_subject    TEXT,                          -- OIDC/SAML subject identifier
    is_active       BOOLEAN NOT NULL DEFAULT true,
    profile         JSONB DEFAULT '{}',
    /* Example profile:
    {
        "phone": "+1-555-0123",
        "title": "Senior Financial Advisor",
        "certifications": ["CFP", "CFA"],
        "notification_preferences": {"email": true, "sms": false}
    }
    */
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, email)
);
CREATE INDEX idx_user_firm ON app_user(firm_id);
```

## Client & Household Management

```sql
CREATE TABLE household (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    name            TEXT NOT NULL,
    primary_advisor_id UUID REFERENCES app_user(id),
    risk_profile    TEXT,
    total_aum       NUMERIC(18,2),
    attributes      JSONB DEFAULT '{}',
    /* Example attributes:
    {
        "service_tier": "platinum",
        "annual_review_month": 3,
        "special_instructions": "Client prefers phone calls over email",
        "referral_source": "existing_client",
        "engagement_score": 78
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_household_firm ON household(firm_id);

CREATE TABLE client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    household_id    UUID REFERENCES household(id),
    client_type     TEXT NOT NULL CHECK (client_type IN ('individual','joint','entity','trust')),
    -- Core identity fields (relational for query performance)
    first_name      TEXT,
    last_name       TEXT,
    entity_name     TEXT,
    email           TEXT,
    country         CHAR(2),                       -- ISO 3166-1
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','prospect','inactive','deceased')),
    -- Flexible identity and contact fields
    personal_info   JSONB DEFAULT '{}',
    /* Example (individual):
    {
        "date_of_birth": "1975-03-15",
        "tax_id_encrypted": "...",
        "citizenship": ["US", "GB"],
        "phone_numbers": [{"type": "mobile", "number": "+1-555-0123"}],
        "addresses": [{
            "type": "home",
            "line1": "123 Main St",
            "city": "New York",
            "state": "NY",
            "postal_code": "10001",
            "country": "US"
        }],
        "employment": {
            "employer": "Acme Corp",
            "title": "VP Engineering",
            "industry": "technology"
        }
    }
    Example (entity/trust):
    {
        "tax_id_encrypted": "...",
        "formation_date": "2010-06-01",
        "formation_jurisdiction": "DE",
        "trust_type": "revocable_living",
        "trustees": ["uuid1", "uuid2"],
        "beneficiaries": ["uuid3"]
    }
    */
    -- KYC and compliance (JSONB for jurisdiction-specific fields)
    kyc_status      TEXT NOT NULL DEFAULT 'pending' CHECK (kyc_status IN ('pending','verified','expired','rejected')),
    compliance_data JSONB DEFAULT '{}',
    /* Example (US - Reg BI):
    {
        "regime": "reg_bi",
        "client_category": "retail",
        "form_crs_delivered": true,
        "form_crs_date": "2026-01-15",
        "accredited_investor": false,
        "political_exposure": false,
        "insider_status": []
    }
    Example (EU - MiFID II):
    {
        "regime": "mifid2",
        "client_category": "professional",
        "appropriateness_test_date": "2025-11-01",
        "target_market_assessment": "growth_moderate_risk",
        "recording_consent": true,
        "cost_disclosure_date": "2026-01-15"
    }
    */
    -- Suitability profile (JSONB for flexibility across regimes)
    suitability     JSONB DEFAULT '{}',
    /* Example:
    {
        "investment_objective": "growth",
        "risk_tolerance": "moderate_high",
        "time_horizon_years": 20,
        "liquidity_needs": "low",
        "investment_experience": "extensive",
        "esg_preferences": {
            "exclude_sectors": ["tobacco", "weapons"],
            "prefer_sectors": ["renewable_energy"],
            "minimum_esg_score": 60
        },
        "tax_sensitivity": "high",
        "assessed_date": "2026-03-01",
        "assessed_by": "uuid",
        "next_review_date": "2027-03-01"
    }
    */
    user_id         UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_client_firm ON client(firm_id);
CREATE INDEX idx_client_household ON client(household_id);
CREATE INDEX idx_client_status ON client(status);
CREATE INDEX idx_client_kyc ON client(kyc_status);
-- GIN index for JSONB queries
CREATE INDEX idx_client_compliance_gin ON client USING gin(compliance_data);
CREATE INDEX idx_client_suitability_gin ON client USING gin(suitability);
```

## Securities & Market Data

```sql
CREATE TABLE security (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticker          TEXT,
    isin            CHAR(12),
    cusip           CHAR(9),
    name            TEXT NOT NULL,
    security_type   TEXT NOT NULL CHECK (security_type IN (
        'equity','bond','mutual_fund','etf','option','future',
        'alternative','real_estate','private_equity','cash_equivalent','other'
    )),
    currency        CHAR(3) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Flexible attributes by security type
    attributes      JSONB DEFAULT '{}',
    /* Example (equity):
    {
        "exchange": "NYSE",
        "sector": "Technology",
        "industry": "Software",
        "market_cap_category": "large_cap",
        "country": "US",
        "sedol": "B1YW440",
        "figi": "BBG000BVPV84",
        "esg_score": 72,
        "dividend_yield": 0.0065
    }
    Example (bond):
    {
        "issuer": "US Treasury",
        "coupon_rate": 0.0425,
        "coupon_frequency": "semi_annual",
        "maturity_date": "2034-11-15",
        "face_value": 1000,
        "credit_rating": "AAA",
        "callable": false,
        "bond_type": "government"
    }
    Example (private_equity):
    {
        "fund_name": "Sequoia Capital Fund XVIII",
        "vintage_year": 2022,
        "commitment": 500000,
        "called_capital": 350000,
        "distribution": 50000,
        "nav_date": "2026-03-31",
        "nav_per_unit": 1.25,
        "valuation_method": "manager_reported"
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_security_ticker ON security(ticker);
CREATE INDEX idx_security_isin ON security(isin);
CREATE INDEX idx_security_type ON security(security_type);
CREATE INDEX idx_security_attrs_gin ON security USING gin(attributes);

CREATE TABLE security_price (
    security_id     UUID NOT NULL REFERENCES security(id),
    price_date      DATE NOT NULL,
    close_price     NUMERIC(18,6) NOT NULL,
    source          TEXT NOT NULL,
    currency        CHAR(3) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (security_id, price_date)
);
```

## Accounts & Portfolios

```sql
CREATE TABLE custodian (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    code            TEXT NOT NULL UNIQUE,
    api_config      JSONB DEFAULT '{}',
    /* Example:
    {
        "api_type": "schwab_trader",
        "base_url": "https://api.schwab.com/trader/v1",
        "auth_method": "oauth2_pkce",
        "supports_realtime": true,
        "data_format": "json",
        "field_mapping": {
            "account_number": "accountNumber",
            "position_quantity": "longQuantity"
        }
    }
    */
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    custodian_id    UUID REFERENCES custodian(id),
    account_number  TEXT NOT NULL,
    account_name    TEXT NOT NULL,
    account_type    TEXT NOT NULL,                  -- unconstrained to support all jurisdictions
    tax_status      TEXT NOT NULL CHECK (tax_status IN ('taxable','tax_deferred','tax_exempt')),
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    inception_date  DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','closed','frozen','pending')),
    is_discretionary BOOLEAN NOT NULL DEFAULT true,
    is_fee_paying   BOOLEAN NOT NULL DEFAULT true,
    -- Jurisdiction and type-specific fields
    attributes      JSONB DEFAULT '{}',
    /* Example (US IRA):
    {
        "account_subtype": "ira_roth",
        "contribution_year": 2026,
        "ytd_contributions": 6500,
        "beneficiary_designation": {
            "primary": [{"name": "Jane Smith", "relationship": "spouse", "pct": 100}],
            "contingent": []
        },
        "rmd_required": false
    }
    Example (UK ISA):
    {
        "account_subtype": "stocks_and_shares_isa",
        "tax_year": "2025-2026",
        "ytd_subscriptions": 15000,
        "isa_allowance_remaining": 5000,
        "manager_reference": "ISA-REF-12345"
    }
    Example (trust account):
    {
        "trust_name": "Smith Family Trust",
        "trust_type": "irrevocable",
        "trustee_ids": ["uuid1"],
        "grantor_id": "uuid2",
        "distribution_rules": "income only to beneficiaries"
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (firm_id, account_number)
);
CREATE INDEX idx_account_firm ON account(firm_id);
CREATE INDEX idx_account_client ON account(client_id);
CREATE INDEX idx_account_status ON account(status);
CREATE INDEX idx_account_attrs_gin ON account USING gin(attributes);

CREATE TABLE model_portfolio (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    name            TEXT NOT NULL,
    risk_level      TEXT,
    benchmark_security_id UUID REFERENCES security(id),
    allocations     JSONB NOT NULL,
    /* Example:
    [
        {"security_id": "uuid", "ticker": "VTI", "target_pct": 40, "min_pct": 35, "max_pct": 45, "drift_pct": 5},
        {"security_id": "uuid", "ticker": "VXUS", "target_pct": 20, "min_pct": 15, "max_pct": 25, "drift_pct": 5},
        {"security_id": "uuid", "ticker": "BND", "target_pct": 30, "min_pct": 25, "max_pct": 35, "drift_pct": 5},
        {"asset_class": "cash", "target_pct": 10, "min_pct": 5, "max_pct": 15}
    ]
    */
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Positions & Transactions

```sql
CREATE TABLE position (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id      UUID NOT NULL REFERENCES account(id),
    security_id     UUID NOT NULL REFERENCES security(id),
    firm_id         UUID NOT NULL,
    quantity        NUMERIC(18,6) NOT NULL,
    cost_basis      NUMERIC(18,2) NOT NULL,
    market_value    NUMERIC(18,2),
    unrealised_gain NUMERIC(18,2),
    as_of_date      DATE NOT NULL,
    currency        CHAR(3) NOT NULL,
    -- Lot-level and type-specific data in JSONB
    lots            JSONB DEFAULT '[]',
    /* Example:
    [
        {
            "lot_id": "uuid",
            "acquisition_date": "2024-03-15",
            "quantity": 50,
            "cost_per_unit": 145.00,
            "cost_basis": 7250.00,
            "holding_period": "long_term",
            "wash_sale": false
        },
        {
            "lot_id": "uuid",
            "acquisition_date": "2026-01-10",
            "quantity": 50,
            "cost_per_unit": 160.00,
            "cost_basis": 8000.00,
            "holding_period": "short_term",
            "wash_sale": false
        }
    ]
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (account_id, security_id, as_of_date)
);
CREATE INDEX idx_position_account ON position(account_id);
CREATE INDEX idx_position_firm ON position(firm_id);
CREATE INDEX idx_position_date ON position(as_of_date);

CREATE TABLE transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id      UUID NOT NULL REFERENCES account(id),
    security_id     UUID REFERENCES security(id),
    firm_id         UUID NOT NULL,
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
    net_amount      NUMERIC(18,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    custodian_ref   TEXT,
    is_cancelled    BOOLEAN NOT NULL DEFAULT false,
    -- Type-specific and custodian-specific fields
    details         JSONB DEFAULT '{}',
    /* Example (buy):
    {
        "commission": 4.95,
        "sec_fee": 0.02,
        "exchange": "NYSE",
        "execution_venue": "ARCA",
        "order_type": "market",
        "time_in_force": "day",
        "fix_exec_id": "EX-20260520-001",
        "lot_assignment": "fifo"
    }
    Example (dividend):
    {
        "dividend_type": "qualified",
        "ex_date": "2026-05-01",
        "record_date": "2026-05-02",
        "pay_date": "2026-05-15",
        "per_share": 0.88,
        "tax_withheld": 0,
        "reinvested": false
    }
    Example (transfer_in):
    {
        "from_custodian": "TD Ameritrade",
        "from_account": "ACCT-OLD-456",
        "transfer_method": "ACATS",
        "cost_basis_method": "specific_id",
        "original_acquisition_dates": [
            {"security_id": "uuid", "date": "2022-06-15", "quantity": 100, "cost": 14500}
        ]
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_txn_account ON transaction(account_id);
CREATE INDEX idx_txn_firm ON transaction(firm_id);
CREATE INDEX idx_txn_trade_date ON transaction(trade_date);
CREATE INDEX idx_txn_type ON transaction(transaction_type);
CREATE INDEX idx_txn_details_gin ON transaction USING gin(details);
```

## Trading & Rebalancing

```sql
CREATE TABLE trade_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    account_id      UUID NOT NULL REFERENCES account(id),
    security_id     UUID NOT NULL REFERENCES security(id),
    side            TEXT NOT NULL CHECK (side IN ('buy','sell')),
    order_type      TEXT NOT NULL CHECK (order_type IN ('market','limit','stop','stop_limit')),
    quantity        NUMERIC(18,6) NOT NULL,
    limit_price     NUMERIC(18,6),
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending','compliance_review','approved','submitted',
        'partial_fill','filled','cancelled','rejected'
    )),
    created_by      UUID NOT NULL REFERENCES app_user(id),
    approved_by     UUID REFERENCES app_user(id),
    -- Execution details and compliance in JSONB
    execution       JSONB DEFAULT '{}',
    /* Example (after fill):
    {
        "filled_quantity": 100,
        "filled_price": 152.35,
        "filled_at": "2026-05-20T14:30:00Z",
        "fix_cl_ord_id": "ORD-20260520-001",
        "fix_exec_id": "EX-20260520-001",
        "venue": "NYSE",
        "transaction_id": "uuid"
    }
    */
    compliance      JSONB DEFAULT '{}',
    /* Example:
    {
        "checks": [
            {"rule": "restricted_security", "result": "pass", "checked_at": "2026-05-20T14:00:00Z"},
            {"rule": "concentration_limit", "result": "pass", "details": "Position would be 4.2% of portfolio (limit: 10%)", "checked_at": "2026-05-20T14:00:00Z"},
            {"rule": "suitability", "result": "pass", "checked_at": "2026-05-20T14:00:01Z"}
        ],
        "approved_at": "2026-05-20T14:15:00Z",
        "approval_notes": "Within model allocation targets"
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_order_firm ON trade_order(firm_id);
CREATE INDEX idx_order_account ON trade_order(account_id);
CREATE INDEX idx_order_status ON trade_order(status);

CREATE TABLE rebalance_batch (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    model_id        UUID REFERENCES model_portfolio(id),
    name            TEXT,
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN ('draft','review','approved','executing','completed','cancelled')),
    config          JSONB NOT NULL DEFAULT '{}',
    /* Example:
    {
        "tax_loss_harvest": true,
        "respect_wash_sale": true,
        "min_trade_amount": 100,
        "cash_buffer_pct": 2,
        "lot_selection": "tax_optimized"
    }
    */
    trades          JSONB NOT NULL DEFAULT '[]',
    /* Example:
    [
        {"account_id": "uuid", "security_id": "uuid", "side": "sell", "quantity": 50, "est_amount": 7500, "reason": "drift_correction", "order_id": null},
        {"account_id": "uuid", "security_id": "uuid", "side": "buy", "quantity": 30, "est_amount": 4500, "reason": "target_allocation", "order_id": null}
    ]
    */
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Performance Reporting

```sql
-- Performance tables are fully relational — no JSONB — because
-- GIPS calculations require predictable, typed numeric fields.

CREATE TABLE account_return (
    account_id      UUID NOT NULL REFERENCES account(id),
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
    PRIMARY KEY (account_id, period_start, period_end)
);

CREATE TABLE composite (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    name            TEXT NOT NULL,
    description     TEXT,
    inception_date  DATE NOT NULL,
    benchmark_security_id UUID REFERENCES security(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE composite_membership (
    composite_id    UUID NOT NULL REFERENCES composite(id),
    account_id      UUID NOT NULL REFERENCES account(id),
    start_date      DATE NOT NULL,
    end_date        DATE,
    PRIMARY KEY (composite_id, account_id, start_date)
);

CREATE TABLE composite_return (
    composite_id    UUID NOT NULL REFERENCES composite(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    asset_weighted_return NUMERIC(12,8) NOT NULL,
    composite_dispersion NUMERIC(12,8),
    total_assets    NUMERIC(18,2),
    number_of_accounts INTEGER,
    benchmark_return NUMERIC(12,8),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (composite_id, period_start, period_end)
);
```

## Billing

```sql
CREATE TABLE fee_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    name            TEXT NOT NULL,
    fee_config      JSONB NOT NULL,
    /* Example (AUM tiered):
    {
        "type": "aum_tiered",
        "frequency": "quarterly",
        "method": "arrears",
        "tiers": [
            {"min": 0, "max": 1000000, "rate_bps": 100},
            {"min": 1000000, "max": 5000000, "rate_bps": 75},
            {"min": 5000000, "max": null, "rate_bps": 50}
        ],
        "household_billing": true,
        "exclude_cash": false
    }
    Example (flat + performance):
    {
        "type": "flat_plus_performance",
        "frequency": "quarterly",
        "flat_amount": 5000,
        "performance_fee_pct": 20,
        "hurdle_rate_pct": 8,
        "high_water_mark": true
    }
    */
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
    status          TEXT NOT NULL DEFAULT 'calculated',
    details         JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
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
    status          TEXT NOT NULL DEFAULT 'open',
    scheduled_at    TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    -- Type-specific fields in JSONB
    details         JSONB DEFAULT '{}',
    /* Example (meeting):
    {
        "location": "Client's office",
        "attendees": ["uuid1", "uuid2"],
        "agenda": "Annual portfolio review",
        "duration_minutes": 60,
        "notes": "Discussed retirement timeline...",
        "follow_up_tasks": ["Review estate plan", "Update beneficiaries"],
        "calendar_event_id": "google_cal_123"
    }
    Example (review - annual compliance):
    {
        "review_type": "annual_suitability",
        "accounts_reviewed": ["uuid1", "uuid2"],
        "suitability_confirmed": true,
        "changes_required": false,
        "next_review_date": "2027-05-01"
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_activity_firm ON activity(firm_id);
CREATE INDEX idx_activity_client ON activity(client_id);

CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    client_id       UUID REFERENCES client(id),
    account_id      UUID REFERENCES account(id),
    document_type   TEXT NOT NULL,
    title           TEXT NOT NULL,
    storage_key     TEXT NOT NULL,
    metadata        JSONB DEFAULT '{}',
    /* Example:
    {
        "file_name": "Q1-2026-Statement.pdf",
        "mime_type": "application/pdf",
        "file_size_bytes": 245000,
        "uploaded_by": "uuid",
        "retention_until": "2031-05-20",
        "tags": ["statement", "quarterly"],
        "generated": true
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_document_client ON document(client_id);
```

## Financial Planning

```sql
CREATE TABLE financial_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL REFERENCES firm(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    created_by      UUID NOT NULL REFERENCES app_user(id),
    name            TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    -- Entire plan configuration in JSONB for maximum flexibility
    goals           JSONB NOT NULL DEFAULT '[]',
    /* Example:
    [
        {
            "goal_type": "retirement",
            "name": "Retirement at 65",
            "target_amount": 3000000,
            "target_date": "2040-03-15",
            "current_funding": 1200000,
            "monthly_contribution": 5000,
            "priority": 1
        },
        {
            "goal_type": "education",
            "name": "College for Sarah",
            "target_amount": 200000,
            "target_date": "2032-09-01",
            "current_funding": 45000,
            "monthly_contribution": 1000,
            "priority": 2
        }
    ]
    */
    assumptions     JSONB NOT NULL DEFAULT '{}',
    /* Example:
    {
        "inflation_rate": 0.03,
        "return_assumptions": {
            "conservative": 0.05,
            "moderate": 0.07,
            "aggressive": 0.09
        },
        "social_security_start_age": 67,
        "social_security_monthly": 2800,
        "tax_rate_current": 0.24,
        "tax_rate_retirement": 0.22,
        "life_expectancy": 90
    }
    */
    results         JSONB DEFAULT '{}',
    /* Example (Monte Carlo):
    {
        "iterations": 10000,
        "success_probability": 0.87,
        "median_ending_balance": 2450000,
        "percentile_10": 1200000,
        "percentile_90": 4100000,
        "shortfall_probability": 0.13,
        "calculated_at": "2026-05-20T10:30:00Z"
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_plan_client ON financial_plan(client_id);
```

## Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firm_id         UUID NOT NULL,
    user_id         UUID,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    action          TEXT NOT NULL,
    changes         JSONB,                         -- {"field": {"old": ..., "new": ...}}
    context         JSONB DEFAULT '{}',
    /* Example:
    {
        "ip_address": "203.0.113.42",
        "user_agent": "Mozilla/5.0...",
        "source": "web_app",
        "correlation_id": "uuid"
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

## Example JSONB Queries

```sql
-- Find all clients with ESG preferences who exclude tobacco
SELECT id, first_name, last_name
FROM client
WHERE firm_id = '<firm_id>'
  AND suitability @> '{"esg_preferences": {"exclude_sectors": ["tobacco"]}}';

-- Find all accounts with Roth IRA subtype
SELECT id, account_number, account_name
FROM account
WHERE firm_id = '<firm_id>'
  AND attributes->>'account_subtype' = 'ira_roth';

-- Find all MiFID II professional clients
SELECT id, first_name, last_name
FROM client
WHERE firm_id = '<firm_id>'
  AND compliance_data @> '{"regime": "mifid2", "client_category": "professional"}';

-- Aggregate dividend income by type
SELECT
    details->>'dividend_type' AS dividend_type,
    SUM(net_amount) AS total
FROM transaction
WHERE firm_id = '<firm_id>'
  AND transaction_type = 'dividend'
  AND trade_date BETWEEN '2026-01-01' AND '2026-12-31'
GROUP BY details->>'dividend_type';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | firm, app_user (roles/permissions in JSONB) |
| Client & Household | 2 | household, client (compliance, suitability in JSONB) |
| Securities & Market Data | 2 | security, security_price |
| Accounts & Portfolios | 3 | custodian, account, model_portfolio |
| Positions & Transactions | 2 | position (lots in JSONB), transaction |
| Trading & Rebalancing | 2 | trade_order, rebalance_batch |
| Performance (GIPS) | 4 | account_return, composite, composite_membership, composite_return |
| Billing | 2 | fee_schedule, billing_record |
| CRM | 2 | activity, document |
| Financial Planning | 1 | financial_plan (goals, assumptions, results all in JSONB) |
| Audit | 1 | audit_log |
| **Total** | **~23** | About 40% fewer tables than fully normalised |

---

## Key Design Decisions

1. **JSONB for jurisdiction-specific compliance data** — rather than creating separate tables for Reg BI, MiFID II, and FCA compliance fields (which would multiply with every new jurisdiction), a single `compliance_data` JSONB column on the client table absorbs all regime-specific fields. GIN indexes ensure queryability.

2. **JSONB for security attributes by type** — equities, bonds, private equity, and real estate all have different attributes. Instead of 5+ type-specific tables, a single `attributes` JSONB column handles all types. The `security_type` column indicates which JSON shape to expect.

3. **Relational for performance calculations** — GIPS composite returns and account returns remain fully relational with typed NUMERIC columns. Financial calculations must be predictable and auditable; JSONB would introduce unnecessary complexity here.

4. **Tax lots embedded in position JSONB** — for most query patterns (portfolio display, rebalancing), you need lots alongside positions. Embedding lots in JSONB avoids a join-heavy query pattern while still allowing extraction for tax reporting.

5. **Permissions as JSONB arrays** — eliminates the role/user_role junction tables from the normalised model. Works well for platforms with fewer than ~50 distinct permissions per user.

6. **Custodian API configuration in JSONB** — each custodian has completely different API configurations, authentication methods, and field mappings. JSONB is the natural fit.

7. **Financial plans as document-like JSONB** — financial planning data (goals, assumptions, Monte Carlo results) is inherently hierarchical and varies by planning scenario. Storing the entire plan as a JSONB document simplifies the application layer and enables versioning by simply saving new rows.

8. **Firm config in JSONB** — branding, compliance settings, billing defaults, and integration configuration are firm-specific and change rarely. A single JSONB column avoids a firm_settings table with dozens of columns.

9. **Model portfolio allocations in JSONB** — target allocations, drift thresholds, and min/max bounds are naturally a list. JSONB keeps the model_portfolio table simple while supporting complex allocation structures.

10. **Application-layer validation is critical** — the trade-off of JSONB flexibility is that the database cannot enforce JSON shapes. JSON Schema validation in the application layer (aligned with the documented JSONB examples above) is essential to prevent drift.
