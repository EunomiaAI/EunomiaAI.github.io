---
layout: default
title: Home
nav_order: 1
description: Governance-first NLQ middleware for LLMs on data warehouses
permalink: /
---

# Eunomia

**Governance-first natural-language query middleware for LLMs on data warehouses.**

LLM-driven analytics that trust the model with sensitive data is theater. Eunomia draws the trust boundary in code: the model is a *producer*, not an *authorizer*.

## Core Architecture

```
   Client (CLI / UI)
        │
        │  Authorization: Bearer <Keycloak JWT>
        ▼
   ┌─────────────────────────────────────┐
   │   eunomia-middleware                │
   │                                     │
   │  1. JWT validation (Keycloak JWKS)  │
   │  2. Role → tag policies (OM)        │
   │  3. RAG (retrieve top-K views)      │
   │  4. LLM generates SQL               │
   │  5. sqlglot AST validation          │
   │  6. Query execution + PII masking   │
   │  7. SSE stream + audit              │
   │                                     │
   └─────────────────────────────────────┘
```

## Key Design Principles

- **Identity is verified, not trusted** — Keycloak JWT validated against realm JWKS
- **Authorization lives in OpenMetadata** — role-to-view access encoded as tag policies
- **RAG ranks; OM authorizes** — Qdrant filters by allowed views at query time; relevance and entitlement stay separate concerns
- **Validation before execution** — sqlglot AST validation against the full authorized scope
- **PII masking per role** — applied before SSE stream to client
- **Audit complete** — every query logged with identity, SQL, execution time, rows returned

## Four Open-Source Repos

| Repo | Purpose |
|---|---|
| **eunomia-middleware** | FastAPI policy enforcement core |
| **eunomia-rag** | Catalog-aware retrieval (Qdrant + sentence-transformers) |
| **eunomia-cli** | Typer CLI with OAuth 2.0 device-code login |
| **eunomia-infrastructure** | Docker-compose stack (Keycloak, OpenMetadata, MySQL, Elasticsearch, Qdrant) + end-to-end verification harness |

## Quick Links

- [Architecture & Design](docs/architecture.md)
- [Getting Started](docs/quickstart.md)
- [API Reference](docs/api.md)
- [FAQ & Troubleshooting](docs/faq.md)
- [GitHub Organization](https://github.com/EunomiaAI)

## License

Apache 2.0
