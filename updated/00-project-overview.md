# Cloud-Native Disaster Recovery Platform --- Project Overview

## 1. What we are building

We are building a **small, safe, Azure-based disaster recovery prototype
for a banking-style web application**, using a **primary region and a
secondary (failover) region**.

The goal is not to build a real banking platform or an enterprise DR
system. The goal is to demonstrate one clear idea:

> **Azure automatically routes users away from a failed region, while AI
> helps people understand *why* the primary failed and what to do about
> it. A human remains responsible for the recovery action the AI
> recommends.**

The project will use only synthetic data and a sandbox environment.

## 2. The simple story

The demo should be easy for someone outside the project to understand:

1.  A small banking-style web app runs in a **primary Azure region**.
2.  A second, idle copy of the same app runs in a **secondary Azure
    region**, ready to take traffic.
3.  Azure Traffic Manager continuously checks the health of the primary
    app. If it stops responding, traffic is **automatically** redirected
    to the secondary region — no human or AI action required for this
    step.
4.  A primary database runs in the primary region. A **cross-region read
    replica** of that database runs in the secondary region, staying in
    sync.
5.  If the application cannot reach the primary database, this is
    treated as a database-level failure: traffic has already moved to
    the secondary region (step 3), and the read replica is **promoted**
    to become the new writable database so the secondary region is
    fully self-sufficient.
6.  Azure monitoring captures what happened in both regions.
7.  **After** the automatic failover has already happened, the AI
    component reviews the monitoring evidence, the recovery runbook and
    the system architecture document, and explains *why* the primary
    likely failed.
8.  AI produces a short explanation and points to the relevant runbook
    step — it does not perform any action itself.
9.  A human operator reviews the AI's diagnosis and carries out the
    runbook-recommended recovery step (for example, confirming/running
    the database promotion, or investigating and repairing the primary
    region).
10. The dashboard shows what happened, which region is currently active,
    how long failover took, whether data was preserved, and what AI
    recommended.

## 3. What makes this an AI + DR project

AI should **not** decide that a disaster has happened, and it should
**not** trigger any failover. In this design, that boundary is easier to
keep than before, because the application-layer failover is handled
entirely by Azure infrastructure (Traffic Manager health probes) — it
happens whether or not AI or a human is even looking at the dashboard.

Use normal Azure rules for things that must be predictable and instant:

-   Is the primary application healthy?
-   Has Traffic Manager already failed traffic over to the secondary
    region?
-   Can the application reach the primary database?
-   Did the backup/replication process succeed?
-   Did the recovery test pass?
-   What was the measured RTO/RPO?

Use AI for the things that benefit from interpretation, **after** the
automatic failover has occurred:

-   Explain why the primary region/application/database most likely
    failed, using approved telemetry.
-   Summarize the incident from logs/alerts across both regions.
-   Identify unusual patterns in recent telemetry.
-   Recommend the relevant recovery-runbook step (e.g. "confirm the
    replica promotion" or "primary region requires investigation before
    failback").
-   Point out missing information or prerequisites.
-   Produce a short technical and executive summary after the drill.

This still follows the facilitator's core design: deterministic
infrastructure controls first, AI as decision support *after the fact*,
and human execution of any recovery action.

## 4. Four workstreams

### Infrastructure & DevOps

Own both Azure regions, networking, VM/application deployment, Azure
Traffic Manager configuration, the primary database and its cross-region
read replica, the database promotion script, monitoring infrastructure,
IaC and recovery automation.

### AI Engineering

Own Azure AI Foundry (Azure OpenAI Service) integration, prompts,
grounding on the runbook **and** the architecture document, AI safety
rules, evaluation tests and the post-failover diagnosis flow.

### Software Development

Own the simple banking-style application, health endpoint, a database
connectivity check, failure-simulation endpoints (app failure and
database-unreachable), synthetic data, and support for switching the
app's database connection target during promotion.

### Testing, QA & Dashboard

Own the test plan, DR drill evidence for **both** failure scenarios,
RTO/RPO measurement, dashboard, screenshots/evidence pack and final
validation.

## 5. What we are deliberately NOT building

To finish in four days, do not build:

-   A real banking system.
-   Multiple Availability Zones inside each region (one VM per region is
    enough to tell the story — the region *is* the redundancy).
-   Kubernetes/Azure Kubernetes Service (AKS).
-   Microservices.
-   Fully automated, unattended database promotion (a Function/Logic App
    that promotes the replica without a human step). This is
    deliberately left as a documented stretch goal, not a requirement,
    because it removes the human-approval control that the project's
    safety principles depend on.
-   Complex event-driven orchestration.
-   Autonomous AI agents that can change infrastructure or trigger
    failover.
-   A sophisticated ML model.
-   A custom vector database.
-   A complex CI/CD platform.
-   Customer data handling.
-   Full regulatory certification.

The prototype should be small enough that the entire team can understand
the complete flow across both regions.

## 6. Recommended Azure shape

**Application path (normal operation)**

`User -> Azure Traffic Manager -> Primary Region VM (Web App) -> Primary Azure Database for PostgreSQL`

**Application path (after automatic failover)**

`User -> Azure Traffic Manager -> Secondary Region VM (Web App) -> Primary DB (if only the app failed) OR Promoted Replica (if the DB was unreachable)`

**Operations path**

`VMs/Database/Traffic Manager (both regions) -> Azure Monitor metrics + logs + alerts -> AI evidence -> Azure AI Foundry (Azure OpenAI), grounded on the runbook + architecture doc -> Diagnosis + recommendation -> Human executes runbook step (e.g. confirm/execute DB promotion, investigate primary)`

**Dashboard**

Use a simple **Azure Dashboard** (or Azure Workbook) in Azure Monitor for
operational metrics, alerts, and which region is currently active. If
the team needs a more polished executive view, the dashboard team can
add a very small web page that reads a prepared JSON summary. Do not
introduce a large BI platform unless the team already knows it.

## 7. Definition of done

The project is successful when the team can demonstrate:

-   The application works normally in the primary region.
-   A controlled app-layer failure can be triggered safely, and Traffic
    Manager automatically redirects traffic to the secondary region.
-   A controlled database-unreachable condition can be triggered safely,
    and results in both automatic traffic failover **and** a scripted,
    human-triggered database promotion.
-   Monitoring detects both types of problem.
-   The dashboard reflects the active region and incident state.
-   AI receives approved evidence **after** failover has occurred and
    produces a grounded explanation/recommendation, citing the runbook
    and/or architecture document.
-   AI does not execute recovery or failover.
-   A human explicitly executes the recommended recovery step.
-   Recovery completes and the application passes health/integrity
    checks.
-   RTO and RPO evidence is captured for both failure scenarios.
-   The team can explain the architecture, security controls, AI
    controls and business value, including which parts are automatic and
    which require a human.

## 8. Source basis

The facilitator documents describe a controlled sandbox prototype using
synthetic data, transparent readiness rules, AI-assisted analysis, human
approval, recovery drills, monitoring, evidence collection and cost
controls. They explicitly exclude production customer data, autonomous
disaster declaration/failover and enterprise compliance certification.

The Azure version in this project keeps that same concept while
replacing AWS-native components with Azure equivalents such as Azure
Traffic Manager, Azure Monitor, Azure Database for PostgreSQL (with a
cross-region read replica), Virtual Machines, Blob Storage, Azure
Functions where useful, and Azure AI Foundry (Azure OpenAI Service).
