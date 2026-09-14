# Cloud-Native Disaster Recovery Platform

## AI-Assisted Disaster Recovery for Banking Services

## Overview

This document specifies the architecture for a cloud-native disaster recovery platform that demonstrates AI-assisted incident triage and recovery recommendation for banking services. The system combines Azure cloud monitoring, LLM-based analysis, human-in-the-loop approval, and Infrastructure as Code execution to automate disaster recovery workflows while maintaining safety and auditability. This RFC defines the complete technical architecture, implementation scope, security boundaries, and realistic constraints for a 5-day student capstone project.

## 1. Project Description

The Cloud-Native Disaster Recovery Platform consists of a simulated banking application with comprehensive operations tooling. At its core is a simple payment processing API representing a critical banking service deployed on Azure infrastructure.

The platform operates through a coordinated pipeline: Azure Monitor continuously observes the banking application and underlying infrastructure, collecting metrics, logs, and health signals. When service degradation or failure occurs, configured alert rules detect threshold violations and trigger an incident workflow. An incident record is created, capturing alert metadata and telemetry context.

The AI analysis subsystem retrieves relevant logs, metrics, and historical patterns from Azure Log Analytics. This contextual data is sent to Azure OpenAI Service (GPT-4o), which performs structured analysis to determine incident severity, identify probable root cause, assess business impact, and recommend an appropriate recovery action from a predefined catalog.

Critically, the AI does not generate infrastructure code dynamically. Instead, it selects from a known set of recovery plans that have been pre-authored, peer-reviewed, and tested by the engineering team. Each recovery plan is a versioned Terraform or Bicep configuration stored in source control.

A human operator reviews the AI recommendation through an operations dashboard that presents the complete evidence package: raw telemetry, AI analysis, severity assessment, confidence score, and the specific recovery plan being proposed. The operator exercises judgment and either approves or rejects the recommendation.

Upon approval, the system triggers a secure IaC pipeline that executes the predefined recovery plan. The pipeline applies infrastructure changes using least-privilege credentials. After execution, automated health checks verify that services have been restored. The incident lifecycle is tracked end-to-end with full audit trails.

This architecture demonstrates the integration of Cloud Engineering, Observability, AI-assisted decision support, Human-in-the-Loop controls, and Infrastructure as Code in a realistic disaster recovery context.

## 2. Business Problem

Banking and financial services operate under strict availability requirements. Service outages directly impact customer transactions, regulatory compliance, and institutional reputation. Traditional disaster recovery approaches face several limitations:

**Manual Triage Inefficiency:** When incidents occur, on-call engineers must manually correlate logs across multiple systems, interpret metrics, and identify patterns. This investigation phase can consume 20-40 minutes during critical outages.

**Knowledge Fragmentation:** Recovery procedures often exist as tribal knowledge, wiki pages, or outdated runbooks. New team members lack context. Senior engineers become bottlenecks.

**Error-Prone Execution:** Manual recovery steps introduce human error risk. Typing mistakes, wrong resource selection, or incomplete procedures can extend outages or cause cascading failures.

**Inconsistent Response:** Different operators may choose different recovery approaches for similar incidents, leading to unpredictable outcomes and difficulty in process improvement.

**Audit and Compliance Gaps:** Manual processes generate sparse audit trails, complicating post-incident reviews and regulatory reporting.

This platform addresses these problems by demonstrating how AI can accelerate the incident triage phase while maintaining human decision authority. It bridges multiple domains—Cloud Engineering, Monitoring, Infrastructure as Code, Disaster Recovery, AI-assisted analysis, and Human-in-the-Loop automation—in a cohesive system that is both more efficient and more auditable than purely manual approaches.

The capstone scope focuses on proving the architecture pattern rather than building production-scale infrastructure, making it an ideal vehicle for demonstrating these concepts in an academic context.

## 3. Minimum Viable Architecture

The MVP architecture consists of seven interconnected layers:

### Simulated Banking Application Layer

A lightweight payment processing API built with Node.js (Express) or Python (FastAPI). The API exposes 2-3 endpoints: POST /api/payments (create payment transaction), GET /api/payments/:id (retrieve transaction status), and GET /health (health check). The service is deployed on Azure App Service or Azure Container Apps with deployment slot configuration enabled. State persistence uses Azure PostgreSQL Flexible Server. The API generates structured application logs and custom metrics that feed into Application Insights.

### Observability and Monitoring Layer

Azure Monitor serves as the central observability platform. Application Insights instruments the payment API, collecting request telemetry, dependency tracking, and custom events. A Log Analytics workspace aggregates logs from all Azure resources. Azure Monitor Metrics captures resource-level signals (CPU, memory, request rate, error rate, latency percentiles). Alert rules are configured with threshold-based conditions—for example, error rate exceeds 10% over a 5-minute window, or availability drops below 95%.

### Alert and Incident Pipeline

When an alert rule fires, it triggers an action group configured with a webhook endpoint. The webhook target is an Azure Function or Logic App that serves as the incident receiver. This function validates the alert payload, creates an incident record in PostgreSQL with status DETECTED, and stores the raw alert data as JSON. The incident creation includes timestamp, alert metadata, and initial severity classification from the alert rule. This decouples incident management from the monitoring system.

### Backend Control Plane

A RESTful API service (Node.js/Express or Python/FastAPI) deployed on Azure App Service that orchestrates the entire recovery workflow. Core responsibilities include:

- **Incident Management:** CRUD operations on incident records
- **Telemetry Retrieval:** Executes KQL queries against Log Analytics to fetch relevant logs and metrics for an incident time window
- **AI Analysis Orchestration:** Constructs prompts with incident context and invokes Azure OpenAI Service
- **Recommendation Validation:** Validates AI response structure and maps recommended actions to known recovery plans
- **Approval Workflow:** Handles operator approval/rejection with audit logging
- **Recovery Execution:** Triggers GitHub Actions workflows via REST API with approved plan identifiers
- **Status Polling:** Monitors pipeline execution status
- **Health Verification:** Executes post-recovery health checks
- **History and Audit:** Provides query endpoints for incident history

The backend enforces all business logic and security boundaries. It is the single source of truth for incident state.

### AI Analysis Service

Azure OpenAI Service with GPT-4o model access. The backend constructs a structured prompt containing:

- Incident alert details (alert rule name, fired threshold, resource affected)
- Recent error logs from Log Analytics (last 15-30 minutes)
- Relevant metric values (error rate trend, latency spike, availability drop)
- Catalog of available recovery actions with brief descriptions

For retrieval-augmented generation (RAG), a lightweight approach is used: embed 1-2 recovery runbook documents into Azure AI Search, or simply include them in the prompt context window since the catalog is small. The LLM analyzes patterns, correlates signals, and returns structured JSON adhering to a defined schema (see Section 5). Temperature is set to 0.3 for deterministic output. The response parser validates schema compliance before persisting to the database.

### Recovery Execution Layer

Predefined infrastructure configurations stored in a Git repository (e.g., `disaster-recovery-platform/recovery-plans/`). Each recovery plan is a directory containing Terraform or Bicep files:

```
recovery-plans/
├── payment-service-failover/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── database-failover/
    ├── main.tf
    └── variables.tf

```

GitHub Actions workflows are configured to accept a `recovery-plan` parameter. After human approval, the backend calls the GitHub Actions API with the plan identifier and incident ID. The workflow checks out the repository, navigates to the specified plan directory, initializes Terraform, and applies the configuration using a dedicated Azure Service Principal. Terraform state is stored in Azure Storage with locking enabled. The workflow reports success/failure status back via API or logs that the backend polls.

### Operations Dashboard Frontend

A single-page application built with React (Vite) or Next.js. The UI is organized into views:

- **Dashboard View:** Service health cards, active incident count, recent metrics charts
- **Incident Detail View:** Full incident context including raw logs, metrics visualization, AI analysis summary, severity badge, confidence indicator, recommended action card
- **Approval Controls:** Prominent approve/reject buttons with operator input field for rejection notes
- **Recovery Status View:** Real-time pipeline execution progress, logs from GitHub Actions, health check results
- **History View:** Paginated list of past incidents with filtering by status, severity, date range

The frontend polls backend APIs for status updates (3-5 second intervals during active recovery). Authentication can use Azure AD for operator identity. The dashboard is deployed as a static site on Azure Storage with Static Website hosting or bundled with the backend App Service.

### Database Schema

Azure PostgreSQL Flexible Server with three core tables:

**incidents**

- `id` (UUID, primary key)
- `title` (VARCHAR)
- `status` (ENUM: DETECTED, ANALYZING, PENDING_APPROVAL, APPROVED, RECOVERING, RESOLVED, REJECTED, RECOVERY_FAILED)
- `severity` (ENUM: CRITICAL, HIGH, MEDIUM, LOW)
- `detected_at` (TIMESTAMP)
- `resolved_at` (TIMESTAMP, nullable)
- `alert_data` (JSONB)

**incident_analyses**

- `id` (UUID, primary key)
- `incident_id` (UUID, foreign key)
- `ai_response_json` (JSONB)
- `recommended_action` (VARCHAR)
- `confidence` (DECIMAL)
- `created_at` (TIMESTAMP)

**recovery_actions**

- `id` (UUID, primary key)
- `incident_id` (UUID, foreign key)
- `plan_name` (VARCHAR)
- `status` (ENUM: PENDING_APPROVAL, APPROVED, REJECTED, EXECUTING, COMPLETED, FAILED, VERIFIED)
- `approved_by` (VARCHAR, nullable)
- `approved_at` (TIMESTAMP, nullable)
- `execution_id` (VARCHAR, nullable, GitHub Actions run ID)
- `started_at` (TIMESTAMP, nullable)
- `completed_at` (TIMESTAMP, nullable)
- `health_check_result` (JSONB, nullable)

Telemetry data (logs, metrics) remains in Azure Monitor and Log Analytics as the source of truth; only incident lifecycle metadata and AI analysis results are stored in PostgreSQL to avoid data duplication and maintain separation of concerns.

## 4. Architecture Diagram

The complete system architecture is visually represented in the companion architecture diagram on the Miro board. The diagram illustrates:

- Azure resources topology (App Services, PostgreSQL, Monitor, OpenAI Service)
- Data flow paths from alert detection through AI analysis to recovery execution
- Human operator touchpoints and approval gates
- IaC pipeline integration with GitHub Actions
- Feedback loops for health verification and incident resolution

Refer to the board diagram for component placement, network boundaries, and end-to-end flow visualization. The textual sections in this RFC provide the implementation details that correspond to each architectural component shown in the diagram.

## 5. Exact Role of AI

The AI subsystem performs analytical and advisory functions only. It does not have direct access to Azure Resource Manager APIs, cannot execute infrastructure changes, and cannot bypass human approval.

### AI Capabilities

**Incident Summarization:** Condenses raw alert payloads, log entries, and metric time series into a human-readable incident summary. Example: "Payment API is experiencing elevated failure rates, with 18% of requests returning 500 errors over the past 10 minutes."

**Root-Cause Analysis:** Identifies the most probable cause by analyzing error patterns, dependency traces, and resource metrics. Example: "Primary application environment appears unhealthy. Application Insights shows increased exception rates in the payment processing module correlating with database connection timeouts."

**Severity Assessment:** Classifies incidents into severity tiers (CRITICAL, HIGH, MEDIUM, LOW) based on impact signals such as error rate magnitude, affected user count, and service criticality.

**Impact Analysis:** Describes business and technical consequences. Example: "Payment transactions are failing for end users. Estimated impact: 200\+ failed transactions per minute. Customer-facing checkout flows are degraded."

**Recovery Recommendation:** Selects the most appropriate predefined recovery plan from the known catalog based on incident characteristics. Example: Recommends FAILOVER_PAYMENT_SERVICE when the primary payment API environment is unhealthy but standby capacity exists.

**Confidence Scoring:** Provides a decimal confidence score (0.0 to 1.0) indicating the AI's certainty in its analysis and recommendation, based on clarity of telemetry signals and pattern match strength.

### Structured Output Format

The AI returns a JSON object conforming to this schema:

```json
{
  "severity": "HIGH",
  "summary": "Payment API is experiencing elevated failure rates with 18% of requests failing.",
  "likelyCause": "Primary application environment is unhealthy. Database connection pool exhaustion detected.",
  "confidence": 0.91,
  "impact": "Payment transactions are failing for end users. Estimated 200+ failed transactions per minute.",
  "recommendedAction": "FAILOVER_PAYMENT_SERVICE",
  "recoveryPlan": "payment-service-failover"
}

```

**Field Definitions:**

- `severity`: One of CRITICAL, HIGH, MEDIUM, LOW
- `summary`: Brief incident description (1-2 sentences)
- `likelyCause`: Probable root cause (2-3 sentences)
- `confidence`: Decimal between 0.0 and 1.0
- `impact`: Business and technical consequences
- `recommendedAction`: Action identifier from the predefined catalog
- `recoveryPlan`: Specific IaC plan name to execute

The backend validates this schema and rejects responses that don't conform. Invalid responses trigger manual review workflows rather than automated execution.

### What AI Explicitly Does NOT Do

- Generate Terraform, Bicep, or any infrastructure code dynamically
- Execute Azure CLI commands or API calls
- Access Azure Resource Manager, subscription, or resource group APIs
- Make autonomous infrastructure changes without human approval
- Have credentials to any Azure resources
- Store or persist incident data (read-only access to telemetry via backend queries)
- Interact directly with GitHub Actions or IaC pipelines
- Override or bypass approval workflows
- Learn or retrain models (uses pre-trained GPT-4o via API)

The AI is a read-only analytical service. All execution authority resides with the backend control plane after explicit human authorization.

## 6. Role of IaC

Infrastructure as Code serves as the deterministic, auditable execution engine for all recovery operations. IaC provides several critical properties for disaster recovery:

### Pre-Authored and Tested Recovery Plans

Every recovery action is implemented as a Terraform or Bicep configuration that is written, peer-reviewed, and tested BEFORE production deployment. This ensures:

- **Correctness:** Recovery logic is validated in staging environments
- **Repeatability:** The same input configuration produces identical infrastructure changes
- **Auditability:** All changes are version-controlled with commit history
- **Rollback Capability:** Git history enables reverting to previous states if needed

### Recovery Plan Structure

Each recovery plan is stored as a self-contained directory in the repository:

```
recovery-plans/payment-service-failover/
├── main.tf
├── variables.tf
├── outputs.tf
└── README.md

```

Example Terraform configuration for payment service failover:

```
# main.tf
resource "azurerm_app_service_slot" "payment_api_failover" {
  name                = "failover"
  app_service_name    = var.app_service_name
  location            = var.location
  resource_group_name = var.resource_group_name
  app_service_plan_id = var.app_service_plan_id
}

resource "azurerm_app_service_slot_swap" "activate_failover" {
  app_service_name    = var.app_service_name
  resource_group_name = var.resource_group_name
  source_slot         = "failover"
  target_slot         = "production"
}

```

### Student Implementation Scope

For the 5-day capstone, students write 1-2 complete recovery plans:

**Plan 1 (Required): Payment Service Failover**

- Provisions or activates a standby deployment slot for the payment API
- Updates traffic routing to direct requests to healthy capacity
- Validation: 20-30 lines of Terraform

**Plan 2 (Stretch Goal): Database Connectivity Restore**

- Re-applies correct firewall rules or network security group rules
- Or promotes a PostgreSQL read replica to primary
- Validation: 15-25 lines of Terraform

Writing more plans is explicitly out of scope. The goal is to demonstrate the pattern, not build a comprehensive runbook library.

### Execution Safety Properties

IaC ensures that recovery execution is:

- **Idempotent:** Re-running the same plan produces the same result without side effects
- **Atomic:** Changes are applied transactionally (Terraform state locking)
- **Scoped:** Service Principal permissions restrict which resources can be modified
- **Traced:** Terraform output logs provide detailed execution audit trail

By constraining recovery to pre-approved IaC configurations, the system prevents the AI from making arbitrary infrastructure changes while maintaining the speed and consistency benefits of automation.

## 7. Human Approval Workflow

The human approval workflow is a REAL control gate, not a cosmetic interface element. It enforces organizational policy that critical infrastructure changes require explicit human authorization.

### Approval Workflow Steps

**1. Incident Detection and AI Analysis** The backend detects or receives an incident, retrieves telemetry, and invokes AI analysis. The AI recommendation is persisted to the `incident_analyses` table with the incident still in ANALYZING status.

**2. Recommendation Storage** The backend creates a record in the `recovery_actions` table with:

- `incident_id`: Link to parent incident
- `plan_name`: The recommended recovery plan identifier
- `status`: PENDING_APPROVAL
- `approved_by`: NULL
- `approved_at`: NULL

The incident status transitions to PENDING_APPROVAL.

**3. Evidence Package Presentation** The frontend fetches the complete incident context via API and displays:

- Alert rule that fired and threshold violation details
- Relevant error logs from the incident time window (formatted with timestamps and severity)
- Metric charts showing the failure pattern (error rate spike, latency increase)
- AI analysis summary with severity badge and confidence indicator
- Identified likely cause with supporting evidence references
- Impact assessment describing business consequences
- Recommended recovery action with plain-language description
- Specific recovery plan name and what infrastructure changes it will apply

**4. Operator Review** The operator examines the evidence package and exercises judgment:

- Does the AI analysis align with the observed symptoms?
- Is the severity classification appropriate?
- Is the recommended action the correct response?
- Are there business considerations (e.g., ongoing maintenance window) that should prevent automatic recovery?
- Is the confidence score sufficiently high?

**5. Approval Action** The operator clicks **APPROVE** or **REJECT**.

**On APPROVE:**

- Frontend sends POST /api/incidents/:id/recovery-actions/:actionId/approve with operator identity
- Backend performs validation checks:
    - Incident is still in PENDING_APPROVAL status
    - Recovery action exists and is still PENDING_APPROVAL
    - Recovery plan identifier maps to a known IaC configuration
    - No duplicate approval has been recorded (idempotency)
- Backend updates recovery_action record:
    - `status` → APPROVED
    - `approved_by` → Operator username/email
    - `approved_at` → Current timestamp
- Backend triggers IaC pipeline execution (see Section 9)
- Incident status transitions to RECOVERING

**On REJECT:**

- Frontend sends POST /api/incidents/:id/recovery-actions/:actionId/reject with operator note
- Backend updates recovery_action record:
    - `status` → REJECTED
    - Stores rejection reason in notes field
- Incident status transitions to REJECTED
- Operator receives notification to handle incident manually
- Incident remains in system for audit and post-mortem analysis

### Approval Security Enforcement

The frontend CANNOT bypass the backend to trigger infrastructure changes. Key security properties:

- Frontend has no direct access to GitHub Actions API tokens
- Frontend has no credentials to Azure Service Principal
- IaC pipelines only execute when backend sends valid, approved execution requests
- Backend validates approval state before every pipeline trigger
- Audit log records all approval/rejection events with operator identity and timestamp

This multi-layer validation ensures that no infrastructure changes occur without explicit human authorization captured in the audit trail.

## 8. AI Recommendation to Recovery Plan Mapping

The mapping between AI recommendations and executable recovery plans is implemented as a static configuration table. This provides deterministic, auditable translation from AI analysis to infrastructure action.

### Mapping Table Structure

*[Table](https://miro.com/app/board/uXjVHnGy4Oc=/?moveToWidget=3458764683631478520&cot=14)*

| AI recommendedAction | Recovery Plan ID | IaC Path | Description |
| --- | --- | --- | --- |
| FAILOVER_PAYMENT_SERVICE | payment-service-failover | recovery-plans/payment-service-failover/ | Activates standby payment API deployment slot and swaps traffic routing to healthy environment |
| RESTORE_DB_CONNECTIVITY | database-connectivity-restore | recovery-plans/database-connectivity-restore/ | Re-applies correct database firewall rules or network security group configuration |
|  |  |  |  |

### Validation Logic

When the backend receives an AI recommendation:

1. **Extract Recommendation:** Parse `recommendedAction` field from AI JSON response
2. **Lookup Mapping:** Query the mapping table for a matching entry
3. **Validation:**
    - If match found: Extract `recovery_plan_id` and `iac_path`
    - If NO match found: Log error, set incident status to MANUAL_REVIEW_REQUIRED, alert operator
4. **Verification:** Confirm that the IaC path exists in the Git repository (optional pre-flight check)
5. **Storage:** Store validated `plan_name` in the recovery_actions table

### Safety Boundary

This mapping table is a critical safety control. It ensures that:

- AI cannot recommend arbitrary actions outside the predefined catalog
- Typos or hallucinations in AI output are caught before execution
- The system fails safe: unrecognized actions trigger manual review rather than proceeding with unknown operations
- Recovery plans can be added/removed by updating the mapping table without modifying AI prompts

For the student capstone, the mapping table contains 1-2 entries corresponding to the implemented recovery plans. This simplicity makes the validation logic straightforward and testable.

## 9. Safe IaC Execution After Approval

IaC execution follows a secure, multi-step workflow that maintains audit trails and enforces least-privilege access.

### Execution Workflow

**Step 1: Approval Validation** After the operator approves a recovery action, the backend validates:

- Recovery action record exists and is in APPROVED status
- Incident is still active (not already resolved)
- No duplicate execution has been initiated (check for existing `execution_id`)

**Step 2: Pipeline Trigger** Backend calls the GitHub Actions API to trigger a workflow run:

```
POST https://api.github.com/repos/{owner}/{repo}/actions/workflows/{workflow_id}/dispatches
Authorization: Bearer {GITHUB_TOKEN}
Content-Type: application/json

{
  "ref": "main",
  "inputs": {
    "recovery_plan": "payment-service-failover",
    "incident_id": "123e4567-e89b-12d3-a456-426614174000",
    "approval_token": "{signed_token}"
  }
}

```

**Workflow Input Parameters:**

- `recovery_plan`: Identifier matching the mapping table
- `incident_id`: UUID for audit correlation
- `approval_token`: Signed JWT containing approval metadata, preventing replay attacks

**Step 3: GitHub Actions Workflow Execution** The workflow (`.github/workflows/recovery-execution.yml`) performs:

```yaml
name: Recovery Execution
on:
  workflow_dispatch:
    inputs:
      recovery_plan:
        required: true
      incident_id:
        required: true
      approval_token:
        required: true

jobs:
  execute-recovery:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Validate approval token
        run: |
          # Verify JWT signature and expiration
          # Decode and validate incident_id matches

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Terraform Init
        working-directory: recovery-plans/${{ inputs.recovery_plan }}
        run: terraform init

      - name: Terraform Apply
        working-directory: recovery-plans/${{ inputs.recovery_plan }}
        run: terraform apply -auto-approve -var="incident_id=${{ inputs.incident_id }}"

      - name: Report Status
        if: always()
        run: |
          # Call backend API to update recovery_action status
          # Include execution logs and outputs

```

**Step 4: Status Polling** Backend polls the GitHub Actions API for run status:

- Query `/repos/{owner}/{repo}/actions/runs/{run_id}` every 15-30 seconds
- Parse `status` field: queued, in_progress, completed
- Parse `conclusion` field: success, failure, cancelled
- Update recovery_action record with real-time status

**Step 5: Execution Completion** When workflow completes:

- Backend fetches execution logs via API
- Updates recovery_action record:
    - `completed_at` → Completion timestamp
    - `status` → COMPLETED or FAILED based on workflow conclusion
- Proceeds to recovery verification (Section 10)

### Security Controls

**Least-Privilege Service Principal** The Azure Service Principal used by GitHub Actions has RBAC permissions scoped to:

- App Service: Contributor role on the payment API App Service only
- PostgreSQL: No access (database changes are out of scope)
- Storage Account: Contributor role only on the Terraform state storage account
- No subscription-level or resource group-level permissions

**Credential Management**

- Service Principal credentials stored in GitHub Secrets, never in code
- Database connection strings stored in Azure Key Vault
- Key Vault access granted to App Services via Managed Identity
- GitHub Personal Access Token (for Actions API) rotated quarterly

**Execution Audit Trail**

- GitHub Actions logs captured and retained for 90 days
- Backend stores execution_id linking to specific workflow run
- All Terraform state changes recorded in Azure Storage with versioning enabled
- recovery_actions table provides queryable audit history

**Idempotency Enforcement**

- Backend checks for existing execution_id before triggering new run
- Terraform state locking prevents concurrent executions of the same plan
- Approval tokens include nonce and expiration to prevent replay

**Error Handling**

- If pipeline fails: Status set to FAILED, operator alerted for manual intervention
- If API call to GitHub fails: Backend retries with exponential backoff (max 3 attempts)
- If status polling times out: Recovery action marked UNKNOWN, manual verification required

This multi-layered approach ensures that IaC execution is both automated and secure, with clear accountability for every infrastructure change.

## 10. Recovery Verification

After IaC execution completes successfully, the system must verify that the recovered infrastructure is actually healthy and the incident is resolved.

### Verification Workflow

**Step 1: Cool-Down Period** Backend waits a configurable stabilization period (default: 60 seconds) to allow:

- New App Service slot to fully warm up
- Application startup to complete
- Health check endpoints to become responsive
- DNS/routing changes to propagate

**Step 2: Health Check Execution** Backend executes a suite of health checks:

**HTTP Health Endpoint Check:**

```
GET https://{payment-api-domain}/health
Expected: 200 OK
Response body: { "status": "healthy", "timestamp": "..." }
Timeout: 10 seconds

```

**Database Connectivity Check:**

```
SELECT 1 FROM incidents LIMIT 1;
Expected: Query succeeds
Timeout: 5 seconds

```

**Synthetic Transaction Test (Optional for Stretch Goal):**

```
POST https://{payment-api-domain}/api/payments
Body: { "amount": 1.00, "currency": "USD", "test": true }
Expected: 200 or 201 with transaction ID
Timeout: 15 seconds

```

Each check is executed with retry logic (3 attempts with 10-second intervals) to handle transient failures during stabilization.

**Step 3: Result Evaluation** Backend aggregates health check results into a structured JSON object:

```json
{
  "timestamp": "2026-09-14T10:30:00Z",
  "overall_status": "PASSED",
  "checks": [
    {
      "name": "http_health_endpoint",
      "status": "PASSED",
      "response_time_ms": 124,
      "details": "HTTP 200, response body indicates healthy"
    },
    {
      "name": "database_connectivity",
      "status": "PASSED",
      "response_time_ms": 45,
      "details": "Database query succeeded"
    },
    {
      "name": "synthetic_transaction",
      "status": "PASSED",
      "response_time_ms": 312,
      "transaction_id": "txn_abc123",
      "details": "Test payment processed successfully"
    }
  ]
}

```

**Step 4: Status Update**

**If All Checks Pass:**

- recovery_actions.status → VERIFIED
- recovery_actions.health_check_result → Store JSON result
- incidents.status → RESOLVED
- incidents.resolved_at → Current timestamp
- Dashboard shows green success indicator

**If Any Check Fails:**

- recovery_actions.status → FAILED
- recovery_actions.health_check_result → Store JSON result with failure details
- incidents.status → RECOVERY_FAILED
- Trigger operator alert notification
- Dashboard shows red failure indicator with check details
- Operator initiates manual troubleshooting

**Step 5: Continuous Monitoring (Post-Recovery)** For 15 minutes after resolution, the backend can optionally poll Azure Monitor metrics to confirm sustained stability:

- Error rate remains below threshold
- Request latency is within normal range
- No new alerts have fired

If degradation recurs during this window, the incident is automatically reopened with status RECURRING_ISSUE.

### Health Check Configuration

Health checks are configurable via backend environment variables:

```
HEALTH_CHECK_COOLDOWN_SECONDS=60
HEALTH_CHECK_RETRY_ATTEMPTS=3
HEALTH_CHECK_RETRY_DELAY_SECONDS=10
HEALTH_CHECK_TIMEOUT_SECONDS=30
POST_RECOVERY_MONITORING_MINUTES=15

```

This allows tuning for different failure scenarios without code changes.

### Failure Scenarios

**Partial Recovery:** If HTTP checks pass but synthetic transaction fails, the system marks recovery as PARTIAL. Operator reviews to determine if this is acceptable (e.g., service is available but degraded).

**False Positive:** If health checks pass but users still report issues, the operator can manually reopen the incident. This highlights the importance of health check comprehensiveness.

**Timeout:** If health checks don't complete within 5 minutes, recovery is marked TIMEOUT and requires manual intervention.

The verification layer ensures that "successful IaC execution" is not confused with "actual service recovery." This distinction is critical for production reliability.

## 11. Recommended Azure Services

The architecture uses a minimal set of Azure services, each chosen for specific justification aligned with capstone constraints (5-day timeline, student budget, learning objectives).

*[Table](https://miro.com/app/board/uXjVHnGy4Oc=/?moveToWidget=3458764683631706953&cot=14)*

| Service | Purpose | Justification |
| --- | --- | --- |
| Azure App Service (x2 instances) | Host the payment API and backend control plane | App Service provides managed compute with built-in deployment slots (enabling failover scenarios), automatic scaling, and integrated Application Insights. Students avoid container orchestration complexity. Standard tier includes staging slots for recovery demos. |
| Azure PostgreSQL Flexible Server | Application database for incident records and audit data | Managed relational database with flexible scaling, automated backups, and built-in high availability options (zone redundancy). Familiar SQL interface. Avoids NoSQL learning curve. |
| Azure Monitor + Application Insights | Metrics collection, log aggregation, and alerting | Native Azure observability with zero additional setup. Application Insights auto-instruments Node.js/Python apps. Log Analytics provides KQL query interface for incident context retrieval. Alert rules integrate directly with Action Groups. |
| Log Analytics Workspace | Centralized log query and analysis | Aggregates logs from all Azure resources into queryable format. KQL enables powerful log correlation for incident investigation. Required backend for Application Insights. |
| Azure OpenAI Service | GPT-4o LLM for incident analysis | Managed LLM service with structured output support. Students avoid managing model hosting. GPT-4o provides strong reasoning for log analysis. East US 2 region recommended for quota availability. |
| Azure Key Vault | Secrets management | Centralized secure storage for database credentials, API keys, Service Principal secrets, and GitHub tokens. Integrates with App Service via Managed Identity (no credential sprawl). |
| GitHub Actions | IaC pipeline execution engine | Free tier sufficient for capstone. Students already familiar with GitHub. Simple YAML workflow syntax. REST API enables programmatic triggering from backend. Alternative: Azure DevOps Pipelines (if students prefer). |
| Azure Storage Account | Terraform state backend | Remote state storage with blob locking ensures safe concurrent IaC operations. Standard LRS tier adequate. Also can host frontend as static website (optional). |
| Azure AI Search (Optional) | RAG vector search for runbook retrieval | Only if implementing retrieval-augmented generation for runbook context. For 1-2 runbooks, in-context prompting is simpler. Include only if time permits. |

### Explicitly Excluded Services

Services that add unnecessary complexity or cost for a 5-day capstone:

- **Azure Kubernetes Service (AKS):** Over-engineered for simple APIs. App Service is sufficient.
- **Azure Front Door / Traffic Manager:** Multi-region routing unnecessary. Single-region failover via App Service slots is adequate.
- **Azure Event Grid / Service Bus:** Webhook-based alerting is simpler. Asynchronous messaging adds complexity without clear benefit.
- **Azure Functions (except alert receiver):** App Services provide simpler deployment model for persistent backend. Function cold starts complicate demo timing.
- **Azure Container Instances / Container Apps:** Unless students specifically want container experience, App Service is easier.
- **Azure Cosmos DB:** PostgreSQL meets all persistence needs. NoSQL adds learning overhead.

### Cost Optimization

Recommended tiers for student budget (<$50 total for 5-day project):
