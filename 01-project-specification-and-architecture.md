# Project Specification & Architecture

## 1. Project name

**Cloud-Native Disaster Recovery Platform Using AI --- Azure Prototype**

## 2. Objective

Build a low-cost Azure prototype that demonstrates how a banking-style
application can remain available during a component failure, how
recovery evidence can be collected, and how AI can help a human operator
understand and respond to an incident.

## 3. Architecture principle

Keep the architecture simple:

> **Availability + Recovery + Observability + AI Decision Support +
> Human Approval**

## 4. Proposed architecture

``` text
                         USERS
                           |
                           v
                  +-------------------+
                  | Azure Application |
                  | Gateway           |
                  +---------+---------+
                            |
                 +----------+----------+
                 |                     |
                 v                     v
          +-------------+       +-------------+
          | VM - Zone 1 |       | VM - Zone 2 |
          | Web App     |       | Web App     |
          +------+------+       +------+------+
                 |                     |
                 +----------+----------+
                            |
                            v
                    +---------------+
                    | Azure Database|
                    | for PostgreSQL|
                    | Flexible Srv  |
                    +-------+-------+
                            |
                    backups/snapshots
                            |
                            v
                     Azure Blob Storage
                    (evidence/artifacts)


 VMs / App Gateway / Azure DB for PostgreSQL
        |
        v
+----------------------+
| Azure Monitor        |
| Metrics + Logs       |
| Alerts + Dashboard   |
+----------+-----------+
           |
           v
   Approved evidence
           |
           v
+----------------------+
| Azure Function /     |
| small AI integration |
| layer                |
+----------+-----------+
           |
           v
+----------------------+
| Azure AI Foundry     |
| (Azure OpenAI)       |
| AI DR Advisor        |
+----------+-----------+
           |
           v
  Recommendation only
           |
           v
+----------------------+
| HUMAN OPERATOR       |
| Approve / Reject     |
+----------+-----------+
           |
           v
+----------------------+
| Recovery workflow    |
| script / Function    |
+----------------------+
```

Microsoft documents using Azure Application Gateway or Azure Load
Balancer across Availability Zones with health probes to stop routing
traffic to unhealthy VM instances. The same "spread across zones + take
unhealthy targets out of rotation" pattern used in the AWS design maps
directly onto Azure Availability Zones and Application Gateway health
probes.

## 5. Application layer

Use:

-   Two small Azure Virtual Machines (or two instances in a Virtual
    Machine Scale Set spread across zones).
-   One simple application running on both.
-   Azure Application Gateway (Layer 7, path-based routing, health
    probes) in front. Azure Load Balancer (Layer 4) is an acceptable
    simpler substitute if Application Gateway adds too much setup time.
-   `/health` endpoint.
-   `/api/accounts` or similar simple endpoint.
-   `/simulate-failure` endpoint for the lab demonstration.

The failure endpoint must be protected and clearly labelled as a
**test-only endpoint**. It should simulate an application failure rather
than perform a dangerous infrastructure action.

A very simple Python Flask/FastAPI application is enough.

## 6. Database approach

For the four-day prototype, do not build complex database replication.

Use **Azure Database for PostgreSQL --- Flexible Server** as the
application database.

Use Azure Database for PostgreSQL automated backups and/or a manual
backup as the recovery mechanism. During the DR demonstration, restore a
recovery database from a known backup and point the application to it.

This gives the team a clear recovery story without building custom
database replication.

Azure Database for PostgreSQL Flexible Server supports automated daily
backups with point-in-time restore, and on-demand backups can be
restored as a new server instance.

### Important interpretation of "two databases"

The original meeting note says "2 databases". For a four-day
implementation, interpret this as:

-   **Primary database:** the database used by the running application.
-   **Recovery database:** a database restored from a protected backup
    during the DR exercise.

Do not try to build a sophisticated active-active database design.

If the facilitator specifically requires two continuously running
databases, revisit this decision before implementation (for example
using Azure Database for PostgreSQL read replicas).

## 7. Monitoring

Use Azure Monitor (Log Analytics + Metrics + Alerts + Dashboards) for:

-   VM CPU and status metrics.
-   Application Gateway healthy/unhealthy backend count.
-   Application logs.
-   Azure Database for PostgreSQL basic health metrics.
-   Azure Monitor alert rules.
-   DR events.
-   Recovery timing evidence.

Azure Monitor supports dashboards and alert rules, making it a good fit
for a low-complexity prototype. Azure currently includes a limited free
allowance for metrics, logs and alerts, subject to the subscription's
overall pricing/usage conditions.

## 8. Readiness score

Do not ask AI to calculate the score.

Use simple deterministic rules.

Example:

  Indicator                    Weight
  -------------------------- --------
  Backup/protection status        30%
  Last recovery test              25%
  Monitoring coverage             15%
  Runbook status                  15%
  Open recovery defects           15%

These weights are only a proposed prototype starting point, not an
industry standard.

Example:

-   Backup healthy = full points.
-   Last recovery test passed = full points.
-   Monitoring enabled = full points.
-   Runbook reviewed = full points.
-   No critical open defect = full points.

The resulting score is then passed to AI so that AI can explain it.

## 9. RTO and RPO

The facilitator document deliberately leaves numerical RTO/RPO targets
open.

For this prototype, the team should agree simple **lab targets** before
the drill.

Example:

-   Lab RTO target: **15 minutes**
-   Lab RPO target: **5 minutes**

These are demonstration targets, not banking requirements.

Measure:

`RTO = recovery start/declaration time -> validated application recovery`

`RPO = last protected data point -> latest data point successfully recovered`

## 10. Recovery flow

### Before failure

1.  Application is healthy.
2.  Backup exists.
3.  Azure Monitor is collecting data.
4.  Dashboard shows green/healthy status.
5.  AI can access approved runbook and summarized telemetry.

### During failure

1.  Trigger the controlled application failure.
2.  Azure Monitor detects the unhealthy state.
3.  Alert rule changes state.
4.  Evidence is collected.
5.  AI receives the approved evidence.
6.  AI explains the incident and suggests the relevant runbook step.

### Human decision

The operator checks:

-   Alert.
-   Application status.
-   Evidence.
-   AI recommendation.
-   Runbook.

Then explicitly chooses:

**Approve recovery** or **Reject / investigate further**.

### Recovery

After approval:

1.  Execute the recovery script/workflow.
2.  Restore the recovery database if needed.
3.  Start or redeploy the application.
4.  Confirm health endpoint.
5.  Confirm expected synthetic records.
6.  Record recovery time.
7.  Update the dashboard.

## 11. Infrastructure as Code

Use **Terraform** (azurerm provider) if the team is already comfortable
with it — Terraform works the same way against Azure as it does against
AWS.

If Terraform would slow the team down significantly, use **Azure Bicep**
or **ARM templates** only if someone already knows them.

The IaC should cover the minimum viable infrastructure:

-   Virtual Network/subnets/network security groups.
-   Virtual Machines.
-   Azure Application Gateway (or Load Balancer).
-   Backend pool / health probes.
-   Azure Database for PostgreSQL.
-   Azure Monitor alert rules/logging where practical.
-   Managed identities / role assignments.
-   Storage account (Blob container) for evidence.

Do not try to IaC every dashboard widget if doing so consumes the team's
final day.

## 12. Security baseline

Minimum controls:

-   Synthetic data only.
-   No production connection.
-   No real banking credentials.
-   Least-privilege role-based access control (RBAC) using managed
    identities.
-   Network security groups restricted to required ports.
-   Database not publicly accessible (private endpoint or
    VNet-integrated access only).
-   Secrets stored in Azure Key Vault.
-   Azure Activity Log / Microsoft Defender for Cloud enabled if
    available in the sandbox subscription.
-   Recovery actions require human approval.
-   AI output labelled as advisory.
-   Project resources tagged with owner, environment and deletion date.
-   Budget/cost alert configured via Azure Cost Management.
-   Resources deleted after the demonstration.

## 13. Operational excellence

The project demonstrates operational excellence through:

-   Infrastructure as Code.
-   Repeatable deployment.
-   Health checks.
-   Centralized logs.
-   Alerts.
-   A documented runbook.
-   A controlled DR drill.
-   Measured RTO/RPO.
-   Evidence collection.
-   Clear ownership.
-   Human approval.
-   Cleanup and cost controls.

## 14. Business value

The demonstration should connect technical work to three outcomes:

### Better business continuity

The application can continue serving users when one application instance
fails, and the team has a defined recovery path for a larger failure.

### Reduced downtime

Monitoring detects problems quickly, while the recovery workflow reduces
manual guesswork.

### Better regulatory preparedness

The prototype creates evidence, timestamps, logs, recovery results and
approval records that can support future governance work.

It should **not** be presented as proof of regulatory compliance.

## 15. Explicit non-goals

Do not implement:

-   Multi-region active-active architecture.
-   Kubernetes / Azure Kubernetes Service (AKS).
-   Service mesh.
-   Complex data replication.
-   Autonomous AI failover.
-   Production customer data.
-   Real banking integrations.
-   Full compliance certification.
-   Complex ML forecasting.
