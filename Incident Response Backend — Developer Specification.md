# Incident Response Backend — Developer Specification

## 1. Purpose

Build a small standalone **Incident Response Backend** using **JavaScript/TypeScript with Express**.

This backend sits between the frontend incident-review interface and the external systems involved in the disaster-recovery flow.

Its responsibilities are intentionally limited to:

1. Retrieving the current incident artifact from Azure Blob Storage.
2. Providing the incident data to the frontend.
3. Accepting an approved recovery request.
4. Triggering the configured GitHub Actions recovery workflow.
5. Exposing the current recovery workflow status to the frontend.

The backend is **not** responsible for:

- monitoring Azure resources;
- querying Application Insights or Log Analytics;
- calling Azure OpenAI;
- generating incident analysis;
- deciding whether recovery should happen;
- executing Terraform;
- directly modifying Azure infrastructure;
- storing a permanent incident database.

The architecture uses Logic Apps and Azure services for incident orchestration, while GitHub Actions performs the approved recovery operation.

---

# 2. Architectural Context

The upstream incident flow is:

```text
Payment API
     ↓
Application Insights
     ↓
Azure Monitor / Log Analytics
     ↓
Azure Monitor Alert
     ↓
Logic App
     ↓
Azure OpenAI
     ↓
Incident JSON
     ↓
Azure Blob Storage
     ↓
current.json
```

The Incident Response Backend then exposes that incident to the frontend:

```text
Azure Blob Storage
      ↓
GET /api/incidents/latest
      ↓
Incident Response Backend
      ↓
Frontend Incident Page
```

After human approval:

```text
Frontend
    ↓
POST /api/recovery/trigger
    ↓
Incident Response Backend
    ↓
GitHub Actions
    ↓
Recovery Workflow
```

The frontend then monitors:

```text
Frontend
    ↓
GET /api/recovery/status
    ↓
Incident Response Backend
    ↓
GitHub Actions
```

The original architecture specifies that the Logic App builds the incident context, sends it to Azure OpenAI, receives a structured recommendation, and publishes that recommendation for human review.

---

# 3. Current Incident Storage Model

For this capstone, use **one current incident artifact**.

Do not build incident history, a database, or an incident-management data model.

Azure Blob Storage should contain:

```text
incidents/
└── current.json
```

The Logic App is responsible for creating/updating this artifact when a new incident is detected and analyzed.

The Incident Response Backend is responsible for reading it.

The backend must **not** generate the incident itself.

---

# 4. Incident Artifact Contract

The backend should expect `current.json` to contain the structured incident recommendation.

Example:

```json
{
  "incidentId": "INC-001",
  "createdAt": "2026-09-16T18:30:00Z",
  "service": "payment-api",
  "severity": "HIGH",
  "errorRate": 18,
  "responseTime": 2.4,
  "summary": "Payment API is experiencing elevated failures.",
  "likelyCause": "The payment service is returning a high volume of HTTP 500 responses.",
  "confidence": 0.91,
  "impact": "Customers may be unable to complete payments.",
  "recommendedAction": "FAILOVER_PAYMENT_SERVICE",
  "status": "PENDING_APPROVAL"
}
```

The exact incident fields should remain aligned with the AI contract defined by the architecture:

```text
severity
summary
likelyCause
confidence
impact
recommendedAction
```

The architecture explicitly constrains AI recommendations to predefined recovery actions rather than allowing AI to generate infrastructure changes.

The additional metadata such as `incidentId`, `createdAt`, and `status` exists to allow the application to identify and display the current incident safely.

---

# 5. API Endpoints

The backend exposes exactly three application endpoints for this workflow.

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/incidents/latest` | Retrieve the current incident artifact |
| POST | `/api/recovery/trigger` | Trigger the approved GitHub recovery workflow |
| GET | `/api/recovery/status` | Return the current recovery workflow status |

---

# 6. Endpoint 1 — Get Current Incident

## `GET /api/incidents/latest`

### Purpose

Retrieve `current.json` from Azure Blob Storage and return the incident data to the frontend.

The frontend calls this endpoint when the Incident Review page loads.

### Request

No request body.

```http
GET /api/incidents/latest
```

### Successful Response

HTTP `200 OK`

```json
{
  "incidentId": "INC-001",
  "createdAt": "2026-09-16T18:30:00Z",
  "service": "payment-api",
  "severity": "HIGH",
  "errorRate": 18,
  "responseTime": 2.4,
  "summary": "Payment API is experiencing elevated failures.",
  "likelyCause": "The payment service is returning a high volume of HTTP 500 responses.",
  "confidence": 0.91,
  "impact": "Customers may be unable to complete payments.",
  "recommendedAction": "FAILOVER_PAYMENT_SERVICE",
  "status": "PENDING_APPROVAL"
}
```

The backend may return the artifact directly or validate/normalize it before returning it.

---

# 7. No Current Incident Exists

This case must be explicitly handled.

When the frontend loads the incident page, there may be **no `current.json` file in Blob Storage**.

This is not necessarily an application error.

It can simply mean:

```text
No incident has been detected yet.
```

The backend should return:

```http
404 Not Found
```

with a predictable response:

```json
{
  "status": "NO_ACTIVE_INCIDENT",
  "message": "No current incident is available."
}
```

The frontend should interpret this response as an **empty incident state**, not as a generic application failure.

The UI should display something similar to:

```text
┌─────────────────────────────────────────┐
│        DISASTER RECOVERY INCIDENT       │
│                                         │
│        No active incident                │
│                                         │
│   The system is currently operating     │
│   without a pending recovery incident.  │
│                                         │
│              [ REFRESH ]                │
└─────────────────────────────────────────┘
```

The page should not display Approve or Reject buttons when there is no incident.

The user should be able to refresh/retry the request.

### Important distinction

The backend should distinguish between:

### Case A — No incident

```text
Blob does not contain current.json
        ↓
404 NO_ACTIVE_INCIDENT
```

### Case B — Blob Storage failure

For example:

- authentication failure;
- permission failure;
- network/storage service failure;
- malformed storage configuration.

This should **not** be reported as `NO_ACTIVE_INCIDENT`.

Return an appropriate server error, for example:

```http
500 Internal Server Error
```

or another appropriate `5xx` response.

Example:

```json
{
  "status": "INCIDENT_RETRIEVAL_FAILED",
  "message": "Unable to retrieve the current incident."
}
```

The frontend should display a technical error state in this case.

This distinction is important because:

```text
No incident
≠
The incident system is broken
```

---

# 8. Blob Storage Configuration

Do not hardcode Blob Storage credentials or resource information.

Use environment variables / App Service configuration.

Suggested configuration:

```text
AZURE_STORAGE_CONTAINER
AZURE_STORAGE_BLOB_NAME
```

Example values:

```text
AZURE_STORAGE_CONTAINER=incidents
AZURE_STORAGE_BLOB_NAME=current.json
```

The storage account configuration should also be provided through secure application configuration.

Where practical, prefer Azure App Service Managed Identity for Blob Storage access rather than embedding long-lived storage credentials in application code.

The exact authentication implementation can be chosen by the backend developer based on the infrastructure team's provisioned resources.

The important requirement is:

> **Blob Storage credentials must never be exposed to the frontend.**

---

# 9. Endpoint 2 — Trigger Recovery

## `POST /api/recovery/trigger`

### Purpose

Trigger the configured GitHub Actions recovery workflow after the human reviewer approves the incident.

### Request

The endpoint must have **no request body**.

```http
POST /api/recovery/trigger
```

Do not require the frontend to send:

```json
{
  "incidentId": "...",
  "recommendedAction": "..."
}
```

The backend already has access to the current incident.

The recovery plan is predefined and the current capstone has only one recovery workflow.

### Behaviour

The backend should:

1. Verify that a current incident exists.
2. Verify that the current incident contains a valid/predefined recovery recommendation.
3. Ensure a recovery workflow is not already running.
4. Trigger the configured GitHub Actions workflow.
5. Record enough information to monitor the triggered run.
6. Return a successful trigger response.

Example:

```json
{
  "status": "TRIGGERED",
  "message": "Recovery workflow triggered successfully",
  "incidentId": "INC-001"
}
```

If the GitHub API provides enough information to identify the run, include it:

```json
{
  "status": "TRIGGERED",
  "message": "Recovery workflow triggered successfully",
  "incidentId": "INC-001",
  "runId": 123456789
}
```

---

# 10. Recovery Trigger Protection

The endpoint must not allow repeated clicks to launch multiple recovery workflows.

Implement basic duplicate-trigger protection.

For example:

```text
PENDING_APPROVAL
       ↓
     APPROVE
       ↓
RECOVERY TRIGGERED
       ↓
No additional recovery trigger allowed
```

If a recovery is already in progress, return an appropriate response such as:

```http
409 Conflict
```

```json
{
  "status": "RECOVERY_ALREADY_RUNNING",
  "message": "A recovery workflow is already in progress."
}
```

For the four-day capstone, an in-memory state is acceptable.

A persistent incident/recovery database is out of scope.

---

# 11. GitHub Configuration

Do not hardcode GitHub-specific values.

Use environment variables:

```text
GITHUB_TOKEN
GITHUB_OWNER
GITHUB_REPOSITORY
GITHUB_WORKFLOW_ID
GITHUB_REF
```

Optionally:

```text
GITHUB_API_BASE_URL
```

The GitHub Personal Access Token must be stored as a secure environment variable in Azure App Service configuration.

Never:

- commit the PAT;
- put it in frontend code;
- return it in an API response;
- log it;
- place it in source control.

The actual workflow name, repository, owner, and branch/ref must be configurable because the GitHub Actions workflow is being implemented separately.

---

# 12. Endpoint 3 — Recovery Status

## `GET /api/recovery/status`

### Purpose

Return the current status of the recovery workflow so that the frontend can display recovery progress.

Example:

```http
GET /api/recovery/status
```

Response:

```json
{
  "incidentId": "INC-001",
  "status": "IN_PROGRESS",
  "runId": 123456789,
  "workflowName": "Disaster Recovery",
  "updatedAt": "2026-09-16T18:32:00Z"
}
```

---

# 13. Normalized Recovery States

The backend should hide GitHub-specific state details from the frontend.

Normalize GitHub workflow states into:

```text
NOT_STARTED
QUEUED
IN_PROGRESS
SUCCESS
FAILED
CANCELLED
UNKNOWN
```

The frontend should consume these normalized values instead of implementing GitHub-specific logic.

Example:

```text
GitHub:
queued
   ↓
Backend:
QUEUED
```

```text
GitHub:
in_progress
   ↓
Backend:
IN_PROGRESS
```

```text
GitHub:
completed + success
   ↓
Backend:
SUCCESS
```

```text
GitHub:
completed + failure
   ↓
Backend:
FAILED
```

---

# 14. Workflow Run Identification

GitHub's workflow-dispatch operation may not directly provide the workflow run ID.

If necessary, the backend may:

1. Dispatch the workflow.
2. Query recent workflow runs.
3. Identify the newly created run using the configured workflow/ref and trigger timing.
4. Store the run ID in memory.
5. Use that run ID for subsequent status queries.

For this four-day capstone, storing the current run ID in memory is acceptable.

Persistent run tracking is out of scope.

---

# 15. Error Handling

The API should return predictable errors.

Examples:

### No incident

```http
404
```

```json
{
  "status": "NO_ACTIVE_INCIDENT",
  "message": "No current incident is available."
}
```

### Recovery already running

```http
409
```

```json
{
  "status": "RECOVERY_ALREADY_RUNNING",
  "message": "A recovery workflow is already in progress."
}
```

### GitHub trigger failure

```http
502
```

```json
{
  "status": "RECOVERY_TRIGGER_FAILED",
  "message": "Unable to trigger the recovery workflow."
}
```

### Incident retrieval failure

```http
500
```

```json
{
  "status": "INCIDENT_RETRIEVAL_FAILED",
  "message": "Unable to retrieve the current incident."
}
```

Do not expose internal credentials, stack traces, or sensitive infrastructure information in responses.

---

# 16. Backend Responsibilities

The backend owns this boundary:

```text
                 INCIDENT RESPONSE BACKEND
┌───────────────────────────────────────────────┐
│                                               │
│  Blob Storage                                  │
│       ↓                                        │
│  GET /api/incidents/latest                    │
│       ↓                                        │
│  Incident data                                 │
│                                               │
│  Approved request                              │
│       ↓                                        │
│  POST /api/recovery/trigger                    │
│       ↓                                        │
│  GitHub Actions                                │
│                                               │
│  GitHub workflow status                        │
│       ↓                                        │
│  GET /api/recovery/status                      │
│       ↓                                        │
│  Frontend                                      │
│                                               │
└───────────────────────────────────────────────┘
```

It does **not** own:

```text
Azure Monitor
Application Insights
Log Analytics
Azure OpenAI
Terraform execution
Azure infrastructure modification
```

---

# 17. Security Requirements

The backend must:

- keep GitHub credentials server-side;
- keep Blob Storage credentials server-side;
- never expose Azure credentials to the frontend;
- validate the incident artifact;
- reject unknown recovery actions;
- prevent duplicate recovery triggers;
- avoid logging secrets;
- configure CORS for the frontend origin;
- use HTTPS in deployed environments.

The AI remains advisory and does not receive infrastructure credentials or directly trigger infrastructure changes.

---

# 18. Suggested Project Structure

```text
incident-response-backend/
│
├── src/
│   ├── server.ts
│   ├── app.ts
│   │
│   ├── routes/
│   │   ├── incident.routes.ts
│   │   └── recovery.routes.ts
│   │
│   ├── controllers/
│   │   ├── incident.controller.ts
│   │   └── recovery.controller.ts
│   │
│   ├── services/
│   │   ├── blob.service.ts
│   │   ├── github.service.ts
│   │   └── recovery.service.ts
│   │
│   ├── types/
│   │   ├── incident.ts
│   │   └── recovery.ts
│   │
│   └── config/
│       └── env.ts
│
├── tests/
│   ├── incident.test.ts
│   ├── recovery-trigger.test.ts
│   └── recovery-status.test.ts
│
├── package.json
├── tsconfig.json
└── README.md
```

---

# 19. Testing Requirements

At minimum, test:

### Incident retrieval

- `current.json` exists → `200`.
- Correct incident JSON is returned.
- Blob does not contain `current.json` → `404 NO_ACTIVE_INCIDENT`.
- Blob access failure → appropriate `5xx`.
- Malformed incident JSON → appropriate error.

### Recovery trigger

- Valid current incident → workflow triggered.
- No incident → trigger rejected.
- Invalid/unknown recovery action → trigger rejected.
- Recovery already running → duplicate trigger rejected.
- GitHub API failure → appropriate error.

### Recovery status

- No recovery started → `NOT_STARTED`.
- Workflow queued → `QUEUED`.
- Workflow running → `IN_PROGRESS`.
- Workflow succeeds → `SUCCESS`.
- Workflow fails → `FAILED`.
- Workflow cancelled → `CANCELLED`.

---

# 20. Acceptance Criteria

The backend is complete when:

- [ ] Express/TypeScript application runs successfully.
- [ ] `GET /api/incidents/latest` retrieves `current.json` from Blob Storage.
- [ ] No incident is handled as `404 NO_ACTIVE_INCIDENT`.
- [ ] Blob Storage failures are distinguished from a missing incident.
- [ ] No Blob Storage credentials are exposed to the frontend.
- [ ] `POST /api/recovery/trigger` requires no request body.
- [ ] GitHub configuration is environment-based.
- [ ] GitHub PAT is never exposed or logged.
- [ ] Duplicate recovery triggers are prevented.
- [ ] `GET /api/recovery/status` returns normalized workflow status.
- [ ] GitHub workflow states are not exposed directly to the frontend.
- [ ] Backend tests cover the main success and failure cases.
- [ ] README documents environment variables, setup, API contracts, and local development.
- [ ] Frontend can consume all three endpoints without knowing that Blob Storage or GitHub exists.

---

# 21. Final Backend Data Flow

```text
                     ┌─────────────────┐
                     │   Logic App     │
                     └────────┬────────┘
                              │
                              ▼
                       Azure OpenAI
                              │
                              ▼
                       Incident JSON
                              │
                              ▼
                    ┌──────────────────┐
                    │ Azure Blob       │
                    │ current.json     │
                    └────────┬─────────┘
                             │
                             │ GET /api/incidents/latest
                             ▼
                 ┌─────────────────────────┐
                 │ Incident Response       │
                 │ Backend                 │
                 └───────────┬─────────────┘
                             │
                             ▼
                         Frontend
                             │
                         APPROVE
                             │
                             │ POST /api/recovery/trigger
                             ▼
                 ┌─────────────────────────┐
                 │ GitHub Actions          │
                 │ Recovery Workflow       │
                 └───────────┬─────────────┘
                             │
                             ▼
                          Recovery
                             │
                             │ GET /api/recovery/status
                             ▼
                         Frontend
```

This keeps the backend small while giving the frontend a single application-facing interface for both **incident retrieval** and **recovery control/status**.