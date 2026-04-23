# Tech Stack Decisions

## 1. Decision Summary

Long-term recommended stack:

- Web UI: Next.js + TypeScript
- BFF / UI Backend: NestJS + TypeScript
- Pipeline Service: TypeScript + Zod
- MCP Service: TypeScript
- Run Metadata Store: PostgreSQL
- Artifact Store: local filesystem in dev, object storage in long-term environments
- Observability: structured logs first, tracing/metrics later

## 2. Web UI

### Recommendation
Use Next.js.

### Why
- strong fit for modern web UI
- good ecosystem for authenticated application UIs
- good SSR/CSR flexibility
- natural TypeScript fit

## 3. BFF / UI Backend

### Recommendation
Use NestJS.

### Why
The BFF is not just a proxy. Long term it needs:
- auth/session mediation
- permission checks
- view-model shaping
- review/finalization endpoints
- frontend-oriented aggregation
- stable modular structure

NestJS is a strong fit for a long-lived UI-facing backend because it provides clear module boundaries and backend structure.

### Non-goal
The BFF must not become a second workflow orchestrator.

## 4. Pipeline Service

### Recommendation
Keep it in TypeScript.

### Why
- already aligned with the current architecture direction
- natural fit for Zod-driven contract validation
- good ecosystem coherence with BFF and UI
- reduced team context switching

## 5. MCP Service

### Recommendation
Keep it as a standalone TypeScript service unless a later team/infrastructure reason requires otherwise.

### Why
- shared contract tooling
- easier DTO/schema alignment
- lower operational complexity inside one ecosystem

## 6. Metadata Store

### Recommendation
PostgreSQL.

### Why
- strong fit for queryable run metadata
- reliable relational model for runs, statuses, review state, artifacts, and audit data

## 7. Artifact Storage

### Development
- local filesystem

### Long term
- S3-compatible object storage or cloud blob storage

### Why
Artifact payloads and generated files are better separated from relational run metadata.

## 8. Observability

### Initial
- structured JSON logs
- runId / traceId / tenantId / nodeId correlation

### Long term
- OpenTelemetry-compatible tracing
- metrics backend
- stage/node latency and retry metrics

## 9. Why Not Split Every Agent into Its Own Service

Do not create classifier-service, policy-service, and consultant-service too early.

Reasons:
- more deployables
- more contracts
- more failure modes
- more tracing complexity
- more latency
- unclear benefit early on

The MCP boundary clearly deserves its own service.
The other workflow nodes should stay inside the Pipeline Service until a strong scaling or ownership reason exists.

## 10. Recommended Production Shape

```text
Next.js UI
   -> NestJS BFF
      -> Pipeline Service
         -> MCP Service
            -> goconut / external systems

Pipeline Service -> PostgreSQL
Pipeline Service -> Artifact Store
```
