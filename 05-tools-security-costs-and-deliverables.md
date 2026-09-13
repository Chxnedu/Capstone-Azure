# Tools, Security, Cost and Final Deliverables

## 1. Recommended toolset

  --------------------------------------------------------------------------
  Area                    Recommended tool           Why
  ----------------------- -------------------------- -----------------------
  Cloud                   Azure                      Team familiarity

  IaC                     Terraform (azurerm)         Simple, readable and
                                                     repeatable

  Compute                 Azure Virtual Machines     Easy to understand for
                                                     the team

  Load balancing          Azure Application Gateway  Simple two-zone
                                                     application path with
                                                     Layer 7 health probes

  Database                Azure Database for         Managed database with
                          PostgreSQL --- Flexible    backup/restore
                          Server                     

  Monitoring              Azure Monitor              Native metrics, logs,
                          (Log Analytics)            alerts and dashboards

  AI                      Azure AI Foundry           Managed
                          (Azure OpenAI Service)     foundation-model access

  AI integration          Azure Functions            Small serverless
                                                     integration

  Evidence                Azure Blob Storage         Cheap/simple artifact
                                                     storage

  Secrets                 Azure Key Vault            Avoid secrets in code

  App                     Python Flask               Very quick to build

  Version control         GitHub/GitLab/Azure Repos  Keep source together
                          already approved by team   

  Diagram                 draw.io / diagrams.net     Fast architecture
                                                     diagram

  Documentation           Markdown                   Easy to maintain

  Testing                 Postman/curl + simple      Fast API testing
                          scripts                    
  --------------------------------------------------------------------------

## 2. What not to add

Avoid adding these unless a reviewer specifically requires them:

-   Azure Kubernetes Service (AKS).
-   Kubernetes.
-   Azure Container Apps (as an orchestration layer for this prototype).
-   Logic Apps / heavy Azure Event Grid orchestration.
-   Azure Cognitive Search as a standalone search product beyond simple
    grounding.
-   Managed Grafana.
-   Azure Machine Learning (full pipelines).
-   Custom vector databases.
-   Complex ML pipelines.
-   Multi-region active-active.
-   Service mesh.

Every extra service adds setup time, cost and failure points.

## 3. Dashboard recommendation

### First choice

**Azure Dashboard / Azure Workbook (in Azure Monitor)**

This is the safest four-day option because it already integrates with
the Azure services being used.

### Second choice

A small HTML dashboard served by the application.

Use this only if the Azure dashboard does not present the story clearly
enough.

### Do not use

Power BI or Azure Managed Grafana unless the team already has access and
experience.

They are unnecessary for the core demonstration.

## 4. Security model

### Identity

Use Azure RBAC with managed identities rather than hard-coded connection
strings or keys wherever possible.

### Network

-   Azure Application Gateway is the public entry point.
-   VMs accept traffic only from the Application Gateway subnet/NSG.
-   Azure Database for PostgreSQL accepts database traffic only from the
    application subnet (via private endpoint or VNet integration).
-   The database should not be publicly accessible.

### Data

Use synthetic records only.

### Secrets

Never put database passwords or Azure credentials in:

-   Git.
-   Terraform variables committed to the repository.
-   Application source.
-   Screenshots.
-   AI prompts.

### AI

Only send approved telemetry to the Azure AI Foundry model.

Do not send:

-   Real customer data.
-   Credentials.
-   Secrets.
-   Production logs.
-   Personal information.

### Recovery

AI recommends.

Human approves.

Script executes.

That is the control boundary.

## 5. Auditability

Capture:

-   Alert timestamp.
-   Failure timestamp.
-   AI response.
-   Evidence supplied to AI.
-   Operator decision.
-   Recovery start.
-   Recovery completion.
-   Verification result.
-   RTO.
-   RPO.
-   Any failed checks.

This creates a simple evidence trail.

## 6. Cost-control approach

The project is intended to be low-cost.

### Main cost risks

-   VMs running continuously.
-   Azure Database for PostgreSQL running continuously.
-   Application Gateway hours.
-   Azure Monitor logs and alerts.
-   Azure AI Foundry / Azure OpenAI model calls.
-   Storage.
-   Public network traffic.

### Controls

-   Use the smallest practical compute/database SKU.
-   Avoid unnecessary traffic.
-   Keep logs for a short period (Log Analytics retention).
-   Keep AI prompts small.
-   Do not run AI continuously every few seconds.
-   Stop/delete resources after testing.
-   Set a budget/cost alert with Azure Cost Management.
-   Tag every resource.
-   Record the deletion date.

Azure Monitor has a free allowance for some metrics, logs and alert
usage, but usage beyond included amounts can incur charges.

Azure AI Foundry / Azure OpenAI pricing varies by model and token usage,
so select a low-cost model suitable for short advisory responses and
verify current pricing before deployment.

Azure Database for PostgreSQL backup/restore is preferable to building
custom database replication for this prototype because it is simpler and
directly demonstrates recoverability. Azure Database for PostgreSQL
Flexible Server supports automated backups, point-in-time restore and
on-demand backups.

## 7. Minimum repository structure

``` text
cloud-dr-capstone/
│
├── app/
│   ├── app.py
│   ├── templates/
│   ├── static/
│   └── requirements.txt
│
├── infra/
│   ├── main.tf
│   ├── networking.tf
│   ├── compute.tf
│   ├── appgateway.tf
│   ├── database.tf
│   ├── monitoring.tf
│   └── variables.tf
│
├── ai/
│   ├── prompt.txt
│   ├── example-events.json
│   └── evaluation.md
│
├── recovery/
│   ├── recovery.sh
│   ├── verification.sh
│   └── runbook.md
│
├── dashboard/
│   └── dashboard-notes.md
│
├── tests/
│   ├── test-plan.md
│   └── test-results.md
│
└── docs/
    ├── architecture.md
    ├── security.md
    └── final-report.md
```

## 8. Final project deliverables

### 1. Architecture diagram

Must show:

-   Users.
-   Azure Application Gateway.
-   Two Availability Zones.
-   Two Virtual Machines.
-   Azure Database for PostgreSQL.
-   Azure Monitor.
-   Azure AI Foundry.
-   Human approval.
-   Recovery workflow.
-   Evidence storage.

### 2. Infrastructure as Code

Must allow the main infrastructure to be reproduced.

### 3. Application

Simple synthetic banking-style workload.

### 4. Disaster recovery runbook

Must explain:

-   Normal state.
-   Failure trigger.
-   Detection.
-   Human decision.
-   Recovery.
-   Verification.
-   RTO/RPO measurement.
-   Rollback/stop condition.
-   Cleanup.

### 5. Monitoring solution

Must show:

-   Health.
-   Errors.
-   Alerts.
-   Recovery state.
-   Readiness.

### 6. AI component

Must show:

-   Input evidence.
-   AI response.
-   Source/runbook grounding.
-   Safety instructions.
-   Evaluation results.
-   Human approval.

### 7. Dashboard

Must communicate the state of the platform clearly.

### 8. Evidence pack

Must contain:

-   Screenshots.
-   Logs.
-   Test results.
-   RTO/RPO.
-   AI evaluation.
-   Operator approval.
-   Final status.

### 9. Final presentation

Recommended story:

**Problem → Architecture → Failure → AI analysis → Human decision →
Recovery → Evidence → Business value → Limitations → Next steps**

## 9. Final limitations to state clearly

This is a capstone prototype.

It demonstrates:

-   DR concepts.
-   Cloud resilience.
-   Monitoring.
-   Recovery testing.
-   AI-assisted decision support.
-   Human oversight.
-   Evidence collection.

It does not prove:

-   Production banking resilience.
-   Regulatory compliance.
-   Enterprise DR readiness.
-   AI reliability in all situations.
-   Recovery performance at real banking scale.

## 10. Azure implementation references

Use current Microsoft Learn documentation as the implementation source
of truth for service-specific configuration.

Relevant areas include:

-   Virtual Machines / Virtual Machine Scale Sets and multi-zone
    Application Gateway routing.
-   Azure Database for PostgreSQL backup and restore.
-   Azure Monitor dashboards, workbooks and alert rules.
-   Azure AI Foundry / Azure OpenAI model invocation.
-   Azure AI Foundry "Add your data" (Azure AI Search grounding).
-   Azure AI Content Safety.

The facilitator's documents remain the source for the project's intended
concept, scope boundaries, human-approval principle, synthetic-data
requirement and demonstration flow.
