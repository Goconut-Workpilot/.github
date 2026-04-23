# API Contracts

## Purpose

This document defines the long-term API contracts between the main platform layers:

- UI ↔ BFF
- BFF ↔ Pipeline Service
- Pipeline Service ↔ MCP Service

The goal is to keep communication boundaries explicit, stable, and aligned with service responsibilities.

---

## A. UI ↔ BFF API

This is the frontend-facing API.

The BFF exists to provide UI-oriented endpoints for run submission, status retrieval, artifact access, aggregated views, and human review actions.

### Core endpoints

```text
POST   /consulting-runs
GET    /consulting-runs/:runId
GET    /consulting-runs/:runId/view
GET    /consulting-runs/:runId/artifacts
GET    /consulting-runs/:runId/result
POST   /consulting-runs/:runId/review-actions
```

### Endpoint intent

#### POST /consulting-runs

Creates a new consulting run.

##### Request

```json
{
  "tenantId": "tenant-123",
  "buildingIds": ["building-a", "building-b"],
  "floorIds": ["floor-1"],
  "dateRange": {
    "from": "2026-01-01T00:00:00Z",
    "to": "2026-01-31T23:59:59Z"
  },
  "surveySource": {
    "type": "google-drive",
    "pathOrId": "drive-file-id"
  },
  "outputMode": {
    "persistArtifacts": true,
    "returnIntermediateOutputs": false
  },
  "options": {
    "reviewRequired": true
  }
}
```

##### Response

```json
{
  "runId": "run_01HXYZ...",
  "status": "queued"
}
```

---

#### GET /consulting-runs/:runId

Returns core run status.

##### Response

```json
{
  "runId": "run_01HXYZ...",
  "status": "running",
  "startedAt": "2026-04-23T10:30:00Z",
  "finishedAt": null,
  "currentNode": "surveyClassification",
  "progress": {
    "completedNodes": 2,
    "totalNodes": 6,
    "percent": 33
  },
  "errors": []
}
```

---

#### GET /consulting-runs/:runId/view

Returns a UI-friendly aggregated response.

##### Response

```json
{
  "run": {
    "runId": "run_01HXYZ...",
    "status": "blocked_for_review"
  },
  "stages": [
    {
      "nodeId": "connector",
      "status": "succeeded"
    },
    {
      "nodeId": "surveyClassification",
      "status": "succeeded"
    },
    {
      "nodeId": "policyGeneration",
      "status": "blocked"
    }
  ],
  "summary": {
    "headline": "Policy draft ready for review",
    "notes": [
      "Connector data loaded successfully",
      "Survey classification completed"
    ]
  },
  "reviewState": {
    "required": true,
    "pendingAction": "approve_policy"
  },
  "artifactLinks": [
    {
      "artifactId": "artifact_policy_01",
      "name": "policy.yml",
      "url": "/consulting-runs/run_01HXYZ.../artifacts/artifact_policy_01"
    }
  ]
}
```

---

#### GET /consulting-runs/:runId/artifacts

Returns artifact references available to the UI.

##### Response

```json
{
  "runId": "run_01HXYZ...",
  "artifacts": [
    {
      "artifactId": "artifact_policy_01",
      "stageName": "policyGeneration",
      "artifactType": "policy_yaml",
      "fileName": "policy.yml"
    },
    {
      "artifactId": "artifact_result_01",
      "stageName": "consultant",
      "artifactType": "consulting_result",
      "fileName": "recommendations.json"
    }
  ]
}
```

---

#### GET /consulting-runs/:runId/result

Returns the final consulting result when available.

##### Response

```json
{
  "runId": "run_01HXYZ...",
  "status": "succeeded",
  "result": {
    "findings": [
      "Too many focus desks relative to observed usage"
    ],
    "recommendations": [
      "Replace underused focus desks with phone booths"
    ],
    "executiveSummary": "The current floor configuration is misaligned with observed and reported workplace behavior."
  }
}
```

---

#### POST /consulting-runs/:runId/review-actions

Allows human review actions.

##### Request

```json
{
  "action": "approve_policy",
  "payload": {
    "comment": "Policy approved by consultant"
  }
}
```

##### Allowed actions

```text
approve_policy
override_policy
finalize_consulting
request_regeneration
```

##### Generic response

```json
{
  "runId": "run_01HXYZ...",
  "status": "running",
  "accepted": true
}
```

---

## B. BFF ↔ Pipeline Service API

This API is internal between the BFF and the Pipeline Service.

It may resemble the UI-facing contract, but it should remain domain-oriented rather than presentation-oriented.

### Core endpoints

```text
POST   /pipeline-runs
GET    /pipeline-runs/:runId
GET    /pipeline-runs/:runId/nodes
GET    /pipeline-runs/:runId/artifacts
GET    /pipeline-runs/:runId/result
POST   /pipeline-runs/:runId/review-actions
POST   /pipeline-runs/:runId/cancel
POST   /pipeline-runs/:runId/retry
```

### Endpoint intent

#### POST /pipeline-runs

Creates a new pipeline run.

##### Request

```json
{
  "tenantId": "tenant-123",
  "buildingIds": ["building-a"],
  "floorIds": ["floor-1"],
  "dateRange": {
    "from": "2026-01-01T00:00:00Z",
    "to": "2026-01-31T23:59:59Z"
  },
  "surveySource": {
    "type": "google-drive",
    "pathOrId": "drive-file-id"
  },
  "outputMode": {
    "persistArtifacts": true,
    "returnIntermediateOutputs": false
  },
  "options": {
    "reviewRequired": true
  }
}
```

##### Response

```json
{
  "runId": "run_01HXYZ...",
  "status": "queued"
}
```

---

#### GET /pipeline-runs/:runId

Returns core run lifecycle status.

##### Response

```json
{
  "runId": "run_01HXYZ...",
  "status": "running",
  "startedAt": "2026-04-23T10:30:00Z",
  "finishedAt": null,
  "currentNode": "policyGeneration",
  "errors": []
}
```

### Run status model

```text
queued
running
blocked_for_review
succeeded
failed
cancelled
superseded
```

---

#### GET /pipeline-runs/:runId/nodes

Returns node-level execution state.

##### Response

```json
{
  "runId": "run_01HXYZ...",
  "nodes": [
    {
      "nodeId": "connector",
      "status": "succeeded",
      "startedAt": "2026-04-23T10:30:00Z",
      "finishedAt": "2026-04-23T10:30:10Z"
    },
    {
      "nodeId": "surveyClassification",
      "status": "running",
      "startedAt": "2026-04-23T10:30:11Z",
      "finishedAt": null
    }
  ]
}
```

### Node status model

```text
pending
running
succeeded
failed
skipped
blocked
cached
```

---

#### GET /pipeline-runs/:runId/artifacts

Returns artifact references generated by the workflow.

##### Response

```json
{
  "runId": "run_01HXYZ...",
  "artifacts": [
    {
      "artifactId": "artifact_connector_01",
      "stageName": "connector",
      "artifactType": "connector_output"
    },
    {
      "artifactId": "artifact_policy_01",
      "stageName": "policyGeneration",
      "artifactType": "policy_yaml"
    }
  ]
}
```

---

#### GET /pipeline-runs/:runId/result

Returns the final domain result.

##### Response

```json
{
  "runId": "run_01HXYZ...",
  "status": "succeeded",
  "result": {
    "findings": [
      "Observed usage does not support current desk allocation"
    ],
    "recommendations": [
      "Reallocate underused desk area to collaborative and phone spaces"
    ],
    "executiveSummary": "The workplace configuration should be adjusted to better fit observed usage and survey-driven profiles."
  }
}
```

---

#### POST /pipeline-runs/:runId/review-actions

Accepts workflow-level review actions.

##### Request

```json
{
  "action": "override_policy",
  "payload": {
    "overrides": {
      "focus_workplace_ratio": 0.3
    },
    "comment": "Adjusted after consultant review"
  }
}
```

---

#### POST /pipeline-runs/:runId/cancel

Cancels an active run when possible.

##### Response

```json
{
  "runId": "run_01HXYZ...",
  "status": "cancelled"
}
```

---

#### POST /pipeline-runs/:runId/retry

Retries a failed or retryable run or node.

##### Response

```json
{
  "runId": "run_01HXYZ...",
  "status": "queued"
}
```

---

## C. Pipeline Service ↔ MCP Service API

This is a service-to-service API and must remain hidden from the UI.

### Design rule

Expose domain-oriented integration endpoints, not GraphQL leakage.

The Pipeline Service should request workplace facts needed for orchestration, while the MCP Service owns external-system transport, authentication, and retrieval mechanics.

### Preferred contract

```text
POST /workplace-snapshot
```

### Endpoint intent

#### POST /workplace-snapshot

Returns a tenant-scoped workplace snapshot for pipeline connector usage.

##### Request

```json
{
  "tenantId": "tenant-123",
  "buildingIds": ["building-a"],
  "floorIds": ["floor-1"],
  "dateRange": {
    "from": "2026-01-01T00:00:00Z",
    "to": "2026-01-31T23:59:59Z"
  },
  "requestedScopes": ["buildings", "bookings", "occupancy"]
}
```

##### Response

```json
{
  "buildings": [
    {
      "id": "building-a",
      "name": "Berlin HQ"
    }
  ],
  "bookings": [
    {
      "id": "booking-1",
      "buildingId": "building-a",
      "floorId": "floor-1",
      "date": "2026-01-15",
      "type": "desk"
    }
  ],
  "occupancySummary": {
    "averageOccupancy": 0.42,
    "peakDays": ["Tuesday", "Wednesday", "Thursday"]
  },
  "sourceMetadata": {
    "sourceSystem": "goconut",
    "tenantId": "tenant-123"
  },
  "fetchedAt": "2026-04-23T10:30:00Z"
}
```

### Error shape

All integration-side errors should use a stable service error format.

```json
{
  "code": "UPSTREAM_TIMEOUT",
  "message": "Timed out while retrieving workplace snapshot",
  "retryable": true,
  "details": {
    "scope": "occupancy"
  }
}
```

### Contract rule

This contract keeps:
- integration mechanics behind the MCP boundary
- transport/auth details out of the Pipeline Service
- the connector stage focused on normalized, usable workplace data

---

## Contract Principles

### 1. Clear ownership
Each API belongs to a specific boundary and should reflect that layer’s responsibility.

### 2. Domain-oriented contracts
The Pipeline Service and MCP Service should exchange domain concepts, not raw transport details.

### 3. Validation at boundaries
All requests and responses should be runtime-validated.

### 4. Stable DTOs
Contracts should evolve through explicit schema changes rather than accidental drift.

### 5. UI-friendly vs domain-friendly separation
The UI-facing API can shape data for frontend needs, while the Pipeline and MCP contracts should remain domain-oriented.

---

## Summary

This document defines three major contract layers:

- **UI ↔ BFF** for frontend-facing operations
- **BFF ↔ Pipeline Service** for workflow and run management
- **Pipeline Service ↔ MCP Service** for deterministic workplace data retrieval

Together, these contracts support a layered platform with explicit ownership, stable communication boundaries, and long-term maintainability.
