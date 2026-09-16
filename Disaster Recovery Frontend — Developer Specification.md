# Disaster Recovery Frontend
## Frontend Developer Specification

**Project:** Cloud-Native Disaster Recovery Platform  
**Component:** Banking Frontend + Incident Review + Recovery Status  
**Technology:** React/Vite or existing project frontend stack  
**Backend Dependencies:** Payment API + Incident Response Backend

---

# 1. Purpose

Update the frontend so the human operator can:

1. view the detected incident and AI recommendation;
2. review the proposed recovery;
3. explicitly approve or reject it;
4. trigger recovery through the Incident Response Backend;
5. monitor recovery progress;
6. see whether recovery succeeded or failed.

The frontend is the **human approval interface**.

The architecture requires human approval to remain a hard gate before infrastructure recovery.

---

# 2. Important Architectural Boundary

The frontend must **not** contain disaster-recovery execution logic.

The frontend does not:

- call GitHub directly;
- execute Terraform;
- call Azure Resource Manager;
- contain Azure credentials;
- contain a GitHub PAT;
- determine which GitHub workflow to execute.

Instead:

```text
Frontend
    ↓
Incident Response Backend
    ↓
GitHub Actions
```

The architecture specifically requires that the frontend not contain Azure credentials and that the GitHub trigger be protected.

---

# 3. Overall Frontend Flow

The user journey should be:

```text
NORMAL BANKING UI
       ↓
SERVICE DEGRADATION
       ↓
INCIDENT REVIEW
       ↓
AI RECOMMENDATION
       ↓
HUMAN APPROVAL
       ↓
RECOVERY TRIGGER
       ↓
RECOVERY STATUS
       ↓
RECOVERY SUCCESS / FAILURE
```

The project's intended sequence is:

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
RECOVERY PLAN
   ↓
SLOT SWAP
   ↓
HEALTH CHECK
   ↓
SUCCESSFUL PAYMENT
```

The frontend is responsible primarily for the **Human Approval** and **status presentation** portions.

---

# 4. Required Pages / Views

The frontend should have three logical UI states/pages.

## 4.1 Banking Payment Page

Existing application UI.

Example:

```text
┌──────────────────────────────────────┐
│          BANK PAYMENT PORTAL         │
│                                      │
│ Amount: [ 5000 ]                     │
│                                      │
│       [ PROCESS PAYMENT ]            │
│                                      │
│ Transaction Status:                  │
│       ✓ Payment Successful           │
│                                      │
│ Service Status: HEALTHY              │
└──────────────────────────────────────┘
```

This reflects the project's existing frontend specification.

---

# 5. Incident Review Page

When an incident requiring human review exists, display a clear incident-review interface.

Example:

```text
┌──────────────────────────────────────────┐
│       DISASTER RECOVERY INCIDENT         │
│                                          │
│ Service: Payment API                     │
│ Status: DEGRADED                         │
│ Severity: HIGH                           │
│                                          │
│ Error Rate: 18%                          │
│ Response Time: 2.4 seconds               │
│                                          │
│ Summary:                                 │
│ Payment API is experiencing elevated     │
│ failures.                                │
│                                          │
│ Likely Cause:                            │
│ Payment service is returning HTTP 500s.  │
│                                          │
│ AI Confidence: 91%                       │
│                                          │
│ Recommended Action:                      │
│ FAILOVER_PAYMENT_SERVICE                 │
│                                          │
│ [ APPROVE RECOVERY ] [ REJECT ]          │
└──────────────────────────────────────────┘
```

This mirrors the architecture's proposed incident review page.

---

# 6. Incident Data

The incident information should contain the AI recommendation fields:

```json
{
  "service": "payment-api",
  "severity": "HIGH",
  "errorRate": 18,
  "responseTime": 2.4,
  "summary": "Payment API is experiencing elevated failures.",
  "likelyCause": "Payment service is returning HTTP 500 responses.",
  "confidence": 0.91,
  "recommendedAction": "FAILOVER_PAYMENT_SERVICE",
  "status": "PENDING_APPROVAL"
}
```

These fields are based on the project's defined incident/recommendation contract.

Display them clearly.

---

# 7. Recommendation Display

The recommended action should be treated as a predefined recovery action.

For example:

```text
Recommended Action

FAILOVER_PAYMENT_SERVICE
```

Optionally provide a human-readable explanation:

```text
Activates the prepared App Service recovery environment.
```

The AI is advisory and the recovery actions are predefined. The frontend must not allow the user to modify the recommendation into an arbitrary infrastructure action.

---

# 8. Approve Button

The primary action is:

```text
APPROVE RECOVERY
```

When clicked:

```text
User clicks APPROVE
       ↓
Disable approval button
       ↓
Show "Starting recovery..."
       ↓
POST /api/recovery/trigger
       ↓
Success?
```

---

# 9. Trigger API

Call:

```http
POST /api/recovery/trigger
```

No GitHub information should be included in the request.

Example frontend request:

```javascript
await fetch(`${INCIDENT_BACKEND_URL}/api/recovery/trigger`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json"
  }
});
```

The frontend does **not** send:

```text
GITHUB_TOKEN
GITHUB_OWNER
GITHUB_REPOSITORY
GITHUB_WORKFLOW_ID
GITHUB_REF
```

---

# 10. Successful Approval Flow

When the backend returns success:

```json
{
  "status": "TRIGGERED",
  "message": "Recovery workflow triggered successfully"
}
```

the frontend should:

```text
Trigger successful
       ↓
Navigate to /recovery-status
```

If the backend returns run information, it may be retained for display, but the frontend should not depend on it being present.

---

# 11. Trigger Failure

If triggering fails:

```text
APPROVE RECOVERY
       ↓
Backend error
       ↓
Stay on Incident Review
       ↓
Display error
```

Example:

```text
Unable to start recovery.

The recovery workflow could not be triggered.
Please try again or investigate manually.
```

Do not tell the user to provide GitHub credentials.

---

# 12. Reject Button

The second action is:

```text
REJECT
```

The rejection flow is:

```text
REJECT
   ↓
No recovery API call
   ↓
No GitHub workflow
   ↓
No infrastructure change
   ↓
Display manual-review state
```

This is critical.

The frontend must **not** call:

```text
POST /api/recovery/trigger
```

when the user rejects the recommendation.

The project's required behaviour is explicitly:

```text
REJECT
   ↓
No infrastructure change
   ↓
Manual investigation
```



---

# 13. Reject UI

After rejection, show something like:

```text
RECOVERY REJECTED

The recommended recovery action was not approved.

No infrastructure change was initiated.

Manual investigation is required.
```

Optionally provide:

```text
[ RETURN TO DASHBOARD ]
```

---

# 14. Recovery Status Page

Create a dedicated recovery-status view.

Route:

```text
/recovery-status
```

The page should show the current recovery stage clearly.

Example:

```text
┌──────────────────────────────────────────┐
│          DISASTER RECOVERY               │
│                                          │
│ Recovery Status                          │
│                                          │
│ ✓ Approval received                      │
│ ✓ Recovery workflow triggered            │
│ ● Recovery in progress                   │
│ ○ Verification                           │
│ ○ Recovery successful                    │
│                                          │
│              [ REFRESH STATUS ]          │
└──────────────────────────────────────────┘
```

---

# 15. Status API

The frontend polls:

```http
GET /api/recovery/status
```

Example response:

```json
{
  "status": "IN_PROGRESS",
  "runId": 123456789,
  "workflowName": "Disaster Recovery",
  "updatedAt": "2026-09-16T18:30:00Z"
}
```

---

# 16. Frontend Status Mapping

The backend exposes these normalized statuses:

```text
NOT_STARTED
QUEUED
IN_PROGRESS
SUCCESS
FAILED
CANCELLED
UNKNOWN
```

Map them into user-friendly UI states.

| Backend Status | Frontend Display |
|---|---|
| `NOT_STARTED` | Waiting for recovery |
| `QUEUED` | Recovery queued |
| `IN_PROGRESS` | Recovery in progress |
| `SUCCESS` | Recovery successful |
| `FAILED` | Recovery failed |
| `CANCELLED` | Recovery cancelled |
| `UNKNOWN` | Status unavailable |

---

# 17. Recovery Progress

The UI should make the recovery sequence visually obvious.

Recommended:

```text
APPROVAL
   ✓
   ↓
RECOVERY TRIGGERED
   ✓
   ↓
RECOVERY IN PROGRESS
   ●
   ↓
VERIFYING
   ○
   ↓
RECOVERY SUCCESSFUL
   ○
```

The project's expected UI progression is:

```text
PENDING APPROVAL
      ↓
RECOVERY IN PROGRESS
      ↓
VERIFYING
      ↓
RECOVERY SUCCESSFUL
```

with the alternative failure path:

```text
RECOVERY FAILED
      ↓
MANUAL INTERVENTION REQUIRED
```



---

# 18. Polling

The frontend should automatically poll the status endpoint.

Recommended interval:

```text
5 seconds
```

Example:

```text
GET /api/recovery/status
       ↓
wait 5 seconds
       ↓
GET /api/recovery/status
       ↓
wait 5 seconds
       ↓
...
```

Stop polling when the backend returns a terminal state:

```text
SUCCESS
FAILED
CANCELLED
```

Do not continue polling indefinitely after recovery has completed.

---

# 19. Manual Refresh

Provide:

```text
[ REFRESH STATUS ]
```

The user should be able to manually refresh the recovery status in addition to automatic polling.

This is useful during the live demonstration.

---

# 20. Successful Recovery State

When:

```json
{
  "status": "SUCCESS"
}
```

display:

```text
┌──────────────────────────────────────────┐
│        ✓ RECOVERY SUCCESSFUL             │
│                                          │
│ Payment service has been recovered.      │
│                                          │
│ Recovery workflow: COMPLETE              │
│                                          │
│ Service Status: HEALTHY                  │
│                                          │
│ [ RETURN TO PAYMENT PORTAL ]             │
└──────────────────────────────────────────┘
```

The backend's `SUCCESS` means the GitHub recovery workflow completed successfully.

The underlying workflow is expected to perform:

```text
Slot Swap
   ↓
GET /health → 200
   ↓
POST /api/payments → SUCCESS
```

The project defines these as the recovery verification steps.

---

# 21. Failed Recovery State

If the backend returns:

```json
{
  "status": "FAILED"
}
```

display:

```text
┌──────────────────────────────────────────┐
│          ✕ RECOVERY FAILED               │
│                                          │
│ The recovery workflow did not complete   │
│ successfully.                            │
│                                          │
│ Manual intervention is required.         │
│                                          │
│ [ REFRESH STATUS ]                       │
└──────────────────────────────────────────┘
```

Do not present a failed recovery as a successful incident resolution.

The project's testing requirements explicitly include a recovery-failure scenario leading to manual review.

---

# 22. Cancelled Recovery

For:

```text
CANCELLED
```

display:

```text
RECOVERY CANCELLED

The recovery workflow was cancelled.

Manual investigation is required.
```

---

# 23. Unknown Status

If:

```text
UNKNOWN
```

display:

```text
RECOVERY STATUS UNAVAILABLE

The system could not determine the current recovery status.

[ REFRESH STATUS ]
```

Do not assume success.

---

# 24. Loading States

Every API call needs an appropriate loading state.

### Trigger

```text
APPROVE RECOVERY
      ↓
STARTING RECOVERY...
```

Disable the button while the request is running.

### Status

```text
Checking recovery status...
```

Do not leave the page visually frozen while waiting.

---

# 25. Prevent Double Approval

Once the user clicks:

```text
APPROVE RECOVERY
```

immediately disable the button.

Example:

```text
[ STARTING RECOVERY... ]
```

This prevents:

```text
click
click
click
```

from sending multiple trigger requests.

The backend also protects against duplicate recovery workflows, but the frontend should prevent unnecessary duplicate requests as well.

---

# 26. Frontend Environment Variables

The frontend needs only the URL of the Incident Response Backend.

For example:

```env
VITE_PAYMENT_API_URL=
VITE_INCIDENT_API_URL=
```

Do **not** add:

```env
VITE_GITHUB_TOKEN=
VITE_AZURE_TOKEN=
VITE_AZURE_CREDENTIALS=
```

There must never be infrastructure credentials in the frontend.

---

# 27. Frontend API Layer

Do not scatter `fetch()` calls throughout components.

Create a small API layer.

Suggested:

```text
src/
├── api/
│   ├── paymentApi.ts
│   └── incidentApi.ts
```

Example:

```text
incidentApi.ts

getRecoveryStatus()
triggerRecovery()
```

This keeps the UI components focused on presentation and state.

---

# 28. Suggested Frontend Structure

```text
src/
│
├── components/
│   ├── IncidentCard.tsx
│   ├── RecoveryStatus.tsx
│   ├── StatusBadge.tsx
│   └── ApprovalButtons.tsx
│
├── pages/
│   ├── PaymentPage.tsx
│   ├── IncidentReviewPage.tsx
│   └── RecoveryStatusPage.tsx
│
├── api/
│   ├── paymentApi.ts
│   └── incidentApi.ts
│
├── types/
│   ├── incident.ts
│   └── recovery.ts
│
└── App.tsx
```

Adapt this structure to the existing frontend project rather than rebuilding the application unnecessarily.

---

# 29. Recovery Type

Define a frontend type corresponding to the backend contract.

Example:

```typescript
type RecoveryStatus =
  | "NOT_STARTED"
  | "QUEUED"
  | "IN_PROGRESS"
  | "SUCCESS"
  | "FAILED"
  | "CANCELLED"
  | "UNKNOWN";
```

Status response:

```typescript
interface RecoveryStatusResponse {
  status: RecoveryStatus;
  runId?: number;
  workflowName?: string;
  updatedAt?: string;
}
```

---

# 30. Incident Type

The incident UI should support:

```typescript
interface Incident {
  service: string;
  severity: string;
  errorRate: number;
  responseTime: number;
  summary: string;
  likelyCause: string;
  confidence: number;
  recommendedAction: string;
  status: string;
}
```

---

# 31. Recommended User Experience

The demonstration should feel like a real operational recovery system.

### Step 1 — Healthy

```text
Service Status: HEALTHY
Payment: SUCCESS
```

### Step 2 — Incident

```text
Service Status: DEGRADED

Incident detected.
AI recommendation available.
```

### Step 3 — Human Review

Display:

```text
Severity
Error rate
Response time
Likely cause
AI confidence
Recommended action
```

Then:

```text
[ APPROVE RECOVERY ] [ REJECT ]
```

### Step 4 — Approval

After approval:

```text
Recovery initiated.
```

Navigate to:

```text
/recovery-status
```

### Step 5 — Recovery

Show:

```text
QUEUED
   ↓
IN PROGRESS
   ↓
VERIFYING
```

### Step 6 — Success

Show:

```text
✓ RECOVERY SUCCESSFUL

Service Status: HEALTHY
Payment Verification: SUCCESS
```

This directly supports the capstone's end-to-end demonstration.

---

# 32. Error Handling

Frontend should handle:

### Backend unavailable

```text
Unable to contact recovery service.
```

### Trigger failure

```text
Recovery could not be started.
Please retry or investigate manually.
```

### Status failure

```text
Unable to retrieve recovery status.
[ REFRESH STATUS ]
```

### Network timeout

```text
Connection to recovery service timed out.
```

Do not expose raw stack traces to the user.

---

# 33. Incident Page Refresh

The incident review page may periodically refresh incident data if required by the existing application design.

However, do not introduce unnecessary polling complexity.

The critical polling requirement is the **recovery status page after approval**.

---

# 35. Frontend Testing

The frontend developer should test:

### Test 1 — Incident displayed

Verify:

```text
Severity
Error rate
Response time
Summary
Likely cause
Confidence
Recommendation
```

are displayed.

### Test 2 — Approve

```text
Click APPROVE
      ↓
POST /api/recovery/trigger
      ↓
Navigate to recovery status
```

### Test 3 — Reject

```text
Click REJECT
      ↓
No trigger API request
      ↓
Manual-review state
```

### Test 4 — Recovery polling

Mock:

```text
QUEUED
IN_PROGRESS
SUCCESS
```

and verify the UI changes correctly.

### Test 5 — Recovery failure

Mock:

```text
FAILED
```

and verify:

```text
RECOVERY FAILED
MANUAL INTERVENTION REQUIRED
```

### Test 6 — Duplicate clicks

Verify that rapid repeated clicks on:

```text
APPROVE RECOVERY
```

do not produce multiple frontend requests.

### Test 7 — Backend unavailable

Verify an appropriate error state.

---

# 36. Acceptance Criteria

The frontend is complete when:

- [ ] Banking payment UI continues to work.
- [ ] Incident information can be displayed.
- [ ] AI recommendation is clearly visible.
- [ ] Human can approve recovery.
- [ ] Human can reject recovery.
- [ ] Reject never triggers recovery.
- [ ] Approve calls `POST /api/recovery/trigger`.
- [ ] GitHub is never called directly by frontend.
- [ ] No Azure/GitHub credentials exist in frontend code.
- [ ] Successful trigger navigates to recovery status.
- [ ] Recovery status is automatically polled.
- [ ] Manual Refresh Status works.
- [ ] `QUEUED` is displayed correctly.
- [ ] `IN_PROGRESS` is displayed correctly.
- [ ] `SUCCESS` is displayed correctly.
- [ ] `FAILED` is displayed correctly.
- [ ] `CANCELLED` is displayed correctly.
- [ ] Unknown/backend-error states are handled.
- [ ] Polling stops after a terminal state.
- [ ] Successful recovery clearly communicates service restoration.
- [ ] Failed recovery clearly communicates manual intervention.
- [ ] Full frontend flow works against the deployed Incident Response Backend.

---

# 37. End-to-End Frontend Flow

The final frontend experience should be:

```text
                 PAYMENT PORTAL
                       │
                       ▼
                  FAILURE
                       │
                       ▼
                 INCIDENT PAGE
                       │
                       ▼
               AI RECOMMENDATION
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
           APPROVE            REJECT
              │                 │
              ▼                 ▼
      Incident Backend     No Recovery
              │                 │
              ▼                 ▼
       GitHub Actions       Manual Review
              │
              ▼
       Recovery Status Page
              │
       ┌──────┴───────┐
       │              │
       ▼              ▼
    SUCCESS         FAILED
       │              │
       ▼              ▼
 HEALTHY +         MANUAL
 PAYMENT           INTERVENTION
 SUCCESS
```

---

# 38. Final Developer Principle

The frontend should make the recovery process **visible and understandable**, but it should not become responsible for executing recovery.

The separation should remain:

```text
FRONTEND
Human interaction
       ↓
INCIDENT BACKEND
Secure recovery trigger/status bridge
       ↓
GITHUB ACTIONS
Recovery execution
       ↓
TERRAFORM / RECOVERY PLAN
Infrastructure configuration
       ↓
AZURE
Actual recovery
```

This keeps the implementation aligned with the capstone's simplified architecture while providing the team with a clear human approval and recovery-status experience.
