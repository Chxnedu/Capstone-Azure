# Cloud-Native Disaster Recovery Platform
## 4-Day Implementation Timeline

### Team Structure

| Team | Primary Responsibility |
|---|---|
| **Infrastructure & DevOps** | Azure infrastructure, Terraform, App Service, monitoring, alerts, Logic Apps, GitHub Actions, CI/CD, secrets |
| **AI** | Azure OpenAI integration, incident analysis prompt, structured recommendation, safety constraints |
| **Software Development** | Banking frontend, Payment API, failure simulation, telemetry instrumentation, incident/review interface |
| **QA / Testing & Dashboard** | Test strategy, integration/E2E testing, failure scenarios, recovery verification, Power BI/dashboard, demo validation |

### Overall Strategy

The four days should follow this progression:

```text
DAY 1
Build the foundations
        ↓
DAY 2
Connect the major components
        ↓
DAY 3
Complete the end-to-end recovery workflow
        ↓
DAY 4
Test, harden, document and rehearse
```

The **non-negotiable MVP** is:

```text
Payment API
    ↓
Telemetry
    ↓
Azure Monitor Alert
    ↓
Logic App
    ↓
Azure OpenAI
    ↓
Human Approval
    ↓
GitHub Actions
    ↓
Recovery Slot / Swap
    ↓
Health Check + Test Payment
    ↓
RESOLVED
```

This directly reflects the architecture's intended recovery sequence.

---

# DAY 1 — BUILD THE FOUNDATIONS

## Goal

By the end of Day 1, **every team should have something running** and the basic infrastructure/application foundations should exist.

Do **not** spend Day 1 trying to connect everything together. Establish stable interfaces between teams first.

---

## Infrastructure & DevOps

### Tasks

1. Create the Azure resource group/project structure.
2. Provision the App Service Plan.
3. Provision the main App Service.
4. Provision the **recovery deployment slot**.
5. Provision Application Insights.
6. Configure Log Analytics.
7. Establish Terraform repository structure.
8. Set up GitHub repository and initial GitHub Actions pipeline.
9. Establish Azure authentication for GitHub Actions.
10. Define required secrets/variables.
11. Document resource names and endpoints for the other teams.

### End-of-Day Deliverables

```text
✓ Azure resource group
✓ App Service Plan
✓ Production App Service
✓ Recovery App Service slot
✓ Application Insights
✓ Log Analytics workspace
✓ Terraform project initialized
✓ GitHub Actions pipeline running
✓ Azure authentication working
✓ Shared environment variables/secrets documented
```

The recovery slot is important because the architecture explicitly uses App Service deployment slots rather than maintaining a separate standby VM.

---

## AI Team

### Tasks

1. Confirm Azure OpenAI access.
2. Identify the deployed model/endpoint.
3. Define the incident-analysis prompt.
4. Define the exact JSON response schema.
5. Implement the initial AI analysis service/workflow interface.
6. Restrict recommendations to known recovery actions.

### Required AI output

```json
{
  "severity": "HIGH",
  "summary": "...",
  "likelyCause": "...",
  "confidence": 0.91,
  "impact": "...",
  "recommendedAction": "FAILOVER_PAYMENT_SERVICE"
}
```

7. Define fallback behavior:

```text
Unknown recommendation
        ↓
MANUAL_REVIEW
        ↓
NO AUTOMATIC RECOVERY
```

### End-of-Day Deliverables

```text
✓ Azure OpenAI access confirmed
✓ Prompt finalized
✓ JSON schema finalized
✓ Known recovery actions defined
✓ AI can analyze sample incident data
✓ AI cannot execute infrastructure actions
✓ Fallback/manual-review behavior defined
```

The architecture specifically requires AI to be **advisory only** and prevents it from receiving Azure credentials or directly modifying infrastructure.

---

# Software Development

## Tasks

### Backend

Build the minimum Payment API:

```text
GET  /health
POST /api/payments
POST /api/failure
```

Optional:

```text
GET /api/payments/:id
GET /incident
```

The `/payments/:id` endpoint should be considered **low priority** if time becomes constrained.

### Failure Simulation

Implement:

```text
failureMode = false
        ↓
Normal payment
        ↓
SUCCESS
```

and:

```text
failureMode = true
        ↓
Payment request
        ↓
HTTP 500
        ↓
FAILED
```

### Frontend

Build the basic banking UI:

```text
BANK PAYMENT PORTAL

Amount: [5000]

[ PROCESS PAYMENT ]

Transaction Status:
Payment Successful

Service Status:
HEALTHY
```

### Telemetry

Instrument:

```text
✓ Requests
✓ Failed requests
✓ Response duration
✓ Exceptions
```

These are deliberately the minimum telemetry requirements in the architecture.

### End-of-Day Deliverables

```text
✓ Payment API running locally
✓ /health working
✓ /api/payments working
✓ /api/failure working
✓ Failure mode working
✓ Frontend calling API
✓ Application telemetry implemented
✓ Application can be deployed to App Service
```

---

# QA / Testing & Dashboard

## Tasks

1. Define acceptance criteria.
2. Create initial test cases.
3. Define the failure scenario.
4. Define expected recovery behavior.
5. Create a simple test checklist.
6. Begin Power BI dashboard design.
7. Identify required Log Analytics/KQL queries.

### Core test scenario

```text
NORMAL
  ↓
Trigger Failure
  ↓
Payment failures increase
  ↓
Alert
  ↓
AI recommendation
  ↓
Human approval
  ↓
Recovery
  ↓
Health check
  ↓
Successful payment
```

### End-of-Day Deliverables

```text
✓ Test plan
✓ E2E test scenario
✓ Acceptance criteria
✓ Failure/recovery test cases
✓ Initial KQL queries
✓ Dashboard wireframe
```

---

# DAY 1 EXIT CRITERIA

Do not move on until these are true:

```text
[ ] Application runs
[ ] Application can be deployed to Azure
[ ] Production + recovery slots exist
[ ] Application Insights receives telemetry
[ ] Failure mode works
[ ] Azure OpenAI can process sample incident data
[ ] GitHub Actions can authenticate to Azure
[ ] Teams have exchanged endpoint/resource information
[ ] Test scenario is agreed by everyone
```

---

# DAY 2 — CONNECT THE COMPONENTS

## Goal

By the end of Day 2, the system should be **detecting failures and generating AI recommendations**.

The critical milestone is:

```text
Failure
   ↓
Telemetry
   ↓
Alert
   ↓
Logic App
   ↓
Azure OpenAI
   ↓
Recommendation
```

---

# Infrastructure & DevOps

## Tasks

1. Configure Application Insights → Log Analytics integration.
2. Configure Azure Monitor alert.
3. Set the demonstration threshold.

Recommended:

```text
Error rate > 10%
Evaluation window = 5 minutes
```

The architecture explicitly proposes this type of threshold and notes that the team should make it easy to trigger during the demonstration.

4. Create the Logic App.
5. Configure the Logic App trigger.
6. Configure Log Analytics query.
7. Build the incident-context payload.
8. Connect Logic App to Azure OpenAI.
9. Store the AI recommendation somewhere accessible to the review UI.

### Preferred state mechanism

If the application needs to retrieve the incident:

```text
Logic App
   ↓
JSON incident artifact
   ↓
Azure Blob Storage
   ↓
Review Page
```

The architecture recommends Blob Storage as the simple hand-off/state mechanism instead of introducing PostgreSQL.

### End-of-Day Deliverables

```text
✓ Azure Monitor alert fires
✓ Logic App receives alert
✓ Logic App queries Log Analytics
✓ Incident context generated
✓ Azure OpenAI receives incident context
✓ AI recommendation returned
✓ Recommendation stored/published
✓ Complete detection → AI flow working
```

---

# AI Team

## Tasks

1. Connect the finalized prompt to Logic Apps.
2. Validate the actual telemetry payload.
3. Improve prompt handling of noisy/incomplete telemetry.
4. Validate JSON response parsing.
5. Add strict action validation.

For example:

```text
IF recommendedAction ==
   FAILOVER_PAYMENT_SERVICE
       ↓
       VALID

ELSE
       ↓
       MANUAL_REVIEW
```

6. Test at least three scenarios:

### Scenario A — Clear failure

```text
High error rate
+
HTTP 500
+
Payment exceptions

→ HIGH
→ FAILOVER_PAYMENT_SERVICE
```

### Scenario B — Weak evidence

```text
Low error rate
+
Insufficient evidence

→ MANUAL_REVIEW
```

### Scenario C — Unknown recommendation

```text
Invalid AI action

→ MANUAL_REVIEW
```

### End-of-Day Deliverables

```text
✓ AI integrated with Logic App
✓ Real telemetry successfully analyzed
✓ Structured JSON returned
✓ Recovery action validation implemented
✓ Fallback behavior tested
✓ AI recommendation visible outside the AI team's environment
```

---

# Software Development

## Tasks

1. Deploy the application to Azure.
2. Confirm Application Insights telemetry from the deployed application.
3. Build the incident/review page.
4. Display:

```text
Service
Status
Severity
Error Rate
Response Time
Likely Cause
AI Confidence
Recommended Action
```

5. Implement the approval/rejection UI.

```text
[ APPROVE RECOVERY ]

[ REJECT ]
```

6. Do **not** put Azure credentials in the frontend.

The architecture explicitly requires the frontend to contain no Azure credentials and treats human approval as a hard gate.

### End-of-Day Deliverables

```text
✓ Production application deployed
✓ Review page working
✓ Incident information displayed
✓ AI recommendation displayed
✓ APPROVE button exists
✓ REJECT button exists
✓ No Azure credentials exposed to frontend
```

---

# QA / Testing & Dashboard

## Tasks

1. Execute the Day 1 test cases against Azure.
2. Verify telemetry appears in Application Insights.
3. Verify Log Analytics queries return useful data.
4. Verify alert fires under failure conditions.
5. Validate AI output against expected results.
6. Start connecting Power BI to Log Analytics.
7. Build initial dashboard.

### Dashboard MVP

```text
SERVICE HEALTH
├── Availability
├── Error Rate
├── Response Time
└── Request Volume

INCIDENT
├── Current Error Rate
├── Severity
├── AI Recommendation
└── AI Confidence
```

Power BI is intended primarily for observability/reporting rather than infrastructure control.

---

# DAY 2 EXIT CRITERIA

The team should be able to demonstrate:

```text
[ ] Trigger failure
[ ] Failed requests appear in telemetry
[ ] Error rate crosses threshold
[ ] Azure Monitor alert fires
[ ] Logic App starts
[ ] Log Analytics data is queried
[ ] AI analyzes the incident
[ ] Recommendation is produced
[ ] Recommendation reaches review page
[ ] Reviewer can see the incident
```

At this point you have a **partial end-to-end system**.

---

# DAY 3 — COMPLETE RECOVERY

## Goal

This is the most important implementation day.

By the end of Day 3, the team must be able to demonstrate:

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

---

# Infrastructure & DevOps

## Tasks

### 1. Finalize recovery Terraform

Terraform should own the infrastructure configuration for:

```text
✓ App Service
✓ Recovery slot
✓ Slot configuration
✓ Application settings
✓ Monitoring resources
✓ Supporting Azure resources
```

The architecture explicitly states that Terraform should be the source of truth for the desired infrastructure configuration, while the recovery slot already exists rather than being recreated during every incident.

### 2. Build recovery workflow

GitHub Actions:

```text
workflow_dispatch
      ↓
Validate recovery plan
      ↓
Azure Login
      ↓
Terraform
      ↓
Recovery operation
      ↓
Health verification
```

### 3. Implement slot swap

Conceptually:

```text
BEFORE

Production → Bad Version
Recovery   → Good Version


AFTER

Production → Good Version
Recovery   → Bad Version
```

The architecture specifically identifies the App Service slot swap as the recovery mechanism.

### 4. Add post-recovery verification

The workflow must verify:

```text
GET /health
POST /api/payments
```

### End-of-Day Deliverables

```text
✓ Recovery Terraform complete
✓ Recovery plan version-controlled
✓ GitHub Actions recovery workflow working
✓ Azure authentication working
✓ Recovery slot swap working
✓ /health verification automated
✓ Test payment verification automated
✓ Recovery result reported
```

---

# AI Team

## Tasks

AI should now become relatively stable.

Focus on:

1. Prompt reliability.
2. JSON parsing reliability.
3. Recommendation validation.
4. Handling missing telemetry.
5. Handling ambiguous incidents.
6. Ensuring AI never triggers recovery directly.

### Final AI contract

```text
INPUT
Incident Context

OUTPUT

{
  severity,
  summary,
  likelyCause,
  confidence,
  impact,
  recommendedAction
}
```

### Critical rule

```text
AI
 ↓
RECOMMENDATION
 ↓
HUMAN
 ↓
APPROVAL
 ↓
RECOVERY
```

Never:

```text
AI
 ↓
AZURE
```

This maintains the architecture's intended safety boundary.

### End-of-Day Deliverables

```text
✓ Production-quality prompt
✓ Reliable structured output
✓ Invalid action handling
✓ Low-confidence handling
✓ AI integration stable
```

---

# Software Development

## Tasks

### 1. Complete approval flow

```text
AI Recommendation
       ↓
Review Page
       ↓
APPROVE
       ↓
Recovery Trigger
```

And:

```text
REJECT
   ↓
No infrastructure change
   ↓
Manual Review
```

### 2. Connect approval to GitHub Actions

The implementation may use whatever secure mechanism the team has selected, but the important behavior is:

```text
APPROVE
   ↓
Authorized GitHub Actions workflow
   ↓
Recovery plan
```

### 3. Display recovery status

The UI should communicate:

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

### End-of-Day Deliverables

```text
✓ Approval → recovery trigger working
✓ Reject → no infrastructure change
✓ Recovery status displayed
✓ Successful recovery displayed
✓ Failed recovery displayed
```

---

# QA / Testing & Dashboard

## Tasks

This is the first day QA should perform **full integration testing**.

### Test 1 — Normal Operation

```text
Health → 200
Payment → SUCCESS
Error rate → Normal
```

### Test 2 — Failure

```text
Enable failure mode
      ↓
Payments fail
      ↓
Telemetry increases
      ↓
Alert fires
```

### Test 3 — AI

```text
Alert
 ↓
Logic App
 ↓
AI
 ↓
HIGH
 ↓
FAILOVER_PAYMENT_SERVICE
```

### Test 4 — Approval

```text
APPROVE
 ↓
Recovery workflow
```

### Test 5 — Rejection

```text
REJECT
 ↓
No recovery
```

### Test 6 — Recovery

```text
Slot swap
 ↓
/health = 200
 ↓
Payment = SUCCESS
```

### Test 7 — Recovery Failure

Intentionally make verification fail and ensure:

```text
RECOVERY FAILED
        ↓
MANUAL REVIEW
```

### Dashboard

Finish the Power BI MVP:

```text
┌─────────────────────────────────┐
│       SERVICE HEALTH            │
├─────────────────────────────────┤
│ Availability                    │
│ Error Rate                      │
│ Response Time                   │
│ Request Volume                  │
├─────────────────────────────────┤
│       CURRENT INCIDENT          │
├─────────────────────────────────┤
│ Severity                        │
│ Error Rate                      │
│ Recommendation                  │
│ AI Confidence                   │
├─────────────────────────────────┤
│       RECOVERY                  │
├─────────────────────────────────┤
│ Recovery Attempts               │
│ Successful Recoveries           │
│ Failed Recoveries               │
│ Recovery Duration               │
└─────────────────────────────────┘
```

### End-of-Day Deliverables

```text
✓ Full E2E test executed
✓ Failure scenario validated
✓ Approval scenario validated
✓ Rejection scenario validated
✓ Recovery scenario validated
✓ Recovery failure scenario validated
✓ Power BI MVP functional
✓ Major defects identified
```

---

# DAY 3 EXIT CRITERIA

This is the **critical milestone**.

The team must be able to run the complete scenario:

```text
1. Application is healthy
       ↓
2. Failure is triggered
       ↓
3. Payments fail
       ↓
4. Telemetry records failures
       ↓
5. Alert fires
       ↓
6. Logic App investigates
       ↓
7. Azure OpenAI analyzes
       ↓
8. Recommendation appears
       ↓
9. Human approves
       ↓
10. GitHub Actions executes
       ↓
11. Recovery slot activated/swapped
       ↓
12. /health passes
       ↓
13. Payment succeeds
       ↓
14. Incident = RESOLVED
```

If this works by the end of Day 3, **the capstone is fundamentally complete**.

---

# DAY 4 — HARDEN, TEST & DEMO

## Goal

**Do not add major new architecture on Day 4.**

Day 4 is for:

```text
Fix
↓
Test
↓
Secure
↓
Document
↓
Rehearse
```

---

# Infrastructure & DevOps

## Tasks

### Reliability

- Fix Terraform issues.
- Fix GitHub Actions failures.
- Verify deployment repeatability.
- Verify recovery workflow repeatability.
- Verify App Service slots.
- Verify monitoring/alerts.
- Confirm all resources are named/documented.

### Security

Verify:

```text
[ ] No Azure credentials in frontend
[ ] Secrets stored securely
[ ] GitHub Actions protected
[ ] AI has no infrastructure credentials
[ ] Recovery requires approval
[ ] Recovery actions are predefined
```

### Demo Environment

Freeze the infrastructure configuration.

```text
NO MAJOR INFRASTRUCTURE CHANGES
```

after the final successful E2E test.

### Deliverables

```text
✓ Stable Azure environment
✓ Stable Terraform
✓ Stable GitHub Actions
✓ Secure secrets
✓ Final infrastructure diagram
✓ Resource inventory
✓ Deployment/recovery instructions
```

---

# AI Team

## Tasks

Run final AI tests:

```text
✓ Clear failure
✓ Ambiguous failure
✓ Low-confidence incident
✓ Invalid recommendation
✓ Missing telemetry
```

Verify that AI:

```text
✓ Produces structured output
✓ Gives reasonable severity
✓ Provides a likely cause
✓ Provides confidence
✓ Selects only known recovery actions
✓ Falls back to manual review when appropriate
✓ Cannot directly modify infrastructure
```

### Deliverables

```text
✓ Final prompt
✓ AI test results
✓ AI safety documentation
✓ Example incident analysis
✓ Example recommendation
```

---

# Software Development

## Tasks

### UI Polish

Focus only on things visible during the demonstration:

```text
✓ Clear HEALTHY state
✓ Clear DEGRADED state
✓ Clear incident information
✓ Clear AI recommendation
✓ Clear APPROVE/REJECT controls
✓ Clear recovery progress
✓ Clear SUCCESS/FAILURE result
```

### Application Reliability

Verify:

```text
/health
/api/payments
/api/failure
```

are stable.

### Demo Reset

Implement/document an easy way to return to:

```text
HEALTHY
```

after every demonstration.

### Deliverables

```text
✓ Final frontend
✓ Final API
✓ Failure simulation
✓ Incident review page
✓ Recovery status UI
✓ Demo reset procedure
```

---

# QA / Testing & Dashboard

## Tasks

QA owns the final **Go/No-Go test**.

### Final E2E Test

Run from a completely clean/known state:

```text
HEALTHY
  ↓
TRIGGER FAILURE
  ↓
WAIT FOR ALERT
  ↓
AI ANALYSIS
  ↓
REVIEW
  ↓
APPROVE
  ↓
RECOVER
  ↓
VERIFY
  ↓
RESOLVED
```

### Capture Evidence

Capture screenshots/logs for:

1. Healthy application.
2. Failed payments.
3. Application Insights telemetry.
4. Azure Monitor alert.
5. Logic App execution.
6. AI recommendation.
7. Approval page.
8. GitHub Actions execution.
9. Recovery slot/swap.
10. Successful health check.
11. Successful payment.
12. Power BI dashboard.

### Final Dashboard

Verify that the dashboard tells the story:

```text
BEFORE
Error Rate ↑
       ↓
INCIDENT
Severity HIGH
       ↓
RECOVERY
Recovery initiated
       ↓
AFTER
Error Rate ↓
Service HEALTHY
```

### Deliverables

```text
✓ Final E2E test report
✓ Test evidence/screenshots
✓ Defect list closed or documented
✓ Power BI dashboard
✓ Demo checklist
✓ Go/No-Go approval
```

---

# DAY 4 FINAL EXIT CRITERIA

The project is ready when:

```text
[✓] Application works
[✓] Failure can be triggered
[✓] Monitoring detects failure
[✓] Alert fires
[✓] Logic App investigates
[✓] AI analyzes incident
[✓] Recommendation is generated
[✓] Human approval is required
[✓] Approval triggers recovery
[✓] Recovery plan executes
[✓] Recovery slot becomes active
[✓] /health returns 200
[✓] Test payment succeeds
[✓] Recovery result is displayed
[✓] Power BI shows operational data
[✓] No credentials are exposed
[✓] E2E demo has been rehearsed
```

---

# Cross-Team Dependencies

This is particularly important because four teams working independently can easily become blocked by one another.

## Dependency Map

```text
                    ┌─────────────────────┐
                    │ Infrastructure      │
                    │ Azure Foundation    │
                    └──────────┬──────────┘
                               │
                ┌──────────────┼──────────────┐
                ↓              ↓              ↓
         Software Dev         AI             QA
         Application      OpenAI Flow     Test/KQL
                │              │              │
                └──────────────┼──────────────┘
                               ↓
                       Integration
                               ↓
                        Recovery Flow
                               ↓
                         Final Testing
```

### Specific hand-offs

| Provider | Consumer | Required Handoff |
|---|---|---|
| Infrastructure | Software | App Service URL, App Insights configuration, deployment details |
| Infrastructure | AI | Azure OpenAI endpoint/model/access details |
| Infrastructure | QA | Log Analytics workspace, KQL access, monitoring resources |
| Software | Infrastructure | Application runtime, ports, environment variables, health endpoint |
| Software | AI | Error formats, telemetry examples, application failure behavior |
| Software | QA | API endpoints, expected responses, failure trigger |
| AI | Software | Recommendation JSON schema |
| AI | QA | Expected AI outputs/test cases |
| Infrastructure | QA | Recovery workflow behavior and expected outputs |
| QA | Everyone | Acceptance criteria and E2E test results |

---

# Daily Stand-Up Structure

Because the timeline is extremely aggressive, have a **15-minute stand-up every morning**.

Each team answers only:

```text
1. What did we complete?
2. What are we doing today?
3. What is blocking us?
4. What do we need from another team?
```

Do not spend the stand-up debugging.

If two teams are blocked:

```text
Stand-up
   ↓
Identify blocker
   ↓
Assign 2–3 people
   ↓
Separate technical session
```

---

# Recommended Git Strategy

Keep the repository structure obvious:

```text
/
├── app/
│   ├── backend/
│   └── frontend/
│
├── infrastructure/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
│
├── recovery-plans/
│   └── payment-service-failover/
│       ├── main.tf
│       ├── variables.tf
│       └── README.md
│
├── workflows/
│   └── disaster-recovery.yml
│
├── ai/
│   ├── prompts/
│   └── schemas/
│
├── tests/
│   ├── api/
│   ├── integration/
│   └── e2e/
│
└── docs/
    ├── architecture.md
    ├── deployment.md
    ├── recovery.md
    └── demo-runbook.md
```

The architecture itself already establishes the concept of a version-controlled predefined `recovery-plans/payment-service-failover` plan.

---

# Priority Rules for the 4-Day Sprint

If you run out of time, use this priority order.

## P0 — MUST WORK

```text
1. Payment API
2. Failure simulation
3. Application telemetry
4. Azure Monitor alert
5. Logic App
6. Azure OpenAI analysis
7. Human approval
8. GitHub Actions
9. Recovery slot
10. Health verification
11. Successful payment after recovery
```

## P1 — SHOULD WORK

```text
12. Incident review UI polish
13. Power BI dashboard
14. Recovery failure handling
15. Detailed recovery status
16. Automated test suite
```

## P2 — NICE TO HAVE

```text
17. Payment lookup endpoint
18. Advanced dashboard visuals
19. Advanced telemetry
20. Sophisticated incident history
21. Additional recovery plans
22. Complex authentication
23. Advanced analytics
```

**Do not sacrifice the P0 end-to-end flow to implement P1/P2 features.**

---

# The Four-Day Definition of Done

At the end of the fourth day, the team should be able to stand in front of the evaluators and say:

> **"We intentionally broke the payment service. Azure detected the degradation through telemetry, investigated the incident through Logic Apps, used Azure OpenAI to analyze the evidence and recommend a predefined recovery action, presented that recommendation to a human for approval, executed the approved recovery through GitHub Actions and our version-controlled infrastructure, and finally verified that the service recovered by checking both health and an actual payment transaction."**

That is the capstone story.

```text
              ┌───────────────────────────┐
              │       DAY 1               │
              │       FOUNDATION          │
              └────────────┬──────────────┘
                           ↓
              ┌───────────────────────────┐
              │       DAY 2               │
              │       DETECT + AI         │
              └────────────┬──────────────┘
                           ↓
              ┌───────────────────────────┐
              │       DAY 3               │
              │       FULL RECOVERY       │
              └────────────┬──────────────┘
                           ↓
              ┌───────────────────────────┐
              │       DAY 4               │
              │       HARDEN + DEMO       │
              └───────────────────────────┘
```

**Most important rule:** by the end of **Day 3**, the complete happy-path demo must already work. Day 4 should never be the first time you attempt the full workflow.