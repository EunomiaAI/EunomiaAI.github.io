---
layout: default
title: Home
nav_order: 1
description: Governance-first NLQ middleware for LLMs on data warehouses
permalink: /
---

<section class="home-hero">
  <div class="eyebrow">Open-source governance middleware for LLM analytics</div>
  <h1>Eunomia</h1>
  <p class="hero-tagline">Let LLMs generate SQL, while authorization, validation, masking, and audit stay enforced in code.</p>

  <div class="hero-actions">
    <a class="btn btn-primary fs-5" href="{{ '/docs/quickstart.html' | relative_url }}">Get Started</a>
    <a class="btn fs-5" href="https://github.com/EunomiaAI">View GitHub</a>
    <a class="btn fs-5" href="{{ '/docs/architecture.html' | relative_url }}">Read Architecture</a>
  </div>

  <div class="oss-cues" aria-label="Project cues">
    <span>Apache 2.0</span>
    <span>GitHub Pages</span>
    <span>Static docs</span>
    <span>No paid hosting</span>
  </div>
</section>

<section class="home-section">
  <div class="section-kicker">Why it exists</div>
  <h2>Prompts are not a security boundary.</h2>
  <p class="section-lede">Eunomia sits between natural-language analytics and your warehouse so the model can propose a query, but never decide what data a user is allowed to touch.</p>

  <div class="why-grid">
    <div class="why-card">
      <h3>The risk</h3>
      <p>LLM analytics can leak sensitive data when the model is trusted to avoid forbidden tables, PII columns, or unauthorized scopes.</p>
    </div>
    <div class="why-card">
      <h3>The boundary</h3>
      <p>Identity, authorization, retrieval filtering, SQL validation, masking, and audit happen outside the prompt path.</p>
    </div>
    <div class="why-card">
      <h3>The result</h3>
      <p>The model becomes a producer of candidate SQL. Eunomia remains the policy enforcement layer before rows are read.</p>
    </div>
  </div>
</section>

<section id="how-it-works" class="home-section architecture-preview" markdown="1">
  <div class="section-kicker">How it works</div>
  <div class="section-heading-row">
    <div>
      <h2>One enforced path from question to result.</h2>
      <p class="section-lede">Every request passes through the same eight-layer pipeline. The LLM output enters at SQL generation, then must survive validation before execution.</p>
    </div>
    <a class="text-link" href="{{ '/docs/architecture.html' | relative_url }}">Architecture details</a>
  </div>

  <a class="architecture-figure" href="#architecture-full" aria-label="Open architecture diagram full size">
    <img src="{{ '/assets/images/eunomia-architecture.svg' | relative_url }}" alt="Eunomia governed natural-language query pipeline">
    <span>Click to expand</span>
  </a>

  <div id="architecture-full" class="diagram-lightbox" aria-label="Expanded architecture diagram">
    <a class="diagram-lightbox-backdrop" href="#how-it-works" aria-label="Close expanded diagram"></a>
    <div class="diagram-lightbox-panel">
      <a class="diagram-lightbox-close" href="#how-it-works" aria-label="Close expanded diagram">Close</a>
      <img src="{{ '/assets/images/eunomia-architecture.svg' | relative_url }}" alt="Eunomia governed natural-language query pipeline expanded">
    </div>
  </div>
</section>

<section class="home-section">
  <div class="section-kicker">Core guarantees</div>
  <h2>Governance controls stay explicit and testable.</h2>

  <div class="principle-grid">
    <div class="principle-card">
      <h3>Verified identity</h3>
      <p>Every request carries a Keycloak JWT validated against JWKS before policy lookup or query generation begins.</p>
    </div>
    <div class="principle-card">
      <h3>Catalog authorization</h3>
      <p>OpenMetadata tag policies define allowed views. The middleware enforces those policies without hardcoded role maps.</p>
    </div>
    <div class="principle-card">
      <h3>Allow-list retrieval</h3>
      <p>Qdrant retrieval is filtered by the user's allowed view set, so relevance ranking cannot widen access.</p>
    </div>
    <div class="principle-card">
      <h3>SQL AST validation</h3>
      <p>sqlglot parses model-generated SQL and checks referenced tables against the authorized scope before execution.</p>
    </div>
    <div class="principle-card">
      <h3>Role-aware masking</h3>
      <p>PII redaction happens at result time based on catalog tags and JWT roles, keeping broad views narrowly exposed.</p>
    </div>
    <div class="principle-card">
      <h3>Audit trail</h3>
      <p>Identity, roles, SQL, allowed views, execution time, returned rows, and masked columns are logged for traceability.</p>
    </div>
  </div>
</section>

<section class="home-section">
  <div class="section-kicker">Project modules</div>
  <h2>Four repositories, one local stack.</h2>

  <div class="repo-grid">
    <a class="repo-card" href="https://github.com/EunomiaAI/eunomia-middleware">
      <div class="repo-topline">
        <span class="repo-name">eunomia-middleware</span>
        <span class="repo-role">Policy core</span>
      </div>
      <p>FastAPI enforcement layer for JWT validation, OpenMetadata policy lookup, LLM orchestration, SQL validation, PII masking, and audit.</p>
    </a>
    <a class="repo-card" href="https://github.com/EunomiaAI/eunomia-rag">
      <div class="repo-topline">
        <span class="repo-name">eunomia-rag</span>
        <span class="repo-role">Retrieval</span>
      </div>
      <p>Catalog-aware retrieval service backed by Qdrant and sentence-transformers, with server-side allow-list filtering.</p>
    </a>
    <a class="repo-card" href="https://github.com/EunomiaAI/eunomia-cli">
      <div class="repo-topline">
        <span class="repo-name">eunomia-cli</span>
        <span class="repo-role">Developer UX</span>
      </div>
      <p>Typer CLI for device-code login, local token caching, automatic refresh, and streamed query progress.</p>
    </a>
    <a class="repo-card" href="https://github.com/EunomiaAI/eunomia-infrastructure">
      <div class="repo-topline">
        <span class="repo-name">eunomia-infrastructure</span>
        <span class="repo-role">Local stack</span>
      </div>
      <p>Docker Compose environment for Keycloak, OpenMetadata, MySQL, Elasticsearch, Qdrant, seeded data, and verification cases.</p>
    </a>
  </div>
</section>

<section class="home-section quickstart-preview">
  <div class="quickstart-copy">
    <div class="section-kicker">Quickstart preview</div>
    <h2>Try the happy path locally.</h2>
    <p class="section-lede">Bring up the infrastructure stack, start the services, log in as a seeded user, and ask a governed analytics question.</p>
    <a class="btn btn-primary fs-5" href="{{ '/docs/quickstart.html' | relative_url }}">Open Getting Started</a>
  </div>

  <div class="terminal-window" aria-label="Eunomia CLI example">
    <div class="terminal-bar">
      <span></span><span></span><span></span>
    </div>
    <pre><code>$ eunomia-cli login
Logged in as finance.alice
roles: [eunomia-finance-user]

$ eunomia-cli ask "What is our daily revenue last week?"
> Validating JWT...              ok
> Fetching authorized views...   ok  2 views
> RAG retrieval...               ok  finance_daily_revenue_view
> Validating AST...              ok  all tables authorized
> Executing query...             ok  7 rows / 45ms
> Applying PII masking...        ok  0 columns masked</code></pre>
  </div>
</section>

<section class="home-footer-note">
  Apache 2.0 licensed. Built as static GitHub Pages documentation with no paid hosting dependency.
</section>
