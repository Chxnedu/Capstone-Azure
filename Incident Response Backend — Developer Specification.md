# Incident Response Backend
## Backend Developer Specification

**Project:** Cloud-Native Disaster Recovery Platform  
**Component:** Incident Response Backend  
**Primary Consumer:** Disaster Recovery Frontend  
**Primary Integration:** GitHub Actions  
**Technology:** Node.js + TypeScript + Express

---

# 1. Purpose

Build a small standalone **Incident Response Backend** that provides the secure bridge between the human approval interface and the GitHub Actions recovery workflow.

The backend's responsibility is intentionally narrow:

```text
Frontend
   ↓
Human clicks APPROVE
   ↓
Incident Response Backend
   ↓
GitHub Actions
   ↓
Recovery Workflow
   ↓
Terraform / Recovery Plan
   ↓
Slot Swap
   ↓
Health Verification
```

The backend must **not** become a general disaster-recovery control plane.

The overall capstone flow remains:

```text
FAILURE
   ↓
DETECTION
   ↓
AI ANALYSIS
   ↓
HUMAN APPROVAL
   ↓
GITHUB ACTIONS
   ↓
TERRAFORM / RECOVERY PLAN
   ↓
SLOT SWAP
   ↓
HEALTH CHECK
   ↓
SUCCESSFUL PAYMENT
```

This is consistent with the project's intended recovery sequence.

---

# 2. Architectural Position

The Incident Response Backend is a **separate service** from the Payment API.

Do not merge its code into the payment application backend.

```text
                     ┌──────────────────────┐
                     │   Banking Frontend   │
                     └──────────┬───────────┘
                                │
                   GET incident/recommendation
                                │
                                ▼
                     ┌──────────────────────┐
                     │ Incident Response    │
                     │ Backend              │
                     │                      │
                     │ POST /api/recovery/  │
                     │       trigger        │
                     │                      │
                     │ GET /api/recovery/   │
                     │       status         │
                     └──────────┬───────────┘
                                │
                         GitHub API
                                │
                                ▼
                     ┌──────────────────────┐
                     │   GitHub Actions     │
                     │ Recovery Workflow    │
                     └──────────┬───────────┘
                                │
                                ▼
                     ┌──────────────────────┐
                     │ Terraform / Recovery │
                     │ Plan                 │
                     └──────────┬───────────┘
                                │
                                ▼
                     ┌──────────────────────┐
                     │ Azure App Service    │
                     │ Recovery Slot        │
                     └──────────────────────┘
```

The architecture explicitly keeps AI advisory and requires human approval before recovery execution.

---

# 3. Scope

## 3.1 Backend MUST

The backend must:

- expose a recovery trigger endpoint;
- expose a recovery status endpoint;
- authenticate with GitHub using a server-side token;
- trigger the configured GitHub Actions recovery workflow;
- prevent the frontend from directly interacting with GitHub;
- keep GitHub configuration server-side;
- track the recovery workflow currently triggered by the application;
- query GitHub for workflow status;
- normalize GitHub workflow states into simple application states;
- return useful errors to the frontend;
- never expose GitHub credentials.

## 3.2 Backend MUST NOT

Do **not** implement:

- payment processing;
- Azure infrastructure management;
- Terraform execution;
- Azure CLI execution;
- AI analysis;
- telemetry querying;
- incident detection;
- a PostgreSQL database;
- dynamic Terraform generation;
- arbitrary GitHub workflow execution;
- arbitrary repository/workflow selection from frontend input;
- direct Azure API calls for infrastructure recovery.

The recovery infrastructure remains the responsibility of GitHub Actions/Terraform. The architecture specifically states that AI must not directly modify infrastructure or trigger recovery.

---

# 4. Technology Requirements

Use:

```text
Node.js
TypeScript
Express
```

Recommended supporting libraries:

```text
express
dotenv
cors
```

For GitHub integration, use either:

- Octokit; or
- a small HTTP client around the GitHub REST API.

Prefer Octokit if the team is already comfortable with it.

---

# 5. API Contract

The backend exposes only two required application endpoints.

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/recovery/trigger` | POST | Trigger the configured recovery workflow |
| `/api/recovery/status` | GET | Return the current recovery workflow status |

No request body is required for the trigger endpoint.

The frontend must **not** send repository names, workflow IDs, branches, tokens, recovery-plan names, or other GitHub configuration.

---

# 6. `POST /api/recovery/trigger`

## Purpose

Called when the human operator clicks:

```text
APPROVE RECOVERY
```

The frontend calls:

```http
POST /api/recovery/trigger
```

with no request body.

The backend then triggers the configured GitHub Actions recovery workflow.

---

## Request

```http
POST /api/recovery/trigger
Content-Type: application/json
```

Body:

```json
{}
```

An empty body is acceptable, but the frontend does not need to send one.

---

## Successful Response

Recommended:

```http
200 OK
```

```json
{
  "status": "TRIGGERED",
  "message": "Recovery workflow triggered successfully"
}
```

If the backend successfully obtains a workflow run ID, it should also return it:

```json
{
  "status": "TRIGGERED",
  "message": "Recovery workflow triggered successfully",
  "runId": 123456789,
  "workflowName": "Disaster Recovery"
}
```

The `runId` is optional because GitHub's workflow-dispatch mechanism may require an additional lookup before the newly created run can be identified.

---

# 7. Trigger Behaviour

The backend should execute this sequence:

```text
POST /api/recovery/trigger
          ↓
Validate backend configuration
          ↓
Authenticate with GitHub
          ↓
Dispatch configured workflow
          ↓
Identify/correlate workflow run
          ↓
Store current run information
          ↓
Return TRIGGERED
```

The frontend should never know the GitHub API details.

---

# 8. GitHub Configuration

The GitHub workflow does **not yet exist**, so do not hardcode workflow-specific values.

Everything that may change when Infrastructure/DevOps creates the workflow must be configurable.

Recommended environment variables:

```env
GITHUB_TOKEN=
GITHUB_OWNER=
GITHUB_REPOSITORY=
GITHUB_WORKFLOW_ID=
GITHUB_REF=
```

Optional:

```env
GITHUB_API_BASE_URL=https://api.github.com
```

The backend code should work without modification once the DevOps team supplies the actual values.

For example:

```env
GITHUB_OWNER=example-org
GITHUB_REPOSITORY=disaster-recovery-platform
GITHUB_WORKFLOW_ID=disaster-recovery.yml
GITHUB_REF=main
```

These are examples only.

**Do not put these example values into the application as defaults.**

---

# 9. GitHub Token Security

`GITHUB_TOKEN` must:

- exist only on the backend;
- never be sent to the frontend;
- never be included in API responses;
- never be printed in logs;
- never be committed to Git;
- never appear in frontend environment variables.

For Azure deployment, store the token in the **Azure App Service application settings / secret configuration** for the Incident Response Backend.

The frontend must have no GitHub credentials.

The architecture explicitly requires that the frontend not contain Azure credentials and that the GitHub Actions trigger be protected.

---

# 10. Workflow Selection

For the current capstone there is only **one recovery workflow**.

Therefore:

```text
POST /api/recovery/trigger
```

is intentionally sufficient.

Do not expose something like:

```text
POST /api/recovery/trigger/:workflow
```

or:

```json
{
  "workflow": "some-workflow"
}
```

The frontend should not be able to select an arbitrary workflow.

The backend selects the configured recovery workflow.

---

# 11. Recovery Status API

## `GET /api/recovery/status`

The frontend uses this endpoint to determine what is happening after approval.

Example:

```http
GET /api/recovery/status
```

Successful response:

```json
{
  "status": "IN_PROGRESS",
  "runId": 123456789,
  "workflowName": "Disaster Recovery",
  "updatedAt": "2026-09-16T18:30:00Z"
}
```

---

# 12. Application-Level Status Values

Do not expose raw GitHub status values directly to the frontend.

Normalize them into:

```text
NOT_STARTED
QUEUED
IN_PROGRESS
SUCCESS
FAILED
CANCELLED
UNKNOWN
```

The frontend can then reliably map these states to UI.

Suggested interpretation:

| Backend Status | Meaning |
|---|---|
| `NOT_STARTED` | No recovery has been triggered |
| `QUEUED` | GitHub has accepted the workflow but execution has not started |
| `IN_PROGRESS` | Recovery workflow is running |
| `SUCCESS` | Recovery workflow completed successfully |
| `FAILED` | Recovery workflow failed |
| `CANCELLED` | Recovery workflow was cancelled |
| `UNKNOWN` | Status could not be determined |

---

# 13. Status Lifecycle

The expected lifecycle is:

```text
NOT_STARTED
     ↓
TRIGGERED
     ↓
QUEUED
     ↓
IN_PROGRESS
     ↓
SUCCESS
```

Failure path:

```text
IN_PROGRESS
     ↓
FAILED
     ↓
Frontend displays:
MANUAL INTERVENTION REQUIRED
```

Cancellation:

```text
IN_PROGRESS
     ↓
CANCELLED
```

The overall UI should correspond to the project's required states:

```text
PENDING APPROVAL
      ↓
RECOVERY IN PROGRESS
      ↓
VERIFYING
      ↓
RECOVERY SUCCESSFUL
```

or:

```text
RECOVERY FAILED
      ↓
MANUAL INTERVENTION REQUIRED
```

These states are explicitly identified in the implementation timeline.

---

# 14. Workflow Run Correlation

One implementation detail must be handled carefully.

Triggering a GitHub Actions workflow does not necessarily give the backend the newly created run ID directly.

Therefore the backend should:

1. Record the trigger timestamp.
2. Dispatch the configured workflow.
3. Query recent runs for the configured workflow.
4. Find the run associated with the new trigger.
5. Store that run ID.
6. Use that run ID for subsequent status requests.

Conceptually:

```text
Trigger time = T

Dispatch workflow
       ↓
Query recent workflow runs
       ↓
Find matching run created after T
       ↓
Store runId
       ↓
GET /api/recovery/status
       ↓
Query run status
```

For this four-day capstone, **in-memory state is acceptable**.

Example:

```ts
let currentRecoveryRun = {
  runId: undefined,
  triggeredAt: undefined,
};
```

This is intentionally a minimal implementation.

Do not introduce PostgreSQL solely for this.

The architecture deliberately removed the incident database from the simplified design.

---

# 15. Important Limitation of In-Memory State

Because the backend is stateless from a persistence perspective:

```text
Backend restart
      ↓
current run information lost
```

That is acceptable for the capstone MVP.

The backend can fall back to querying the latest configured workflow run if necessary.

A persistent recovery-history store is **out of scope** for this four-day implementation.

---

# 16. Error Handling

Use meaningful HTTP status codes.

### Configuration error

If required environment variables are missing:

```http
500 Internal Server Error
```

```json
{
  "status": "ERROR",
  "message": "Recovery service is not configured correctly"
}
```

Do not reveal which secret/configuration value is missing to the client if doing so creates unnecessary information exposure.

---

### GitHub authentication failure

```http
502 Bad Gateway
```

```json
{
  "status": "ERROR",
  "message": "Unable to authenticate with recovery service"
}
```

Do not return the GitHub error body or token.

---

### Workflow trigger failure

```http
502 Bad Gateway
```

```json
{
  "status": "ERROR",
  "message": "Unable to trigger recovery workflow"
}
```

---

### GitHub status lookup failure

```http
502 Bad Gateway
```

```json
{
  "status": "ERROR",
  "message": "Unable to retrieve recovery status"
}
```

---

# 17. Prevent Duplicate Recovery Triggers

The backend should avoid accidentally launching multiple recovery workflows from repeated button clicks.

Recommended behaviour:

```text
POST /trigger
      ↓
Is recovery already QUEUED/IN_PROGRESS?
      ↓
YES → reject duplicate trigger
      ↓
NO → trigger workflow
```

For example:

```http
409 Conflict
```

```json
{
  "status": "ALREADY_RUNNING",
  "message": "A recovery workflow is already in progress"
}
```

This protects the recovery pipeline from double-clicks or repeated frontend requests.

---

# 18. Frontend-to-Backend Security Boundary

The frontend communicates only with:

```text
Incident Response Backend
```

Never:

```text
Frontend → GitHub
```

Never:

```text
Frontend → Azure Resource Manager
```

Never:

```text
Frontend → Terraform
```

The intended boundary is:

```text
Human
  ↓
Frontend
  ↓
Incident Backend
  ↓
Authorized GitHub Workflow
  ↓
Recovery Infrastructure
```

---

# 19. CORS

Configure CORS to permit requests from the deployed frontend origin.

During development, localhost can be allowed.

Example configuration:

```env
FRONTEND_ORIGIN=
```

Then configure Express accordingly.

Do not use unrestricted CORS in the final deployment unless the team has a specific reason.

---

# 20. Logging

Log operational events such as:

```text
Recovery trigger requested
Recovery workflow dispatched
Recovery run identified
Recovery status queried
Recovery completed
Recovery failed
```

Example:

```text
INFO Recovery workflow trigger requested
INFO Recovery workflow dispatched
INFO Recovery run identified: 123456789
INFO Recovery status: IN_PROGRESS
INFO Recovery status: SUCCESS
```

Never log:

```text
GITHUB_TOKEN
Authorization headers
Secrets
```

---

# 21. Suggested Project Structure

```text
incident-backend/
│
├── src/
│   ├── server.ts
│   ├── app.ts
│   │
│   ├── routes/
│   │   └── recovery.routes.ts
│   │
│   ├── controllers/
│   │   └── recovery.controller.ts
│   │
│   ├── services/
│   │   └── github.service.ts
│   │
│   ├── config/
│   │   └── env.ts
│   │
│   ├── types/
│   │   └── recovery.types.ts
│   │
│   └── state/
│       └── recovery.state.ts
│
├── tests/
│   ├── recovery.trigger.test.ts
│   └── recovery.status.test.ts
│
├── .env.example
├── package.json
├── tsconfig.json
└── README.md
```

Keep the implementation simple.

---

# 22. Environment Configuration

`.env.example`:

```env
PORT=3000

FRONTEND_ORIGIN=

GITHUB_TOKEN=
GITHUB_OWNER=
GITHUB_REPOSITORY=
GITHUB_WORKFLOW_ID=
GITHUB_REF=
```

Do not commit `.env`.

Add it to `.gitignore`.

---

# 23. Azure App Service Deployment

The backend should be deployable as a normal Node.js application on Azure App Service.

The application must:

- listen on the configured `PORT`;
- bind appropriately for App Service;
- receive GitHub configuration through environment variables;
- receive the GitHub PAT through App Service secret configuration;
- not rely on local persistent filesystem state;
- not require PostgreSQL.

The backend itself should expose a simple operational health endpoint if useful for App Service monitoring:

```http
GET /health
```

Example:

```json
{
  "status": "healthy"
}
```

This is the **Incident Response Backend's health endpoint** and is separate from the Payment API's `/health` endpoint.

---

# 24. Testing Requirements

The backend developer must create tests for at least:

### Test 1 — Successful trigger

```text
POST /api/recovery/trigger
        ↓
GitHub mocked as successful
        ↓
200
TRIGGERED
```

### Test 2 — Missing configuration

```text
GITHUB_TOKEN missing
        ↓
Trigger
        ↓
500
```

### Test 3 — GitHub authentication failure

```text
GitHub returns 401/403
        ↓
502
```

### Test 4 — Workflow trigger failure

```text
GitHub dispatch fails
        ↓
502
```

### Test 5 — Status lookup

Mock:

```text
queued
in_progress
completed/success
completed/failure
cancelled
```

and verify they map correctly to:

```text
QUEUED
IN_PROGRESS
SUCCESS
FAILED
CANCELLED
```

### Test 6 — Duplicate trigger

```text
Recovery already IN_PROGRESS
        ↓
POST /trigger
        ↓
409
```

### Test 7 — Token protection

Verify that:

- token is never included in response;
- token is never included in error response;
- token is never logged.

---

# 25. Acceptance Criteria

The backend is complete when all of the following work:

- [ ] Node.js/TypeScript/Express application runs locally.
- [ ] GitHub configuration comes entirely from environment variables.
- [ ] GitHub PAT is server-side only.
- [ ] `POST /api/recovery/trigger` works without a request body.
- [ ] Trigger endpoint invokes the configured GitHub Actions workflow.
- [ ] Frontend does not communicate directly with GitHub.
- [ ] Backend can identify the triggered workflow run.
- [ ] `GET /api/recovery/status` returns recovery status.
- [ ] GitHub statuses are normalized to application statuses.
- [ ] Duplicate recovery triggers are prevented.
- [ ] GitHub failures are handled cleanly.
- [ ] Secrets never appear in logs or responses.
- [ ] Backend deploys successfully to Azure App Service.
- [ ] Backend can operate without PostgreSQL.
- [ ] Tests cover trigger, status, failure, and security cases.

---

# 26. Integration Handoff to DevOps

The backend developer should finish with a clear list of values that DevOps must provide.

```text
GITHUB_TOKEN
GITHUB_OWNER
GITHUB_REPOSITORY
GITHUB_WORKFLOW_ID
GITHUB_REF
```

The backend code should not need to change when these values become available.

The intended handoff is:

```text
Backend Developer
      ↓
Environment-variable contract
      ↓
Infrastructure / DevOps
      ↓
Actual GitHub workflow values
      ↓
Azure App Service configuration
      ↓
End-to-end integration
```

---

# 27. Integration Handoff to Frontend

Frontend developers only need to know:

```text
POST /api/recovery/trigger
GET  /api/recovery/status
```

They should never need:

```text
GitHub token
GitHub owner
GitHub repository
GitHub workflow ID
Azure credentials
Terraform information
```

The frontend implementation is specified in the companion frontend document.

---

# 28. Final Backend Flow

The complete backend responsibility is:

```text
                 HUMAN APPROVAL
                       │
                       ▼
            POST /api/recovery/trigger
                       │
                       ▼
             Incident Response API
                       │
                       ▼
                GitHub API
                       │
                       ▼
             GitHub Actions Run
                       │
                       ▼
             Recovery / Terraform
                       │
                       ▼
                Slot Swap
                       │
                       ▼
             Health Verification
                       │
                       ▼
             GET /api/recovery/status
                       │
                       ▼
                  Frontend
```

Keep this service small. Its job is to securely connect the human approval action to the predefined recovery workflow and report the workflow's progress.