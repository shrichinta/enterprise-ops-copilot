# Architecture deep dive

*Enterprise Ops & Invoice Reconciliation Copilot · by [Shridhar Ch](https://shrichinta.github.io)*

## 1. Data model
The ERP is the system of record for approved (net) amounts. Vendor PDFs are untrusted evidence.
```mermaid
erDiagram
  VENDORS ||--o{ SHIPMENTS : ships
  VENDORS ||--o{ INVOICES : bills
  SHIPMENTS ||--|| INVOICES : "billed by"
  INVOICES ||--o{ AUDIT_LOGS : "flagged in"
  VENDORS {
    text vendor_name PK
    text trn "15 digits, UAE TRN"
  }
  SHIPMENTS {
    text shipment_id PK
    text vendor_name FK
    text status "DELIVERED, IN_TRANSIT, CUSTOMS_HOLD, DELAYED"
    real declared_value_aed
    text updated_at
  }
  INVOICES {
    text invoice_id PK
    text shipment_id FK
    real billed_amount_aed "approved net, excl. VAT"
    text payment_status
    real vendor_cost_margin "manager only"
  }
  AUDIT_LOGS {
    int log_id PK
    text invoice_id FK
    text issue_type "fixed enum"
    text details
    text status "OPEN or RESOLVED"
    text flagged_at
  }
```
The audit log enforces idempotency **in the database**, not in application code:
```sql
CREATE UNIQUE INDEX ux_audit_open ON audit_logs(invoice_id, issue_type) WHERE status = 'OPEN';
```

## 2. Model resilience: circuit breaker
A pinned, pre-approved primary and secondary model. If both are unavailable, the system degrades to deterministic audits instead of failing.
```mermaid
stateDiagram-v2
  direction LR
  [*] --> Primary
  Primary --> Primary: success
  Primary --> Retrying: 503 / 429 / 500
  Retrying --> Primary: retry succeeds (backoff 2s, 4s)
  Retrying --> Secondary: retries exhausted, circuit opens
  Secondary --> Primary: cooldown elapsed, primary healthy
  Secondary --> Degraded: secondary also fails
  Degraded --> Primary: next request, cooldown elapsed
  note right of Degraded
    Deterministic reconcile + flag,
    RBAC still enforced,
    answer marked "degraded mode"
  end note
```

## 3. Target deployment (scripts included: `Dockerfile`, `deploy.sh`)
```mermaid
%%{init: {"flowchart": {"curve": "basis"}}}%%
flowchart LR
  U["Browser<br/>analyst / manager"] -->|HTTPS + login| CR
  subgraph GCP["Google Cloud project"]
    CR["Cloud Run service<br/>container, non-root, max 1 instance"]
    SM["Secret Manager<br/>demo password"]
    SA["Service account<br/>roles/aiplatform.user only"]
    VX["Vertex AI<br/>Gemini primary + secondary"]
    CB["Cloud Build<br/>build from source"]
  end
  CB -->|image| CR
  SM -->|mounted as env| CR
  CR -->|runs as| SA
  SA -->|IAM, no API keys| VX
  classDef ext fill:#EAF2FB,stroke:#1F6FEB,color:#0B2545
  classDef run fill:#E6F7F5,stroke:#2EC4B6,color:#0B2545
  classDef sec fill:#FDECEC,stroke:#D64545,color:#4A0D0D
  classDef ai fill:#FFF5E0,stroke:#F2B134,color:#4A3400
  class U ext
  class CR,CB run
  class SM,SA sec
  class VX ai
  style GCP fill:#FFFFFF,stroke:#475569,stroke-dasharray:4 3
```
| Control | Implementation |
|---|---|
| Identity to Vertex AI | service account with `roles/aiplatform.user`; no API keys in the container |
| Secrets | Secret Manager, mounted as env at runtime |
| Container | non-root user, minimal `python:3.12-slim` image |
| Cost guardrail | `max-instances 1`, `min-instances 0`, budget alert |

## 4. Design decisions
| Decision | Why |
|---|---|
| Arithmetic in code, not the LLM | auditable and repeatable; the LLM can't hallucinate a variance |
| Role bound server-side, not a tool argument | a prompt injection can't escalate privileges |
| Allow-list RBAC, not deny-list | new sensitive columns stay hidden by default |
| Evidence gate on the write tool | the model can only flag what a deterministic check found |
| Lexical + exact-ID retrieval before embeddings | invoice IDs and 15-digit TRNs are where pure vector search misses |
| Degraded mode + nightly batch | the business process doesn't depend on LLM availability |
| Tools exposed over MCP | the model provider is swappable (Gemini, Azure OpenAI, Claude) for data-residency needs |

## 5. Known limits of this build
| Area | This build | For a real client |
|---|---|---|
| Database | SQLite, synthetic data | Cloud SQL / Postgres or the client's ERP via read-only API |
| Auth | Gradio login + secret | SSO (Entra ID / Google) via Identity-Aware Proxy; role from token claims |
| Retrieval | BM25 + exact-ID filters | add dense vectors with RRF fusion for free-text questions |
| Capacity | on-demand model access | provisioned throughput + agreed SLA |

<sub>© 2026 Shridhar Ch · All rights reserved. Viewing permitted; reuse requires written permission.</sub>
