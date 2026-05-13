---
layout: default
title: FAQ & Troubleshooting
---

# FAQ & Troubleshooting

## General Questions

### What is Eunomia?

Eunomia is a **governance-first natural-language query middleware** that sits between an LLM and a data warehouse. It enforces role-based access control so the model can generate SQL, but the middleware validates every query before it touches the warehouse.

**Core idea:** The model is a *producer*, not an *authorizer*. The trust boundary is in code, not in the prompt.

---

### Why is it called Eunomia?

[Eunomia](https://en.wikipedia.org/wiki/Eunomia) is one of the Horae (Hours) in Greek mythology, goddess of order and law. Fitting for a governance layer. 🏛️

---

### What warehouse backends does Eunomia support?

Currently: **MySQL**.

Roadmap: PostgreSQL, Snowflake, BigQuery. The architecture is backend-agnostic; adding a new warehouse mainly requires swapping the SQL dialect in sqlglot and the driver.

---

### Can I use a different LLM?

Yes. The middleware currently defaults to **Gemini**, but it's pluggable. You can swap in:
- Claude (via Anthropic SDK)
- OpenAI (gpt-4, gpt-3.5)
- Open-source models (Llama, Mixtral via Ollama)
- Any model with a REST API

See `src/llm.py` in `eunomia-middleware` for the abstraction.

---

### Does Eunomia work with private/on-prem Keycloak?

Yes. Point `KEYCLOAK_URL` in your config to any Keycloak instance (public or private). The middleware fetches the JWKS endpoint to validate tokens — it just needs network access.

---

### Can I use a different identity provider?

Currently: **Keycloak OIDC** is baked in.

The middleware validates JWTs against Keycloak's JWKS. Swapping this out requires:
1. Changing the JWKS fetch endpoint
2. Adjusting claim extraction (roles, user ID, etc.)

Other OIDC providers (Okta, Auth0, Azure AD) would work with minimal changes.

---

## Setup & Installation

### The Keycloak realm won't initialize

Check the Keycloak logs:

```bash
docker-compose logs keycloak
```

Common issues:
- **Port 8080 already in use** — `docker ps`, find the conflicting container, kill it
- **Database migration timeout** — Keycloak can take 2-3 minutes on first startup. Wait and retry.
- **Disk space** — `docker system df` to check

---

### OpenMetadata is stuck on "Loading..."

The OM database (MySQL 5.7) can take a while to initialize. Check:

```bash
docker-compose logs openmetadata
docker-compose logs openmetadata_db
```

Wait 2-3 minutes, then refresh the browser. If it persists:

```bash
docker-compose restart openmetadata openmetadata_db
```

---

### RAG service fails to start

Check if Qdrant is running:

```bash
cd eunomia-rag
docker-compose logs qdrant
```

And the RAG service logs:

```bash
# In the RAG terminal, check the Python traceback
```

Common issues:
- **Qdrant port 6333 in use** — find and kill conflicting container
- **Missing environment variables** — check `eunomia-rag/.env` has `KEYCLOAK_URL`, `OPENMETADATA_URL`, etc.

---

### Middleware can't connect to RAG service

The middleware needs to reach the RAG service at `http://localhost:8001` by default. Check:

1. **RAG service is running** — `curl http://localhost:8001/health`
2. **Network connectivity** — if running in Docker, use `host.docker.internal:8001` instead of `localhost:8001`
3. **Firewall** — ensure port 8001 is not blocked

---

## Authentication & Authorization

### `eunomia-cli login` hangs

The device-code flow opens your browser. If it doesn't:

1. Check your default browser is set
2. Manually open the URL printed in the terminal
3. Enter the code on the Keycloak login page

If it times out after 600s, the code expires. Re-run `eunomia-cli login` to get a new one.

---

### JWT validation fails ("invalid_token")

Causes:
1. **Token expired** — the CLI auto-refreshes; if it fails, re-authenticate
2. **Wrong Keycloak realm** — ensure `KEYCLOAK_REALM` in middleware config matches the one in Keycloak
3. **Clock skew** — system time out of sync. Run `ntpdate -s time.nist.gov` (Mac/Linux) or sync via Windows Settings

---

### User is "not authorized" even though they have a role

Check:
1. **OpenMetadata tag policies** — the role must have a policy attached to at least one view. Go to OpenMetadata → Data → Tags → look for your role name
2. **View tags** — the view must be tagged with something the role can access. If a view has no tags, it's inaccessible to everyone
3. **Role name mismatch** — JWT role claims must *exactly* match OM policy role names (case-sensitive)

Example:
- JWT has role `eunomia-finance-user`
- OM policy has role `eunomia-finance-user` with access to `finance_*` views
- Query succeeds ✓

But if:
- JWT has role `Eunomia-Finance-User` (different case)
- OM policy is `eunomia-finance-user`
- Query fails ✗

---

### "No authorized views for this query"

The RAG service found views in the index but none matched the user's allowed set. This can happen if:

1. **User has no roles** — check `eunomia-cli whoami`
2. **Roles have no OM policies** — add policies in OpenMetadata
3. **Policies don't cover any views** — tag some views with role names
4. **Query mismatch** — user asking about a domain (e.g., "marketing") they're not authorized for

---

## Queries & Results

### LLM generates invalid SQL

The middleware runs sqlglot validation; if it fails, you'll see:

```
> Validating SQL...
  ✗ Validation failed: Table "bad_table_name" not in allowed set
```

Causes:
1. **LLM hallucination** — the model made up a table name. Rephrase the question to be more specific.
2. **Typo in view name** — ask for the exact view name: `eunomia-cli ask "What views start with finance?"`
3. **Schema mismatch** — table exists in warehouse but OM catalog is out of sync. Run `curl -X POST http://localhost:8001/v1/index-refresh ...` to refresh the RAG index.

**Workaround:** Skip the LLM and write SQL directly in the CLI (future feature; currently requires using the REST API).

---

### Query times out

MySQL is slow or hanging. Check:

1. **Connection pool** — is the middleware exhausting MySQL connections? `SHOW PROCESSLIST;` in MySQL
2. **Query complexity** — is the LLM generating a huge JOIN or subquery? Check the printed SQL.
3. **Warehouse size** — if the table is huge, even a simple query takes time. Add a `LIMIT` to the prompt: *"Top 10 daily revenues last week"*

---

### Results are missing columns / data

**PII masking is hiding them.** Check:

1. Run `eunomia-cli whoami` — do you have the `eunomia-pii-unmask` role?
2. If not, columns tagged as PII in OpenMetadata are redacted
3. Ask an admin to grant you the `eunomia-pii-unmask` role if you need to see sensitive data

Check which columns are masked:

```bash
eunomia-cli ask "..."
# Look for: "Masked Columns: [email, ssn, ...]"
```

---

### Query result is empty

Possible reasons:

1. **WHERE clause too restrictive** — the LLM generated `WHERE 1=0` or a condition that matches nothing
2. **No data in the view** — the view exists but is empty. Try `SELECT COUNT(*) FROM ...`
3. **User doesn't have view access** — but then you'd see "not authorized" earlier

Ask the question differently or check the data manually in MySQL.

---

## Performance & Optimization

### Queries are slow

Bottleneck analysis (in order of likelihood):

1. **MySQL query** — the warehouse itself is slow. Add indices, check query plan with `EXPLAIN`.
2. **RAG retrieval** — Qdrant vector search is slow. This is rare unless you have millions of vectors. Check Qdrant logs.
3. **LLM latency** — Gemini API is slow (network). Try switching to a local model (Ollama) for testing.
4. **Middleware overhead** — JWT validation, OM policy fetch, sqlglot parsing. Negligible unless doing hundreds of queries/second.

**Typical breakdown (for a fast query):**
- JWT validation: ~5ms
- OM policy fetch: ~20ms
- RAG retrieval: ~30ms
- LLM SQL generation: ~500ms (network bound)
- SQL execution: ~50ms
- **Total: ~600ms**

---

### How do I optimize RAG ranking?

The RAG service uses `sentence-transformers/all-MiniLM-L6-v2` out of the box. To improve ranking:

1. **Improve view descriptions** — the indexer synthesizes docs from OM. Add better descriptions in OpenMetadata.
2. **Switch embeddings model** — edit `src/embeddings.py` to use a larger, more accurate model (e.g., `all-mpnet-base-v2`). Trade-off: slower, bigger model.
3. **Fine-tune** — if you have domain-specific queries, fine-tune the embeddings on your query/view pairs. Advanced.

---

### Can I cache LLM generations?

Currently: no. Every query goes to the LLM.

Future: add LLM prompt caching (common queries → cache hit → faster response).

---

## Troubleshooting Guide

### General: "Something is broken, where do I start?"

```bash
# 1. Health check all services
curl http://localhost:8000/health          # Middleware
curl http://localhost:8001/health          # RAG
curl http://localhost:8080                 # Keycloak (loads login page)
curl http://localhost:8585                 # OpenMetadata (loads login page)

# 2. Check logs
docker-compose logs -f keycloak | head -50
docker-compose logs -f openmetadata | head -50

# 3. Verify your token
eunomia-cli whoami

# 4. Retry with verbose output
eunomia-cli ask "simple query" --verbose
```

---

### I lost my token or it expired

```bash
eunomia-cli logout
eunomia-cli login
# Re-authenticate
```

---

### Keycloak realm got messed up

Reset it:

```bash
cd eunomia-infrastructure
docker-compose down
docker volume rm eunomia-infrastructure_keycloak_data
docker-compose up -d keycloak
# Wait 2 minutes for initialization
docker-compose logs keycloak | grep "Realm 'eunomia' imported"
```

---

### I want to add a new user / role

1. Go to `http://localhost:8080` (Keycloak admin console)
2. Log in as `admin` / `admin`
3. Navigate to Realm: `eunomia` → Users → Add User
4. Set a password, assign roles
5. The roles are auto-synced to OM on next policy refresh

---

## Contributing & Support

### I found a bug

1. Check the [GitHub Issues](https://github.com/EunomiaAI) for existing reports
2. Open a new issue with:
   - Reproduction steps
   - OS / Docker version
   - Relevant logs (`docker-compose logs`, CLI output, Python traceback)
   - Expected vs. actual behavior

### I have a feature request

Open an issue or discussion on GitHub. We track ideas in the [Roadmap](https://github.com/EunomiaAI/eunomia-middleware/issues?q=label:roadmap).

### Where can I ask questions?

- **GitHub Discussions** (not yet set up; consider opening one)
- **Issues** with `question` label
- **Email** — contact maintainers in repo README

---

## Glossary

| Term | Definition |
|---|---|
| **JWT** | JSON Web Token — signed credentials issued by Keycloak |
| **JWKS** | JSON Web Key Set — Keycloak's public signing keys |
| **OM** | OpenMetadata — the catalog + policy decision point |
| **RAG** | Retrieval-Augmented Generation — find relevant data before generating |
| **Qdrant** | Vector database for semantic search |
| **sqlglot** | SQL parsing library; used for validation |
| **PII** | Personally Identifiable Information (email, SSN, etc.) |
| **SSE** | Server-Sent Events — real-time streaming to client |

---

## Additional Resources

- [Architecture & Design](architecture.md) — deep dive on each layer
- [Getting Started](quickstart.md) — step-by-step setup
- [API Reference](api.md) — REST endpoints & CLI commands
- [GitHub Repos](https://github.com/EunomiaAI) — source code
- [Apache 2.0 License](https://github.com/EunomiaAI/eunomia-middleware/blob/master/LICENSE)
