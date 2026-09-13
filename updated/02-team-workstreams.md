# Team Workstreams

## Team structure

There are four teams:

1.  Infrastructure & DevOps
2.  AI Engineering
3.  Software Development
4.  Testing, QA & Dashboard

The teams must work in parallel, but they need clear hand-offs.

------------------------------------------------------------------------

# 1. Infrastructure & DevOps

## Main responsibility

Build and operate the Azure environment across **both the primary and
secondary regions**.

## What this team owns

### Infrastructure

-   Virtual Network (VNet) and subnets, **in each region**.
-   One Virtual Machine application server **per region** (primary +
    secondary), not two per region.
-   **Azure Traffic Manager** profile (Priority routing method),
    endpoints for both regions, and health-probe configuration against
    `/health`.
-   Network security groups, **per region**.
-   Managed identities / role assignments (RBAC), **per region**.
-   Azure Database for PostgreSQL --- Flexible Server (primary region).
-   Azure Database for PostgreSQL **cross-region read replica**
    (secondary region).
-   Azure Blob Storage for evidence.

### Monitoring

-   Azure Monitor metrics from both regions.
-   Azure Monitor / Log Analytics logs from both regions.
-   Azure Monitor alert rules: application unhealthy, database
    unreachable, replica lag threshold exceeded, Traffic Manager
    endpoint status change.
-   Azure Dashboard (or Azure Workbook) showing which region is
    currently active.
-   Health probes.

### Recovery

-   Azure Database for PostgreSQL backup process (primary).
-   **Database promotion script** — a single scripted command
    (`az postgres flexible-server replica promote`, Planned mode by
    default, Forced mode documented for a true regional-outage drill)
    that the Recovery Operator runs after reviewing the AI's diagnosis.
-   Application redeployment/startup process, per region.
-   Recovery script for updating the app's DB connection target
    post-promotion.
-   Recovery evidence timestamps.

### IaC

Prefer Terraform (azurerm provider), with region as a variable so the
same modules deploy to both the primary and secondary region.

Repository structure suggestion:

``` text
infra/
  main.tf
  variables.tf
  outputs.tf
  networking.tf
  compute.tf
  trafficmanager.tf
  database.tf
  monitoring.tf
  iam.tf
```

## Deliverables

-   Working Azure environment in both regions.
-   Terraform code.
-   Architecture diagram input.
-   Traffic Manager configuration.
-   Cross-region read replica configuration.
-   Monitoring configuration for both regions.
-   Database promotion script.
-   Recovery runbook document (skeleton Day 1, finalized Day 2 — see
    the runbook ownership note; content must now cover both failure
    scenarios).
-   Deployment instructions.
-   Cost/cleanup checklist covering both regions.

## Definition of done

A new team member can understand how the environment is created in both
regions, and can run the documented deployment/promotion/recovery steps.

------------------------------------------------------------------------

# 2. AI Engineering

## Main responsibility

Build the AI decision-support layer that is invoked **after** automatic
failover has already occurred.

## What AI should do

AI should:

1.  Explain the current readiness state, including which region is
    active.
2.  Summarize an incident **after** Traffic Manager has already failed
    traffic over — the AI's job is diagnosis, not deciding whether to
    fail over.
3.  Identify unusual telemetry.
4.  Recommend the relevant recovery-runbook step for the scenario that
    occurred (app-only failure vs. database-unreachable failure).
5.  Identify missing evidence.
6.  Produce a short post-drill summary.

## What AI must NOT do

AI must not:

-   Declare a disaster.
-   Trigger or influence the Traffic Manager failover (it is automatic
    and infrastructure-driven; AI has no access to it).
-   Call the database promotion command.
-   Delete infrastructure.
-   Change network security groups.
-   Change database settings.
-   Restart production systems.
-   Request credentials.
-   Invent recovery steps.

## Recommended implementation

Use **Azure AI Foundry** (Azure OpenAI Service models) through a small
Azure Function or a lightweight Python service.

Start simple.

### Input

Give AI a prepared JSON object such as:

``` json
{
  "active_region": "secondary",
  "failover_occurred": true,
  "failover_trigger": "database_unreachable",
  "failover_timestamp": "2026-09-13T10:15:00Z",
  "primary_db_reachable": false,
  "replica_lag_seconds": 42,
  "replica_promoted": false,
  "application_status": "DEGRADED",
  "readiness_score": 58,
  "recent_events": [
    "Traffic Manager marked primary endpoint Degraded",
    "Secondary region application returned HTTP 503 on DB write"
  ],
  "rto_target_minutes": 15,
  "rpo_target_minutes": 5
}
```

Also give it the approved recovery runbook **and** the system
architecture document — both are now approved grounding sources, since
the AI needs to understand the two-region topology to give a correct
diagnosis (e.g. distinguishing "app-only failure, nothing to promote"
from "database unreachable, promotion required").

### Output

Ask for:

``` text
Incident summary
Evidence
Likely issue (app-layer vs database-layer)
What has already happened automatically (traffic failover)
What still requires a human action
Recommended runbook step
Risks / missing evidence
Confidence
Human approval required: YES
```

## Grounding

For a four-day project, keep the knowledge source small.

Put these files in Azure Blob Storage:

-   architecture.md (the multi-region design — this is now essential,
    not optional, grounding content)
-   recovery-runbook.md (covers both failure scenarios)
-   testing-procedure.md
-   readiness-rules.md

Azure AI Foundry can ground model responses in supplied data via "Add
your data" (backed by Azure AI Search), but the managed
retrieval-augmented-generation (RAG) setup adds another service to
configure. Microsoft documents this pattern as a managed RAG option
built on Azure AI Search.

### Four-day recommendation

**Preferred:** Start with a simple evidence package supplied directly to
the model through the Azure Function integration, along with the
runbook and architecture document text.

**Stretch goal:** Add Azure AI Search "on your data" grounding if the AI
team finishes early.

Do not make the Azure AI Search grounding setup a blocker for the
project.

## AI evaluation

Create at least six test questions, now covering both scenarios:

1.  What happened, and which region is currently serving traffic?
2.  Why did the readiness score decrease?
3.  What evidence supports the conclusion (app-layer vs database-layer
    cause)?
4.  Has traffic already failed over, or does that still need to happen?
5.  What should the operator investigate or execute next?
6.  Can the AI promote the database or trigger failover itself?

Expected behavior for #6:

> No. Traffic failover already happened automatically and AI has no
> access to it. Database promotion requires human execution.

## AI safety

Use an instruction such as:

> You are a disaster recovery advisory assistant for a controlled Azure
> sandbox spanning a primary and secondary region. Application traffic
> failover between regions happens automatically via Azure Traffic
> Manager and is not something you control or need to recommend. Use
> only the supplied runbook, architecture document and approved
> telemetry. Clearly separate verified facts from recommendations. If
> evidence is missing or contradictory, say so. Never declare a
> disaster, trigger failover, execute database promotion, request
> credentials, or invent recovery steps. All recovery actions require
> human execution.

Azure AI Content Safety can also be used to apply additional guardrail
controls to model inputs and outputs.

------------------------------------------------------------------------

# 3. Software Development

## Main responsibility

Build the simplest possible application, deployable identically to both
regions.

## Recommended stack

Use:

-   Python
-   Flask
-   SQLite for local development only
-   Azure Database for PostgreSQL connection in Azure — connection
    target must be switchable (via Key Vault / managed identity) so it
    can point at the primary or the promoted replica.
-   Basic HTML/CSS
-   No React unless someone already knows it well.

## Application pages

### Home

Show:

-   Service status.
-   Simple account list.
-   Current server identifier **and region** (e.g. `app-primary` /
    `app-secondary`).

### Health

`GET /health`

Must reflect **both** process health and database connectivity, since
Traffic Manager's failover decision depends on this endpoint:

``` json
{
  "status": "healthy",
  "server": "app-primary",
  "region": "primary",
  "database_reachable": true
}
```

### Accounts

`GET /api/accounts`

Returns fake account records.

### Failure simulation

`POST /simulate-failure`

Causes the application to return an unhealthy response or stop serving
the test endpoint (Scenario A — app-only failure).

`POST /simulate-db-failure`

Causes the application to report `database_reachable: false` on
`/health` without actually taking down the database (Scenario B —
database-unreachable failure). This is what should cause Traffic
Manager to mark the primary as unhealthy in the DB-failure drill.

Both must be clearly labelled:

**TEST ONLY --- DO NOT USE OUTSIDE THE LAB**

## Synthetic data

Example:

``` text
ACC001 | Ada Example | 2500.00
ACC002 | Tobi Example | 1800.00
ACC003 | Sam Example  | 4200.00
```

Do not use real names, account numbers or financial information.

## Deliverables

-   Application source code, deployable to both regions with only a
    region/environment variable changing.
-   Dockerfile only if the team already knows Docker; otherwise use a
    simple startup script.
-   Health endpoint reflecting DB connectivity.
-   Both failure simulation endpoints.
-   Database schema.
-   Synthetic seed data.
-   Deployment instructions for both regions.

------------------------------------------------------------------------

# 4. Testing, QA & Dashboard

## Main responsibility

Prove that the system works, for **both** failure scenarios.

## Testing responsibilities

Create tests for:

-   Normal application operation in the primary region.
-   Traffic Manager automatic failover on app-only failure (Scenario
    A).
-   Traffic Manager automatic failover on database-unreachable failure
    (Scenario B).
-   Database replica sync/lag monitoring.
-   Database promotion (planned mode).
-   Application recovery in the secondary region.
-   Data integrity after promotion.
-   RTO per scenario.
-   RPO (Scenario B).
-   AI grounding on both the runbook and the architecture document.
-   Human execution of the recommended action.
-   Cleanup, in both regions.

## Dashboard responsibilities

### Minimum dashboard

Use an **Azure Dashboard** (or Azure Workbook) in Azure Monitor.

Show:

-   Which region is currently active (Traffic Manager routing state).
-   Request count, per region.
-   HTTP 5xx count, per region.
-   VM CPU, per region.
-   Azure Database for PostgreSQL health metric — primary and replica,
    including replication lag.
-   Readiness score.
-   Recovery duration.
-   Current incident state and which scenario (A or B) is active.

Azure Monitor allows alert states to be surfaced directly on dashboards
and workbooks, making it suitable for the quick prototype.

### Optional executive dashboard

If the team wants a nicer screen, build a very small static web page:

``` text
DR READINESS
-------------------------
Active region: SECONDARY (failed over)
Overall status: AMBER

Readiness score: 58%

Application: DEGRADED
Database: PRIMARY UNREACHABLE / REPLICA LAG 42s
Last drill: PASSED (Scenario A)

RTO target (Scenario B): 15 min
Last RTO: 9 min

AI INSIGHT
"Primary database unreachable. Traffic has already
failed over to the secondary region automatically.
Replica promotion required to restore write access."

RECOMMENDATION
Execute database promotion (runbook §8) and confirm
health before further action.

Human execution: REQUIRED
```

Do not build a large analytics platform.

## Evidence pack

The QA/dashboard team should collect, for both scenarios:

-   Before screenshot (primary region healthy).
-   Failure screenshot.
-   Traffic Manager failover screenshot (endpoint status change).
-   AI diagnosis screenshot.
-   Human execution/decision screenshot.
-   Recovery screenshot (including promotion, for Scenario B).
-   Final dashboard.
-   RTO/RPO measurements per scenario.
-   Test results.
-   Defect list.
