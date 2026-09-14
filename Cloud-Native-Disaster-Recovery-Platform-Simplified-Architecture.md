# Cloud-Native Disaster Recovery Platform
## Simplified 4-Day Capstone Architecture

> **Purpose:** A practical, cloud-native disaster recovery demonstration for a simulated banking payment service.
>
> **Core flow:** **Detect → Investigate → Recommend → Approve → Recover → Verify**
>
> This architecture intentionally simplifies the original facilitator recommendation. The goal is to demonstrate the business problem and required deliverables without building a large custom control-plane application.

---

## 1. Executive Summary

The platform demonstrates how a banking payment service can detect a service failure, use AI to analyze the incident and recommend a known recovery action, obtain human approval, execute a predefined infrastructure recovery plan, and verify that the service has actually recovered.

The platform will use:

- **Azure App Service** for the simulated banking application.
- **Application Insights** for application telemetry.
- **Azure Monitor + Log Analytics** for monitoring, querying, and alerting.
- **Azure Logic Apps** for lightweight workflow orchestration.
- **Azure OpenAI** for incident analysis and recovery recommendation.
- **A simple incident/review web page** for human approval.
- **GitHub Actions** to execute the approved recovery workflow.
- **Terraform** to manage the recovery infrastructure configuration.
- **Power BI** for operational visualization where useful.
- **Application health checks** to verify recovery.

The design deliberately removes:

- A custom control-plane backend.
- A PostgreSQL incident database.
- An Azure Function used only as an alert receiver.
- A VM-based primary/standby architecture.
- A complex event/messaging layer.

The application itself remains small. The recovery workflow is implemented primarily with managed Azure services and GitHub Actions.

---

# 2. Business Problem

Banking services must remain available during outages because downtime can prevent customers from completing transactions and can negatively affect business continuity and customer trust.

The platform addresses four operational problems:

1. **Slow incident detection**
2. **Manual investigation of logs and metrics**
3. **Inconsistent recovery decisions**
4. **Manual and error-prone recovery execution**

The intended business outcome is:

```text
Faster Detection
       ↓
Faster Diagnosis
       ↓
Consistent Recovery
       ↓
Reduced Downtime
       ↓
Improved Business Continuity
       ↓
Better Auditability
```

---

# 3. Design Principles

The implementation follows these principles:

### 3.1 Keep the banking application simple

The application exists to generate realistic telemetry and provide a service that can fail and recover. It is not intended to be a full banking platform.

### 3.2 Avoid a custom control-plane backend

The original design introduced a second backend responsible for incident management, telemetry retrieval, AI orchestration, approval, GitHub Actions triggering, status polling, and health verification.

For a four-day capstone, this creates unnecessary development and integration work.

Instead, **Azure Logic Apps and GitHub Actions provide the orchestration required for the demonstration.**

### 3.3 AI is advisory, not autonomous

Azure OpenAI does not receive Azure credentials and does not directly modify infrastructure.

AI:

- analyzes telemetry;
- summarizes the incident;
- identifies a probable cause;
- estimates severity;
- recommends a known recovery plan.

AI does **not**:

- write Terraform dynamically;
- execute Azure CLI;
- modify Azure resources;
- trigger infrastructure changes directly.

### 3.4 Recovery actions are predefined

Recovery plans are written and tested before an incident occurs.

Example:

```text
recovery-plans/
└── payment-service-failover/
    ├── main.tf
    ├── variables.tf
    └── README.md
```

The AI selects a known recovery plan rather than inventing infrastructure changes.

### 3.5 Human approval remains a hard gate

The recovery plan is not executed simply because AI recommends it.

The sequence is:

```text
AI Recommendation
       ↓
Human Review
       ↓
APPROVE
       ↓
GitHub Actions
       ↓
Terraform
```

---

# 4. High-Level Architecture

```text
                         ┌───────────────┐
                         │     Users     │
                         └───────┬───────┘
                                 │
                                 ▼
                       ┌───────────────────┐
                       │  Banking Frontend │
                       │   Simple Web UI   │
                       └─────────┬─────────┘
                                 │
                                 ▼
                       ┌───────────────────┐
                       │   Payment API     │
                       │  Azure App Service│
                       └─────────┬─────────┘
                                 │
                         Telemetry
                                 │
                                 ▼
                 ┌──────────────────────────────┐
                 │ Application Insights         │
                 │ + Azure Monitor              │
                 │ + Log Analytics              │
                 └──────────────┬───────────────┘
                                │
                         Alert threshold
                                │
                                ▼
                       ┌─────────────────┐
                       │  Azure Monitor  │
                       │      Alert      │
                       └────────┬────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │  Azure Logic    │
                       │      Apps       │
                       └───────┬─────────┘
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
                 ▼                           ▼
       Query Log Analytics          Send incident context
                 │                           │
                 └─────────────┬─────────────┘
                               ▼
                       ┌─────────────────┐
                       │  Azure OpenAI   │
                       │ Incident        │
                       │ Analysis        │
                       └────────┬────────┘
                                │
                         Recommendation
                                │
                                ▼
                    ┌────────────────────────┐
                    │ Incident Review Page   │
                    │                        │
                    │ Severity: HIGH         │
                    │ Confidence: 91%        │
                    │ Recommendation:        │
                    │ PAYMENT_FAILOVER       │
                    │                        │
                    │ [APPROVE] [REJECT]     │
                    └────────────┬───────────┘
                                 │
                              APPROVE
                                 │
                                 ▼
                       ┌─────────────────┐
                       │  GitHub Actions │
                       └────────┬────────┘
                                │
                                ▼
                           ┌─────────┐
                           │Terraform│
                           └────┬────┘
                                │
                                ▼
                    ┌────────────────────────┐
                    │ Azure App Service      │
                    │ Recovery Configuration │
                    └────────────┬───────────┘
                                 │
                                 ▼
                         Health Verification
                                 │
                    ┌────────────┴────────────┐
                    ▼                         ▼
                 SUCCESS                    FAILURE
                    │                         │
                    ▼                         ▼
               RESOLVED                 MANUAL REVIEW
```

---

# 5. Banking Application

## 5.1 Purpose

The banking application is a deliberately small payment simulator.

Its purpose is to:

- provide a customer-facing service;
- generate realistic request telemetry;
- provide a controlled failure mechanism;
- demonstrate monitoring;
- provide a service that can be recovered.

It does not need:

- real payment processing;
- authentication;
- a real banking database;
- customer accounts;
- payment gateways;
- complex transaction processing.

---

# 6. Frontend Specification

The frontend can be a simple React/Vite application or another lightweight web frontend.

Example UI:

```text
┌──────────────────────────────────────┐
│          BANK PAYMENT PORTAL         │
│                                      │
│ Amount: [ 5000 ]                     │
│                                      │
│       [ PROCESS PAYMENT ]            │
│                                      │
│ Transaction Status:                  │
│                                      │
│       ✓ Payment Successful           │
│                                      │
│ Service Status: HEALTHY              │
└──────────────────────────────────────┘
```

The frontend calls the payment API.

It should not contain disaster-recovery logic.

---

# 7. Payment API Specification

The backend is the application's **payment API**. It is not the disaster-recovery control plane.

Recommended endpoints:

| Endpoint | Method | Purpose |
|---|---|---|
| `/health` | GET | Application health check |
| `/api/payments` | POST | Simulate a payment |
| `/api/payments/:id` | GET | Retrieve simulated payment status |
| `/api/failure` | POST | Enable/disable simulated failure mode |
| `/incident` | GET | Display the latest AI incident recommendation for the review page |

The `/incident` endpoint is part of the simplified design only if the team chooses to host the approval/review page with the application.

---

# 8. Endpoint Behaviour

## 8.1 `GET /health`

Normal response:

```json
{
  "status": "healthy"
}
```

The endpoint should return HTTP 200 when the application is healthy.

When failure mode is active, it can return:

```json
{
  "status": "unhealthy"
}
```

with an appropriate non-200 status.

This endpoint is used for recovery verification.

---

## 8.2 `POST /api/payments`

Example request:

```json
{
  "amount": 5000
}
```

Normal response:

```json
{
  "transactionId": "TXN-12345",
  "status": "SUCCESS",
  "amount": 5000
}
```

When failure mode is enabled, the endpoint intentionally returns an error:

```json
{
  "status": "FAILED",
  "message": "Payment service temporarily unavailable"
}
```

The application therefore remains reachable, but the **critical payment function is degraded**.

This is preferable to completely killing the application because it allows Azure Monitor and Application Insights to observe the failure normally.

---

## 8.3 `GET /api/payments/:id`

Optional endpoint for demonstrating transaction lookup.

Example:

```text
GET /api/payments/TXN-12345
```

Response:

```json
{
  "transactionId": "TXN-12345",
  "status": "SUCCESS"
}
```

If time is limited, this endpoint can be omitted.

---

## 8.4 `POST /api/failure`

This is a controlled demonstration endpoint.

Example:

```text
POST /api/failure
```

It changes the application into failure mode.

For example:

```text
failureMode = true
```

From this point:

```text
POST /api/payments
        ↓
HTTP 500
        ↓
Payment Failed
```

The endpoint can optionally accept:

```json
{
  "enabled": true
}
```

This makes the demo easy to control.

---

# 9. Application Telemetry

The application developers configure the application to send telemetry to **Application Insights**.

The infrastructure team provisions/configures the Azure Application Insights resource and supplies the required connection configuration.

Therefore:

> **Infrastructure creates and configures the monitoring resource; developers instrument the application to send application telemetry to it.**

The application should emit only the telemetry necessary for this project.

## Required telemetry

### 9.1 Requests

Every API request should produce request telemetry.

Example:

```text
POST /api/payments
Status: 200
Duration: 145 ms
```

or:

```text
POST /api/payments
Status: 500
Duration: 210 ms
```

This allows Azure Monitor/Application Insights to calculate:

- request volume;
- failed request count;
- failure rate;
- response time.

---

## 9.2 Failures / Exceptions

When the payment processing code encounters an application error, the application should record an exception.

Example conceptually:

```javascript
try {
    processPayment();
} catch (error) {
    appInsights.trackException({
        exception: error
    });

    throw error;
}
```

The exact SDK implementation depends on the chosen application language.

The purpose is simply to give AI useful evidence such as:

```text
PaymentProcessingError
DatabaseConnectionError
PaymentServiceUnavailable
```

For the four-day implementation, the team does not need a large catalogue of custom exception types.

---

## 9.3 Response Time

Application Insights already receives request duration information.

For example:

```text
GET /health       50 ms
POST /api/payments 180 ms
```

A sudden increase can indicate degradation.

---

## 9.4 What we are deliberately NOT requiring

We do not need to build a complicated custom telemetry system.

Avoid implementing numerous:

- custom events;
- custom metrics;
- business analytics events;
- distributed tracing scenarios;
- elaborate logging frameworks.

The minimum useful telemetry is:

```text
Requests
   +
Failed requests
   +
Response duration
   +
Application exceptions
```

That is sufficient for the capstone.

---

# 10. Application Insights vs Azure Monitor

These services have different roles.

```text
Application
     │
     │ application telemetry
     ▼
Application Insights
     │
     ▼
Azure Monitor / Log Analytics
     │
     ├── Query telemetry
     ├── Create alert rules
     ├── Monitor error rate
     └── Monitor response time
```

### Application Insights

Primarily provides application-level telemetry such as:

- requests;
- failures;
- response duration;
- exceptions.

### Azure Monitor

Provides the broader monitoring and alerting capability.

For this project, Azure Monitor will:

- evaluate monitoring conditions;
- trigger alerts;
- provide the operational monitoring layer.

Log Analytics provides the query layer used to retrieve incident telemetry.

---

# 11. Failure Simulation

The failure should be **functional degradation**, not total application unavailability.

Normal:

```text
100 payment requests
       ↓
98 successful
2 failed

Error rate = 2%
```

Failure mode:

```text
100 payment requests
       ↓
80 successful
20 failed

Error rate = 20%
```

This creates a clear signal for Azure Monitor.

The application itself remains reachable, meaning:

- monitoring remains active;
- the failure can be observed;
- the recovery workflow can run;
- the recovery result can be demonstrated.

---

# 12. Azure Monitor Alert

The infrastructure team configures an Azure Monitor alert rule.

Example:

```text
Metric:
Failed requests / request failure rate

Condition:
Error rate > 10%

Evaluation window:
5 minutes

Action:
Trigger Logic App workflow
```

The threshold is **defined by the project team**.

It is not automatically chosen by Azure.

For the demonstration, the threshold should be intentionally easy to trigger.

Example:

```text
Normal:
Error rate < 10%

Failure:
Error rate > 10%

       ↓

🚨 Azure Monitor Alert
```

---

# 13. Logic Apps

Azure Logic Apps replaces most of the custom control-plane backend from the original design.

Its purpose is to orchestrate the workflow between Azure Monitor, Log Analytics, Azure OpenAI, and the approval mechanism.

Conceptually:

```text
Azure Monitor Alert
        ↓
    Logic App
        ↓
Query Log Analytics
        ↓
Build incident context
        ↓
Call Azure OpenAI
        ↓
Receive structured recommendation
        ↓
Publish recommendation
```

The Logic App acts as the **workflow engine**, not as a custom application backend.

---

# 14. What data goes to Azure OpenAI?

The AI should receive enough context to understand the incident without receiving the entire Log Analytics workspace.

The Logic App should gather a small incident window, for example the last 5–15 minutes.

Example context:

```text
INCIDENT

Service:
Payment API

Alert:
Payment API error rate exceeded 10%

Current error rate:
18%

Average response time:
2.4 seconds

Recent failed requests:
POST /api/payments → HTTP 500
POST /api/payments → HTTP 500
POST /api/payments → HTTP 500

Recent exceptions:
PaymentProcessingError
PaymentServiceUnavailable

Time:
2026-09-14 14:30 UTC
```

The AI then receives this context together with the recovery-plan catalogue.

---

# 15. AI Prompt

The AI should be constrained to the available recovery actions.

Example:

```text
You are an incident analysis assistant for a simulated
banking payment service.

Analyze the incident telemetry below.

Available recovery actions:

1. FAILOVER_PAYMENT_SERVICE
   Activates the prepared App Service recovery environment.

2. MANUAL_REVIEW
   Used when available evidence is insufficient.

Return JSON only.

Incident:
{{incident_context}}
```

Expected response:

```json
{
  "severity": "HIGH",
  "summary": "Payment API is experiencing elevated failures.",
  "likelyCause": "The payment service is returning a high volume of HTTP 500 responses.",
  "confidence": 0.91,
  "impact": "Customers may be unable to complete payments.",
  "recommendedAction": "FAILOVER_PAYMENT_SERVICE"
}
```

The AI should **not** generate Terraform.

---

# 16. AI Safety Boundary

The AI has no Azure infrastructure credentials.

```text
                    Azure OpenAI
                         │
                 Analysis only
                         │
                         ▼
                  Recommendation
                         │
                         X
                         │
                NO DIRECT AZURE ACCESS
```

The AI cannot:

- execute Terraform;
- modify App Service;
- start/stop infrastructure;
- call Azure Resource Manager;
- directly trigger the recovery workflow.

The AI recommendation must correspond to one of the predefined recovery actions.

If it does not:

```text
Unknown recommendation
        ↓
Manual review
        ↓
No recovery execution
```

---

# 17. Incident Review / Approval Page

A full operations dashboard is not required.

The simplest operational control interface is a small web page.

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

---

# 18. `/incident` Endpoint

If the review page is served by the banking application's frontend, the application can expose:

```text
GET /incident
```

The endpoint should return the latest incident/recommendation information available to the application.

Example:

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

### Important architectural question

Because we are removing the incident database, the team must decide where this small piece of incident state is temporarily stored.

For the capstone, the simplest options are:

1. **A small JSON artifact/object produced by the Logic App and stored in Azure Blob Storage**, which the application reads.
2. A lightweight configuration/state store if already available.
3. If the dashboard can consume Logic App output directly, avoid adding `/incident` altogether.

**Preferred:** use Blob Storage as the simple hand-off/state mechanism if the application must retrieve the recommendation.

This keeps PostgreSQL out of the architecture.

---

# 19. Power BI

Power BI is useful as the **visual analytics/dashboard layer**, but it does not need to be the approval mechanism.

Power BI can visualize data from Log Analytics by using Azure Monitor Logs queries. Microsoft provides a supported integration for exporting Log Analytics query results into Power BI reports and dashboards.

Recommended Power BI visuals:

### Service health

```text
Availability
Error rate
Average response time
Request volume
```

### Incident trend

```text
Incidents over time
Failed requests over time
```

### Current incident

```text
Current error rate
Current severity
Current recommendation
AI confidence
```

### Recovery outcome

```text
Recovery attempts
Successful recoveries
Failed recoveries
Recovery duration
```

The Power BI dashboard is therefore primarily for:

> **Observability, reporting, and demonstrating business/operational value.**

It does not need to control the infrastructure.

---

# 20. Power BI Data Flow

A simple model is:

```text
Application
     ↓
Application Insights
     ↓
Log Analytics
     ↓
KQL Queries
     ↓
Power BI
     ↓
Operational Dashboard
```

Power BI can use Log Analytics query results to build reports and dashboards.

For this capstone, avoid building a complicated data warehouse or ETL pipeline.

---

# 21. Human Approval

The approval mechanism should remain a real control gate.

The sequence is:

```text
AI Recommendation
       ↓
Incident Review Page
       ↓
Human reads:
- logs
- metrics
- severity
- likely cause
- confidence
- recommended recovery
       ↓
    APPROVE
       ↓
GitHub Actions
```

Or:

```text
    REJECT
       ↓
No infrastructure change
       ↓
Manual investigation
```

The frontend must not contain Azure credentials.

The GitHub Actions trigger must be protected.

---

# 22. GitHub Actions

GitHub Actions becomes the execution pipeline.

Example workflow:

```yaml
name: Disaster Recovery

on:
  workflow_dispatch:
    inputs:
      recovery_plan:
        required: true
        type: choice
        options:
          - payment-service-failover

jobs:
  recover:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        working-directory: recovery-plans/${{ inputs.recovery_plan }}
        run: terraform init

      - name: Terraform Apply
        working-directory: recovery-plans/${{ inputs.recovery_plan }}
        run: terraform apply -auto-approve
```

The actual workflow should be adjusted to match the team's Azure authentication and Terraform state design.

---

# 23. App Service Recovery Design

The application will use **Azure App Service deployment slots** instead of primary and standby VMs.

Example:

```text
                    App Service
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
        Production Slot       Recovery Slot
          ACTIVE                 READY
```

Normal operation:

```text
Users
  ↓
Production
```

Recovery operation:

```text
Users
  ↓
Recovery Slot
```

The recovery slot should contain the known-good application version.

---

# 24. Why App Service Slots?

This avoids the complexity of:

- maintaining a second VM;
- paying for a second VM continuously;
- VM boot time;
- configuring load balancers;
- managing VM images;
- configuring health probes;
- managing VM networking;
- manually updating standby servers.

Azure App Service deployment slots are designed to support staging and swapping application versions. Microsoft documents slot swaps as a way to move a warmed and validated application into production. citeturn0search2turn0search1

Deployment slots are available on supported App Service tiers; the project should select a tier that supports the required slot. citeturn0search2

---

# 25. What exactly does Terraform do during recovery?

Terraform should own the **desired infrastructure configuration**.

For the recovery environment, Terraform can manage:

- the App Service;
- the deployment slot;
- slot configuration;
- application settings;
- monitoring configuration;
- supporting Azure resources.

The important distinction is:

> **Terraform does not need to recreate the application from scratch during every incident.**

The recovery slot already exists.

The recovery workflow uses the prepared slot and performs the required slot transition.

Microsoft provides a Terraform deployment-slot example specifically demonstrating using Terraform to provision slots and swap between them. citeturn0search6

---

# 26. Recovery Terraform Concept

A simplified Terraform structure:

```text
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
└── recovery-plans/
    └── payment-service-failover/
        ├── main.tf
        └── variables.tf
```

The infrastructure configuration can create the App Service and slot.

Conceptually:

```hcl
resource "azurerm_linux_web_app" "payment_api" {
  name                = var.app_name
  resource_group_name = var.resource_group_name
  location            = var.location
  service_plan_id     = var.service_plan_id

  site_config {
    application_stack {
      node_version = "22-lts"
    }
  }
}

resource "azurerm_linux_web_app_slot" "recovery" {
  name           = "recovery"
  app_service_id = azurerm_linux_web_app.payment_api.id

  site_config {
    application_stack {
      node_version = "22-lts"
    }
  }
}
```

The exact application-stack configuration should be changed to match the actual runtime selected by the software team.

---

# 27. Slot Swap

The recovery operation is conceptually:

```text
BEFORE

Production → Version A
Recovery   → Version B


AFTER SWAP

Production → Version B
Recovery   → Version A
```

This is important:

> A slot swap changes which version is serving production traffic. It does not mean Terraform creates a new server during the incident.

Azure supports swapping a deployment slot with production through its management APIs/CLI. citeturn0search14turn0search2

For the capstone, the recovery workflow can invoke the Azure slot swap operation after approval.

Example Azure CLI operation:

```bash
az webapp deployment slot swap \
  --resource-group <resource-group> \
  --name <app-name> \
  --slot recovery \
  --target-slot production
```

Microsoft documents this exact slot-swap operation. citeturn0search2

### Important implementation note

The team should test the exact Terraform/provider implementation before the final demo. Current AzureRM supports Linux Web App slots, and Microsoft's current Terraform documentation includes deployment-slot provisioning and swapping as a supported workflow. citeturn0search7turn0search6

For a four-day project, it is acceptable for the GitHub Actions recovery workflow to invoke the Azure slot-swap API/CLI while Terraform remains the source of truth for the infrastructure configuration. The key is that the recovery operation is version-controlled, repeatable, auditable, and only triggered after approval.

---

# 28. Recovery Sequence

The complete recovery flow becomes:

```text
1. Failure mode enabled
        ↓
2. Payment requests begin failing
        ↓
3. Application Insights records failed requests
        ↓
4. Azure Monitor observes error rate
        ↓
5. Error rate crosses configured threshold
        ↓
6. Azure Monitor fires alert
        ↓
7. Logic App starts
        ↓
8. Logic App queries Log Analytics
        ↓
9. Logic App builds incident context
        ↓
10. Logic App sends context to Azure OpenAI
        ↓
11. AI returns analysis + recommendation
        ↓
12. Recommendation is made available to reviewer
        ↓
13. Human reviews evidence
        ↓
14. Human clicks APPROVE
        ↓
15. GitHub Actions recovery workflow starts
        ↓
16. Recovery workflow executes approved recovery plan
        ↓
17. App Service recovery slot becomes production
        ↓
18. Wait for stabilization
        ↓
19. GET /health
        ↓
20. Test payment
        ↓
21. Recovery confirmed
```

---

# 29. Recovery Verification

Recovery is not considered successful simply because the Terraform/GitHub Actions job succeeded.

The system should verify the application itself.

Minimum checks:

### Check 1 — Health endpoint

```text
GET /health
```

Expected:

```text
HTTP 200
status = healthy
```

### Check 2 — Payment transaction

```text
POST /api/payments
```

Expected:

```text
HTTP 200/201
status = SUCCESS
```

### Optional Check 3 — Error rate

Verify that the error rate has returned below the alert threshold.

Example:

```text
Before recovery:
Error rate = 18%

After recovery:
Error rate = 1%
```

---

# 30. Recovery Result

If all checks pass:

```text
┌─────────────────────────────┐
│     RECOVERY SUCCESSFUL     │
│                             │
│ /health        ✓            │
│ Test payment   ✓            │
│ Error rate     ✓            │
│                             │
│ Incident: RESOLVED          │
└─────────────────────────────┘
```

If checks fail:

```text
┌─────────────────────────────┐
│     RECOVERY FAILED         │
│                             │
│ /health        ✓            │
│ Test payment   ✗            │
│ Error rate     ✗            │
│                             │
│ Manual intervention needed  │
└─────────────────────────────┘
```

---

# 31. End-to-End Example

This is the scenario the team should build and test.

### Normal state

```text
Payment API
Error rate: 1%
Status: HEALTHY
```

### Failure triggered

Operator calls:

```text
POST /api/failure
```

Application enters failure mode.

Payment requests begin returning HTTP 500.

### Monitoring

Application Insights records:

```text
Failed requests ↑
Response time ↑
Exceptions ↑
```

Azure Monitor calculates:

```text
Error rate = 18%
```

Configured threshold:

```text
10%
```

Alert fires.

### Automation

Logic App:

```text
Alert received
     ↓
Query last 15 minutes of telemetry
     ↓
Build incident context
     ↓
Send to Azure OpenAI
```

### AI

AI returns:

```json
{
  "severity": "HIGH",
  "confidence": 0.91,
  "recommendedAction": "FAILOVER_PAYMENT_SERVICE"
}
```

### Human

Engineer reviews:

```text
Error rate: 18%
Likely cause: Payment service failure
Confidence: 91%
Recommendation: FAILOVER_PAYMENT_SERVICE
```

Engineer clicks:

```text
APPROVE
```

### Recovery

GitHub Actions:

```text
Checkout
    ↓
Azure Login
    ↓
Terraform
    ↓
Activate/swap recovery slot
```

### Verification

```text
GET /health → 200
POST /api/payments → SUCCESS
Error rate → normal
```

Result:

```text
🟢 SERVICE RESTORED
```

---

# 32. Final Simplified Architecture

The architecture can therefore be summarized as:

```text
                    ┌─────────────┐
                    │    USER     │
                    └──────┬──────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │ Banking Frontend│
                  └────────┬────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │ Payment API     │
                  │ Azure App       │
                  │ Service         │
                  └────────┬────────┘
                           │
                      Telemetry
                           │
                           ▼
              ┌─────────────────────────┐
              │ Application Insights    │
              │ Azure Monitor           │
              │ Log Analytics           │
              └────────────┬────────────┘
                           │
                     Alert fires
                           │
                           ▼
                  ┌─────────────────┐
                  │ Azure Logic App │
                  └────────┬────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │   Azure OpenAI  │
                  │   AI Analysis   │
                  └────────┬────────┘
                           │
                    Recommendation
                           │
                           ▼
                  ┌─────────────────┐
                  │ Incident Review │
                  │     Page        │
                  └────────┬────────┘
                           │
                        APPROVE
                           │
                           ▼
                  ┌─────────────────┐
                  │ GitHub Actions  │
                  └────────┬────────┘
                           │
                           ▼
                     ┌──────────┐
                     │ Terraform│
                     └────┬─────┘
                          │
                          ▼
                ┌─────────────────────┐
                │ App Service Recovery│
                │        Slot         │
                └──────────┬──────────┘
                           │
                           ▼
                    Health Checks
                           │
                    ┌──────┴──────┐
                    ▼             ▼
                 SUCCESS        FAILURE
                    │             │
                    ▼             ▼
                RESOLVED      MANUAL REVIEW


        ────────────────────────────────────
                 POWER BI
        ────────────────────────────────────
        Reads monitoring data from
        Log Analytics for visualization
        and operational reporting.
