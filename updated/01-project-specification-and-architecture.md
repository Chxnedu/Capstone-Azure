# Project Specification & Architecture

## 1. Project name

**Cloud-Native Disaster Recovery Platform Using AI --- Azure Prototype
(Multi-Region)**

## 2. Objective

Build a low-cost Azure prototype that demonstrates how a banking-style
application can automatically fail over from a primary region to a
secondary region, how a cross-region database can be promoted when the
primary database is unreachable, how recovery evidence can be
collected, and how AI can help a human operator understand and respond
to the incident **after** the automatic failover has occurred.

## 3. Architecture principle

Keep the architecture simple:

> **Automatic Regional Failover + Recovery + Observability + AI
> Decision Support (post-failover) + Human-Executed Remediation**

## 4. Proposed architecture

``` text
                              USERS
                                |
                                v
                    +---------------------------+
                    | Azure Traffic Manager     |
                    | (Priority routing,        |
                    |  health probes on /health)|
                    +-------------+-------------+
                                  |
                 +----------------+----------------+
                 |                                 |
                 v                                 v
        +-----------------+               +-----------------+
        | PRIMARY REGION  |               | SECONDARY REGION|
        | VM - Web App    |               | VM - Web App    |
        | (Priority 1)    |               | (Priority 2,    |
        |                 |               |  warm standby)  |
        +--------+--------+               +--------+--------+
                 |                                  |
                 v                                  v
        +-----------------+   cross-region   +-----------------+
        | Azure Database  | <--replication-->| Read Replica     |
        | for PostgreSQL  |                  | (Azure Database  |
        | Flexible Server |                  | for PostgreSQL) |
        | (PRIMARY)       |    promote-on-   | (SECONDARY,     |
        |                 |    failure       |  read-only until|
        |                 |                  |  promoted)      |
        +--------+--------+                  +--------+--------+
                 |                                     |
          backups/snapshots                     backups/snapshots
                 |                                     |
                 v                                     v
          Azure Blob Storage (evidence/artifacts) -- one shared account,
          or one per region if the team prefers region isolation


 VMs / Traffic Manager / Azure DB for PostgreSQL (both regions)
        |
        v
+----------------------+
| Azure Monitor        |
| Metrics + Logs       |
| Alerts + Dashboard   |
+----------+-----------+
           |
           v
   Approved evidence (which region is active, DB reachability,
   replica lag, failover timestamps)
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
| Grounded on:         |
|  - recovery-runbook  |
|  - this architecture |
|    document          |
+----------+-----------+
           |
           v
  Diagnosis + recommendation only
  (invoked AFTER automatic failover already happened)
           |
           v
+----------------------+
| HUMAN OPERATOR       |
| Reviews diagnosis,   |
| executes recommended |
| runbook step (e.g.   |
| confirm/execute DB   |
| promotion)           |
+----------+-----------+
           |
           v
+----------------------+
| Recovery / promotion |
| script                |
+----------------------+
```

**Key change from the single-region design:** the application-layer
failover (which region receives traffic) is handled automatically by
Azure Traffic Manager's health probing, with no human or AI step in the
loop. AI is only invoked *after* failover has already happened, to
explain the root cause and recommend the next runbook step. The human
operator's job shifts from "approve the failover" to "execute the
recommended recovery/remediation step" (most importantly, database
promotion, which Azure does not perform automatically).

Microsoft documents Azure Traffic Manager's Priority routing method as
a way to send traffic to a primary endpoint and automatically fail over
to a backup endpoint in another region based on endpoint health checks.
Microsoft's guidance on Azure Database for PostgreSQL Flexible Server
read replicas states that promoting a cross-region replica for
disaster recovery is a user-initiated action, not a service-driven one
— there is no managed feature that promotes it for you automatically.

## 5. Application layer

Use:

-   **One small Azure Virtual Machine per region** (primary region +
    secondary region) — not two VMs per region. The region itself is
    the redundancy boundary for this prototype; do not also build
    multi-Availability-Zone redundancy inside each region.
-   One simple application running on both VMs, sharing the same code
    and configuration pattern.
-   **Azure Traffic Manager**, Priority routing method, as the single
    public entry point. Configure the primary region's VM (or its
    public IP/DNS name) as Priority 1 and the secondary region's VM as
    Priority 2. Traffic Manager health-probes `/health` on both
    endpoints and automatically redirects DNS resolution to the
    secondary endpoint if the primary stops responding.
-   `/health` endpoint — must also reflect database connectivity (see
    below), not just "the process is running."
-   `/api/accounts` or similar simple endpoint.
-   `/simulate-failure` endpoint — simulates an application-level
    failure (test-only).
-   `/simulate-db-failure` endpoint — simulates the application being
    unable to reach its database (test-only), used for the
    database-failover drill scenario.

Both failure endpoints must be protected and clearly labelled as
**test-only endpoints**. They must simulate failure conditions rather
than perform a dangerous infrastructure action.

A very simple Python Flask/FastAPI application is enough, deployed
identically to both regions.

### Why not Azure Front Door instead of Traffic Manager?

Azure Front Door (Layer 7, faster failover, more configuration) would
also work, but Traffic Manager (DNS-based Priority routing) is simpler
to set up in a 4-day window and is sufficient to demonstrate the DR
concept. Note the trade-off for the team: Traffic Manager failover
depends on DNS TTL, so expect failover to take roughly the TTL value
plus probe interval (tune both down for the demo, e.g. a 10-second fast
health-check interval and a short TTL), not an instant cutover.

### Public exposure note

Because there is no longer an Application Gateway in front of each
region's single VM, Traffic Manager's endpoints point directly at each
VM's public IP. Restrict inbound NSG rules on each VM to the ports
required (HTTP/HTTPS) and treat this as a lab-only simplification — a
production design would front each region with its own load balancer or
gateway so VMs are never directly internet-facing.

## 6. Database approach

### The core question: how do we get "promote a cross-region replica" on Azure?

Use **Azure Database for PostgreSQL --- Flexible Server** with a
**cross-region read replica**:

-   Deploy the primary Flexible Server in the primary region.
-   Deploy a **read replica of that server in the secondary region**.
    Azure keeps it in near-real-time asynchronous sync with the
    primary (typical replication lag target well under 5 minutes for
    this kind of setup).
-   When the primary database is unreachable, **promote the read
    replica** using the `az postgres flexible-server replica promote`
    command (or the Portal's "Replication" > "Promote" action on the
    server).
-   Two promotion modes matter for this project:
    -   **Planned / SwitchOver** — waits for the replica to fully
        catch up before promoting, no data loss, use for a controlled
        drill.
    -   **Forced / Standalone** — promotes immediately without waiting
        for full sync, may lose the most recent uncommitted changes;
        use only to simulate a genuine regional outage where the
        primary is unreachable and cannot be waited on.
-   After promotion, update the application's database connection
    setting (via Key Vault / managed identity, never a hard-coded
    string) to point at the promoted server.

**This promotion step is not automatic on Azure** — Microsoft's own
documentation states this explicitly. This project treats that as a
feature, not a limitation: it keeps the human-approval control boundary
that the rest of this project already requires. The promotion is a
**single scripted command**, run by the Recovery Operator after
reviewing the AI's diagnosis, so it is fast (well within the lab RTO
target) without being unattended.

### Two failure scenarios this creates

| Scenario | What fails | What happens automatically | What the human must do |
| --- | --- | --- | --- |
| A — App-only failure | The VM/application process in the primary region | Traffic Manager reroutes traffic to the secondary region's app, which talks to the still-healthy primary database across regions | Investigate and repair the primary region; plan failback |
| B — Database unreachable | The application in the primary region can't reach the primary database | Traffic Manager reroutes traffic to the secondary region's app (same mechanism as Scenario A) | Run the replica promotion command, update the app's DB connection target, verify, then plan failback |

### Do not build

-   A fully automated, unattended promotion pipeline (e.g. an Azure
    Function that watches DB health and calls promote without a human
    step). This is listed as an optional Day-4-or-later stretch goal
    only — see §15.
-   Bidirectional/multi-master replication.
-   A second, independent failover chain for the database (one primary
    + one cross-region replica is enough).

## 7. Monitoring

Use Azure Monitor (Log Analytics + Metrics + Alerts + Dashboards) for:

-   VM CPU and status metrics, **in both regions**.
-   Traffic Manager endpoint status (which endpoint is currently
    considered healthy/degraded, and which one is receiving traffic).
-   Application logs, **in both regions**.
-   Azure Database for PostgreSQL basic health metrics for **both** the
    primary server and the read replica, including replication lag.
-   Azure Monitor alert rules for: application unhealthy, database
    unreachable, replica lag exceeding a threshold.
-   DR events (failover start/end, promotion start/end).
-   Recovery timing evidence.

Azure Monitor supports dashboards and alert rules across multiple
regions from a single Log Analytics workspace, which keeps this
manageable for a low-complexity prototype. Azure currently includes a
limited free allowance for metrics, logs and alerts, subject to the
subscription's overall pricing/usage conditions.

## 8. Readiness score

Do not ask AI to calculate the score.

Use simple deterministic rules.

Example:

  Indicator                          Weight
  --------------------------------- --------
  Backup / replica sync status          30%
  Last recovery test                    25%
  Monitoring coverage (both regions)    15%
  Runbook status                        15%
  Open recovery defects                 15%

These weights are only a proposed prototype starting point, not an
industry standard. "Backup / replica sync status" now reflects whether
the cross-region replica is healthy and within an acceptable lag
window, not just whether a backup job succeeded.

The resulting score is then passed to AI so that AI can explain it.

## 9. RTO and RPO

The facilitator document deliberately leaves numerical RTO/RPO targets
open. Because this design now has two distinct failure scenarios with
different recovery mechanics, agree **two sets of lab targets** before
the drill:

  Scenario                    Lab RTO target   Lab RPO target
  --------------------------  ---------------  ---------------
  A — App-only failure        5 minutes        0 (no data loss; DB untouched)
  B — Database unreachable    15 minutes        5 minutes

These are demonstration targets, not banking requirements.

Measure:

`RTO = failure trigger timestamp -> validated application recovery (traffic serving correctly from whichever region is active)`

`RPO (Scenario B only) = last replicated data point on the replica before promotion -> latest data point that existed on the primary before the failure`

## 10. Recovery flow

### Before failure

1.  Application is healthy in the primary region; secondary region is
    running as a warm standby, receiving no traffic.
2.  Primary database is healthy; read replica is in sync in the
    secondary region.
3.  Azure Monitor is collecting data from both regions.
4.  Traffic Manager reports the primary endpoint as healthy.
5.  Dashboard shows green/healthy status and "Primary region active."
6.  AI can access the approved runbook and architecture document as
    grounding sources, and summarized telemetry on request.

### During failure — Scenario A (app-only)

1.  Trigger the controlled application failure (`/simulate-failure`) in
    the primary region.
2.  Traffic Manager's health probe detects the unhealthy endpoint and
    **automatically** redirects traffic to the secondary region. No
    human or AI action is required for this step.
3.  Azure Monitor records the failover event.
4.  Evidence is collected.
5.  AI receives the approved evidence **after** the failover has
    already occurred and explains what happened and why, citing the
    runbook and architecture document.

### During failure — Scenario B (database unreachable)

1.  Trigger the controlled database-unreachable condition
    (`/simulate-db-failure`) in the primary region.
2.  Traffic Manager's health probe (via `/health`, which reflects DB
    connectivity) detects the unhealthy primary endpoint and
    **automatically** redirects traffic to the secondary region.
3.  The secondary region's application still cannot reach a writable
    database (the replica is read-only until promoted).
4.  Evidence is collected, including replication lag and DB
    reachability status.
5.  AI receives the approved evidence and explains the likely cause,
    citing the runbook's database-promotion step.

### Human decision

The operator checks:

-   The active alert(s).
-   Which region Traffic Manager is currently routing to.
-   The evidence package.
-   The AI diagnosis and recommendation.
-   The runbook's recovery steps.

Then explicitly executes the recommended action, e.g.:

**Execute database promotion** (Scenario B) or **Investigate and repair
primary region** (Scenario A) — followed by **Reject / investigate
further** if the evidence doesn't support the recommendation.

### Recovery

After the human executes the recommended step:

1.  (Scenario B only) Promote the read replica; update the
    application's database connection target.
2.  Confirm the active region's health endpoint reports healthy,
    including database connectivity.
3.  Confirm expected synthetic records are present and current.
4.  Record recovery completion time.
5.  Update the dashboard.
6.  Document a failback plan (out of scope to execute live in the
    4-day window, but the runbook should describe it — see §15).

## 11. Infrastructure as Code

Use **Terraform** (azurerm provider) if the team is already comfortable
with it.

If Terraform would slow the team down significantly, use **Azure Bicep**
or **ARM templates** only if someone already knows them.

The IaC should cover the minimum viable infrastructure, **now spanning
two regions**:

-   Virtual Network/subnets/network security groups (per region).
-   One Virtual Machine per region.
-   Azure Traffic Manager profile, endpoints and health-probe
    configuration.
-   Azure Database for PostgreSQL --- Flexible Server (primary region).
-   Azure Database for PostgreSQL cross-region read replica (secondary
    region).
-   Azure Monitor alert rules/logging for both regions where practical.
-   Managed identities / role assignments (per region).
-   Storage account(s) (Blob container) for evidence.

Do not try to IaC every dashboard widget if doing so consumes the
team's final day. Do not attempt to codify the promotion automation —
the promote command is run manually/scripted by the operator, not by
Terraform.

## 12. Security baseline

Minimum controls:

-   Synthetic data only.
-   No production connection.
-   No real banking credentials.
-   Least-privilege role-based access control (RBAC) using managed
    identities, in both regions.
-   Network security groups restricted to required ports on each
    region's VM.
-   Databases (primary and replica) not publicly accessible (private
    endpoint or VNet-integrated access only).
-   Secrets stored in Azure Key Vault (per region, or one vault with
    cross-region access if the team's sandbox allows it).
-   Azure Activity Log / Microsoft Defender for Cloud enabled if
    available in the sandbox subscription.
-   Database promotion and any other recovery action require a human
    operator; AI never calls the promote command or any infrastructure
    API directly.
-   AI output labelled as advisory and dated/timestamped relative to
    when the automatic failover occurred.
-   Project resources tagged with owner, environment, region role
    (primary/secondary) and deletion date.
-   Budget/cost alert configured via Azure Cost Management, covering
    both regions.
-   Resources deleted after the demonstration, in both regions.

## 13. Operational excellence

The project demonstrates operational excellence through:

-   Infrastructure as Code, spanning two regions.
-   Repeatable deployment.
-   Automatic, infrastructure-driven traffic failover.
-   Health checks that reflect both application and database state.
-   Centralized logs across regions.
-   Alerts.
-   A documented runbook covering both failure scenarios.
-   A controlled DR drill for both scenarios.
-   Measured RTO/RPO per scenario.
-   Evidence collection.
-   Clear ownership.
-   Human-executed recovery actions.
-   Cleanup and cost controls.

## 14. Business value

The demonstration should connect technical work to three outcomes:

### Better business continuity

The application keeps serving users automatically when the entire
primary region fails, not just a single instance, and the team has a
defined, tested path to make the database in the secondary region
writable.

### Reduced downtime

Automatic traffic failover removes the slowest, most error-prone part
of manual DR (deciding to reroute traffic), while AI-assisted diagnosis
speeds up the part that still needs a human: understanding what
actually happened and completing the database recovery.

### Better regulatory preparedness

The prototype creates evidence, timestamps, logs, recovery results and
operator decision records — for both an application-only failure and a
full regional/database failure — that can support future governance
work.

It should **not** be presented as proof of regulatory compliance.

## 15. Explicit non-goals

Do not implement:

-   Multiple Availability Zones inside each region (redundancy comes
    from the two regions, not from zone-level duplication as well).
-   Kubernetes / Azure Kubernetes Service (AKS).
-   Service mesh.
-   Bidirectional/active-active database replication.
-   Fully automated, unattended database promotion (no human step). A
    scripted, human-triggered promotion is the requirement; an
    automated trigger is an optional stretch goal only, not a
    deliverable.
-   Live execution of a failback (secondary → primary) during the
    4-day window. Document the failback plan in the runbook, but do not
    build and rehearse it unless time remains after the primary drill.
-   Autonomous AI failover or AI-triggered promotion.
-   Production customer data.
-   Real banking integrations.
-   Full compliance certification.
-   Complex ML forecasting.
