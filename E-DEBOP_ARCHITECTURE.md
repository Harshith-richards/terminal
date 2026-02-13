# Enterprise Digital Engineering & BIM Operations Platform (E-DEBOP)

## 1) Executive Architecture Vision
E-DEBOP is a mission-critical, multi-tenant, cloud-native engineering management platform for mega-project delivery (airports, metro rail, hospitals, industrial plants, smart cities). The architecture is designed for 100,000+ users, 10,000+ concurrent sessions, TB-scale BIM storage, strict compliance, and multi-country operations.

**Architecture style**
- Microservices + event-driven architecture
- API Gateway + BFF (Backend-for-Frontend)
- Polyglot persistence:
  - MongoDB: project/domain documents
  - PostgreSQL: finance, accounting, ledger-grade data
  - Redis: distributed cache, session state, rate limiting, locks
  - Elasticsearch: search/indexing
  - S3-compatible object storage: file binaries and large models
- Messaging backbone:
  - Kafka: high-volume event streams, analytics feeds, audit pipelines
  - RabbitMQ: workflow/task queues, retries, delayed jobs
- Kubernetes-first deployment with service mesh-ready networking

---

## 2) Multi-Tenant Organizational Model & Isolation

### Hierarchy
- Tenant (Company)
  - Branch
    - Department
      - Project

### Isolation Strategy
- **Primary isolation key** on every record: `tenantId`, plus `branchId`, `departmentId`, `projectId` where relevant.
- **Data partitioning**
  - MongoDB: shard key includes `tenantId` (and optionally `projectId` for hot datasets).
  - PostgreSQL: schema-per-tenant for enterprise-grade isolation OR row-level security with `tenant_id` for large-scale shared clusters.
  - Elasticsearch: index-per-tenant for strict isolation (or aliases with document-level security).
  - Object storage: bucket/prefix policy (`tenant/{tenantId}/project/{projectId}/...`).
- **Security boundaries**
  - Tenant-scoped encryption keys (KMS envelope encryption).
  - Tenant-aware API guards and query middleware.
  - Cross-tenant access denied by default.

---

## 3) Microservice Landscape (Domain-Driven)

### Core Platform Services
1. **Identity & Access Service (IAM)**
   - SSO, OAuth2, JWT/refresh token, MFA
   - RBAC + ABAC + permission matrix
   - Session policies and adaptive risk checks

2. **Tenant & Organization Service**
   - Company/branch/department provisioning
   - Policy templates, locale/timezone/currency defaults

3. **Project Lifecycle Service**
   - Tender → Proposal → BOQ → Contract → Design → Coordination → Construction → Handover/FM
   - WBS, cost breakdown, resources, milestones, critical path metadata

4. **BIM Document & Model Service**
   - Versioned model/document metadata
   - Check-in/check-out, file lock, revisions, approval workflows
   - Model audit history and transmittals

5. **File Ingestion & Storage Service**
   - Large-file chunked upload orchestrator
   - Virus scan, file hash, OCR/metadata extraction, storage lifecycle

6. **Clash & Coordination Service**
   - Navisworks clash tracking
   - Model review comments, status workflows, assignment

7. **MEP/HVAC Engineering Service**
   - Equipment schedules, duct sizing logs, load calculations
   - Technical submittals, BOQ tracking, RFIs, method statements

8. **Site & Field Operations Service**
   - Inspection logs, NCR, DPR, snag lists, safety logs
   - Geo-tagged photos and issue tracking

9. **Procurement & Vendor Service**
   - Vendor onboarding/evaluation, PO, invoice, material approvals
   - Contract terms and payment milestones

10. **Finance & Budget Service**
   - Budget allocation, cost tracking, revenue forecast, margin analysis
   - Multi-currency, tax rules, payroll/timesheet interfaces
   - PostgreSQL-backed transactional integrity

11. **HR & Workforce Service**
   - Onboarding, certifications, skill matrix, attendance, overtime, performance

12. **Compliance & Audit Service**
   - Immutable audit trail, access/IP history, retention policy enforcement
   - ISO/GDPR evidence records and traceability

13. **Collaboration & Communication Service**
   - Real-time chat (Socket.io), channels, announcements, meetings
   - Notification orchestration: email/SMS/in-app/push

14. **Search Service**
   - Elasticsearch indexing and query APIs
   - Unified search by project/drawing/revision/vendor

15. **Analytics & AI Insights Service**
   - KPI dashboards (CEO/PD/Finance/HR/BIM)
   - Delay prediction, cost overrun alerts, resource overload, AI risk scoring

### Platform Infrastructure Services
- API Gateway
- BFF Service (web/mobile)
- Config Service
- Secrets/KMS integration adapter
- Logging/Observability service
- Workflow/Job service

---

## 4) Event-Driven Design

### Event Backbone Patterns
- **Kafka topics** for domain events and analytics fan-out:
  - `project.lifecycle.updated`
  - `bim.file.versioned`
  - `approval.status.changed`
  - `finance.invoice.posted`
  - `site.ncr.created`
  - `audit.event.logged`
- **RabbitMQ queues** for command-style async jobs:
  - file conversion, report generation, notification delivery, OCR, PDF stamping

### Reliability Patterns
- Outbox pattern for guaranteed event publication
- Idempotency keys for commands (upload chunk, approval actions, payment posting)
- Dead-letter queues + replay tooling
- Saga orchestration for cross-service processes (e.g., contract award → budget baseline → vendor package initiation)

---

## 5) API Structure (Enterprise Standard)

### North-South APIs (Gateway)
- `/api/v1/auth/*`
- `/api/v1/tenants/*`
- `/api/v1/projects/*`
- `/api/v1/bim/*`
- `/api/v1/mep/*`
- `/api/v1/site/*`
- `/api/v1/procurement/*`
- `/api/v1/finance/*`
- `/api/v1/hr/*`
- `/api/v1/compliance/*`
- `/api/v1/collab/*`
- `/api/v1/search/*`
- `/api/v1/analytics/*`

### API Governance
- OpenAPI/Swagger per service + gateway contract catalog
- Versioned APIs + backward compatibility policy
- Pagination + cursor APIs for large datasets
- Correlation IDs propagated (`x-correlation-id`)
- Tenant context headers (`x-tenant-id`, validated against token claims)

---

## 6) Authorization Model (RBAC + ABAC + Matrix)

### RBAC Role Groups
- Organization: Super Enterprise Admin, Global Director, Compliance Officer
- Project: Project Director, BIM Manager, MEP Lead, HVAC Lead, Structural Lead, Site Coordinator
- Engineering: Design Engineer, Draftsman, Modeler, QA Engineer, Planner
- External: Client Viewer, Consultant, Vendor, Contractor

### ABAC Attributes
- User attributes: role, discipline, clearance, certifications, employment status
- Resource attributes: project stage, discipline, document sensitivity, ownership
- Context attributes: location, IP risk, device posture, time window

### Permission Matrix Model
- `module.action.scope` pattern (e.g., `bim.document.approve.project`, `finance.invoice.post.branch`)
- Policy evaluation engine:
  1. Validate tenant boundary
  2. Evaluate deny overrides
  3. Evaluate RBAC grants
  4. Evaluate ABAC constraints
  5. Emit policy decision audit

---

## 7) MongoDB Schema Design (Primary Document Store)

### Key Collections
1. `tenants`
   - `_id`, `tenantId`, `name`, `status`, `countryProfiles[]`, `kmsKeyRef`, `createdAt`
2. `organizations`
   - `_id`, `tenantId`, `branchId`, `departmentId`, `name`, `parentRef`, `metadata`
3. `users`
   - `_id`, `tenantId`, `employeeCode`, `profile`, `roles[]`, `attributes`, `mfa`, `status`
4. `projects`
   - `_id`, `tenantId`, `projectId`, `name`, `stage`, `wbs[]`, `costBreakdown[]`, `riskMatrix[]`, `kpis`
5. `documents`
   - `_id`, `tenantId`, `projectId`, `docNumber`, `docType`, `discipline`, `currentRevision`, `lockState`, `tags[]`
6. `document_versions`
   - `_id`, `tenantId`, `projectId`, `documentId`, `revision`, `s3Key`, `checksum`, `uploadedBy`, `approvedBy`
7. `clashes`
   - `_id`, `tenantId`, `projectId`, `modelRefs[]`, `severity`, `status`, `assignee`, `dueDate`
8. `site_logs`
   - `_id`, `tenantId`, `projectId`, `logType`, `geo`, `attachments[]`, `createdBy`, `createdAt`
9. `rfis`
   - `_id`, `tenantId`, `projectId`, `discipline`, `question`, `status`, `slaDue`, `responses[]`
10. `audit_events`
   - `_id`, `tenantId`, `actor`, `action`, `resource`, `before`, `after`, `ip`, `timestamp`, `hashChainRef`

### MongoDB Patterns
- Soft delete with `deletedAt` + archival pipelines
- Document versioning split into base (`documents`) + immutable versions (`document_versions`)
- Bucket strategy for high-frequency telemetry/log data

---

## 8) PostgreSQL Schema Domains (Finance-Critical)

### Core Tables
- `ledger_accounts`
- `cost_centers`
- `budgets`
- `purchase_orders`
- `invoices`
- `payment_milestones`
- `tax_rules`
- `fx_rates`
- `timesheet_entries`
- `payroll_runs`

### Finance Controls
- ACID transactions for posting and settlements
- Double-entry ledger postings
- Immutable journal entries with reversal-only corrections
- Fiscal period lock controls and maker-checker approvals

---

## 9) Indexing Strategy

### MongoDB Indexes
- Compound tenant-first indexes:
  - `projects: { tenantId: 1, projectId: 1 }`
  - `documents: { tenantId: 1, projectId: 1, docNumber: 1 }` (unique per project)
  - `document_versions: { tenantId: 1, documentId: 1, revision: -1 }`
  - `audit_events: { tenantId: 1, timestamp: -1 }`
- TTL indexes for temporary operational data (upload sessions, ephemeral tokens)

### PostgreSQL Indexes
- B-tree on `(tenant_id, project_id, status)` for operational filters
- Partial indexes for open invoices / pending approvals
- BRIN for very large time-series finance/audit tables

### Elasticsearch Mapping
- Indices:
  - `tenant_{id}_documents`
  - `tenant_{id}_drawings`
  - `tenant_{id}_vendors`
- Keyword fields for exact drawing/revision/vendor lookup
- Text analyzers for full-text (technical terms, multilingual support)

---

## 10) Backend Folder Structure (Node.js + Express)

```text
services/
  api-gateway/
  bff-web/
  iam-service/
  tenant-service/
  project-service/
  bim-service/
  file-service/
  clash-service/
  mep-hvac-service/
  site-service/
  procurement-service/
  finance-service/
  hr-service/
  compliance-service/
  collab-service/
  search-service/
  analytics-service/

packages/
  shared-kernel/
    src/
      types/
      errors/
      logger/
      auth/
      events/
      observability/
  config/
  eslint-config/
  tsconfig/

infra/
  docker/
  k8s/
  helm/
  terraform/

docs/
  adr/
  api/
  security/
```

### Service Internal Structure
```text
src/
  app.ts
  routes/
  controllers/
  services/
  repositories/
  domain/
  policies/
  events/
  jobs/
  middleware/
  validators/
  utils/
  config/
  tests/
```

---

## 11) Frontend Architecture (React + TypeScript)

```text
apps/web/
  src/
    app/
      routes/
      layouts/
      providers/
    modules/
      dashboard/
      projects/
      bim/
      mep/
      site/
      procurement/
      finance/
      hr/
      compliance/
      collab/
      analytics/
    components/
      ui/            # ShadCN components
      datagrid/      # 50+ column enterprise grid
      gantt/
      kanban/
      filters/
      charts/
    store/           # Redux Toolkit slices
    api/             # React Query clients
    hooks/
    lib/
    theme/
```

### UI/UX Requirements Realization
- Desktop-first enterprise layout with mega sidebar + contextual top nav
- Role-aware menu rendering from permissions API
- Theme engine for dark/light mode with tenant branding
- Gantt + Kanban + dense data grids for project control rooms

---

## 12) Security Architecture

- OAuth2 + OIDC SSO with enterprise IdPs (Azure AD/Okta/Keycloak)
- JWT access tokens (short-lived) + rotating refresh tokens
- MFA enforcement policies by role/risk
- End-to-end encryption in transit (TLS 1.3) and at rest (KMS-backed)
- Object storage server-side encryption + optional client-side encryption for highly sensitive models
- WAF + API rate limiting + bot detection
- Fine-grained API guards with centralized policy decision point (PDP)
- Full audit trail with tamper-evident hash-chain references
- GDPR controls: consent, right-to-access, right-to-erasure workflows, retention policies

---

## 13) Docker, Kubernetes, and Deployment Strategy

### Docker Baseline
- One Dockerfile per service (multi-stage build)
- Distroless runtime images where possible
- SBOM generation + image signing in CI

### Kubernetes
- Namespaces per environment (`dev`, `staging`, `prod`)
- Helm charts/Kustomize overlays
- HPA on CPU/memory/custom metrics (queue lag, p95 latency)
- StatefulSets for data systems where self-managed
- Ingress via NGINX + cert-manager TLS

### CI/CD
- Pipeline stages:
  1. Lint + unit tests
  2. SAST + dependency scanning
  3. Build + sign images
  4. Integration tests
  5. Deploy to staging
  6. E2E smoke + security gates
  7. Progressive prod rollout (canary/blue-green)

---

## 14) Scalability Strategy

### Horizontal Scaling
- Stateless microservices scaled by request rate and queue depth
- WebSocket/chat scaled with Redis adapter and sticky-session strategy at ingress
- Read replicas for MongoDB/PostgreSQL

### Data Scaling
- MongoDB sharding by `tenantId` + workload-aware secondary shard dimensions
- Elasticsearch hot-warm architecture for cost/performance
- Tiered object storage lifecycle (hot → cool → archive)

### Performance Patterns
- Redis caching for:
  - permission snapshots
  - dashboard aggregates
  - frequently accessed project/document metadata
- Async processing for heavy tasks (model conversion, report generation)
- CDN acceleration for static assets and controlled document previews

---

## 15) Observability, Audit, and Compliance

- Central logging pipeline: Fluent Bit → Elasticsearch/OpenSearch
- Metrics: Prometheus + Grafana with SLOs (latency, error rate, saturation)
- Tracing: OpenTelemetry (distributed trace across gateway/services/jobs)
- Audit event standards with immutable retention and legal hold
- Compliance dashboards for ISO workflows, NCR trends, access anomalies

---

## 16) Testing & Quality Strategy

- Unit tests per service (domain logic, policy engine, repository behavior)
- Integration tests with ephemeral MongoDB/PostgreSQL/Redis/Kafka containers
- Contract tests between API Gateway and microservices
- Load testing scenarios:
  - 10k concurrent sessions
  - burst uploads of large BIM files
  - high-frequency audit streams
- Security test plan:
  - penetration testing
  - token/session abuse tests
  - privilege escalation tests
  - multi-tenant boundary tests

---

## 17) Reference API Examples

### BIM Document Check-In
- `POST /api/v1/bim/documents/{id}/checkout`
- `POST /api/v1/bim/uploads/initiate`
- `POST /api/v1/bim/uploads/{sessionId}/chunk`
- `POST /api/v1/bim/uploads/{sessionId}/complete`
- `POST /api/v1/bim/documents/{id}/checkin`

### RFI Flow
- `POST /api/v1/mep/rfis`
- `PATCH /api/v1/mep/rfis/{id}/assign`
- `PATCH /api/v1/mep/rfis/{id}/respond`
- `PATCH /api/v1/mep/rfis/{id}/close`

### Finance Posting
- `POST /api/v1/finance/invoices`
- `POST /api/v1/finance/invoices/{id}/approve`
- `POST /api/v1/finance/invoices/{id}/post-ledger`

---

## 18) Enterprise Rollout Roadmap

### Phase 1 (Foundation)
- IAM, Tenant, Project, BIM docs, basic search, core audit

### Phase 2 (Operational Excellence)
- MEP/HVAC, Site ops, Procurement, Finance core, analytics MVP

### Phase 3 (Advanced Intelligence)
- AI risk engine, predictive delays, advanced optimization, FM handover digital twin integrations

---

## 19) Why This Is Enterprise-Grade (Not CRUD)
- Domain-driven service boundaries aligned with EPC/BIM workflows
- Multi-tenant isolation + compliance-grade audit architecture
- Mixed transactional/document/search storage aligned with workload semantics
- Async, resilient processing for heavy engineering artifacts
- Governance and security controls suitable for billion-dollar capital programs
