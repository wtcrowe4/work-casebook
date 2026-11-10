# Work Casebook — Thomas Crowe

Sanitized write-ups of production work I build and run as **Senior Software Engineer** at [CalibrationWands.com](https://calibrationwands.com) (a division of Fastec Services LLC). Architecture and outcomes only — no proprietary source, credentials, or internal infrastructure detail.

> **Context.** CalibrationWands.com is a B2B e-commerce manufacturer of NIST-traceable calibration test standards, serving food, pharmaceutical, and textile quality-control customers. I build and operate its technical stack: storefront, infrastructure, internal platform, business-system integrations, and AI automation.

---

## 1. Self-hosted AI automation — a fleet of production agents

**Problem.** A range of repetitive, judgment-light business operations — reading purchase orders, generating documents, routing orders, preparing machine files — consumed hours of manual work daily and didn't scale.

**Approach.** Designed and deployed six self-hosted AI agents that handle these workflows end to end, built on infrastructure I architect and run myself.

```mermaid
flowchart TD
    I[Business events: orders, POs, documents] --> O[Agent orchestration — LangGraph + custom Python]
    O --> A1[Order processing]
    O --> A2[PO → catalog SKU mapping]
    O --> A3[Document handling]
    O --> A4[CNC G-code generation]
    O --> S[Self-hosted serving — vLLM + Ollama]
    R[(pgvector RAG — embedded knowledge base)] --> O
    O --> L[Langfuse — tracing + evals]
```

- Six agents orchestrated with **LangGraph** and custom Python, several exposed through Slack (Socket Mode)
- Models served on a self-hosted stack (**vLLM** and **Ollama**) — full control over the models, the data, and pipeline behavior
- Responses grounded in a **pgvector** retrieval pipeline so agents reason over real source documents, not model memory
- Local models fine-tuned with **Unsloth/PyTorch**, training tracked in Weights & Biases; quantization (AWQ/GPTQ) for throughput
- All agent runs traced and evaluated in **Langfuse**

**Stack.** LangGraph · vLLM · Ollama · pgvector · PyTorch · Unsloth · Langfuse · Python · Docker

**Outcome.** Repetitive operations automated on self-hosted infrastructure, keeping sensitive business data in-house and independent of third-party APIs.

> Self-hosted lab; personal and work infrastructure kept strictly separate.

---

## 2. Test Standards Hub — order, certificate & production platform

**Problem.** Orders lived in two disconnected systems (WooCommerce + QuickBooks Desktop). Certificate generation and CNC engraving artwork were manual, and serialized test-standard tracking had no system of record.

**Approach.** Built an internal full-stack platform that becomes the single source of truth and drives production end to end.

```mermaid
flowchart TD
    W[WooCommerce orders] -->|webhook over Cloudflare Tunnel| H[Test Standards Hub]
    Q[QuickBooks Desktop] --> H
    H --> C[Cryptographically signed certificates — pyHanko]
    H --> S[(Serialized test numbers — relational DB)]
    H --> E[SVG engraving + STL files] --> M[CNC machine]
    H --> P[Account-gated customer document portal]
    H --> R[PrintNode — shop-floor printing]
```

- New orders trigger the platform through a **webhook over a Cloudflare Tunnel** — no exposed origin
- Auto-generates calibration certificates (Word-template merge via Adobe PDF Services), AES-256 locks them, and signs them with **pyHanko** for tamper-evident verification
- Embeds a measurement-capture module: per-unit micrometer readings against NIST-traceable references, bringing the workflow into conformance with ISO 10012
- Tracks serialized test numbers for every standard sold, backed by a relational database
- Generates SVG engraving and STL files sent to the CNC machine — a software-to-machine production pipeline
- Triggers **PrintNode** shop-floor printing and serves finished certificates through an account-gated customer portal

**Stack.** React/Vite · Express · Prisma · PostgreSQL · pyHanko · Cloudflare Tunnel · PrintNode · SVG/STL

**Outcome.** 3,800+ certificates auto-generated, the core of 500+ hours of manual work eliminated per year — one source of truth across two systems; manual certificate, printing, and engraving steps replaced by an automated order-to-manufacturing pipeline.

---

## 3. Production migration + infrastructure hardening

**Problem.** The production storefront needed a more reliable, performant home, with stronger operational practices around it.

**Approach.** Planned and executed a live cutover to dedicated DigitalOcean NVMe hosting — no data loss, no customer-visible downtime — then hardened the surrounding operations.

```mermaid
flowchart LR
    A[Live WooCommerce store] -->|full export + sync| B[DigitalOcean NVMe host]
    B --> C[Cloudflare DNS + CDN + SSL]
    C --> D[Live storefront]
    B --> E[Redis Object Cache Pro]
    B --> F[Email deliverability: SPF/DKIM/DMARC]
    B --> G[Bedrock + React staging — Git-tracked deploys]
```

- DNS + CDN cutover via Cloudflare; SSL re-issue; recurring outages cut to near zero
- Email deliverability restored (SPF/DKIM/DMARC); Stripe webhooks verified end to end
- Three-tier backup/disaster-recovery (off-site cloud, server-local, provider snapshots) with verified restores
- Database tuning (InnoDB buffer pool, connection limits, timeouts) for production stability
- Currently migrating off a plugin-heavy WordPress setup onto a **Bedrock + React** architecture, rebuilding tooling in-house to eliminate dependency drift

**Stack.** DigitalOcean · Cloudflare · Linux · Redis · Bedrock · React · MySQL

**Outcome.** Faster, reliable hosting; near-zero outages; a safe staging path and verified recovery for ongoing changes.

---

## 4. Accounting & inventory integration + AI order intake

**Problem.** Orders, invoices, and stock lived in separate systems. Every web order and emailed purchase order had to be re-keyed into accounting by hand, and inventory was only reconciled after the fact.

**Approach.** Connected the order platform to the accounting and inventory systems, and put an LLM intake pipeline in front of the inbox — with a human approving anything before it posts.

```mermaid
flowchart LR
    E[Emailed POs + quote requests] --> AI[LLM intake: extract, match part numbers, dedupe]
    AI --> HA{Human approval}
    W[Web orders] --> H[Order platform]
    HA --> H
    H -->|qbXML via Web Connector| Q[QuickBooks Desktop: invoices, estimates, items, customers]
    H -->|on certificate generation| Z[Zoho Inventory: stock + build planning]
    H --> L[(Audit change log with revert)]
```

- **QuickBooks Desktop** integration over Web Connector / qbXML: invoices, estimates, items, customers, and inventory adjustments, with per-write verification and an audit change log that can revert a write
- Built behind a **provider-agnostic accounting interface**, so a future move off QuickBooks Desktop swaps an adapter instead of rewriting the platform
- **Zoho Inventory** decremented and builds planned automatically as certificates are generated, so stock tracks production in real time
- **AI order intake:** an LLM pipeline turns emailed purchase orders and quote requests into validated orders: customer part-number matching, duplicate-invoice detection, prompt-injection guards, and a **human approval step** before anything reaches accounting

**Stack.** Node/Express · PostgreSQL · QuickBooks Desktop (qbXML) · Zoho Inventory API · LLM pipeline (self-hosted + guarded prompts)

**Outcome.** Orders flow from inbox or storefront into accounting and inventory without re-keying; every automated write is verified, logged, and reversible.

---

## 5. Platform engineering: CI/CD & observability

**Problem.** A growing set of internal services needed safe, repeatable releases and fast visibility when something broke, without a dedicated ops team.

**Approach.** Built a delivery pipeline and observability layer around the internal platform.

```mermaid
flowchart LR
    G[Merge request] --> CI[GitLab CI on Docker runners]
    CI --> T[Tests + gitleaks secret scan + LLM-assisted review]
    T --> D[Release-swap deploy, zero restart]
    D --> HC{Post-deploy health check + schema-drift check}
    HC -->|fail| RB[Keep previous release]
    HC -->|pass| P[Live]
    P --> S[Sentry/Bugsink error tracking] --> A[Alerts to Slack]
```

- **GitLab CI** on Docker runners; every merge runs tests, **gitleaks** secret scanning, and an LLM-assisted code review
- **Zero-restart release-swap deploys** with post-deploy health checks and schema-drift detection
- **Sentry/Bugsink** error tracking with alert routing to Slack
- Internal tools published behind **Cloudflare Tunnel + Access** on an Oracle Cloud ARM host: no open inbound ports

**Stack.** GitLab CI · Docker · Sentry/Bugsink · Cloudflare Tunnel + Access · Oracle Cloud · Linux/systemd

**Outcome.** Releases that verify themselves after deploy, secrets caught before merge, and errors that surface in Slack before a customer reports them.

---

## 6. Custom storefront engineering & data integrity

**Problem.** The storefront needed configurable, validated product SKUs across many product lines, and order data needed to stay correct under real load.

**Approach.** Engineered a custom SKU-configurator system and hardened the data layer behind it.

```mermaid
flowchart LR
    U[Product options] --> J[SKU configurator — JS + PHP hooks]
    J --> C[Cart meta] --> O[Order meta] --> P[PDF + email]
    DB[(Production DB)] --> G[SQL integrity guards]
```

- Dynamic SKU configurator across 10 product lines (JavaScript + PHP WooCommerce hooks) generating valid SKUs in real time, with cart → order-meta → email → PDF integration
- Custom Dompdf invoice/packing-slip templates; validated with Playwright end-to-end tests
- **SQL data recovery & integrity:** recovered 1,534 order records with targeted SQL against the production database, diagnosed the inconsistency at the schema level, and added PHP validation guards to prevent recurrence

**Stack.** JavaScript · PHP · WooCommerce · Dompdf · Playwright · SQL

**Outcome.** Configurable, validated SKUs across the catalog; order data integrity restored and protected at the code level.

---

### Notes

These summaries describe work I performed. Proprietary source code, credentials, customer data, pricing, and internal infrastructure details are intentionally omitted.