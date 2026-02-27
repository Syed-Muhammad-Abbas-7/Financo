🏗️ AI Financial Co-Pilot — Complete System Architecture & Design
Role: World-Class Solution Architect | Stack: Next.js · FastAPI · PostgreSQL · OpenAI Philosophy: Every design decision serves the owner, not the accountant.

🧠 Architecture Philosophy
Three rules govern every decision here:

Tenant-First — business_id is on every table. Multi-tenancy is a first-class citizen, not an afterthought.
Immutability for Finance — Transactions are never truly deleted. Audit trails exist on everything that matters.
No Accounting DNA — No charts of accounts, no debits/credits, no journal entries. The domain language is cashflow, entities, invoices, expenses.
📐 System Layer Map
┌──────────────────────────────────────────────────────────────┐
│  PRESENTATION LAYER  (Next.js + Tailwind + ShadCN)           │
│  Dashboard · Invoice UI · Chat · Reports · Mobile PWA        │
└────────────────────────┬─────────────────────────────────────┘
                         │ HTTPS / REST
┌────────────────────────▼─────────────────────────────────────┐
│  APPLICATION LAYER  (FastAPI · Python 3.12)                  │
│  Auth · Transactions · Invoices · Expenses · Forecasts       │
│  Middleware: JWT Validation · Business Context · Rate Limit  │
└──────────┬──────────────────────────────┬────────────────────┘
           │                              │
┌──────────▼──────────┐     ┌─────────────▼──────────────────┐
│  INTELLIGENCE LAYER  │     │  DATA LAYER                    │
│  AI Service Modules  │     │  PostgreSQL (primary)          │
│  Classification      │     │  Redis (cache + AI sessions)   │
│  Forecasting         │     │  File Storage (PDFs, receipts) │
│  Anomaly Detection   │     └────────────────────────────────┘
│  Conversational AI   │
└─────────────────────┘
🗄️ DATABASE SCHEMA
Engine: PostgreSQL 15+ · UUIDs everywhere · JSONB for flexibility · Soft deletes via deleted_at

👤 users
CopyCREATE TABLE users (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email           TEXT UNIQUE NOT NULL,
  password_hash   TEXT NOT NULL,
  full_name       TEXT NOT NULL,
  avatar_url      TEXT,
  is_active       BOOLEAN DEFAULT TRUE,
  last_login_at   TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);
🏢 businesses
The tenant root. Every other table hangs off business_id.

CopyCREATE TABLE businesses (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id    UUID NOT NULL REFERENCES users(id),
  name        TEXT NOT NULL,
  slug        TEXT UNIQUE NOT NULL,         -- url-safe identifier
  currency    CHAR(3) DEFAULT 'USD',        -- ISO 4217
  timezone    TEXT DEFAULT 'UTC',
  logo_url    TEXT,
  industry    TEXT,
  settings    JSONB DEFAULT '{}',           -- invoice branding, tax rates, alert thresholds
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ DEFAULT NOW(),
  deleted_at  TIMESTAMPTZ                   -- soft delete
);
settings JSONB shape:

Copy{
  "invoice_prefix": "INV",
  "default_payment_terms_days": 30,
  "tax_rate_default": 0.10,
  "liquidity_alert_threshold": 5000,
  "invoice_branding": { "primary_color": "#1A56DB", "footer_text": "Thank you!" },
  "enable_multi_currency": false
}
🔐 user_business_roles
Enables shared access — accountants, co-founders, employees.

CopyCREATE TABLE user_business_roles (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  business_id  UUID NOT NULL REFERENCES businesses(id) ON DELETE CASCADE,
  role         TEXT NOT NULL CHECK (role IN ('owner', 'editor', 'viewer')),
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, business_id)
);
🏦 accounts
Bank accounts, cash wallets, credit lines — all unified here.

CopyCREATE TABLE accounts (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id       UUID NOT NULL REFERENCES businesses(id),
  name              TEXT NOT NULL,
  type              TEXT NOT NULL CHECK (type IN ('bank','cash','credit','wallet')),
  currency          CHAR(3) DEFAULT 'USD',
  current_balance   NUMERIC(18,2) DEFAULT 0,
  available_balance NUMERIC(18,2) DEFAULT 0,  -- excludes pending
  is_active         BOOLEAN DEFAULT TRUE,
  last_synced_at    TIMESTAMPTZ,
  metadata          JSONB DEFAULT '{}',        -- bank connection info
  created_at        TIMESTAMPTZ DEFAULT NOW(),
  updated_at        TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_accounts_business ON accounts(business_id);
💳 transactions
The heartbeat of the system. Central to every feature.

CopyCREATE TABLE transactions (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id       UUID NOT NULL REFERENCES businesses(id),
  account_id        UUID NOT NULL REFERENCES accounts(id),
  entity_id         UUID REFERENCES entities(id),          -- auto or manual mapped
  category_id       UUID REFERENCES categories(id),
  invoice_id        UUID REFERENCES invoices(id),
  expense_id        UUID REFERENCES expenses(id),
  recurring_rule_id UUID REFERENCES recurring_rules(id),

  amount            NUMERIC(18,2) NOT NULL,                 -- always positive
  direction         TEXT NOT NULL CHECK (direction IN ('inflow','outflow')),
  type              TEXT NOT NULL CHECK (type IN (
                      'income','expense','transfer',
                      'owner_withdrawal','tax','financing'
                    )),
  description       TEXT,
  reference_number  TEXT,
  transaction_date  DATE NOT NULL,
  value_date        DATE,
  status            TEXT DEFAULT 'pending' CHECK (status IN ('pending','cleared','reconciled')),

  -- AI fields
  is_anomaly        BOOLEAN DEFAULT FALSE,
  anomaly_score     FLOAT DEFAULT 0,            -- 0.0 to 1.0
  ai_confidence     FLOAT DEFAULT 0,            -- classification confidence
  ai_label          TEXT,                        -- AI suggested category label

  tags              TEXT[] DEFAULT '{}',
  metadata          JSONB DEFAULT '{}',          -- raw bank import data
  version           INT DEFAULT 1,               -- increments on edit (for audit)

  created_at        TIMESTAMPTZ DEFAULT NOW(),
  updated_at        TIMESTAMPTZ DEFAULT NOW(),
  deleted_at        TIMESTAMPTZ                  -- soft delete only
);

CREATE INDEX idx_txn_business_date  ON transactions(business_id, transaction_date DESC);
CREATE INDEX idx_txn_entity         ON transactions(entity_id);
CREATE INDEX idx_txn_category       ON transactions(category_id);
CREATE INDEX idx_txn_status         ON transactions(business_id, status);
CREATE INDEX idx_txn_anomaly        ON transactions(business_id, is_anomaly) WHERE is_anomaly = TRUE;
Copy
📋 transaction_audit_log
Immutable. No updates, no deletes. Every mutation is recorded.

CopyCREATE TABLE transaction_audit_log (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_id UUID NOT NULL REFERENCES transactions(id),
  changed_by     UUID REFERENCES users(id),
  change_type    TEXT NOT NULL CHECK (change_type IN ('create','update','delete')),
  old_values     JSONB,
  new_values     JSONB,
  ip_address     INET,
  changed_at     TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audit_txn ON transaction_audit_log(transaction_id);
🤝 entities
Customers AND suppliers in one table. Clean, simple, powerful.

CopyCREATE TABLE entities (
  id                     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id            UUID NOT NULL REFERENCES businesses(id),
  type                   TEXT NOT NULL CHECK (type IN ('customer','supplier','both')),
  name                   TEXT NOT NULL,
  email                  TEXT,
  phone                  TEXT,
  address                JSONB DEFAULT '{}',    -- {street, city, country, postal}
  tax_id                 TEXT,
  payment_terms_days     INT DEFAULT 30,

  -- Computed/cached scores (updated by background job)
  reliability_score      FLOAT DEFAULT 100,     -- 0-100
  avg_payment_delay_days FLOAT DEFAULT 0,
  total_revenue          NUMERIC(18,2) DEFAULT 0,
  total_spend            NUMERIC(18,2) DEFAULT 0,
  payment_count          INT DEFAULT 0,
  late_payment_count     INT DEFAULT 0,

  tags                   TEXT[] DEFAULT '{}',
  notes                  TEXT,
  created_at             TIMESTAMPTZ DEFAULT NOW(),
  updated_at             TIMESTAMPTZ DEFAULT NOW(),
  deleted_at             TIMESTAMPTZ
);

CREATE INDEX idx_entity_business      ON entities(business_id);
CREATE INDEX idx_entity_type          ON entities(business_id, type);
CREATE INDEX idx_entity_reliability   ON entities(business_id, reliability_score);
🗂️ categories
System defaults + user-defined. Hierarchical (parent/child).

CopyCREATE TABLE categories (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID REFERENCES businesses(id),  -- NULL = system default
  name        TEXT NOT NULL,
  type        TEXT NOT NULL CHECK (type IN ('income','expense','transfer')),
  icon        TEXT,
  color       CHAR(7),                          -- hex color
  parent_id   UUID REFERENCES categories(id),  -- subcategories
  is_system   BOOLEAN DEFAULT FALSE,
  sort_order  INT DEFAULT 0,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);
Default system categories seeded on startup:

Income: Sales Revenue, Service Income, Refunds, Financing, Other Income
Expense: Rent, Salaries, Subscriptions, Marketing, Utilities, Tax, Supplies, Loan Repayment
📦 products_services
Invoice line items are built from here.

CopyCREATE TABLE products_services (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID NOT NULL REFERENCES businesses(id),
  name        TEXT NOT NULL,
  description TEXT,
  type        TEXT NOT NULL CHECK (type IN ('product','service')),
  unit_price  NUMERIC(18,2) NOT NULL DEFAULT 0,
  unit        TEXT DEFAULT 'unit',              -- 'hour', 'unit', 'month', 'kg'
  tax_rate    FLOAT DEFAULT 0,
  category_id UUID REFERENCES categories(id),
  is_active   BOOLEAN DEFAULT TRUE,
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ DEFAULT NOW()
);
🧾 invoices
CopyCREATE TABLE invoices (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id       UUID NOT NULL REFERENCES businesses(id),
  invoice_number    TEXT NOT NULL,
  entity_id         UUID NOT NULL REFERENCES entities(id),  -- customer
  recurring_rule_id UUID REFERENCES recurring_rules(id),

  status            TEXT DEFAULT 'draft' CHECK (status IN (
                      'draft','sent','viewed','partial','paid','overdue','cancelled'
                    )),
  issue_date        DATE NOT NULL DEFAULT CURRENT_DATE,
  due_date          DATE NOT NULL,

  subtotal          NUMERIC(18,2) NOT NULL DEFAULT 0,
  tax_amount        NUMERIC(18,2) DEFAULT 0,
  discount_amount   NUMERIC(18,2) DEFAULT 0,
  total_amount      NUMERIC(18,2) NOT NULL,
  paid_amount       NUMERIC(18,2) DEFAULT 0,

  currency          CHAR(3) DEFAULT 'USD',
  notes             TEXT,
  payment_terms     TEXT,
  template_config   JSONB DEFAULT '{}',          -- logo, colors, footer
  pdf_url           TEXT,

  last_sent_at      TIMESTAMPTZ,
  last_viewed_at    TIMESTAMPTZ,
  created_at        TIMESTAMPTZ DEFAULT NOW(),
  updated_at        TIMESTAMPTZ DEFAULT NOW(),
  deleted_at        TIMESTAMPTZ,

  UNIQUE(business_id, invoice_number)
);

CREATE INDEX idx_inv_business_status ON invoices(business_id, status);
CREATE INDEX idx_inv_due_date        ON invoices(business_id, due_date);
CREATE INDEX idx_inv_entity          ON invoices(entity_id);
Copy
📑 invoice_items
CopyCREATE TABLE invoice_items (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invoice_id  UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
  product_id  UUID REFERENCES products_services(id),
  description TEXT NOT NULL,
  quantity    NUMERIC(10,3) NOT NULL DEFAULT 1,
  unit_price  NUMERIC(18,2) NOT NULL,
  tax_rate    FLOAT DEFAULT 0,
  discount    FLOAT DEFAULT 0,               -- percentage 0-1
  line_total  NUMERIC(18,2) NOT NULL,        -- computed: qty * price * (1-discount) * (1+tax)
  sort_order  INT DEFAULT 0,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);
💰 invoice_payments
Tracks partial and full payments against an invoice.

CopyCREATE TABLE invoice_payments (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invoice_id     UUID NOT NULL REFERENCES invoices(id),
  transaction_id UUID REFERENCES transactions(id),
  amount         NUMERIC(18,2) NOT NULL,
  payment_date   DATE NOT NULL,
  payment_method TEXT,                       -- 'bank_transfer', 'cash', 'card', 'crypto'
  reference      TEXT,
  notes          TEXT,
  created_at     TIMESTAMPTZ DEFAULT NOW()
);
💸 expenses
CopyCREATE TABLE expenses (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id       UUID NOT NULL REFERENCES businesses(id),
  entity_id         UUID REFERENCES entities(id),     -- supplier
  category_id       UUID REFERENCES categories(id),
  transaction_id    UUID REFERENCES transactions(id),
  recurring_rule_id UUID REFERENCES recurring_rules(id),

  amount            NUMERIC(18,2) NOT NULL,
  currency          CHAR(3) DEFAULT 'USD',
  description       TEXT NOT NULL,
  expense_date      DATE NOT NULL,
  due_date          DATE,
  status            TEXT DEFAULT 'pending' CHECK (status IN (
                      'pending','scheduled','paid','overdue'
                    )),
  receipt_url       TEXT,
  tax_deductible    BOOLEAN DEFAULT FALSE,
  tags              TEXT[] DEFAULT '{}',
  notes             TEXT,

  created_at        TIMESTAMPTZ DEFAULT NOW(),
  updated_at        TIMESTAMPTZ DEFAULT NOW(),
  deleted_at        TIMESTAMPTZ
);

CREATE INDEX idx_exp_business_date   ON expenses(business_id, expense_date DESC);
CREATE INDEX idx_exp_status          ON expenses(business_id, status);
CREATE INDEX idx_exp_entity          ON expenses(entity_id);
🔁 recurring_rules
One table drives all recurring patterns — invoices, expenses, transactions.

CopyCREATE TABLE recurring_rules (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id         UUID NOT NULL REFERENCES businesses(id),
  name                TEXT NOT NULL,
  rule_type           TEXT NOT NULL CHECK (rule_type IN ('invoice','expense','transaction')),
  frequency           TEXT NOT NULL CHECK (frequency IN (
                        'daily','weekly','biweekly','monthly','quarterly','yearly','custom'
                      )),
  interval_days       INT,                    -- used when frequency = 'custom'
  start_date          DATE NOT NULL,
  end_date            DATE,
  next_execution_date DATE,
  last_execution_date DATE,
  config              JSONB NOT NULL,         -- template: entity_id, amount, items, category, etc.
  is_active           BOOLEAN DEFAULT TRUE,
  execution_count     INT DEFAULT 0,
  created_at          TIMESTAMPTZ DEFAULT NOW(),
  updated_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_recur_next_exec ON recurring_rules(next_execution_date) WHERE is_active = TRUE;
📊 budgets
CopyCREATE TABLE budgets (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id       UUID NOT NULL REFERENCES businesses(id),
  name              TEXT NOT NULL,
  category_id       UUID REFERENCES categories(id),
  entity_id         UUID REFERENCES entities(id),
  amount            NUMERIC(18,2) NOT NULL,
  period            TEXT NOT NULL CHECK (period IN ('weekly','monthly','quarterly','yearly')),
  start_date        DATE NOT NULL,
  end_date          DATE,
  alert_threshold   FLOAT DEFAULT 0.80,       -- alert at 80% of budget consumed
  created_at        TIMESTAMPTZ DEFAULT NOW(),
  updated_at        TIMESTAMPTZ DEFAULT NOW()
);
🔮 forecasts
Snapshot of a forecast run — never mutated after creation.

CopyCREATE TABLE forecasts (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id           UUID NOT NULL REFERENCES businesses(id),
  scenario_id           UUID REFERENCES scenarios(id),
  model_type            TEXT DEFAULT 'baseline' CHECK (model_type IN (
                          'baseline','probabilistic','scenario'
                        )),
  forecast_period_start DATE NOT NULL,
  forecast_period_end   DATE NOT NULL,
  confidence_score      FLOAT,               -- model confidence 0-1
  data                  JSONB NOT NULL,       -- see structure below
  parameters            JSONB DEFAULT '{}',  -- inputs used: horizon, weights, etc.
  generated_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_forecast_business ON forecasts(business_id, generated_at DESC);
data JSONB shape:

Copy{
  "summary": {
    "opening_balance": 12400,
    "closing_expected": 8100,
    "closing_best": 11500,
    "closing_worst": 4200,
    "deficit_risk_date": "2025-03-15",
    "deficit_probability": 0.32
  },
  "daily": [
    {
      "date": "2025-02-28",
      "inflow_expected": 2000,
      "outflow_expected": 1400,
      "net_expected": 600,
      "balance_expected": 13000,
      "balance_best": 14200,
      "balance_worst": 11800
    }
  ]
}
🎭 scenarios
User-defined what-if simulations.

CopyCREATE TABLE scenarios (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id     UUID NOT NULL REFERENCES businesses(id),
  name            TEXT NOT NULL,
  description     TEXT,
  type            TEXT DEFAULT 'custom' CHECK (type IN (
                    'expense_delay','revenue_acceleration',
                    'cost_cut','revenue_loss','custom'
                  )),
  parameters      JSONB NOT NULL,            -- see shape below
  result_snapshot JSONB,                     -- last simulation result
  is_archived     BOOLEAN DEFAULT FALSE,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);
parameters JSONB shape:

Copy{
  "adjustments": [
    { "type": "delay_expense", "entity_id": "uuid", "days": 14 },
    { "type": "accelerate_invoice", "invoice_id": "uuid", "days": -7 },
    { "type": "add_one_time_expense", "amount": 5000, "date": "2025-03-01" }
  ]
}
💡 ai_insights
Proactive intelligence surface — system-generated, not user-created.

CopyCREATE TABLE ai_insights (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id         UUID NOT NULL REFERENCES businesses(id),
  type                TEXT NOT NULL CHECK (type IN (
                        'anomaly','trend','recommendation',
                        'risk_alert','performance','opportunity'
                      )),
  priority            TEXT DEFAULT 'medium' CHECK (priority IN ('low','medium','high','critical')),
  title               TEXT NOT NULL,
  summary             TEXT NOT NULL,         -- 1-2 sentence plain English
  detail              TEXT,                  -- full explanation
  related_entity_type TEXT,                  -- 'transaction','invoice','expense','entity','account'
  related_entity_id   UUID,
  data_payload        JSONB DEFAULT '{}',    -- supporting data for the insight
  is_read             BOOLEAN DEFAULT FALSE,
  is_dismissed        BOOLEAN DEFAULT FALSE,
  expires_at          TIMESTAMPTZ,
  created_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_insights_business    ON ai_insights(business_id, created_at DESC);
CREATE INDEX idx_insights_unread      ON ai_insights(business_id, is_read) WHERE is_read = FALSE;
CREATE INDEX idx_insights_priority    ON ai_insights(business_id, priority);
🚨 alerts
Operational triggers — liquidity drops, overdue invoices, budget overruns.

CopyCREATE TABLE alerts (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id         UUID NOT NULL REFERENCES businesses(id),
  type                TEXT NOT NULL CHECK (type IN (
                        'liquidity_risk','overdue_invoice','budget_exceeded',
                        'anomaly_detected','forecast_deficit','payment_due'
                      )),
  severity            TEXT DEFAULT 'warning' CHECK (severity IN ('info','warning','critical')),
  title               TEXT NOT NULL,
  message             TEXT NOT NULL,
  related_entity_type TEXT,
  related_entity_id   UUID,
  action_url          TEXT,                  -- deep link to relevant screen
  is_read             BOOLEAN DEFAULT FALSE,
  is_dismissed        BOOLEAN DEFAULT FALSE,
  triggered_at        TIMESTAMPTZ DEFAULT NOW(),
  created_at          TIMESTAMPTZ DEFAULT NOW()
);
💬 chat_sessions + chat_messages
CopyCREATE TABLE chat_sessions (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID NOT NULL REFERENCES businesses(id),
  user_id     UUID NOT NULL REFERENCES users(id),
  title       TEXT,                          -- AI-generated title after first exchange
  summary     TEXT,                          -- session summary for memory
  topic_tags  TEXT[] DEFAULT '{}',
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE chat_messages (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id       UUID NOT NULL REFERENCES chat_sessions(id) ON DELETE CASCADE,
  role             TEXT NOT NULL CHECK (role IN ('user','assistant','system')),
  content          TEXT NOT NULL,
  data_references  JSONB DEFAULT '{}',       -- {transactions: [], invoices: [], metrics: {}}
  reasoning_chain  JSONB DEFAULT '[]',       -- [{step: "check_balance", result: 12400}, ...]
  confidence_score FLOAT,
  tokens_used      INT,
  created_at       TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_chat_sessions_business ON chat_sessions(business_id, updated_at DESC);
CREATE INDEX idx_chat_messages_session  ON chat_messages(session_id, created_at ASC);
🧠 ai_memory
Long-term business memory for the Co-Pilot.

CopyCREATE TABLE ai_memory (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id      UUID NOT NULL REFERENCES businesses(id),
  memory_type      TEXT NOT NULL CHECK (memory_type IN (
                     'preference','fact','pattern','context'
                   )),
  key              TEXT NOT NULL,            -- e.g., 'preferred_report_style', 'risk_tolerance'
  value            JSONB NOT NULL,           -- flexible memory value
  importance_score FLOAT DEFAULT 0.5,        -- 0-1, higher = recalled more often
  last_accessed_at TIMESTAMPTZ,
  created_at       TIMESTAMPTZ DEFAULT NOW(),
  updated_at       TIMESTAMPTZ DEFAULT NOW(),

  UNIQUE(business_id, memory_type, key)
);
📈 stability_scores
Historical record of business health — computed nightly.

CopyCREATE TABLE stability_scores (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID NOT NULL REFERENCES businesses(id),
  score       FLOAT NOT NULL,               -- 0-100
  components  JSONB NOT NULL,               -- breakdown
  computed_at TIMESTAMPTZ DEFAULT NOW()
);
components JSONB shape:

Copy{
  "cash_buffer_ratio": { "score": 72, "value": 0.45, "label": "Good" },
  "revenue_concentration": { "score": 55, "value": 0.68, "label": "Moderate Risk" },
  "expense_predictability": { "score": 88, "value": 0.91, "label": "Excellent" },
  "cashflow_volatility": { "score": 61, "value": 0.38, "label": "Moderate" }
}
📥 import_jobs
Tracks CSV/bank-import processing.

CopyCREATE TABLE import_jobs (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id     UUID NOT NULL REFERENCES businesses(id),
  account_id      UUID NOT NULL REFERENCES accounts(id),
  source_type     TEXT CHECK (source_type IN ('csv','bank_api','manual')),
  status          TEXT DEFAULT 'pending' CHECK (status IN (
                    'pending','processing','completed','failed'
                  )),
  row_count       INT DEFAULT 0,
  processed_count INT DEFAULT 0,
  error_log       JSONB DEFAULT '[]',
  file_url        TEXT,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);
🔌 API SPECIFICATION
Base URL: https://api.copilot.app/api/v1 All endpoints require: Authorization: Bearer <jwt_token> + X-Business-ID: <uuid> (middleware injects business context)

🔐 AUTH — /auth
Method	Path	Description
POST	/auth/register	Create user account
POST	/auth/login	Returns access_token + refresh_token
POST	/auth/refresh	Rotate access token
POST	/auth/logout	Invalidate refresh token
GET	/auth/me	Current user profile
PUT	/auth/me	Update profile
POST	/auth/change-password	Change password
POST /auth/login Response:

Copy{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "expires_in": 900,
  "user": {
    "id": "uuid",
    "email": "owner@biz.com",
    "full_name": "Alex Reyes",
    "businesses": [
      { "id": "uuid", "name": "Studio Flow", "role": "owner" }
    ]
  }
}
🏢 BUSINESSES — /businesses
Method	Path	Description
GET	/businesses	List user's businesses
POST	/businesses	Create new business
GET	/businesses/:id	Business detail
PUT	/businesses/:id	Update business
GET	/businesses/:id/settings	Get settings JSONB
PUT	/businesses/:id/settings	Patch settings
GET	/businesses/:id/stability-score	Latest stability score + history
GET	/businesses/:id/dashboard	Master dashboard aggregation endpoint
GET /businesses/:id/dashboard Response:

Copy{
  "cash_position": {
    "available": 12400,
    "committed": 4800,
    "incoming_expected": 6200,
    "net_free": 7600
  },
  "period_summary": {
    "total_inflow": 18500,
    "total_outflow": 11200,
    "net_cashflow": 7300
  },
  "alerts_count": { "critical": 1, "warning": 3 },
  "overdue_invoices": { "count": 2, "total": 3400 },
  "upcoming_expenses": { "next_7_days": 2100 },
  "stability_score": 74,
  "top_insights": [...]
}
🏦 ACCOUNTS — /accounts
Method	Path	Description
GET	/accounts	List all accounts
POST	/accounts	Create account
GET	/accounts/:id	Account detail
PUT	/accounts/:id	Update account
DELETE	/accounts/:id	Soft delete
POST	/accounts/:id/sync	Trigger balance refresh
GET	/accounts/:id/balance-history	Daily balance series for charts
💳 TRANSACTIONS — /transactions
Method	Path	Description
GET	/transactions	List with filters
POST	/transactions	Create manual transaction
GET	/transactions/:id	Detail
PUT	/transactions/:id	Update (triggers audit log)
DELETE	/transactions/:id	Soft delete (audit logged)
POST	/transactions/import	Batch import (CSV / bank data)
GET	/transactions/:id/audit-log	Full change history
POST	/transactions/:id/classify	Re-trigger AI classification
GET	/transactions/summary	Aggregated stats
GET	/transactions/cashflow-timeline	Daily net cashflow series
GET /transactions Query Params:

?direction=inflow|outflow
&type=income|expense|transfer
&status=pending|cleared|reconciled
&entity_id=uuid
&category_id=uuid
&date_from=2025-01-01
&date_to=2025-03-31
&is_anomaly=true
&search=invoice+description
&page=1&limit=50
&sort=transaction_date:desc
GET /transactions/cashflow-timeline Response:

Copy{
  "period": "2025-01-01 to 2025-03-31",
  "granularity": "daily",
  "series": [
    {
      "date": "2025-02-01",
      "inflow": 3200,
      "outflow": 1800,
      "net": 1400,
      "balance": 13800
    }
  ],
  "totals": { "inflow": 28000, "outflow": 17400, "net": 10600 }
}
🧾 INVOICES — /invoices
Method	Path	Description
GET	/invoices	List with filters
POST	/invoices	Create invoice
GET	/invoices/:id	Invoice detail with items
PUT	/invoices/:id	Update (only if draft)
DELETE	/invoices/:id	Soft delete
POST	/invoices/:id/send	Mark as sent + trigger email
POST	/invoices/:id/record-payment	Record full or partial payment
POST	/invoices/:id/generate-pdf	Generate and store PDF
GET	/invoices/:id/payments	Payment history
POST	/invoices/auto-match	Match bank transactions to open invoices
GET	/invoices/performance	Collection metrics dashboard
POST /invoices Request:

Copy{
  "entity_id": "uuid",
  "issue_date": "2025-02-27",
  "due_date": "2025-03-27",
  "currency": "USD",
  "notes": "Project Alpha — Phase 2",
  "items": [
    {
      "product_id": "uuid",
      "description": "UI Design — 40hrs",
      "quantity": 40,
      "unit_price": 85,
      "tax_rate": 0.10,
      "discount": 0
    }
  ]
}
GET /invoices/performance Response:

Copy{
  "total_outstanding": 14200,
  "overdue_count": 3,
  "overdue_amount": 4800,
  "collection_rate_30d": 0.84,
  "avg_payment_cycle_days": 22,
  "aging": {
    "0_30": 6400,
    "31_60": 3000,
    "61_90": 1800,
    "90_plus": 3000
  }
}
💸 EXPENSES — /expenses
Method	Path	Description
GET	/expenses	List with filters
POST	/expenses	Create expense
GET	/expenses/:id	Expense detail
PUT	/expenses/:id	Update
DELETE	/expenses/:id	Soft delete
POST	/expenses/:id/mark-paid	Mark as paid + optionally link transaction
GET	/expenses/summary	Category breakdown + trends
🤝 ENTITIES — /entities
Method	Path	Description
GET	/entities	List (filter by type)
POST	/entities	Create entity
GET	/entities/:id	Entity detail
PUT	/entities/:id	Update
DELETE	/entities/:id	Soft delete
GET	/entities/:id/transactions	All transactions for this entity
GET	/entities/:id/score	Reliability score + history
GET	/entities/:id/profile	Full 360° entity profile
GET /entities/:id/profile Response:

Copy{
  "entity": { "id": "uuid", "name": "Acme Corp", "type": "customer" },
  "reliability_score": 82,
  "avg_payment_delay_days": 7,
  "total_revenue": 48000,
  "open_invoices": 3,
  "open_amount": 6400,
  "last_transaction_date": "2025-02-15",
  "payment_history": [
    { "month": "2025-01", "paid": 12000, "avg_delay": 5 }
  ],
  "insights": [
    { "type": "trend", "message": "Payment speed improved 40% over last 90 days" }
  ]
}
🔮 FORECASTS — /forecasts
Method	Path	Description
GET	/forecasts/current	Latest active forecast
POST	/forecasts/generate	Trigger new forecast run
GET	/forecasts/:id	Historical forecast detail
GET	/forecasts/history	List past forecasts
POST /forecasts/generate Request:

Copy{
  "horizon_days": 60,
  "model_type": "probabilistic",
  "include_scenarios": ["uuid-scenario-1"]
}
🎭 SCENARIOS — /scenarios
Method	Path	Description
GET	/scenarios	List saved scenarios
POST	/scenarios	Create scenario
GET	/scenarios/:id	Detail
PUT	/scenarios/:id	Update
DELETE	/scenarios/:id	Delete
POST	/scenarios/:id/simulate	Run simulation, get result
POST	/scenarios/compare	Compare multiple scenarios side-by-side
POST /scenarios/compare Request:

Copy{ "scenario_ids": ["uuid-1", "uuid-2", "uuid-3"] }
Response:

Copy{
  "baseline_balance_30d": 8100,
  "comparisons": [
    { "id": "uuid-1", "name": "Delay rent 2 weeks", "balance_30d": 9600, "delta": +1500 },
    { "id": "uuid-2", "name": "Collect Invoice A early", "balance_30d": 10800, "delta": +2700 }
  ]
}
📊 BUDGETS — /budgets
Method	Path	Description
GET	/budgets	List
POST	/budgets	Create
GET	/budgets/:id	Detail
PUT	/budgets/:id	Update
DELETE	/budgets/:id	Delete
GET	/budgets/:id/performance	Spent vs budgeted
🔁 RECURRING RULES — /recurring-rules
Method	Path	Description
GET	/recurring-rules	List all rules
POST	/recurring-rules	Create rule
GET	/recurring-rules/:id	Detail
PUT	/recurring-rules/:id	Update
DELETE	/recurring-rules/:id	Delete
POST	/recurring-rules/:id/pause	Pause execution
POST	/recurring-rules/:id/resume	Resume
💡 INSIGHTS — /insights
Method	Path	Description
GET	/insights	List (filter by type/priority)
GET	/insights/:id	Detail
POST	/insights/:id/dismiss	Dismiss insight
GET	/insights/summary	Insight counts by priority
🚨 ALERTS — /alerts
Method	Path	Description
GET	/alerts	List active alerts
POST	/alerts/:id/read	Mark read
POST	/alerts/:id/dismiss	Dismiss
GET	/alerts/unread-count	Badge count for UI
🤖 AI CO-PILOT — /ai
Method	Path	Description
GET	/ai/sessions	List chat sessions
POST	/ai/sessions	Start new session
GET	/ai/sessions/:id	Session detail
GET	/ai/sessions/:id/messages	All messages in session
POST	/ai/sessions/:id/messages	Send message (streaming SSE)
DELETE	/ai/sessions/:id	Delete session
GET	/ai/memory	View AI business memory
DELETE	/ai/memory/:key	Remove a memory entry
POST	/ai/analyze	Ad-hoc analysis request
POST /ai/sessions/:id/messages Request:

Copy{
  "content": "What will my cash look like in 30 days if my top 2 clients pay late?",
  "context_hints": {
    "include_forecasts": true,
    "include_invoices": true
  }
}
Response (SSE stream):

data: {"type": "reasoning", "step": "Fetching open invoices for top 2 clients..."}
data: {"type": "reasoning", "step": "Loading 30-day forecast baseline..."}
data: {"type": "content", "text": "Based on your current forecast, if Acme Corp and Studio Nine..."}
data: {"type": "data_card", "payload": {"type": "forecast_chart", "data": {...}}}
data: {"type": "done", "message_id": "uuid", "tokens_used": 420}
📋 REPORTS — /reports
Method	Path	Description
GET	/reports/cashflow	Cashflow statement (P&L lite)
GET	/reports/receivables	Receivables aging report
GET	/reports/payables	Payables summary
GET	/reports/entity-performance	Revenue/expense by entity
POST	/reports/export	Export to PDF or Excel
🧠 AI SERVICE MODULE DESIGN
ai/
├── classifier.py         # Transaction type & category classification
├── anomaly_detector.py   # Isolation Forest + rule-based spike detection
├── forecaster.py         # Time-series projection engine
├── scenario_engine.py    # What-if simulation logic
├── entity_scorer.py      # Customer/supplier reliability score computation
├── stability_scorer.py   # Business health index computation
├── chat_agent.py         # OpenAI orchestration + context injection
├── memory_manager.py     # Long-term memory read/write
├── reasoning_builder.py  # Constructs deterministic reasoning chains
└── insight_generator.py  # Scheduled insight surface generation
Classifier flow:

Raw Transaction Description
         │
         ▼
  Rule-Based Match (vendor dictionary)
         │ no match
         ▼
  OpenAI: classify into {type, category, entity}
  with business context injected
         │
         ▼
  Confidence score stored in metadata JSONB
  User correction → stored as training signal
Chat Agent context injection:

Copysystem_prompt = f"""
You are the AI Financial Co-Pilot for {business.name}.
Current date: {today}
Cash position: ${available_cash} available, ${committed_cash} committed
Open invoices: {open_invoice_count} totaling ${open_invoice_total}
Top expense categories this month: {top_categories}
Recent anomalies: {recent_anomalies}
Business memory: {memory_summary}
Respond with deterministic, plain-language financial guidance.
Always cite the data source you used. Never say 'I think' — say 'Based on your data...'
"""
📁 PROJECT FOLDER STRUCTURE
copilot/
├── frontend/                   # Next.js App
│   ├── app/
│   │   ├── (auth)/             # login, register
│   │   ├── (dashboard)/        # main app shell
│   │   │   ├── page.tsx        # master dashboard
│   │   │   ├── cashflow/
│   │   │   ├── invoices/
│   │   │   ├── expenses/
│   │   │   ├── entities/
│   │   │   ├── forecasts/
│   │   │   ├── scenarios/
│   │   │   ├── reports/
│   │   │   └── ai-chat/
│   ├── components/
│   │   ├── ui/                 # ShadCN base
│   │   ├── charts/             # Recharts wrappers
│   │   ├── cards/              # CashCard, InvoiceCard, AlertCard
│   │   ├── forms/              # InvoiceForm, ExpenseForm
│   │   └── ai/                 # ChatWindow, MessageBubble, ReasoningCard
│   ├── hooks/                  # useBusinessContext, useAlerts, useChat
│   ├── lib/
│   │   ├── api.ts              # Axios client with business_id injection
│   │   └── auth.ts
│   └── store/                  # Zustand stores
│
└── backend/                    # FastAPI
    ├── app/
    │   ├── main.py
    │   ├── config.py
    │   ├── middleware/
    │   │   ├── auth.py         # JWT validation
    │   │   └── business.py     # X-Business-ID → request context
    │   ├── routers/
    │   │   ├── auth.py
    │   │   ├── businesses.py
    │   │   ├── accounts.py
    │   │   ├── transactions.py
    │   │   ├── invoices.py
    │   │   ├── expenses.py
    │   │   ├── entities.py
    │   │   ├── forecasts.py
    │   │   ├── scenarios.py
    │   │   ├── budgets.py
    │   │   ├── insights.py
    │   │   ├── alerts.py
    │   │   ├── reports.py
    │   │   └── ai.py
    │   ├── services/            # Business logic
    │   │   ├── cashflow.py
    │   │   ├── invoice_service.py
    │   │   ├── matching_service.py
    │   │   └── recurring_service.py
    │   ├── ai/                  # AI modules (see above)
    │   ├── schemas/             # Pydantic models
    │   ├── models/              # SQLAlchemy ORM
    │   ├── tasks/               # Background jobs (APScheduler)
    │   │   ├── forecast_runner.py
    │   │   ├── recurring_executor.py
    │   │   ├── score_updater.py
    │   │   └── insight_generator.py
    │   └── db/
    │       ├── session.py
    │       └── migrations/     # Alembic
⚡ BACKGROUND JOBS (APScheduler)
Job	Schedule	Function
recurring_executor	Every hour	Check recurring_rules.next_execution_date, spawn invoices/expenses
forecast_runner	Nightly 2AM	Regenerate probabilistic forecast for all active businesses
score_updater	Nightly 3AM	Recompute entity reliability scores + business stability score
insight_generator	Nightly 4AM	Detect anomalies, trends, risks → write to ai_insights
alert_checker	Every 15 min	Check overdue invoices, budget thresholds, liquidity → write to alerts
overdue_reminders	Daily 9AM	Auto-send payment reminders for overdue invoices
🔑 KEY ARCHITECTURAL DECISIONS
1. Unified entities table for customers AND suppliers Rather than two tables that share 90% of schema, a single table with type = customer | supplier | both keeps joins simple and enables entities that are both (e.g., a freelancer you hire who also buys from you).

2. transactions as the single source of truth Invoices and expenses are management constructs. The actual money movement is always a transaction. This prevents double-counting and keeps the cash position calculation pure: SELECT SUM(CASE WHEN direction='inflow' THEN amount ELSE -amount END).

3. JSONB for AI metadata, NOT separate tables AI confidence scores, reasoning chains, and classification results live in JSONB columns inside existing tables. No join overhead, easy to extend, and the AI layer can enrich data without migrations.

4. Forecasts are snapshots, not live calculations forecasts rows are immutable records of a forecast run at a point in time. The UI always reads the latest, but you can replay history. This enables "compare forecast vs actual" without any state management complexity.

5. X-Business-ID header pattern Every API request carries the active business context in a header. Backend middleware validates ownership and injects into request.state.business_id. This eliminates a whole class of cross-tenant bugs — no route needs to handle multi-business logic manually.

6. SSE for AI chat responses The AI Co-Pilot streams responses via Server-Sent Events rather than waiting for a full response. Users see reasoning steps in real time. This makes the AI feel alive, not laggy.

7. Stability Score as a first-class entity Rather than computing on-demand in every dashboard call, the stability score is computed nightly and stored. The dashboard API call is instant — no AI computation in the hot path.

🚀 HACKATHON MVP PRIORITY ORDER
Phase 1 (Day 1-2): Foundation
  ✅ Auth + Business creation
  ✅ Accounts + Manual transactions
  ✅ Categories (seeded)
  ✅ Basic dashboard cashflow

Phase 2 (Day 3-4): Core Revenue + Expense
  ✅ Invoice CRUD + PDF + Send
  ✅ Expense CRUD
  ✅ Entities (customer + supplier)
  ✅ Recurring rules (invoice + expense)

Phase 3 (Day 5-6): Intelligence
  ✅ AI transaction classifier (OpenAI)
  ✅ Anomaly detection (rule-based first)
  ✅ Cashflow forecast (baseline)
  ✅ Stability score (basic)

Phase 4 (Day 7): Co-Pilot
  ✅ AI Chat with business context injection
  ✅ AI Memory (key/value, no embeddings needed for MVP)
  ✅ Alerts + Insights (rule-based triggers)
  ✅ Scenario simulator (simple parameter adjustments)