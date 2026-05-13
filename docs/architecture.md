---
layout: default
title: Architecture & Design
---

# Architecture & Design

## Overview

Eunomia sits between an LLM (Gemini, swappable) and a MySQL data warehouse. It enforces a clear trust boundary: the model generates SQL, but the middleware validates every query against role-based access policies before execution.

## The Flow

### 1. Identity: Keycloak OIDC

Every request carries a `Bearer <JWT>` token issued by Keycloak.

- **Validation**: JWT signature checked against the realm's JWKS endpoint
- **Claims**: `iss`, `exp`, `aud` verified; `realm_access.roles` extracted
- **No hardcoded role map** — all authorization decisions come from OpenMetadata

**Example JWT payload:**
```json
{
  "iss": "http://keycloak:8080/realms/eunomia",
  "exp": 1715534100,
  "aud": "eunomia-client",
  "realm_access": {
    "roles": ["eunomia-finance-user", "eunomia-pii-unmask"]
  },
  "sub": "user123",
  "preferred_username": "finance.alice"
}
```

---

### 2. Authorization: OpenMetadata Tag Policies

Once roles are extracted, the middleware queries OpenMetadata for the user's **allowed view set**:

```
GET /api/v1/search/query?q=...&filters=tag_policy.{role1,role2}
```

OpenMetadata returns only the views that:
- Exist in the warehouse
- Have a tag policy allowing at least one of the user's roles

**Example:**
- User `finance.alice` has roles `[eunomia-finance-user]`
- OM returns views tagged `finance_access` with policy `allow: eunomia-finance-user`
- Views tagged `executive_summary` with policy `allow: eunomia-exec` are excluded

---

### 3. RAG: Catalog-Aware Retrieval

Given the user's NL query and their allowed-view list, the RAG service ranks the most relevant views.

```
POST /v1/retrieve
{
  "query": "what is our daily revenue last week",
  "allowed_views": ["finance_daily_revenue_view", "finance_summary"],
  "k": 2
}
```

**RAG's Qdrant filter:**
```json
{
  "must": [
    { "key": "name", "match": { "value": "<one of allowed_views>" } }
  ]
}
```

The index *cannot* return a view the user isn't entitled to, even if it's in the vector store.

**Stack:**
- **sentence-transformers/all-MiniLM-L6-v2** for query encoding (384-d, unit-normalized)
- **Qdrant** with payload filtering on `name`
- Indexer runs on cron, pulls full OM catalog, synthesizes per-view docs

---

### 4. LLM: SQL Generation

The LLM (Gemini) receives:
- The user's NL question
- The top-K view schemas + OM descriptions
- A system prompt constraining it to SELECT-only queries

```python
system_prompt = """
You are a data analyst. Generate a single MySQL SELECT query.
You may only query these views:
- finance_daily_revenue_view: daily revenue by order date
- finance_summary: monthly aggregates

Never use INSERT, UPDATE, DELETE, DROP, CREATE.
Return only the query, no explanation.
"""
```

---

### 5. Validation: sqlglot AST

Before *any* query touches the warehouse, the middleware validates it:

1. **Parse** the SQL with sqlglot
2. **Extract** all table/view names
3. **Check** that every name is in the user's allowed set
4. **Deny** if there's any mismatch

```python
from sqlglot import parse
from sqlglot.expressions import Table

sql = "SELECT * FROM finance_daily_revenue_view WHERE order_date >= CURDATE() - INTERVAL 5 DAY"
ast = parse(sql)[0]

allowed_views = {"finance_daily_revenue_view", "finance_summary"}
referenced_views = {t.name for t in ast.find_all(Table)}

assert referenced_views <= allowed_views  # ✓ passes
```

This prevents:
- The model hallucinating a view name
- SQL injection tricks
- Cross-user data leaks via UNION or subquery

---

### 6. Execution & PII Masking

Once validated, the query runs on MySQL. The result is then:

1. **Scanned** for columns tagged as PII in OpenMetadata
2. **Redacted** based on the user's roles
   - If user has `eunomia-pii-unmask`, show unmasked
   - Otherwise, mask (hash, truncate, or replace with `***`)
3. **Streamed** back to the client via SSE

```json
{
  "order_id": 12345,
  "customer_email": "***",     // masked (user lacks eunomia-pii-unmask)
  "total": 99.99
}
```

---

### 7. Audit

Every query is logged to a audit table:

```sql
INSERT INTO audit_log (user_id, roles, query, allowed_views, execution_ms, rows_returned, masked_columns, timestamp)
VALUES (...)
```

Auditors can later trace:
- Who ran what query
- Which views were accessed
- How long it took
- How many rows
- What was masked

---

## Design Choices

### Why RAG ranks; OM authorizes

Separating concerns:
- **RAG** = relevance engine (which views are most useful for this question?)
- **OM** = entitlement engine (which views can this user see?)

If they were mixed, an index drift could become a security bug. By filtering the Qdrant query with the allow-list as a server-side payload condition, we guarantee that no unauthorized view ever reaches the LLM, even if the index is stale.

### Why sqlglot validation, not trust the model

LLMs are non-deterministic. They occasionally hallucinate table names, construct UNION queries, or use subqueries in unexpected ways. The sqlglot AST check is a hard gate that catches these before they reach the warehouse.

### Why PII masking is separate from view-level access

Some views contain PII but are still useful in redacted form. For example, a "customers" view might include email addresses, but some users can see the masked version. This lets you expose broader data while protecting sensitive columns per-role.

---

## Stack Summary

| Layer | Technology |
|---|---|
| **Identity** | Keycloak (OIDC) |
| **Authorization** | OpenMetadata (tag policies) |
| **RAG** | Qdrant + sentence-transformers |
| **LLM** | Gemini (swappable) |
| **Validation** | sqlglot |
| **Warehouse** | MySQL |
| **API** | FastAPI (eunomia-middleware) |
| **Client** | Typer CLI (eunomia-cli) |

---

## End-to-End Example

```
User: "What is our daily revenue last week?"
      ↓
[1] Keycloak validates JWT → roles = [eunomia-finance-user]
      ↓
[2] OM query → allowed_views = [finance_daily_revenue_view, finance_summary]
      ↓
[3] RAG retrieves top-1 view: finance_daily_revenue_view (high relevance + authorized)
      ↓
[4] LLM generates: SELECT order_date, total_revenue FROM finance_daily_revenue_view 
                   WHERE order_date >= CURDATE() - INTERVAL 7 DAY
      ↓
[5] sqlglot validates: table "finance_daily_revenue_view" ∈ allowed_views ✓
      ↓
[6] MySQL executes query → 7 rows returned
      ↓
[7] PII masking: no masked columns in result
      ↓
[8] Audit logged + SSE streamed to client
      ↓
Result:
  order_date   total_revenue
  2026-05-07   45000.00
  2026-05-08   52000.00
  ...
```

---

## Security Boundary

The trust boundary is **clear and enforced in code**, not in natural language or model behavior:

- ✅ **Verified**: JWT signature, exp, aud
- ✅ **Authorized**: OM policies applied to view list
- ✅ **Validated**: sqlglot AST check before warehouse access
- ✅ **Masked**: PII redacted per role
- ✅ **Audited**: complete trace logged

An attacker cannot:
- Forge a JWT (Keycloak JWKS validates)
- Access a view outside their role (OM policies + RAG filtering + sqlglot validation)
- Inject SQL (sqlglot AST is mandatory)
- Leak PII (redacted per role)
- Hide their tracks (audit logging)
