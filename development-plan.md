# Wealth Management Platform — Phased Development Plan

> Project: 224-wealth-management-platform · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | Python 3.12+ | Financial domain benefits from NumPy/pandas for performance calculations, rich ecosystem for LLM integrations (openai, anthropic SDKs), Pydantic for strict financial data validation, and strong ORM support. Python dominates quantitative finance tooling. |
| API framework | FastAPI 0.115+ | Automatic OpenAPI 3.1 spec generation (required by standards.md — Addepar, Orion, OpenWealth all publish OAS specs), async support for custodian data sync and LLM calls, Pydantic v2 integration for request/response validation. |
| Database | PostgreSQL 16 | Multi-tenant RLS, JSONB with GIN indexes for jurisdiction-specific compliance data, NUMERIC types for financial precision, partitioning for audit logs. All four data model suggestions target PostgreSQL. |
| ORM / query builder | SQLAlchemy 2.0 + Alembic | Mature migration tooling (critical for production schema evolution), async session support, type-mapped models align with Pydantic. |
| Task queue | Celery 5 + Redis | Async workloads: custodian data sync, rebalancing calculations, performance return calculations, billing runs, LLM-powered report generation. Redis also serves as cache layer. |
| Frontend | Next.js 15 (React 19) + TypeScript | Dashboard-heavy UI with real-time portfolio views, advisor workspaces, and client portal — all require rich interactivity. Next.js App Router with server components reduces client bundle for data-heavy pages. |
| UI components | shadcn/ui + Tailwind CSS 4 | Financial dashboards need data tables, charts, forms, and modals. shadcn/ui provides unstyled primitives; Tailwind enables consistent, rapid styling. |
| Charting | Recharts 2 | Portfolio allocation pie charts, performance line charts, drift bar charts. React-native, composable, lightweight. |
| Authentication | NextAuth.js 5 (Auth.js) + OIDC | Supports local credentials, OIDC/SAML for enterprise SSO (Okta, Azure AD), and JWT sessions. FAPI 2.0 compliance requires PKCE — Auth.js supports this natively. |
| Testing (backend) | pytest + pytest-asyncio + factory_boy | pytest is standard; factory_boy generates realistic financial test fixtures; pytest-asyncio for async endpoint tests. |
| Testing (frontend) | Vitest + Playwright | Vitest for unit/component tests; Playwright for E2E flows (advisor dashboard, client portal, trade workflow). |
| Code quality | Ruff (lint + format) + mypy (strict) + ESLint + Prettier | Ruff replaces flake8+isort+black in one tool. mypy strict mode catches financial calculation type errors. |
| Containerisation | Docker + Docker Compose | Self-hosted deployment option (family offices, compliance-sensitive firms). Compose orchestrates API, worker, database, Redis, and frontend. |
| Package manager | uv (Python) + pnpm (Node.js) | uv is 10-100x faster than pip for dependency resolution. pnpm for deterministic Node.js installs. |
| LLM integration | Anthropic Python SDK + OpenAI SDK | AI-native features: suitability report generation, financial plan drafting, conversational client portal. Support multiple providers for flexibility. |
| Key libraries | pandas, numpy (calculations), cryptography (PII encryption), httpx (async HTTP for custodian APIs), python-jose (JWT), jinja2 (report templates), weasyprint (PDF generation) |

### Project Structure

```
wealth-management-platform/
├── pyproject.toml                    # Python project config (uv)
├── Dockerfile                        # Multi-stage build: API + worker
├── docker-compose.yml                # Full stack: API, worker, db, redis, frontend
├── alembic.ini                       # Migration config
├── alembic/
│   └── versions/                     # Migration scripts
├── src/
│   └── wealthmgmt/
│       ├── __init__.py
│       ├── main.py                   # FastAPI application factory
│       ├── config.py                 # Settings (Pydantic BaseSettings)
│       ├── database.py               # SQLAlchemy engine, session, base
│       ├── security.py               # Auth, JWT, RBAC, encryption
│       ├── middleware.py             # Tenant context, audit logging
│       ├── models/                   # SQLAlchemy ORM models
│       │   ├── __init__.py
│       │   ├── firm.py               # Firm, AppUser, Role
│       │   ├── client.py             # Client, Household
│       │   ├── account.py            # Account, Custodian
│       │   ├── security.py           # Security, SecurityPrice
│       │   ├── portfolio.py          # ModelPortfolio, Position, Lot
│       │   ├── transaction.py        # Transaction
│       │   ├── trading.py            # TradeOrder, RebalanceProposal
│       │   ├── compliance.py         # SuitabilityAssessment, ComplianceRule, ComplianceCheck
│       │   ├── performance.py        # AccountReturn, Composite, CompositeReturn
│       │   ├── billing.py            # FeeSchedule, BillingRecord
│       │   ├── crm.py                # Activity, Document
│       │   ├── planning.py           # FinancialGoal, PlanningScenario
│       │   └── audit.py              # AuditLog
│       ├── schemas/                  # Pydantic request/response schemas
│       │   ├── __init__.py
│       │   ├── firm.py
│       │   ├── client.py
│       │   ├── account.py
│       │   ├── portfolio.py
│       │   ├── trading.py
│       │   ├── compliance.py
│       │   ├── performance.py
│       │   ├── billing.py
│       │   ├── crm.py
│       │   └── planning.py
│       ├── api/                      # FastAPI routers
│       │   ├── __init__.py
│       │   ├── deps.py               # Shared dependencies (auth, db session, tenant)
│       │   ├── v1/
│       │   │   ├── __init__.py
│       │   │   ├── auth.py
│       │   │   ├── firms.py
│       │   │   ├── clients.py
│       │   │   ├── accounts.py
│       │   │   ├── portfolios.py
│       │   │   ├── trading.py
│       │   │   ├── compliance.py
│       │   │   ├── performance.py
│       │   │   ├── billing.py
│       │   │   ├── crm.py
│       │   │   └── planning.py
│       │   └── router.py             # Aggregates all v1 routers
│       ├── services/                 # Business logic layer
│       │   ├── __init__.py
│       │   ├── client_service.py
│       │   ├── portfolio_service.py
│       │   ├── rebalancing_service.py
│       │   ├── performance_service.py
│       │   ├── compliance_service.py
│       │   ├── billing_service.py
│       │   ├── planning_service.py
│       │   └── reporting_service.py
│       ├── integrations/             # External system connectors
│       │   ├── __init__.py
│       │   ├── custodian_base.py     # Abstract custodian interface
│       │   ├── schwab.py             # Schwab Trader API
│       │   ├── plaid.py              # Plaid Investments API
│       │   ├── morningstar.py        # Market data
│       │   └── openwealth.py         # OpenWealth API
│       ├── ai/                       # LLM-powered features
│       │   ├── __init__.py
│       │   ├── prompts.py            # Prompt templates
│       │   ├── suitability.py        # AI suitability analysis
│       │   ├── planning.py           # AI financial planning
│       │   └── reporting.py          # AI report generation
│       └── tasks/                    # Celery async tasks
│           ├── __init__.py
│           ├── celery_app.py
│           ├── sync_positions.py
│           ├── calculate_returns.py
│           ├── run_billing.py
│           └── generate_reports.py
├── tests/
│   ├── conftest.py                   # Fixtures: test DB, factories, auth
│   ├── factories/                    # factory_boy model factories
│   │   ├── __init__.py
│   │   ├── firm.py
│   │   ├── client.py
│   │   ├── account.py
│   │   ├── portfolio.py
│   │   └── trading.py
│   ├── unit/
│   │   ├── test_performance_calc.py
│   │   ├── test_rebalancing.py
│   │   ├── test_billing.py
│   │   ├── test_compliance.py
│   │   └── test_encryption.py
│   ├── integration/
│   │   ├── test_client_api.py
│   │   ├── test_account_api.py
│   │   ├── test_trading_api.py
│   │   └── test_auth.py
│   └── fixtures/                     # Static test data (CSV, JSON)
│       ├── sample_positions.json
│       ├── sample_transactions.csv
│       └── sample_prices.csv
├── frontend/
│   ├── package.json
│   ├── pnpm-lock.yaml
│   ├── next.config.ts
│   ├── tsconfig.json
│   ├── tailwind.config.ts
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── (auth)/
│   │   │   │   ├── login/page.tsx
│   │   │   │   └── register/page.tsx
│   │   │   ├── (advisor)/                # Advisor workspace
│   │   │   │   ├── dashboard/page.tsx
│   │   │   │   ├── clients/
│   │   │   │   ├── portfolios/
│   │   │   │   ├── trading/
│   │   │   │   ├── compliance/
│   │   │   │   └── billing/
│   │   │   └── (client)/                 # Client portal
│   │   │       ├── portal/page.tsx
│   │   │       └── documents/
│   │   ├── components/
│   │   │   ├── ui/                       # shadcn/ui primitives
│   │   │   ├── portfolio/
│   │   │   ├── trading/
│   │   │   ├── charts/
│   │   │   └── layout/
│   │   ├── lib/
│   │   │   ├── api-client.ts             # Typed API client
│   │   │   ├── auth.ts
│   │   │   └── utils.ts
│   │   └── types/
│   │       └── index.ts                  # Shared TypeScript types
│   └── tests/
│       ├── components/
│       └── e2e/
└── docs/
    └── api/                              # Auto-generated OpenAPI docs
```

---

## Phase 1: Foundation & Data Layer

### Purpose
Establish the project scaffolding, database schema, configuration system, and core data access patterns. After this phase, all database tables exist with migrations, the FastAPI application boots, and basic CRUD operations work against the database. This is the foundation everything else builds on.

### Tasks

#### 1.1 — Project Scaffolding & Configuration

**What**: Set up the Python project with uv, FastAPI application factory, and Pydantic settings.

**Design**:

```python
# src/wealthmgmt/config.py
from pydantic_settings import BaseSettings
from pydantic import Field, SecretStr

class Settings(BaseSettings):
    model_config = {"env_prefix": "WM_", "env_file": ".env"}

    # Application
    app_name: str = "Wealth Management Platform"
    debug: bool = False
    api_version: str = "v1"
    cors_origins: list[str] = ["http://localhost:3000"]

    # Database
    database_url: str = "postgresql+asyncpg://wm:wm@localhost:5432/wealthmgmt"
    database_echo: bool = False
    database_pool_size: int = 20

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # Security
    secret_key: SecretStr = Field(default=...)
    jwt_algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    refresh_token_expire_days: int = 7
    encryption_key: SecretStr = Field(default=...)  # Fernet key for PII encryption

    # Celery
    celery_broker_url: str = "redis://localhost:6379/1"
    celery_result_backend: str = "redis://localhost:6379/2"
```

```python
# src/wealthmgmt/main.py
from fastapi import FastAPI
from wealthmgmt.config import Settings
from wealthmgmt.api.router import api_router
from wealthmgmt.database import engine
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    yield
    # shutdown
    await engine.dispose()

def create_app(settings: Settings | None = None) -> FastAPI:
    settings = settings or Settings()
    app = FastAPI(
        title=settings.app_name,
        version=settings.api_version,
        lifespan=lifespan,
    )
    app.state.settings = settings
    app.include_router(api_router, prefix=f"/api/{settings.api_version}")
    return app
```

```python
# src/wealthmgmt/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase
import uuid

class Base(DeclarativeBase):
    pass

engine = create_async_engine(settings.database_url, pool_size=settings.database_pool_size)
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        yield session
```

Docker Compose:
```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: wealthmgmt
      POSTGRES_USER: wm
      POSTGRES_PASSWORD: wm
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
  api:
    build: .
    command: uvicorn wealthmgmt.main:app --host 0.0.0.0 --port 8000 --reload
    ports: ["8000:8000"]
    depends_on: [db, redis]
    env_file: .env
  worker:
    build: .
    command: celery -A wealthmgmt.tasks.celery_app worker -l info
    depends_on: [db, redis]
    env_file: .env
volumes:
  pgdata:
```

**Testing**:
- `Unit: Settings loads from environment variables with correct types and defaults`
- `Unit: Settings raises validation error when SECRET_KEY is missing`
- `Integration: FastAPI app starts and returns 200 on GET /api/v1/health`
- `Integration: Database connection pool initialises and executes SELECT 1`
- `Integration: Docker Compose services start and health-check pass`

#### 1.2 — Core Database Schema & Migrations

**What**: Implement the full database schema based on Data Model Suggestion 1 (Entity-Centric Normalized Relational) with Alembic migrations.

**Design**:

The schema adopts Data Model Suggestion 1 because it provides the strongest referential integrity, the most natural alignment with OpenWealth and FDX standards, and straightforward mapping to REST API resources. JSONB is used sparingly (compliance rule parameters, fee tier definitions, planning scenario parameters) following the principles from Data Model Suggestion 3 where variability genuinely requires it.

Core tables (~40 tables across 12 categories as specified in Data Model Suggestion 1):

**Identity & Multi-Tenancy**: `firm`, `app_user`, `role`, `user_role` — with PostgreSQL Row Level Security on all tenant-scoped tables. `firm_id` on every table.

**Client & Household**: `household`, `client`, `client_advisor` — client has `client_category` (MiFID II: retail/professional/eligible_counterparty), `kyc_status`, encrypted `tax_id_encrypted` (BYTEA).

**Reference Data**: `country` (ISO 3166-1), `currency` (ISO 4217), `asset_class` (hierarchical), `security` (with ISIN/CUSIP/SEDOL/FIGI identifiers), `security_price`, `exchange_rate`.

**Accounts & Portfolios**: `account` (with GIPS fields: `is_discretionary`, `is_fee_paying`), `custodian` (with `api_type`), `model_portfolio`, `model_portfolio_allocation`, `account_model_assignment`.

**Positions & Transactions**: `position`, `transaction` (14 transaction types), `lot` (tax lot tracking with wash sale flag).

**Trading & Rebalancing**: `trade_order` (FIX Protocol aligned: `fix_cl_ord_id`, `fix_exec_id`), `rebalance_proposal`, `rebalance_proposal_trade`.

**Performance (GIPS)**: `account_return` (TWR + MWR), `composite`, `composite_membership`, `composite_return` (asset-weighted return, dispersion).

**Compliance**: `suitability_assessment` (Reg BI / MiFID II fields), `compliance_rule` (JSONB parameters), `compliance_check`.

**Billing**: `fee_schedule` (JSONB tiers), `billing_record`.

**CRM**: `activity`, `document` (with `retention_until` for MiFID II 5-year retention).

**Financial Planning**: `financial_goal`, `planning_scenario` (JSONB parameters and results).

**Audit**: `audit_log` (append-only, partitioned by month, captures before/after JSONB changes).

All SQLAlchemy models use:
- UUID primary keys (`mapped_column(UUID, primary_key=True, default=uuid.uuid4)`)
- `created_at` and `updated_at` timestamps
- `firm_id` foreign key with index on all tenant-scoped tables

The full DDL is defined in Data Model Suggestion 1 and should be implemented as-is in SQLAlchemy models, with the initial Alembic migration generating the complete schema.

**Testing**:
- `Unit: All SQLAlchemy models can be instantiated with required fields`
- `Unit: Foreign key relationships resolve correctly (e.g., client.household navigates to Household)`
- `Integration: Alembic upgrade head creates all tables in empty database`
- `Integration: Alembic downgrade base drops all tables cleanly`
- `Integration: Row Level Security policies prevent cross-firm data access`
- `Integration: Unique constraints prevent duplicate (firm_id, account_number) pairs`
- `Integration: Check constraints reject invalid enum values (e.g., client_type='invalid')`

#### 1.3 — PII Encryption Utilities

**What**: Implement application-level encryption for sensitive financial data (tax IDs, SSNs) using Fernet symmetric encryption.

**Design**:

```python
# src/wealthmgmt/security.py (encryption section)
from cryptography.fernet import Fernet
from wealthmgmt.config import Settings

class PiiEncryptor:
    def __init__(self, key: bytes):
        self._fernet = Fernet(key)

    def encrypt(self, plaintext: str) -> bytes:
        """Encrypt a PII string, returning bytes for BYTEA storage."""
        return self._fernet.encrypt(plaintext.encode("utf-8"))

    def decrypt(self, ciphertext: bytes) -> str:
        """Decrypt BYTEA back to plaintext string."""
        return self._fernet.decrypt(ciphertext).decode("utf-8")

    @classmethod
    def generate_key(cls) -> str:
        """Generate a new Fernet key (base64-encoded)."""
        return Fernet.generate_key().decode("utf-8")
```

The encryptor is injected via FastAPI dependency injection. Tax IDs are encrypted before storage and decrypted only when explicitly requested with appropriate RBAC permissions.

**Testing**:
- `Unit: encrypt → decrypt roundtrip preserves original plaintext`
- `Unit: decrypt with wrong key raises InvalidToken`
- `Unit: encrypted output differs for same plaintext (Fernet uses random IV)`
- `Unit: generate_key produces valid base64 Fernet key`
- `Integration: Client with encrypted tax_id stores BYTEA in database, decrypts correctly on read`

#### 1.4 — Audit Logging Middleware

**What**: Implement automatic audit trail capture for all database mutations, writing to the `audit_log` table.

**Design**:

```python
# src/wealthmgmt/middleware.py
from sqlalchemy import event
from wealthmgmt.models.audit import AuditLog

class AuditContext:
    """Thread-local context for the current user and request metadata."""
    user_id: uuid.UUID | None
    ip_address: str | None
    user_agent: str | None

def audit_after_flush(session, flush_context):
    """SQLAlchemy event listener that captures changes after flush."""
    for obj in session.new:
        session.add(AuditLog(
            firm_id=getattr(obj, "firm_id", None),
            user_id=AuditContext.user_id,
            entity_type=obj.__tablename__,
            entity_id=obj.id,
            action="create",
            changes=None,
            ip_address=AuditContext.ip_address,
            user_agent=AuditContext.user_agent,
        ))
    for obj in session.dirty:
        changes = {}
        insp = inspect(obj)
        for attr in insp.attrs:
            hist = attr.history
            if hist.has_changes():
                changes[attr.key] = {"old": hist.deleted[0] if hist.deleted else None,
                                     "new": hist.added[0] if hist.added else None}
        if changes:
            session.add(AuditLog(
                firm_id=getattr(obj, "firm_id", None),
                user_id=AuditContext.user_id,
                entity_type=obj.__tablename__,
                entity_id=obj.id,
                action="update",
                changes=changes,
                ip_address=AuditContext.ip_address,
                user_agent=AuditContext.user_agent,
            ))
    for obj in session.deleted:
        session.add(AuditLog(
            firm_id=getattr(obj, "firm_id", None),
            user_id=AuditContext.user_id,
            entity_type=obj.__tablename__,
            entity_id=obj.id,
            action="delete",
            changes=None,
            ip_address=AuditContext.ip_address,
            user_agent=AuditContext.user_agent,
        ))
```

A FastAPI middleware sets `AuditContext` from the incoming request before each handler runs, and clears it afterward.

**Testing**:
- `Unit: Creating a client generates an audit_log entry with action="create"`
- `Unit: Updating a client's email generates audit_log with old/new values in changes JSONB`
- `Unit: Deleting a client generates audit_log with action="delete"`
- `Integration: API request to update client produces audit_log with correct ip_address and user_id`
- `Integration: Audit log entries cannot be updated or deleted (append-only constraint verified)`

---

## Phase 2: Authentication, Authorization & Tenant Management

### Purpose
Implement secure multi-tenant authentication and role-based access control. After this phase, users can register firms, authenticate, manage users and roles, and all API requests are scoped to the current firm with proper authorization checks.

### Tasks

#### 2.1 — JWT Authentication

**What**: Implement JWT-based authentication with access and refresh tokens, supporting local credentials and OIDC.

**Design**:

```python
# src/wealthmgmt/security.py (auth section)
from datetime import datetime, timedelta
from jose import jwt, JWTError
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

class AuthService:
    def __init__(self, settings: Settings):
        self.secret_key = settings.secret_key.get_secret_value()
        self.algorithm = settings.jwt_algorithm
        self.access_expire = timedelta(minutes=settings.access_token_expire_minutes)
        self.refresh_expire = timedelta(days=settings.refresh_token_expire_days)

    def create_access_token(self, user_id: str, firm_id: str, permissions: list[str]) -> str:
        payload = {
            "sub": user_id,
            "firm_id": firm_id,
            "permissions": permissions,
            "type": "access",
            "exp": datetime.utcnow() + self.access_expire,
            "iat": datetime.utcnow(),
        }
        return jwt.encode(payload, self.secret_key, algorithm=self.algorithm)

    def create_refresh_token(self, user_id: str) -> str:
        payload = {
            "sub": user_id,
            "type": "refresh",
            "exp": datetime.utcnow() + self.refresh_expire,
            "iat": datetime.utcnow(),
        }
        return jwt.encode(payload, self.secret_key, algorithm=self.algorithm)

    def verify_token(self, token: str) -> dict:
        """Decode and validate JWT. Raises JWTError on failure."""
        return jwt.decode(token, self.secret_key, algorithms=[self.algorithm])

    def hash_password(self, password: str) -> str:
        return pwd_context.hash(password)

    def verify_password(self, plain: str, hashed: str) -> bool:
        return pwd_context.verify(plain, hashed)
```

API endpoints:

| Method | Path | Request | Response |
|--------|------|---------|----------|
| POST | `/api/v1/auth/register` | `{firm_name, email, password, display_name}` | `{access_token, refresh_token, user}` |
| POST | `/api/v1/auth/login` | `{email, password}` | `{access_token, refresh_token, user}` |
| POST | `/api/v1/auth/refresh` | `{refresh_token}` | `{access_token, refresh_token}` |
| POST | `/api/v1/auth/logout` | (bearer token) | `204 No Content` |

Registration creates both a `firm` and the first `app_user` with admin role. The `firm` is populated with default roles (admin, advisor, compliance_officer, operations, client, readonly) and a default set of permissions per role.

**Testing**:
- `Unit: create_access_token produces JWT with correct claims (sub, firm_id, permissions, exp)`
- `Unit: verify_token rejects expired tokens with JWTError`
- `Unit: verify_token rejects tokens signed with wrong key`
- `Unit: hash_password produces bcrypt hash; verify_password matches`
- `Integration: POST /auth/register creates firm + user, returns valid tokens`
- `Integration: POST /auth/login with valid credentials returns tokens`
- `Integration: POST /auth/login with wrong password returns 401`
- `Integration: POST /auth/refresh with valid refresh token returns new access token`
- `Integration: POST /auth/refresh with expired refresh token returns 401`

#### 2.2 — RBAC & Permission Middleware

**What**: Implement role-based access control enforced via FastAPI dependencies and PostgreSQL Row Level Security.

**Design**:

```python
# src/wealthmgmt/api/deps.py
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

bearer_scheme = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Security(bearer_scheme),
    db: AsyncSession = Depends(get_db),
    auth: AuthService = Depends(get_auth_service),
) -> AppUser:
    """Extract and validate JWT, return the authenticated user."""
    payload = auth.verify_token(credentials.credentials)
    user = await db.get(AppUser, payload["sub"])
    if not user or not user.is_active:
        raise HTTPException(status_code=401, detail="User not found or inactive")
    return user

def require_permission(permission: str):
    """Dependency factory that checks the user has a specific permission."""
    async def check(user: AppUser = Depends(get_current_user)) -> AppUser:
        user_permissions = await get_user_permissions(user)
        if permission not in user_permissions:
            raise HTTPException(status_code=403, detail=f"Missing permission: {permission}")
        return user
    return check

class TenantContext:
    """Sets PostgreSQL session variable for RLS enforcement."""
    @staticmethod
    async def set_tenant(db: AsyncSession, firm_id: uuid.UUID):
        await db.execute(text(f"SET app.current_firm_id = '{firm_id}'"))
```

Permissions follow a `resource.action` pattern: `client.read`, `client.write`, `trade.execute`, `trade.approve`, `compliance.read`, `compliance.write`, `billing.read`, `billing.write`, `admin.manage_users`.

Default role-permission mappings:
- **admin**: all permissions
- **advisor**: `client.read`, `client.write`, `portfolio.read`, `portfolio.write`, `trade.execute`, `planning.read`, `planning.write`, `crm.read`, `crm.write`
- **compliance_officer**: `client.read`, `portfolio.read`, `trade.approve`, `compliance.read`, `compliance.write`, `audit.read`
- **operations**: `client.read`, `portfolio.read`, `billing.read`, `billing.write`, `crm.read`
- **client**: `portfolio.read` (own accounts only), `document.read` (own documents)
- **readonly**: `client.read`, `portfolio.read`, `compliance.read`

**Testing**:
- `Unit: require_permission("trade.execute") allows advisor, blocks readonly user`
- `Unit: TenantContext.set_tenant sets PostgreSQL session variable`
- `Integration: Advisor accessing another firm's clients gets empty results (RLS)`
- `Integration: Client user can only see their own accounts`
- `Integration: Admin can access all resources within their firm`
- `Integration: Missing authorization header returns 401`
- `Integration: Expired token returns 401`

#### 2.3 — Firm & User Management API

**What**: CRUD endpoints for managing firms, users, roles, and role assignments.

**Design**:

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/api/v1/firms/current` | authenticated | Get current firm details |
| PATCH | `/api/v1/firms/current` | `admin.manage_firm` | Update firm settings |
| GET | `/api/v1/users` | `admin.manage_users` | List users in firm |
| POST | `/api/v1/users` | `admin.manage_users` | Invite/create user |
| GET | `/api/v1/users/{id}` | `admin.manage_users` | Get user details |
| PATCH | `/api/v1/users/{id}` | `admin.manage_users` | Update user |
| DELETE | `/api/v1/users/{id}` | `admin.manage_users` | Deactivate user |
| GET | `/api/v1/roles` | `admin.manage_users` | List roles |
| POST | `/api/v1/roles` | `admin.manage_users` | Create custom role |
| PUT | `/api/v1/users/{id}/roles` | `admin.manage_users` | Assign roles to user |

```python
# src/wealthmgmt/schemas/firm.py
class UserCreate(BaseModel):
    email: EmailStr
    display_name: str = Field(min_length=1, max_length=200)
    role_ids: list[uuid.UUID]
    send_invite: bool = True

class UserResponse(BaseModel):
    id: uuid.UUID
    email: str
    display_name: str
    is_active: bool
    roles: list[RoleResponse]
    last_login_at: datetime | None
    created_at: datetime
```

**Testing**:
- `Integration: POST /users creates user with specified roles`
- `Integration: POST /users with duplicate email returns 409 Conflict`
- `Integration: DELETE /users/{id} soft-deletes (sets is_active=false)`
- `Integration: Non-admin user cannot access /users endpoints (403)`
- `Integration: Created user appears in GET /users list`
- `Integration: Role assignment updates user permissions immediately`

---

## Phase 3: Client & Account Management

### Purpose
Implement the core CRM and account management functionality. After this phase, advisors can onboard clients, manage households, link accounts, and the system tracks full client profiles including KYC status and suitability data. This is the foundation for portfolio management and compliance.

### Tasks

#### 3.1 — Client CRUD & Household Management

**What**: Full client lifecycle management including onboarding, profile updates, household grouping, and KYC tracking.

**Design**:

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/api/v1/clients` | `client.read` | List clients with filtering/pagination |
| POST | `/api/v1/clients` | `client.write` | Create client |
| GET | `/api/v1/clients/{id}` | `client.read` | Get client detail |
| PATCH | `/api/v1/clients/{id}` | `client.write` | Update client |
| GET | `/api/v1/households` | `client.read` | List households |
| POST | `/api/v1/households` | `client.write` | Create household |
| PUT | `/api/v1/households/{id}/members` | `client.write` | Assign clients to household |
| GET | `/api/v1/households/{id}/summary` | `client.read` | Household AUM, members, accounts |
| PUT | `/api/v1/clients/{id}/advisor` | `client.write` | Assign advisor to client |

```python
# src/wealthmgmt/schemas/client.py
class ClientCreate(BaseModel):
    client_type: Literal["individual", "joint", "entity", "trust"]
    first_name: str | None = None  # required for individual/joint
    last_name: str | None = None
    entity_name: str | None = None  # required for entity/trust
    email: EmailStr | None = None
    phone: str | None = None
    date_of_birth: date | None = None
    country: str = Field(min_length=2, max_length=2)  # ISO 3166-1
    client_category: Literal["retail", "professional", "eligible_counterparty"] = "retail"
    household_id: uuid.UUID | None = None
    tax_id: str | None = None  # encrypted before storage

    @model_validator(mode="after")
    def validate_names(self) -> "ClientCreate":
        if self.client_type in ("individual", "joint") and not (self.first_name and self.last_name):
            raise ValueError("first_name and last_name required for individual/joint clients")
        if self.client_type in ("entity", "trust") and not self.entity_name:
            raise ValueError("entity_name required for entity/trust clients")
        return self

class ClientResponse(BaseModel):
    id: uuid.UUID
    client_type: str
    first_name: str | None
    last_name: str | None
    entity_name: str | None
    email: str | None
    country: str
    client_category: str
    kyc_status: str
    household: HouseholdBrief | None
    advisors: list[AdvisorBrief]
    created_at: datetime

class ClientListParams(BaseModel):
    """Query parameters for client listing."""
    search: str | None = None  # searches name, email
    client_type: str | None = None
    kyc_status: str | None = None
    household_id: uuid.UUID | None = None
    advisor_id: uuid.UUID | None = None
    page: int = Field(default=1, ge=1)
    per_page: int = Field(default=25, ge=1, le=100)
    sort_by: str = "last_name"
    sort_order: Literal["asc", "desc"] = "asc"

class HouseholdSummary(BaseModel):
    id: uuid.UUID
    name: str
    primary_advisor: AdvisorBrief | None
    members: list[ClientBrief]
    total_accounts: int
    total_aum: Decimal
    risk_profile: str | None
```

Pagination uses `Link` header (RFC 8288) for consistency with JSON:API conventions used by Addepar.

**Testing**:
- `Unit: ClientCreate validates individual requires first_name + last_name`
- `Unit: ClientCreate validates entity requires entity_name`
- `Unit: ClientCreate rejects invalid country code`
- `Integration: POST /clients creates client with encrypted tax_id`
- `Integration: GET /clients returns paginated list with Link header`
- `Integration: GET /clients?search=Smith filters by name`
- `Integration: POST /households creates household; PUT members assigns clients`
- `Integration: GET /households/{id}/summary returns aggregated AUM`
- `Integration: Client with kyc_status="expired" appears in KYC dashboard filter`
- `E2E: Create client → assign to household → verify household AUM updates`

#### 3.2 — Account Management

**What**: Investment account CRUD with custodian linking, model portfolio assignment, and account lifecycle management.

**Design**:

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/api/v1/accounts` | `portfolio.read` | List accounts |
| POST | `/api/v1/accounts` | `portfolio.write` | Create account |
| GET | `/api/v1/accounts/{id}` | `portfolio.read` | Get account with positions summary |
| PATCH | `/api/v1/accounts/{id}` | `portfolio.write` | Update account |
| GET | `/api/v1/clients/{id}/accounts` | `portfolio.read` | Client's accounts |
| PUT | `/api/v1/accounts/{id}/model` | `portfolio.write` | Assign model portfolio |
| GET | `/api/v1/custodians` | `portfolio.read` | List configured custodians |
| POST | `/api/v1/custodians` | `admin.manage_firm` | Configure custodian |

```python
# src/wealthmgmt/schemas/account.py
class AccountCreate(BaseModel):
    client_id: uuid.UUID
    custodian_id: uuid.UUID | None = None
    account_number: str = Field(min_length=1, max_length=50)
    account_name: str = Field(min_length=1, max_length=200)
    account_type: Literal[
        "individual", "joint", "ira_traditional", "ira_roth", "401k", "403b",
        "sep_ira", "trust", "corporate", "partnership", "custodial",
        "education_529", "other"
    ]
    tax_status: Literal["taxable", "tax_deferred", "tax_exempt"]
    base_currency: str = Field(default="USD", min_length=3, max_length=3)
    inception_date: date
    is_discretionary: bool = True
    is_fee_paying: bool = True

class AccountResponse(BaseModel):
    id: uuid.UUID
    client: ClientBrief
    custodian: CustodianBrief | None
    account_number: str
    account_name: str
    account_type: str
    tax_status: str
    base_currency: str
    inception_date: date
    status: str
    is_discretionary: bool
    total_value: Decimal | None
    assigned_model: ModelPortfolioBrief | None
    created_at: datetime
```

**Testing**:
- `Integration: POST /accounts creates account linked to client and custodian`
- `Integration: POST /accounts with duplicate (firm, account_number) returns 409`
- `Integration: GET /clients/{id}/accounts returns only that client's accounts`
- `Integration: PUT /accounts/{id}/model assigns model portfolio`
- `Integration: PATCH /accounts/{id} with status="closed" updates account`
- `Integration: Account creation generates audit log entry`

#### 3.3 — Reference Data Management

**What**: Seed and manage securities, asset classes, currencies, and countries.

**Design**:

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/api/v1/securities` | `portfolio.read` | Search securities |
| POST | `/api/v1/securities` | `admin.manage_firm` | Create security |
| GET | `/api/v1/securities/{id}` | `portfolio.read` | Security detail |
| GET | `/api/v1/securities/{id}/prices` | `portfolio.read` | Price history |
| POST | `/api/v1/securities/{id}/prices` | `admin.manage_firm` | Upload prices |
| GET | `/api/v1/asset-classes` | `portfolio.read` | List asset class hierarchy |
| GET | `/api/v1/currencies` | authenticated | List currencies |
| GET | `/api/v1/countries` | authenticated | List countries |

Security search supports lookup by ticker, ISIN, CUSIP, or name. Pagination and filtering by security_type and asset_class.

An Alembic data migration seeds ISO 3166-1 countries (~250 rows), ISO 4217 currencies (~180 rows), and a default asset class hierarchy (equity > US large cap / US mid cap / US small cap / international developed / emerging markets; fixed_income > government / corporate / municipal / high_yield; alternatives > real_estate / private_equity / commodities / hedge_funds; cash_equivalent).

**Testing**:
- `Integration: GET /securities?ticker=AAPL returns matching security`
- `Integration: GET /securities?isin=US0378331005 returns Apple`
- `Integration: POST /securities creates security with all identifier fields`
- `Integration: POST /securities/{id}/prices bulk-loads daily prices`
- `Integration: GET /asset-classes returns hierarchical tree structure`
- `Integration: Seeded reference data includes USD, EUR, GBP currencies and US, GB, DE countries`

#### 3.4 — CRM Activities & Documents

**What**: Activity logging (calls, meetings, notes, tasks) and document management for client files.

**Design**:

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/api/v1/activities` | `crm.read` | List activities (filterable by client, type, date) |
| POST | `/api/v1/activities` | `crm.write` | Create activity |
| PATCH | `/api/v1/activities/{id}` | `crm.write` | Update activity |
| GET | `/api/v1/clients/{id}/activities` | `crm.read` | Client activity timeline |
| POST | `/api/v1/documents` | `document.write` | Upload document (multipart) |
| GET | `/api/v1/documents/{id}` | `document.read` | Get document metadata |
| GET | `/api/v1/documents/{id}/download` | `document.read` | Download document file |
| GET | `/api/v1/clients/{id}/documents` | `document.read` | Client documents |

Documents are stored in local filesystem (development) or S3-compatible object storage (production), referenced by `storage_key` in the database. MiFID II mandates 5-year minimum retention; the `retention_until` field enforces this.

```python
class ActivityCreate(BaseModel):
    client_id: uuid.UUID | None = None
    household_id: uuid.UUID | None = None
    activity_type: Literal["call", "email", "meeting", "note", "task", "review", "document", "other"]
    subject: str = Field(min_length=1, max_length=500)
    description: str | None = None
    scheduled_at: datetime | None = None
    status: Literal["open", "in_progress", "completed", "cancelled"] = "open"
```

**Testing**:
- `Integration: POST /activities creates activity linked to client`
- `Integration: GET /clients/{id}/activities returns chronological timeline`
- `Integration: POST /documents uploads file and stores metadata`
- `Integration: GET /documents/{id}/download returns file with correct Content-Type`
- `Integration: Document retention_until defaults to 5 years from creation`
- `Unit: Activity status transitions are validated (cannot go from cancelled to open)`

---

## Phase 4: Portfolio Management & Positions

### Purpose
Implement portfolio tracking, position management, model portfolios, and the transaction ledger. After this phase, the system can track what securities are held in each account, record all transactions, manage tax lots, and define model portfolios with target allocations.

### Tasks

#### 4.1 — Position & Transaction Management

**What**: Record and query positions and transactions for investment accounts.

**Design**:

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/api/v1/accounts/{id}/positions` | `portfolio.read` | Current positions for account |
| GET | `/api/v1/accounts/{id}/positions/summary` | `portfolio.read` | Aggregated position summary |
| POST | `/api/v1/accounts/{id}/transactions` | `portfolio.write` | Record transaction |
| GET | `/api/v1/accounts/{id}/transactions` | `portfolio.read` | Transaction history |
| GET | `/api/v1/accounts/{id}/lots` | `portfolio.read` | Tax lot detail |
| GET | `/api/v1/households/{id}/positions` | `portfolio.read` | Household consolidated positions |

```python
# src/wealthmgmt/schemas/portfolio.py
class TransactionCreate(BaseModel):
    security_id: uuid.UUID | None = None  # null for cash transactions
    transaction_type: Literal[
        "buy", "sell", "dividend", "interest", "contribution", "withdrawal",
        "transfer_in", "transfer_out", "fee", "tax", "split", "merger",
        "reinvestment", "return_of_capital", "other"
    ]
    trade_date: date
    settlement_date: date | None = None
    quantity: Decimal | None = None
    price: Decimal | None = None
    gross_amount: Decimal
    commission: Decimal = Decimal("0")
    fees: Decimal = Decimal("0")
    currency: str = "USD"
    custodian_ref: str | None = None
    description: str | None = None

class PositionResponse(BaseModel):
    id: uuid.UUID
    security: SecurityBrief
    quantity: Decimal
    cost_basis: Decimal
    market_value: Decimal | None
    unrealised_gain: Decimal | None
    weight_pct: Decimal | None  # position weight in portfolio
    as_of_date: date
    lots: list[LotBrief]

class PositionSummary(BaseModel):
    total_market_value: Decimal
    total_cost_basis: Decimal
    total_unrealised_gain: Decimal
    total_realised_gain_ytd: Decimal
    positions_count: int
    asset_allocation: list[AssetAllocationItem]  # by asset class

class AssetAllocationItem(BaseModel):
    asset_class: str
    market_value: Decimal
    weight_pct: Decimal
    target_pct: Decimal | None  # from model portfolio if assigned
    drift_pct: Decimal | None   # actual - target
```

**Position update logic**: When a buy/sell transaction is recorded, the service layer:
1. Creates the `transaction` row
2. For buys: creates/updates `lot` (new tax lot with acquisition date and cost basis)
3. For sells: closes lots using configured method (FIFO by default), calculates realised gain, sets `is_short_term` based on holding period
4. Updates `position` row (quantity, cost_basis, market_value)
5. Publishes position-change event for downstream recalculation (performance, drift)

Tax lot selection methods: FIFO (default), LIFO, highest-cost, lowest-cost, specific identification.

**Testing**:
- `Unit: Buy transaction creates new lot with correct cost basis`
- `Unit: Sell transaction (FIFO) closes oldest lot first`
- `Unit: Sell of 150 shares across two lots (100 + 50) calculates blended cost basis`
- `Unit: Short-term vs long-term classification based on holding period (365 days)`
- `Unit: Wash sale flag set when repurchase within 30 days of loss sale`
- `Integration: POST /accounts/{id}/transactions records buy, updates position`
- `Integration: GET /accounts/{id}/positions returns current holdings with market values`
- `Integration: GET /accounts/{id}/positions/summary returns asset allocation`
- `Integration: GET /households/{id}/positions aggregates across all household accounts`
- `Fixture-based: Load sample_transactions.csv, verify position state matches expected`

#### 4.2 — Model Portfolio Management

**What**: Define model portfolios with target allocations and drift thresholds; assign models to accounts.

**Design**:

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/api/v1/models` | `portfolio.read` | List model portfolios |
| POST | `/api/v1/models` | `portfolio.write` | Create model portfolio |
| GET | `/api/v1/models/{id}` | `portfolio.read` | Model detail with allocations |
| PATCH | `/api/v1/models/{id}` | `portfolio.write` | Update model |
| PUT | `/api/v1/models/{id}/allocations` | `portfolio.write` | Set target allocations |
| GET | `/api/v1/models/{id}/drift` | `portfolio.read` | Drift analysis for all assigned accounts |

```python
class ModelPortfolioCreate(BaseModel):
    name: str = Field(min_length=1, max_length=200)
    description: str | None = None
    risk_level: Literal["conservative", "moderate_conservative", "moderate",
                        "moderate_aggressive", "aggressive"] | None = None
    benchmark_id: uuid.UUID | None = None

class AllocationTarget(BaseModel):
    security_id: uuid.UUID | None = None   # specific security
    asset_class_id: uuid.UUID | None = None  # or asset class level
    target_weight: Decimal = Field(ge=0, le=100)
    min_weight: Decimal | None = None
    max_weight: Decimal | None = None
    drift_threshold: Decimal | None = None  # trigger rebalance when exceeded

class DriftAnalysis(BaseModel):
    account_id: uuid.UUID
    account_name: str
    total_value: Decimal
    allocations: list[DriftItem]
    max_drift: Decimal
    needs_rebalance: bool

class DriftItem(BaseModel):
    security: SecurityBrief | None
    asset_class: str | None
    target_pct: Decimal
    actual_pct: Decimal
    drift_pct: Decimal  # actual - target
    drift_amount: Decimal  # dollar drift
    exceeds_threshold: bool
```

Drift calculation: for each account assigned to a model, compare actual asset allocation weights to target weights. Flag allocations where `abs(actual - target) > drift_threshold`.

**Testing**:
- `Unit: Allocation weights must sum to 100%`
- `Unit: Drift calculation correctly identifies over/under-weight positions`
- `Unit: needs_rebalance=True when any allocation exceeds its drift_threshold`
- `Integration: POST /models creates model; PUT allocations sets targets`
- `Integration: GET /models/{id}/drift shows drift for all assigned accounts`
- `Integration: Account with 45% equity (target 40%, threshold 5%) shows exceeds_threshold=True`

#### 4.3 — Market Data & Position Valuation

**What**: Daily position valuation using security prices, with batch update capability.

**Design**:

```python
# src/wealthmgmt/services/portfolio_service.py
class PortfolioService:
    async def update_market_values(self, firm_id: uuid.UUID, as_of_date: date) -> int:
        """
        Update market_value and unrealised_gain for all positions in the firm
        using the latest security prices as of the given date.

        Returns count of positions updated.
        """
        # 1. Get all active positions for firm
        # 2. For each position, look up security_price for as_of_date (or latest before)
        # 3. market_value = quantity * close_price * exchange_rate (if multi-currency)
        # 4. unrealised_gain = market_value - cost_basis
        # 5. Update household total_aum via trigger or service call
        ...

    async def get_asset_allocation(self, account_id: uuid.UUID) -> list[AssetAllocationItem]:
        """
        Calculate current asset allocation for an account by summing
        position market values grouped by asset class.
        """
        ...
```

Celery task `sync_prices` runs daily (configurable schedule) to:
1. Fetch latest prices from configured data source (Morningstar API or CSV upload initially)
2. Insert into `security_price` table
3. Call `update_market_values` for each firm
4. Trigger drift recalculation for all model-assigned accounts

**Testing**:
- `Unit: Market value = quantity * price for same-currency positions`
- `Unit: Multi-currency position applies exchange rate correctly`
- `Unit: Missing price for as_of_date falls back to most recent prior date`
- `Unit: Unrealised gain = market_value - cost_basis`
- `Integration: Bulk price upload via POST /securities/{id}/prices updates all affected positions`
- `Integration: Household AUM reflects sum of all account positions`
- `Fixture-based: Load sample_prices.csv, run valuation, verify expected market values`

---

## Phase 5: Trading & Compliance Engine

### Purpose
Implement the trade order workflow with pre-trade compliance checking, suitability assessment tracking, and compliance rule management. After this phase, advisors can create trade orders that are automatically checked against compliance rules before approval, and all compliance decisions are documented for Reg BI / MiFID II audit trails.

### Tasks

#### 5.1 — Trade Order Workflow

**What**: Full trade order lifecycle from creation through compliance review to execution.

**Design**:

Trade order state machine:
```
pending → compliance_review → approved → submitted → partial_fill → filled
                            ↘ rejected
         → cancelled (from any pre-submitted state)
```

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| POST | `/api/v1/orders` | `trade.execute` | Create trade order |
| GET | `/api/v1/orders` | `portfolio.read` | List orders (filterable) |
| GET | `/api/v1/orders/{id}` | `portfolio.read` | Order detail with compliance |
| POST | `/api/v1/orders/{id}/approve` | `trade.approve` | Approve order |
| POST | `/api/v1/orders/{id}/reject` | `trade.approve` | Reject order |
| POST | `/api/v1/orders/{id}/cancel` | `trade.execute` | Cancel order |
| POST | `/api/v1/orders/{id}/fill` | `trade.execute` | Record fill (manual or from custodian) |

```python
class TradeOrderCreate(BaseModel):
    account_id: uuid.UUID
    security_id: uuid.UUID
    side: Literal["buy", "sell"]
    order_type: Literal["market", "limit", "stop", "stop_limit"]
    quantity: Decimal = Field(gt=0)
    limit_price: Decimal | None = None
    stop_price: Decimal | None = None
    time_in_force: Literal["day", "gtc", "ioc", "fok"] = "day"

class TradeOrderFill(BaseModel):
    filled_quantity: Decimal
    filled_price: Decimal
    settlement_date: date | None = None
    fix_exec_id: str | None = None
    custodian_ref: str | None = None

class TradeOrderResponse(BaseModel):
    id: uuid.UUID
    account: AccountBrief
    security: SecurityBrief
    side: str
    order_type: str
    quantity: Decimal
    limit_price: Decimal | None
    status: str
    compliance_checks: list[ComplianceCheckResult]
    created_by: UserBrief
    approved_by: UserBrief | None
    filled_quantity: Decimal | None
    filled_price: Decimal | None
    created_at: datetime
    updated_at: datetime
```

When a trade order is created:
1. Status set to `pending`
2. Compliance service runs all active compliance rules for the account
3. If all pass → status becomes `approved` (if auto-approve is enabled) or `compliance_review`
4. If any fail → status becomes `compliance_review` with failures documented
5. Compliance officer can override failures with documented reason
6. Upon approval, order status moves to `approved`
7. Fill records execution details and creates corresponding `transaction` and updates `position`/`lot`

**Testing**:
- `Unit: Order creation validates limit_price required for limit orders`
- `Unit: Order creation validates sell quantity <= current position quantity`
- `Unit: Fill creates buy transaction and new lot`
- `Unit: Fill creates sell transaction and closes lots (FIFO)`
- `Unit: Cannot approve order that has unresolved compliance failures`
- `Integration: POST /orders creates order, runs compliance, returns result`
- `Integration: POST /orders/{id}/approve transitions status and creates audit entry`
- `Integration: POST /orders/{id}/fill records execution, updates position`
- `Integration: POST /orders/{id}/cancel from pending state succeeds`
- `Integration: POST /orders/{id}/cancel from filled state fails with 409`

#### 5.2 — Compliance Rule Engine

**What**: Configurable pre-trade and post-trade compliance rules with automated checking.

**Design**:

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/api/v1/compliance/rules` | `compliance.read` | List compliance rules |
| POST | `/api/v1/compliance/rules` | `compliance.write` | Create rule |
| PATCH | `/api/v1/compliance/rules/{id}` | `compliance.write` | Update rule |
| GET | `/api/v1/compliance/checks` | `compliance.read` | List compliance checks (audit trail) |
| POST | `/api/v1/compliance/checks/{id}/override` | `compliance.write` | Override failed check |

Rule types (from Data Model Suggestion 1):

```python
class ComplianceRuleType(str, Enum):
    RESTRICTED_SECURITY = "restricted_security"       # Block trades in specific securities
    CONCENTRATION_LIMIT = "concentration_limit"       # Max % of portfolio in one security
    SECTOR_LIMIT = "sector_limit"                     # Max % in a sector/asset class
    TRADE_APPROVAL = "trade_approval"                 # Require manual approval above threshold
    HOLDING_PERIOD = "holding_period"                 # Min holding period before sell
    ESG_EXCLUSION = "esg_exclusion"                   # Block ESG-excluded securities

# Example rule parameters (JSONB)
restricted_security_params = {
    "security_ids": ["uuid1", "uuid2"],
    "reason": "Insider trading restriction - CEO personal account"
}

concentration_limit_params = {
    "max_weight_pct": 10.0,
    "scope": "single_security"  # or "sector", "asset_class"
}

sector_limit_params = {
    "restricted_sectors": ["tobacco", "weapons", "gambling"],
    "max_weight_pct": 0  # full exclusion
}
```

```python
# src/wealthmgmt/services/compliance_service.py
class ComplianceService:
    async def check_trade(
        self, order: TradeOrder, account: Account, db: AsyncSession
    ) -> list[ComplianceCheck]:
        """
        Run all active compliance rules against a proposed trade.
        Returns list of ComplianceCheck results.
        """
        rules = await self._get_active_rules(account.firm_id, db)
        checks = []
        for rule in rules:
            result = await self._evaluate_rule(rule, order, account, db)
            check = ComplianceCheck(
                firm_id=account.firm_id,
                trade_order_id=order.id,
                rule_id=rule.id,
                account_id=account.id,
                result=result.status,  # pass, fail, warning
                details=result.details,
            )
            checks.append(check)
        return checks

    async def _evaluate_rule(
        self, rule: ComplianceRule, order: TradeOrder, account: Account, db: AsyncSession
    ) -> RuleResult:
        match rule.rule_type:
            case "restricted_security":
                return self._check_restricted(rule, order)
            case "concentration_limit":
                return await self._check_concentration(rule, order, account, db)
            case "sector_limit":
                return await self._check_sector(rule, order, account, db)
            case "holding_period":
                return await self._check_holding_period(rule, order, account, db)
            case "esg_exclusion":
                return await self._check_esg(rule, order, db)
```

**Testing**:
- `Unit: restricted_security rule blocks trade in restricted security`
- `Unit: restricted_security rule passes for non-restricted security`
- `Unit: concentration_limit rule fails when post-trade weight > max_weight_pct`
- `Unit: concentration_limit rule passes when post-trade weight <= max_weight_pct`
- `Unit: holding_period rule fails for sell within minimum holding days`
- `Unit: esg_exclusion rule blocks buy in excluded sector`
- `Integration: POST /compliance/rules creates rule visible in pre-trade checks`
- `Integration: Trade order with compliance failure shows failed checks in response`
- `Integration: Override creates audit trail with overrider identity and reason`

#### 5.3 — Suitability Assessment & Tracking

**What**: Client suitability questionnaire, assessment recording, and Reg BI / MiFID II documentation.

**Design**:

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| POST | `/api/v1/clients/{id}/suitability` | `compliance.write` | Record suitability assessment |
| GET | `/api/v1/clients/{id}/suitability` | `compliance.read` | Get current suitability profile |
| GET | `/api/v1/clients/{id}/suitability/history` | `compliance.read` | Assessment history |
| GET | `/api/v1/compliance/suitability/expiring` | `compliance.read` | Assessments expiring within N days |

```python
class SuitabilityCreate(BaseModel):
    investment_objective: Literal[
        "capital_preservation", "income", "growth_income",
        "growth", "aggressive_growth", "speculation"
    ]
    risk_tolerance: Literal["low", "moderate_low", "moderate", "moderate_high", "high"]
    time_horizon: Literal["short", "medium", "long", "very_long"]
    liquidity_needs: Literal["high", "moderate", "low"] | None = None
    annual_income_range: str | None = None
    net_worth_range: str | None = None
    investment_experience: Literal["none", "limited", "moderate", "extensive"] | None = None
    # Reg BI / MiFID II specific
    conflicts_disclosed: bool = False
    basis_for_recommendation: str | None = None
    expires_at: date | None = None  # defaults to 12 months from assessment

class SuitabilityResponse(BaseModel):
    id: uuid.UUID
    client_id: uuid.UUID
    assessed_by: UserBrief
    assessment_date: date
    investment_objective: str
    risk_tolerance: str
    time_horizon: str
    status: str  # active, superseded, expired
    expires_at: date | None
    created_at: datetime
```

When a new assessment is recorded, the previous active assessment's status is set to `superseded`. Expired assessments are flagged by a daily Celery task that checks `expires_at`.

**Testing**:
- `Unit: New assessment supersedes previous active assessment`
- `Unit: Expired assessment flagged when expires_at < today`
- `Integration: POST /clients/{id}/suitability records assessment with all Reg BI fields`
- `Integration: GET /compliance/suitability/expiring?days=30 returns assessments expiring soon`
- `Integration: Suitability assessment creates audit log entry`

---

## Phase 6: Rebalancing Engine

### Purpose
Implement the portfolio rebalancing engine that generates trade proposals to bring accounts back to target model allocations, with tax-loss harvesting recommendations. After this phase, advisors can trigger rebalancing for individual accounts or in bulk, review proposed trades, and execute approved rebalance batches.

### Tasks

#### 6.1 — Rebalancing Proposal Generator

**What**: Algorithm that compares current positions to model targets and generates optimal trade proposals.

**Design**:

```python
# src/wealthmgmt/services/rebalancing_service.py
class RebalancingService:
    async def generate_proposal(
        self,
        account_ids: list[uuid.UUID],
        model_id: uuid.UUID,
        options: RebalanceOptions,
        db: AsyncSession,
    ) -> RebalanceProposal:
        """
        Generate rebalance proposal for given accounts against model portfolio.

        Algorithm:
        1. Load model allocations (target weights)
        2. For each account:
           a. Load current positions and market values
           b. Calculate current weights
           c. Calculate drift per allocation
           d. If tax_loss_harvest: identify positions with unrealised losses > threshold
           e. Generate trades to close drift:
              - Sell overweight positions (prioritise loss harvesting if enabled)
              - Buy underweight positions
              - Respect min_trade_amount to avoid dust trades
              - Respect cash buffer (keep N% in cash)
        3. Aggregate trades into proposal
        """
        ...

class RebalanceOptions(BaseModel):
    tax_loss_harvest: bool = False
    min_trade_amount: Decimal = Decimal("100")
    cash_buffer_pct: Decimal = Decimal("2.0")
    lot_selection: Literal["fifo", "lifo", "highest_cost", "tax_optimized"] = "tax_optimized"
    respect_wash_sale: bool = True
    exclude_securities: list[uuid.UUID] = []
```

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| POST | `/api/v1/rebalance/propose` | `trade.execute` | Generate rebalance proposal |
| GET | `/api/v1/rebalance/{id}` | `portfolio.read` | Get proposal detail |
| POST | `/api/v1/rebalance/{id}/approve` | `trade.approve` | Approve proposal |
| POST | `/api/v1/rebalance/{id}/execute` | `trade.execute` | Execute approved proposal (creates trade orders) |
| DELETE | `/api/v1/rebalance/{id}` | `trade.execute` | Cancel proposal |

```python
class RebalanceProposalResponse(BaseModel):
    id: uuid.UUID
    model: ModelPortfolioBrief
    status: str
    tax_loss_harvest: bool
    trades: list[ProposedTrade]
    summary: RebalanceSummary
    created_by: UserBrief
    created_at: datetime

class ProposedTrade(BaseModel):
    account: AccountBrief
    security: SecurityBrief
    side: str
    quantity: Decimal
    estimated_amount: Decimal
    reason: str  # drift_correction, tax_loss_harvest, cash_raise
    estimated_tax_impact: Decimal | None  # for sells

class RebalanceSummary(BaseModel):
    total_accounts: int
    total_trades: int
    total_buy_amount: Decimal
    total_sell_amount: Decimal
    estimated_tax_savings: Decimal | None  # from tax-loss harvesting
```

Execution: when a proposal is executed, each proposed trade becomes a `trade_order` that flows through the standard compliance check workflow (Phase 5.1). The proposal tracks which trades have been executed via `trade_order_id` on `rebalance_proposal_trade`.

**Testing**:
- `Unit: Account with 50% equity (target 40%) generates sell orders to reduce to 40%`
- `Unit: Account with 30% bonds (target 40%) generates buy orders`
- `Unit: min_trade_amount filters out trades below $100`
- `Unit: cash_buffer_pct reserves 2% of portfolio in cash`
- `Unit: tax_loss_harvest identifies positions with unrealised losses > $200`
- `Unit: Wash sale check prevents harvesting if same security sold within 30 days`
- `Unit: lot_selection="tax_optimized" sells highest-cost lots first for loss harvesting`
- `Integration: POST /rebalance/propose generates proposal with correct trades`
- `Integration: POST /rebalance/{id}/execute creates trade orders for each proposed trade`
- `Integration: Each generated trade order goes through compliance checks`
- `Fixture-based: Portfolio with known drift → verify exact proposed trades match expected`

---

## Phase 7: Performance Reporting (GIPS-Aligned)

### Purpose
Implement performance return calculations, composite management, and GIPS-compliant reporting. After this phase, the system calculates time-weighted and money-weighted returns for individual accounts, manages GIPS composites, and generates performance reports.

### Tasks

#### 7.1 — Return Calculation Engine

**What**: Calculate time-weighted returns (TWR) and money-weighted returns (MWR/IRR) for accounts.

**Design**:

```python
# src/wealthmgmt/services/performance_service.py
class PerformanceService:
    def calculate_twr(
        self,
        beginning_value: Decimal,
        ending_value: Decimal,
        cash_flows: list[CashFlow],
    ) -> Decimal:
        """
        Modified Dietz method for time-weighted return.

        TWR = (ending_value - beginning_value - net_cash_flow) /
              (beginning_value + weighted_cash_flow)

        where weighted_cash_flow = sum(cf.amount * (days_remaining / total_days))
        """
        ...

    def calculate_mwr(
        self,
        beginning_value: Decimal,
        ending_value: Decimal,
        cash_flows: list[CashFlow],
    ) -> Decimal:
        """
        Money-weighted return (IRR) using Newton's method.
        Solves: beginning_value * (1+r)^n + sum(cf * (1+r)^(n-t)) = ending_value
        """
        ...

    async def calculate_period_returns(
        self,
        account_id: uuid.UUID,
        period_start: date,
        period_end: date,
        db: AsyncSession,
    ) -> AccountReturn:
        """
        Calculate returns for a specific period.
        Loads positions at start/end, transactions during period, and computes TWR/MWR.
        """
        ...

    async def calculate_cumulative_return(
        self,
        account_id: uuid.UUID,
        start_date: date,
        end_date: date,
        db: AsyncSession,
    ) -> Decimal:
        """
        Chain monthly TWR returns to get cumulative return over arbitrary period.
        cumulative = product(1 + monthly_twr) - 1
        """
        ...

class CashFlow(BaseModel):
    date: date
    amount: Decimal  # positive = inflow, negative = outflow
```

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/api/v1/accounts/{id}/performance` | `portfolio.read` | Account returns (monthly/quarterly/annual) |
| POST | `/api/v1/performance/calculate` | `admin.manage_firm` | Trigger return calculation for period |
| GET | `/api/v1/performance/summary` | `portfolio.read` | Firm-wide performance summary |

Celery task `calculate_returns` runs monthly after month-end to compute returns for all active accounts. Results stored in `account_return` table.

**Testing**:
- `Unit: TWR for period with no cash flows = (end - start) / start`
- `Unit: TWR with mid-period contribution uses Modified Dietz weighting`
- `Unit: MWR converges to expected IRR for known cash flow series`
- `Unit: Cumulative return chains monthly returns correctly`
- `Unit: Benchmark return stored alongside account return`
- `Fixture-based: Known portfolio scenario from GIPS guidance → expected TWR matches`
- `Integration: POST /performance/calculate computes and stores returns`
- `Integration: GET /accounts/{id}/performance returns monthly, QTD, YTD, ITD`

#### 7.2 — GIPS Composite Management

**What**: Manage GIPS composites, composite membership, and composite-level return calculations.

**Design**:

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/api/v1/composites` | `portfolio.read` | List composites |
| POST | `/api/v1/composites` | `compliance.write` | Create composite |
| GET | `/api/v1/composites/{id}` | `portfolio.read` | Composite detail |
| PUT | `/api/v1/composites/{id}/members` | `compliance.write` | Add/remove account membership |
| GET | `/api/v1/composites/{id}/returns` | `portfolio.read` | Composite returns |

GIPS composite return calculation:
```python
async def calculate_composite_return(
    self, composite_id: uuid.UUID, period_start: date, period_end: date, db: AsyncSession
) -> CompositeReturn:
    """
    Asset-weighted composite return per GIPS 2020.

    1. Get all accounts that were members for the full period
    2. Get each account's beginning value and TWR for the period
    3. Asset-weighted return = sum(account_twr * account_beginning_value) / sum(beginning_values)
    4. Composite dispersion = asset-weighted standard deviation of account returns
    """
    ...
```

GIPS rules enforced:
- Only discretionary, fee-paying accounts eligible for composites
- Account must be member for full reporting period to be included
- Composite dispersion reported when >= 6 accounts

**Testing**:
- `Unit: Asset-weighted return calculation with 3 accounts matches manual computation`
- `Unit: Composite dispersion calculation correct for known return distribution`
- `Unit: Non-discretionary account excluded from composite`
- `Unit: Account joining mid-period excluded from that period's composite return`
- `Integration: POST /composites creates composite; PUT members adds accounts`
- `Integration: GET /composites/{id}/returns shows annual returns with dispersion`

---

## Phase 8: Billing & Fee Management

### Purpose
Implement AUM-based fee calculation, billing runs, and fee schedule management. After this phase, firms can define fee schedules, assign them to accounts, run billing calculations, and generate billing records.

### Tasks

#### 8.1 — Fee Schedule & Billing Engine

**What**: Configurable fee schedules with tiered AUM pricing and automated billing runs.

**Design**:

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/api/v1/billing/fee-schedules` | `billing.read` | List fee schedules |
| POST | `/api/v1/billing/fee-schedules` | `billing.write` | Create fee schedule |
| PUT | `/api/v1/accounts/{id}/fee-schedule` | `billing.write` | Assign fee schedule to account |
| POST | `/api/v1/billing/calculate` | `billing.write` | Run billing for period |
| GET | `/api/v1/billing/records` | `billing.read` | List billing records |
| POST | `/api/v1/billing/records/{id}/approve` | `billing.write` | Approve billing record |

```python
# src/wealthmgmt/services/billing_service.py
class BillingService:
    def calculate_aum_fee(
        self,
        billable_aum: Decimal,
        tiers: list[dict],
        billing_frequency: str,
    ) -> Decimal:
        """
        Calculate fee using tiered AUM schedule.

        Example tiers: [
            {"min_aum": 0, "max_aum": 1000000, "rate_bps": 100},
            {"min_aum": 1000000, "max_aum": 5000000, "rate_bps": 75},
            {"min_aum": 5000000, "max_aum": null, "rate_bps": 50}
        ]

        For quarterly billing at $2M AUM:
        - First $1M: 1M * 100bps / 4 = $2,500
        - Next $1M: 1M * 75bps / 4 = $1,875
        - Total: $4,375
        """
        ...

    async def run_billing(
        self,
        firm_id: uuid.UUID,
        period_start: date,
        period_end: date,
        db: AsyncSession,
    ) -> list[BillingRecord]:
        """
        Run billing for all fee-paying accounts in the firm.
        1. Get all active, fee-paying accounts with assigned fee schedules
        2. For each account, get billable AUM (average daily balance or period-end)
        3. Calculate fee using the assigned fee schedule
        4. Create billing_record with status='calculated'
        """
        ...
```

Billing frequency options: monthly, quarterly, semi_annual, annual. Billing method: advance (beginning-of-period AUM) or arrears (end-of-period AUM).

**Testing**:
- `Unit: Tiered fee calculation for $500K AUM = 500K * 100bps = $5000/yr = $1250/qtr`
- `Unit: Tiered fee for $2M AUM spans two tiers correctly`
- `Unit: Tiered fee for $10M AUM spans all three tiers`
- `Unit: Monthly billing divides annual fee by 12; quarterly by 4`
- `Unit: Advance billing uses beginning-of-period AUM`
- `Integration: POST /billing/calculate creates billing records for all eligible accounts`
- `Integration: POST /billing/records/{id}/approve updates status and creates audit entry`
- `Integration: Non-fee-paying accounts are excluded from billing run`

---

## Phase 9: Custodian Integration Layer

### Purpose
Build the integration framework for connecting to custodian APIs (Schwab, Plaid) and synchronising account data. After this phase, the platform can pull real account positions, transactions, and balances from external custodians.

### Tasks

#### 9.1 — Custodian Integration Framework

**What**: Abstract interface for custodian integrations with concrete Schwab and Plaid implementations.

**Design**:

```python
# src/wealthmgmt/integrations/custodian_base.py
from abc import ABC, abstractmethod

class CustodianIntegration(ABC):
    """Base class for custodian API integrations."""

    @abstractmethod
    async def authenticate(self, credentials: dict) -> AuthToken:
        """Authenticate with the custodian API."""
        ...

    @abstractmethod
    async def get_accounts(self, auth: AuthToken) -> list[CustodianAccount]:
        """Fetch list of accounts from custodian."""
        ...

    @abstractmethod
    async def get_positions(self, auth: AuthToken, account_id: str) -> list[CustodianPosition]:
        """Fetch current positions for an account."""
        ...

    @abstractmethod
    async def get_transactions(
        self, auth: AuthToken, account_id: str, start_date: date, end_date: date
    ) -> list[CustodianTransaction]:
        """Fetch transactions for a date range."""
        ...

class CustodianAccount(BaseModel):
    external_id: str
    account_number: str
    account_type: str
    total_value: Decimal | None

class CustodianPosition(BaseModel):
    security_identifier: str  # ticker or CUSIP
    identifier_type: str  # "ticker", "cusip", "isin"
    quantity: Decimal
    market_value: Decimal
    cost_basis: Decimal | None

class CustodianTransaction(BaseModel):
    external_id: str
    transaction_type: str
    trade_date: date
    settlement_date: date | None
    security_identifier: str | None
    quantity: Decimal | None
    price: Decimal | None
    amount: Decimal
    description: str | None
```

```python
# src/wealthmgmt/integrations/schwab.py
class SchwabIntegration(CustodianIntegration):
    """
    Schwab Trader API integration.
    Auth: OAuth 2.0 with PKCE (three-legged auth code flow).
    Access tokens: 30 min; refresh tokens: 7 days.
    Base URL: https://api.schwabapi.com/trader/v1
    """
    ...

# src/wealthmgmt/integrations/plaid.py
class PlaidIntegration(CustodianIntegration):
    """
    Plaid Investments API integration.
    Auth: API Key + Plaid Link flow for user consent.
    Endpoints: /investments/holdings/get, /investments/transactions/get
    """
    ...
```

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| POST | `/api/v1/integrations/custodians/{id}/sync` | `admin.manage_firm` | Trigger manual sync |
| GET | `/api/v1/integrations/custodians/{id}/status` | `admin.manage_firm` | Sync status and last sync time |
| POST | `/api/v1/integrations/custodians/{id}/connect` | `admin.manage_firm` | Initiate OAuth flow |
| GET | `/api/v1/integrations/custodians/{id}/callback` | — | OAuth callback handler |

**Testing**:
- `Unit (mocked API): Schwab get_positions maps API response to CustodianPosition`
- `Unit (mocked API): Plaid get_transactions maps response to CustodianTransaction`
- `Unit (mocked API): OAuth token refresh works when access token expired`
- `Integration (mocked API): Full sync cycle: fetch positions → reconcile → update database`
- `Integration (mocked API): New positions from custodian create position records`
- `Integration (mocked API): Quantity changes in custodian positions update local positions`

#### 9.2 — Position Reconciliation

**What**: Reconcile custodian-provided positions with locally tracked positions, flagging discrepancies.

**Design**:

```python
class ReconciliationService:
    async def reconcile(
        self, account_id: uuid.UUID, custodian_positions: list[CustodianPosition], db: AsyncSession
    ) -> ReconciliationResult:
        """
        Compare custodian positions with local positions.
        1. Match by security identifier (ticker/CUSIP/ISIN)
        2. Compare quantities and market values
        3. Flag discrepancies beyond tolerance (e.g., >$1 or >0.01 shares)
        4. Identify new positions (in custodian but not local)
        5. Identify closed positions (in local but not custodian)
        """
        ...

class ReconciliationResult(BaseModel):
    account_id: uuid.UUID
    matched: int
    discrepancies: list[Discrepancy]
    new_positions: list[CustodianPosition]
    closed_positions: list[PositionBrief]
    reconciled_at: datetime

class Discrepancy(BaseModel):
    security: SecurityBrief
    field: str  # "quantity" or "market_value"
    local_value: Decimal
    custodian_value: Decimal
    difference: Decimal
```

Celery task `sync_positions` runs on a configurable schedule (default: daily at 6 AM) to sync all connected custodian accounts.

**Testing**:
- `Unit: Matching positions with identical quantities → matched, no discrepancies`
- `Unit: Quantity difference of 0.001 shares → discrepancy flagged`
- `Unit: New custodian position not in local → listed in new_positions`
- `Unit: Local position not in custodian → listed in closed_positions`
- `Integration (mocked): Full sync creates reconciliation result with correct counts`

---

## Phase 10: Financial Planning

### Purpose
Implement goals-based financial planning with scenario modelling and Monte Carlo simulation. After this phase, advisors can create financial plans, set goals, run projections, and show clients their probability of meeting financial objectives.

### Tasks

#### 10.1 — Goals-Based Planning Engine

**What**: Financial goal management and scenario modelling with Monte Carlo simulation.

**Design**:

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/api/v1/clients/{id}/goals` | `planning.read` | List client goals |
| POST | `/api/v1/clients/{id}/goals` | `planning.write` | Create goal |
| PATCH | `/api/v1/goals/{id}` | `planning.write` | Update goal |
| POST | `/api/v1/clients/{id}/scenarios` | `planning.write` | Create planning scenario |
| GET | `/api/v1/scenarios/{id}` | `planning.read` | Get scenario with results |
| POST | `/api/v1/scenarios/{id}/run` | `planning.write` | Run Monte Carlo simulation |

```python
# src/wealthmgmt/services/planning_service.py
class PlanningService:
    def run_monte_carlo(
        self,
        current_portfolio_value: Decimal,
        monthly_contribution: Decimal,
        annual_return_mean: float,
        annual_return_std: float,
        inflation_rate: float,
        years: int,
        target_amount: Decimal,
        iterations: int = 10000,
    ) -> MonteCarloResult:
        """
        Run Monte Carlo simulation for retirement/goal planning.

        For each iteration:
        1. Simulate annual returns from normal distribution(mean, std)
        2. Apply inflation-adjusted contributions
        3. Compound portfolio value over the time horizon
        4. Record ending balance

        Returns percentile distribution and success probability.
        """
        # Use numpy for vectorised simulation
        rng = np.random.default_rng()
        annual_returns = rng.normal(annual_return_mean, annual_return_std, (iterations, years))
        # ... vectorised portfolio projection ...

class MonteCarloResult(BaseModel):
    iterations: int
    success_probability: float  # % of iterations meeting target
    median_ending_balance: Decimal
    percentile_10: Decimal
    percentile_25: Decimal
    percentile_75: Decimal
    percentile_90: Decimal
    shortfall_probability: float
    mean_shortfall_amount: Decimal | None  # average shortfall in failing scenarios
```

Planning scenarios store their parameters and results in `planning_scenario` table (JSONB for parameters and results, as defined in Data Model Suggestion 1).

**Testing**:
- `Unit: Monte Carlo with 7% mean, 0% std, 30 years → deterministic result`
- `Unit: Monte Carlo with high std → wider percentile spread`
- `Unit: success_probability increases with higher contributions`
- `Unit: Inflation adjustment reduces real ending balance`
- `Unit: 10,000 iterations produces stable (±2%) success probability on repeated runs`
- `Integration: POST /scenarios/{id}/run calculates and stores results`
- `Integration: GET /clients/{id}/goals returns goals with progress toward target`

---

## Phase 11: Frontend — Advisor Dashboard & Client Portal

### Purpose
Build the web application providing advisor workspaces and client-facing portal. After this phase, advisors can manage their practice through a web interface, and clients can view their portfolios and documents.

### Tasks

#### 11.1 — Frontend Scaffolding & Auth

**What**: Next.js project setup with authentication, layout, and API client.

**Design**:

Next.js App Router with two route groups:
- `(auth)` — login, register pages (unauthenticated)
- `(advisor)` — advisor workspace (requires advisor/admin/compliance/operations role)
- `(client)` — client portal (requires client role)

```typescript
// frontend/src/lib/api-client.ts
class ApiClient {
  private baseUrl: string;
  private token: string | null;

  async get<T>(path: string, params?: Record<string, string>): Promise<T>;
  async post<T>(path: string, body: unknown): Promise<T>;
  async patch<T>(path: string, body: unknown): Promise<T>;
  async delete(path: string): Promise<void>;

  // Automatic token refresh on 401
  private async handleResponse(response: Response): Promise<unknown>;
}
```

Auth flow: login page → POST /auth/login → store tokens in httpOnly cookies → redirect to dashboard. NextAuth.js handles session management and token refresh.

**Testing**:
- `E2E: User can register, login, and see dashboard`
- `E2E: Expired session redirects to login page`
- `E2E: Client user sees client portal, not advisor workspace`
- `Component: Login form validates email and password fields`
- `Component: Navigation shows role-appropriate menu items`

#### 11.2 — Advisor Dashboard

**What**: Main advisor workspace with portfolio overview, client list, recent activity, and alerts.

**Design**:

Dashboard page (`/dashboard`) shows:
- **AUM summary**: total firm AUM, AUM by advisor, monthly change
- **Client list widget**: top 10 clients by AUM with quick-access links
- **Recent activity**: latest 10 activities across all clients
- **Alerts panel**: expiring KYC, accounts needing rebalance (drift > threshold), upcoming reviews
- **Performance summary**: firm-level YTD performance vs benchmark

Key pages:
- `/clients` — searchable, sortable client list with filters
- `/clients/[id]` — client detail with tabs: Profile, Accounts, Activity, Documents, Suitability, Planning
- `/portfolios` — portfolio browser with positions, allocation charts
- `/portfolios/[id]` — account detail with positions table, transaction history, performance chart
- `/trading` — order blotter with status filters, create order form
- `/trading/rebalance` — rebalancing interface: select model, select accounts, generate/review/execute
- `/compliance` — compliance dashboard: pending approvals, rule management, suitability tracker
- `/billing` — billing run interface, fee schedule management

Components:
- `PortfolioAllocationChart` — pie/donut chart of asset allocation (Recharts)
- `PerformanceChart` — line chart of cumulative returns vs benchmark
- `DriftTable` — data table showing model drift per allocation
- `OrderBlotter` — sortable, filterable table of trade orders
- `ClientTimeline` — chronological activity timeline

**Testing**:
- `E2E: Dashboard loads with AUM summary and client list`
- `E2E: Client detail page shows all tabs with correct data`
- `E2E: Portfolio page shows positions with allocation chart`
- `E2E: Creating a trade order from the trading page works end-to-end`
- `Component: PortfolioAllocationChart renders with sample data`
- `Component: DriftTable highlights allocations exceeding threshold`
- `Component: OrderBlotter filters by status`

#### 11.3 — Client Portal

**What**: Client-facing portal for viewing portfolio, downloading documents, and reviewing financial plans.

**Design**:

Client portal pages (`/portal/*`):
- `/portal` — portfolio summary: total value, asset allocation chart, YTD performance
- `/portal/accounts` — list of accounts with balances
- `/portal/accounts/[id]` — account positions and recent transactions
- `/portal/documents` — downloadable statements, tax forms, agreements
- `/portal/planning` — financial goals with progress bars and Monte Carlo results

Client portal is read-only. Clients cannot create trades, modify accounts, or change settings. The portal uses the same API endpoints but with `client` role permissions that restrict data to the authenticated client's accounts only.

Design principles:
- Clean, simple layout (not the dense advisor workspace)
- Mobile-responsive for smartphone access
- Performance charts use simplified views (no benchmark overlay unless requested)
- Document download links with expiring pre-signed URLs

**Testing**:
- `E2E: Client logs in and sees portfolio summary with correct total value`
- `E2E: Client can download a PDF statement`
- `E2E: Client cannot access other clients' accounts`
- `E2E: Client portal is responsive on mobile viewport`
- `Component: Financial goal progress bar renders correctly`

---

## Phase 12: AI-Native Features

### Purpose
Implement the AI-powered capabilities that differentiate this platform from incumbents: AI-generated suitability reports, financial plan drafting, and GIPS-compliant performance commentary generation. After this phase, the platform leverages LLMs to automate advisor workflows that incumbents handle poorly.

### Tasks

#### 12.1 — AI Suitability Report Generation

**What**: LLM-powered generation of Reg BI / MiFID II suitability documentation from client profile data.

**Design**:

```python
# src/wealthmgmt/ai/suitability.py
class AISuitabilityService:
    async def generate_suitability_report(
        self, client_id: uuid.UUID, db: AsyncSession
    ) -> str:
        """
        Generate a suitability report document using LLM.

        Inputs assembled from database:
        - Client suitability assessment (risk tolerance, objectives, time horizon)
        - Current portfolio positions and allocation
        - Model portfolio targets
        - Any compliance flags or recent changes

        Output: structured markdown report covering:
        1. Client profile summary
        2. Investment objective alignment
        3. Risk tolerance assessment
        4. Portfolio suitability analysis
        5. Recommendations and basis
        6. Conflict disclosures (Reg BI)
        """
        ...
```

System prompt template:
```
You are a compliance documentation assistant for a registered investment advisor.
Generate a Reg BI / MiFID II suitability report for the following client.
The report must document the basis for investment recommendations,
disclose any conflicts of interest, and confirm the recommendation
is in the client's best interest.

Use formal, regulatory-appropriate language.
Include specific data from the client profile and portfolio.
Do not provide investment advice — document the suitability analysis.
```

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| POST | `/api/v1/ai/suitability-report` | `compliance.write` | Generate suitability report |
| GET | `/api/v1/ai/suitability-report/{id}` | `compliance.read` | Get generated report |

Reports are stored as `document` records (type `suitability_report`) and linked to the client.

**Testing**:
- `Unit (mocked LLM): Report includes client name, risk tolerance, and investment objective`
- `Unit (mocked LLM): Report includes conflict disclosure section`
- `Unit (mocked LLM): Report references specific portfolio allocations`
- `Integration (mocked LLM): POST /ai/suitability-report generates and stores document`
- `Integration (mocked LLM): Generated report appears in client's document list`

#### 12.2 — AI Performance Commentary

**What**: LLM-generated GIPS-compliant performance commentary from portfolio return data.

**Design**:

```python
# src/wealthmgmt/ai/reporting.py
class AIReportingService:
    async def generate_performance_commentary(
        self, composite_id: uuid.UUID, period_end: date, db: AsyncSession
    ) -> str:
        """
        Generate narrative performance commentary for a GIPS composite.

        Inputs:
        - Composite returns (current period + trailing periods)
        - Benchmark returns
        - Market context (sector performance, economic indicators)
        - Attribution data (if available)

        Output: professional performance commentary suitable for
        client quarterly reports or GIPS composite presentations.
        """
        ...
```

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| POST | `/api/v1/ai/performance-commentary` | `portfolio.write` | Generate commentary |

**Testing**:
- `Unit (mocked LLM): Commentary references composite return and benchmark`
- `Unit (mocked LLM): Commentary uses GIPS-appropriate terminology`
- `Unit (mocked LLM): Commentary includes period identification (Q1 2026)`
- `Integration (mocked LLM): Generated commentary stored as document`

#### 12.3 — AI Financial Plan Drafting

**What**: LLM-assisted financial plan generation from client goals and scenario results.

**Design**:

```python
# src/wealthmgmt/ai/planning.py
class AIPlanningService:
    async def draft_financial_plan(
        self, client_id: uuid.UUID, scenario_id: uuid.UUID, db: AsyncSession
    ) -> str:
        """
        Generate a personalised financial plan narrative.

        Inputs:
        - Client profile (age, income, risk tolerance)
        - Financial goals (retirement, education, etc.)
        - Monte Carlo simulation results
        - Current portfolio and allocation

        Output: narrative financial plan covering:
        1. Client situation summary
        2. Goals and priorities
        3. Investment strategy recommendation
        4. Projected outcomes (referencing Monte Carlo)
        5. Action items and next steps
        """
        ...
```

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| POST | `/api/v1/ai/financial-plan` | `planning.write` | Draft financial plan |

**Testing**:
- `Unit (mocked LLM): Plan references Monte Carlo success probability`
- `Unit (mocked LLM): Plan includes specific goal amounts and dates`
- `Unit (mocked LLM): Plan does not provide specific security recommendations`
- `Integration (mocked LLM): Generated plan stored and linked to client`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Data Layer          ─── required by everything
    │
Phase 2: Authentication & Authorization   ─── requires Phase 1
    │
Phase 3: Client & Account Management      ─── requires Phase 2
    │
    ├── Phase 4: Portfolio Management      ─── requires Phase 3
    │       │
    │       ├── Phase 5: Trading & Compliance ─── requires Phase 4
    │       │       │
    │       │       └── Phase 6: Rebalancing  ─── requires Phase 5
    │       │
    │       ├── Phase 7: Performance Reporting ─── requires Phase 4
    │       │                                     (can parallel Phase 5-6)
    │       │
    │       └── Phase 8: Billing              ─── requires Phase 4
    │                                             (can parallel Phase 5-7)
    │
    ├── Phase 9: Custodian Integration     ─── requires Phase 4
    │                                          (can parallel Phase 5-8)
    │
    ├── Phase 10: Financial Planning       ─── requires Phase 3
    │                                          (can parallel Phase 4-9)
    │
    ├── Phase 11: Frontend                 ─── requires Phase 5 (trading UI)
    │                                          recommended after Phase 7-8
    │
    └── Phase 12: AI-Native Features       ─── requires Phase 5, 7, 10
                                               (can parallel Phase 11)
```

Parallelism opportunities:
- **Phases 5, 7, 8** can be developed concurrently after Phase 4 (they share the position/transaction data layer but have independent business logic)
- **Phase 9** (custodian integration) can be developed in parallel with Phases 5-8 once Phase 4 establishes the position data model
- **Phase 10** (financial planning) depends only on Phase 3 (clients) and can be developed in parallel with Phases 4-9
- **Phases 11 and 12** can be developed concurrently once their backend dependencies are complete

---

## Definition of Done (per phase)

1. All tasks implemented with production-quality code.
2. All unit tests pass (`pytest --tb=short`).
3. All integration tests pass against test database.
4. Ruff linting passes with zero errors (`ruff check .`).
5. Ruff formatting passes (`ruff format --check .`).
6. mypy strict mode passes (`mypy --strict src/`).
7. Docker build succeeds (`docker compose build`).
8. All services start cleanly (`docker compose up`).
9. Feature works end-to-end via API (manual or Playwright test).
10. New database schema changes have Alembic migrations (upgrade + downgrade).
11. New API endpoints appear in auto-generated OpenAPI spec at `/docs`.
12. New configuration options documented in `.env.example`.
13. Audit logging captures all data mutations for the new feature.
14. No hardcoded secrets or credentials in source code.
15. All financial calculations use `Decimal` types — no floating point.
