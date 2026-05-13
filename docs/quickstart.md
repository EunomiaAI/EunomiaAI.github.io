---
layout: default
title: Getting Started
nav_order: 3
---

# Getting Started

This guide walks you through setting up Eunomia locally and running your first query.

## Prerequisites

- **Docker** & **docker-compose** (for the infrastructure stack)
- **Python 3.11+** (for the middleware, RAG service, and CLI)
- **Git**

## 1. Bring Up the Infrastructure Stack

Clone and start the full stack (Keycloak, OpenMetadata, MySQL, Elasticsearch, Qdrant):

```bash
git clone https://github.com/EunomiaAI/eunomia-infrastructure.git
cd eunomia-infrastructure

# Start all services
docker-compose up -d

# Wait for services to be healthy (2-3 minutes)
docker-compose ps
```

**Services running:**
- **Keycloak** on `http://localhost:8080` (identity provider)
- **OpenMetadata** on `http://localhost:8585` (catalog + policies)
- **MySQL** on `localhost:3306` (analytics warehouse)
- **Elasticsearch** on `localhost:9200` (OM search)
- **Qdrant** (via RAG service) on `localhost:6333`

### Seeded Content

The compose automatically seeds:
- **Keycloak realm** (`eunomia`) with users, roles, and OAuth clients
- **OpenMetadata** with tag policies for role-based access
- **MySQL** with sample views and data

**Test users:**
- `finance.alice` / `password` (role: `eunomia-finance-user`)
- `executive.bob` / `password` (role: `eunomia-exec`)

---

## 2. Clone & Start the RAG Service

```bash
git clone https://github.com/EunomiaAI/eunomia-rag.git
cd eunomia-rag

# Install dependencies
pip install -r requirements.txt

# Start the service
python -m uvicorn src.api:app --port 8001
```

**RAG service running on** `http://localhost:8001`

The service auto-initializes its Qdrant index on startup by pulling the catalog from OpenMetadata.

---

## 3. Clone & Start the Middleware

```bash
git clone https://github.com/EunomiaAI/eunomia-middleware.git
cd eunomia-middleware

# Install dependencies
pip install -r requirements.txt

# Start the middleware
python -m uvicorn src.api:app --host 0.0.0.0 --port 8000
```

**Middleware running on** `http://localhost:8000`

---

## 4. Install & Configure the CLI

```bash
git clone https://github.com/EunomiaAI/eunomia-cli.git
cd eunomia-cli

# Install in editable mode
pip install -e .

# Verify installation
eunomia-cli --version
```

---

## 5. Log In

```bash
eunomia-cli login
```

This opens a device-code flow:
```
Open this URL in your browser:
  http://localhost:8080/realms/eunomia/device?user_code=ABCD-EFGH

Enter the code: ABCD-EFGH
```

1. Copy the URL into your browser
2. Log in as `finance.alice` / `password`
3. Confirm the device code
4. Return to the terminal — you're logged in!

**Output:**
```
Logged in as finance.alice
  email               finance.alice@open-metadata.org
  realm_access.roles  eunomia-finance-user
  token_expires_in    3600s
```

Your token is cached locally at `~/.eunomia/token.json` (mode 0600).

---

## 6. Ask Your First Question

```bash
eunomia-cli ask "What is our daily revenue last week?"
```

**You should see:**

```
Querying: What is our daily revenue last week?

> Authenticating & Fetching Roles...
  roles: [eunomia-finance-user]

> Authorized Views (2):
  - finance_daily_revenue_view
  - finance_summary

> Finding Relevant Views (RAG)...
  Retrieved: finance_daily_revenue_view (score: 0.87)

> Generating SQL (Attempt 1)...
  SELECT order_date, total_revenue
    FROM finance_daily_revenue_view
   WHERE order_date >= CURDATE() - INTERVAL 7 DAY
   ORDER BY order_date ASC

> Validating SQL...
  ✓ All tables in allowed set

> Executing Query...
  Execution time: 45ms
  Rows returned: 7

> Applying PII Masking...
  No masked columns

Execution Complete!

┌────────────┬─────────────────┐
│ order_date │ total_revenue   │
├────────────┼─────────────────┤
│ 2026-05-07 │ 45000.00        │
│ 2026-05-08 │ 52000.00        │
│ 2026-05-09 │ 48500.00        │
│ 2026-05-10 │ 61200.00        │
│ 2026-05-11 │ 55800.00        │
│ 2026-05-12 │ 59300.00        │
│ 2026-05-13 │ 47900.00        │
└────────────┴─────────────────┘
```

---

## 7. Try a PII-Masked Query

Switch to the `executive.bob` user:

```bash
eunomia-cli logout
eunomia-cli login
# Log in as executive.bob / password
```

Now ask a query that includes sensitive data:

```bash
eunomia-cli ask "Show me customer emails from our top 10 customers"
```

If `executive.bob` doesn't have the `eunomia-pii-unmask` role, email addresses will be masked:

```
┌──────────────────┬──────────────┐
│ customer_name    │ email        │
├──────────────────┼──────────────┤
│ Acme Corp        │ ***          │
│ TechStart Inc    │ ***          │
│ ...              │ ...          │
└──────────────────┴──────────────┘
```

---

## 8. Verify the Full Flow

Run the end-to-end test suite:

```bash
cd eunomia-infrastructure
python verify_phase_d.py
```

This runs 30 test cases covering:
- JWT validation
- Authorization via OM policies
- RAG filtering
- SQL generation & validation
- PII masking
- Audit logging

Expected output: `30 passed` ✓

---

## Troubleshooting

### Services won't start

Check Docker:
```bash
docker-compose logs keycloak
docker-compose logs openmetadata
```

### JWT validation fails

Keycloak may still be initializing. Wait 30s and retry:
```bash
eunomia-cli ask "..."
```

### RAG returns no results

The Qdrant index may not have been initialized. Check the RAG service logs:
```bash
curl http://localhost:8001/health
```

If unhealthy, restart it:
```bash
# In the eunomia-rag terminal, Ctrl+C then restart
```

### "User not authorized for any views"

Check which roles your user has:
```bash
eunomia-cli login  # logs roles during login
```

Then verify those roles have OM tag policies. See [Architecture → Authorization](architecture.md).

---

## Next Steps

- Read the [Architecture & Design](architecture.md) for deep dives on each layer
- Check the [API Reference](api.md) for REST endpoints and payloads
- Explore the [FAQ](faq.md) for common questions
- Look at the source code in each repo's `src/` directory

---

## Local Development

### Reloading Code

All three services (middleware, RAG, CLI) are in editable installs. Changes take effect on restart:

```bash
# In each service terminal:
# Ctrl+C to stop, then re-run the python -m uvicorn command
```

### Resetting the Database

To start fresh (wipe all audit logs, queries, OM state):

```bash
cd eunomia-infrastructure
docker-compose down
docker volume rm eunomia-infrastructure_keycloak_data eunomia-infrastructure_openmetadata_db
docker-compose up -d
```

Then re-run the initialization script:
```bash
python seed_openmetadata.py
```

---

## Architecture Diagram

```
┌──────────────────┐
│   eunomia-cli    │
│  (Typer CLI)     │
└────────┬─────────┘
         │ Bearer JWT (Keycloak)
         ▼
┌──────────────────────────────────────┐
│   eunomia-middleware (FastAPI)       │
│                                      │
│  1. JWT validation                   │
│  2. OM policy → allowed_views        │
│  3. Call RAG /retrieve               │
│  4. LLM → SQL generation             │
│  5. sqlglot validation               │
│  6. MySQL execution                  │
│  7. PII masking                      │
│  8. SSE response + audit             │
└────────┬────────────────────────────┘
         │
    ┌────┴────┐
    │          │
    ▼          ▼
  RAG       MySQL
(Qdrant)   (Warehouse)
```
