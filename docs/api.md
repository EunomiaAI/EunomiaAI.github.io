---
layout: default
title: API Reference
---

# API Reference

## Middleware API (`eunomia-middleware`)

The middleware exposes a REST API for querying the warehouse via natural language.

### Base URL

```
http://localhost:8000
```

### Authentication

All requests require a `Bearer` token from Keycloak:

```
Authorization: Bearer <JWT>
```

The JWT is validated against Keycloak's JWKS endpoint and its `exp`, `iss`, and `aud` claims.

---

## Endpoints

### `POST /api/v1/query`

Execute a natural-language query.

**Request:**

```json
{
  "query": "What is our daily revenue last week?",
  "k": 1
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `query` | string | ✓ | Natural-language question |
| `k` | int | Optional (default: 1) | Top-K views to use for LLM prompt |

**Response (200 OK):**

Streamed as Server-Sent Events (SSE).

```
event: status
data: {"stage": "authenticating", "message": "Validating JWT..."}

event: status
data: {"stage": "authorizing", "message": "Fetching allowed views from OpenMetadata..."}

event: status
data: {"stage": "retrieving", "message": "Querying RAG for top-K views..."}
data: {"retrieved_views": ["finance_daily_revenue_view"], "scores": [0.87]}

event: status
data: {"stage": "generating", "message": "Calling LLM for SQL generation..."}

event: sql
data: {"sql": "SELECT order_date, total_revenue FROM finance_daily_revenue_view WHERE order_date >= CURDATE() - INTERVAL 7 DAY"}

event: status
data: {"stage": "validating", "message": "Running sqlglot AST validation..."}

event: status
data: {"stage": "executing", "message": "Running query on warehouse..."}
data: {"execution_ms": 45, "rows_returned": 7}

event: status
data: {"stage": "masking", "message": "Applying PII masking..."}
data: {"masked_columns": []}

event: results
data: [
  {"order_date": "2026-05-07", "total_revenue": 45000.00},
  {"order_date": "2026-05-08", "total_revenue": 52000.00},
  ...
]

event: complete
data: {"audit_id": "aud-12345", "total_time_ms": 320}
```

**Error (400 Bad Request):**

```json
{
  "error": "query_required",
  "message": "Missing required field: query"
}
```

**Error (401 Unauthorized):**

```json
{
  "error": "invalid_token",
  "message": "JWT signature validation failed"
}
```

**Error (403 Forbidden):**

```json
{
  "error": "not_authorized",
  "message": "User has no authorized views for this query"
}
```

---

### `GET /api/v1/authorized-views`

Retrieve the list of views the authenticated user can access.

**Request:**

```
GET /api/v1/authorized-views
Authorization: Bearer <JWT>
```

**Response (200 OK):**

```json
{
  "user": "finance.alice",
  "roles": ["eunomia-finance-user"],
  "authorized_views": [
    {
      "name": "finance_daily_revenue_view",
      "description": "Daily revenue aggregated by order date",
      "schema": "SELECT order_date, total_revenue FROM ...",
      "tags": ["pii:none"]
    },
    {
      "name": "finance_summary",
      "description": "Monthly financial summary",
      "schema": "SELECT month, revenue, expenses FROM ...",
      "tags": ["pii:none"]
    }
  ]
}
```

---

### `GET /health`

Health check endpoint.

**Request:**

```
GET /health
```

**Response (200 OK):**

```json
{
  "status": "healthy",
  "components": {
    "keycloak": "ok",
    "openmetadata": "ok",
    "rag_service": "ok",
    "mysql": "ok"
  }
}
```

---

## RAG Service API (`eunomia-rag`)

The RAG service retrieves the most relevant views for a given query.

### Base URL

```
http://localhost:8001
```

### `POST /v1/retrieve`

Retrieve top-K most relevant views.

**Request:**

```json
{
  "query": "What is our daily revenue last week?",
  "allowed_views": ["finance_daily_revenue_view", "finance_summary"],
  "k": 1
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `query` | string | ✓ | Natural-language query |
| `allowed_views` | array | ✓ | List of view names the user is authorized to see |
| `k` | int | Optional (default: 1) | Number of top results to return |

**Response (200 OK):**

```json
{
  "query": "What is our daily revenue last week?",
  "retrieved": [
    {
      "name": "finance_daily_revenue_view",
      "score": 0.87,
      "description": "Daily revenue aggregated by order date",
      "schema": "...",
      "tags": ["finance", "revenue"]
    }
  ],
  "execution_ms": 45
}
```

---

### `POST /v1/index-refresh`

Manually refresh the Qdrant index from OpenMetadata. Normally runs on cron, but this allows admin-triggered updates.

**Request:**

```json
{
  "keycloak_service_account_token": "<service-account-jwt>"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `keycloak_service_account_token` | string | ✓ | JWT from Keycloak service account (not a user token) |

**Response (200 OK):**

```json
{
  "indexed_views": 12,
  "execution_ms": 3200,
  "status": "success"
}
```

---

### `GET /health`

Health check endpoint.

**Request:**

```
GET /health
```

**Response (200 OK):**

```json
{
  "status": "healthy",
  "qdrant_ready": true,
  "vector_count": 12,
  "last_index_refresh": "2026-05-13T10:30:00Z"
}
```

---

## CLI Commands (`eunomia-cli`)

### `eunomia-cli login`

Authenticate with Keycloak via OAuth 2.0 Device Code flow.

```bash
eunomia-cli login
```

Outputs:
```
Open this URL in your browser:
  http://localhost:8080/realms/eunomia/device?user_code=ABCD-EFGH

Enter the code: ABCD-EFGH
Polling for authentication... (expires in 600s)

Logged in as finance.alice
  email               finance.alice@open-metadata.org
  realm_access.roles  eunomia-finance-user
  token_expires_in    3600s
```

Token is cached at `~/.eunomia/token.json` with mode 0600 (read/write owner only).

---

### `eunomia-cli ask <query>`

Ask a natural-language question to the warehouse.

```bash
eunomia-cli ask "What is our daily revenue last week?"
```

Streams progress and results in real-time. Output:

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
[results table]
```

---

### `eunomia-cli logout`

Clear the cached token and log out.

```bash
eunomia-cli logout
```

---

### `eunomia-cli whoami`

Display the currently logged-in user and roles.

```bash
eunomia-cli whoami
```

Output:
```
Logged in as finance.alice
  email               finance.alice@open-metadata.org
  realm_access.roles  eunomia-finance-user
  token_expires_in    2850s
```

---

### `eunomia-cli config show`

Display CLI configuration (cache location, keycloak URL, etc.).

```bash
eunomia-cli config show
```

Output:
```
Configuration:
  keycloak_url         http://localhost:8080
  keycloak_realm       eunomia
  middleware_url       http://localhost:8000
  rag_url              http://localhost:8001
  cache_dir            /Users/anuj/.eunomia
  token_path           /Users/anuj/.eunomia/token.json
```

---

## Error Handling

### Common Errors

| Status | Error Code | Message | Resolution |
|---|---|---|---|
| 401 | `invalid_token` | JWT signature validation failed | Re-authenticate with `eunomia-cli login` |
| 401 | `token_expired` | JWT has expired | Token auto-refreshes; if fails, re-authenticate |
| 403 | `not_authorized` | User has no authorized views | Check OpenMetadata tag policies for your role |
| 400 | `invalid_query` | Query is empty or malformed | Rephrase the question |
| 400 | `sql_validation_failed` | Generated SQL uses unauthorized tables | LLM model quality issue; rephrase question |
| 502 | `rag_service_unavailable` | RAG service not responding | Check RAG service status: `curl http://localhost:8001/health` |
| 502 | `warehouse_unavailable` | MySQL warehouse not responding | Check MySQL is running |

### Retry Logic

The CLI automatically retries:
- Token refresh if `token_expired`
- SQL generation (up to 3 attempts) if `sql_validation_failed`
- RAG calls (up to 2 attempts) with exponential backoff

Manual retries: re-run `eunomia-cli ask "..."` — the query is idempotent (every run gets logged separately).

---

## Audit Logging

Every query execution is logged to the `audit_log` table in MySQL:

```sql
SELECT * FROM audit_log WHERE user_id = 'finance.alice';
```

Columns:
- `audit_id` — unique identifier
- `user_id` — subject from JWT
- `roles` — JSON array of roles at execution time
- `query` — user's NL question
- `allowed_views` — JSON array of authorized views
- `generated_sql` — the SQL the LLM generated (before validation)
- `execution_ms` — query execution time on warehouse
- `rows_returned` — result row count
- `masked_columns` — JSON array of which columns were masked
- `timestamp` — UTC datetime of execution

---

## Rate Limiting

Currently, no built-in rate limiting. For production, configure:

- **Keycloak** token rate limits
- **OpenMetadata** API quotas
- **MySQL** connection pool limits
- **Middleware** request queue depth

See deployment guides in each repo's `README.md`.
