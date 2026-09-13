# Four-Day Implementation Guide

## Day 1 --- Foundation and first working application (both regions)

### Goal

By the end of Day 1, the application should be running in the primary
region, a second copy deployed (idle) in the secondary region, and the
basic architecture should exist.

### Infrastructure & DevOps

Build:

-   Azure subscription/resource group and resource tagging, **for both
    the primary and secondary region**.
-   Virtual Network (VNet) per region.
-   One Virtual Machine per region (not two per region).
-   Network security groups, per region.
-   Azure Database for PostgreSQL --- Flexible Server in the **primary**
    region.
-   Basic managed identities / role assignments, per region.
-   Draft the Azure Traffic Manager profile (Priority routing) — full
    health-probe wiring happens Day 2.

Start Terraform, parameterized by region.

### Software Development

Build:

-   Simple web page.
-   `/health` (must eventually report DB connectivity — stub this for
    Day 1, wire it up fully Day 2).
-   `/api/accounts`.
-   Synthetic database data.
-   `/simulate-failure`.
-   `/simulate-db-failure` (can be a stub on Day 1).

Deploy the application manually first to **both** regions if needed.

### AI Engineering

Prepare:

-   AI role and boundaries, now including "invoked only after automatic
    failover."
-   Prompt draft.
-   Example incident JSON, including region/failover fields.
-   Six AI evaluation cases (covering both scenarios).

Contribute to the Day 1 runbook skeleton (see below) — specifically the
human-execution wording and what "evidence" the AI needs.

Do not spend Day 1 building a complex RAG system.

### Testing & Dashboard

Create:

-   Test plan covering both failure scenarios.
-   Dashboard wireframe, including "active region" indicator.
-   RTO/RPO measurement sheet, with separate rows for Scenario A and
    Scenario B.
-   Evidence folder structure.

### Cross-team: Runbook skeleton

**Infrastructure & DevOps** drafts the recovery runbook skeleton today,
with input from Software Development, Testing/QA and AI Engineering
(~1–2 hrs, end of day). This replaces a standalone "example runbook"
placeholder — see the runbook draft for the required structure. Mark
anything that depends on Day 2 infrastructure as `TBD (Day 2)`.

### End-of-day deliverable

**A user can reach the application through the primary region's public
endpoint, an idle copy of the app exists in the secondary region, and a
runbook skeleton exists covering both failure scenarios.**

------------------------------------------------------------------------

# Day 2 --- Traffic Manager, cross-region replica, monitoring and first AI call

## Goal

Make the system observable, prove that Traffic Manager fails traffic
over automatically, stand up the cross-region database replica, and get
the first AI diagnosis working.

### Infrastructure & DevOps

Configure:

-   Azure Traffic Manager: both regional endpoints, Priority routing,
    health probes against `/health` with a short interval for demo
    purposes.
-   Azure Database for PostgreSQL **cross-region read replica** in the
    secondary region.
-   Azure Monitor / Log Analytics logs from both regions.
-   Azure Monitor alert rules: application unhealthy, database
    unreachable, replica lag threshold.
-   The database promotion script (`az postgres flexible-server replica
    promote`, planned mode default).
-   Recovery script for updating the app's DB connection target
    post-promotion.

Run the first controlled failover test: trigger `/simulate-failure` and
confirm Traffic Manager automatically redirects to the secondary
region.

### Software Development

Improve:

-   `/health` to genuinely reflect database connectivity.
-   `/simulate-db-failure` to make the app report `database_reachable:
    false` without touching the actual database.
-   Application logging, per region.
-   Database connection handling, including a config path that can be
    repointed to the promoted replica.
-   Recovery startup process.

### AI Engineering

Implement:

``` text
test incident JSON (post-failover)
        |
        v
Azure Function
        |
        v
Azure AI Foundry model, grounded on runbook + architecture doc
        |
        v
AI diagnosis + recommendation
```

Get the AI to return a structured response, and confirm it correctly
states that traffic failover has already happened automatically.

### Testing & Dashboard

Build the first Azure Dashboard (or Workbook).

Show:

-   Active region indicator.
-   Healthy/unhealthy status per region.
-   Error count.
-   CPU.
-   Alert state.
-   Replica lag.
-   Basic readiness score.

### End-of-day deliverable

**The team can trigger either controlled failure, see Azure Traffic
Manager automatically redirect traffic to the secondary region, see
Azure Monitor detect the underlying issue, and obtain an AI diagnosis.
The recovery runbook is finalized (v1) and ready to be used as an AI
grounding source.**

------------------------------------------------------------------------

# Day 3 --- End-to-end DR drill (both scenarios)

## Goal

Connect all pieces into the full story, for both the app-only and
database-unreachable scenarios.

### Infrastructure & DevOps

Complete:

-   Database promotion workflow, tested end-to-end.
-   Application recovery process in the secondary region.
-   Evidence collection across both regions.
-   Terraform cleanup/redeploy testing.

### Software Development

Fix:

-   Application recovery issues.
-   Configuration issues (especially DB connection repointing).
-   Failure endpoint issues.

### AI Engineering

Add:

-   Runbook + architecture-document grounding, confirmed working for
    both scenarios.
-   Evidence section distinguishing app-layer vs database-layer cause.
-   Confidence/uncertainty wording.
-   Human-execution wording (not "approval," since failover is already
    automatic — the human executes the recommended remediation).
-   Evaluation results for all six-plus test cases.

Optional:

-   Azure AI Search "on your data" grounding.

Only do this if the core flow is already working.

### Testing & Dashboard

Run the complete drill, **twice** (once per scenario):

**Scenario A — app-only failure**

1.  Healthy baseline (primary active).
2.  Trigger `/simulate-failure`.
3.  Traffic Manager automatically redirects to the secondary region.
4.  Alert appears.
5.  AI analyzes evidence and confirms nothing further is needed for
    traffic, but recommends investigating the primary.
6.  Record RTO (target: 5 minutes).

**Scenario B — database unreachable**

1.  Healthy baseline (primary active).
2.  Trigger `/simulate-db-failure`.
3.  Traffic Manager automatically redirects to the secondary region.
4.  Alert appears (database unreachable + replica lag).
5.  AI analyzes evidence and recommends the promotion step, citing the
    runbook.
6.  Human reviews and executes the promotion command.
7.  Recovery script updates the app's DB connection target.
8.  Application returns to full read/write service.
9.  Data integrity is checked against the replica's last synced state.
10. RTO/RPO recorded (targets: 15 minutes / 5 minutes).

### End-of-day deliverable

**A complete DR demonstration works for both scenarios: automatic
traffic failover in both cases, and human-executed database promotion
for the database-failure scenario, guided by an AI diagnosis grounded
in the runbook and architecture document.**

------------------------------------------------------------------------

# Day 4 --- Hardening, evidence and presentation

## Goal

Make the project understandable and demonstrable.

### Infrastructure & DevOps

Finish:

-   IaC cleanup, for both regions.
-   Security review, including the public-exposure trade-off from
    removing per-region load balancers (see the architecture and
    security docs).
-   Cost review across both regions.
-   Resource tagging (including primary/secondary region role).
-   Cleanup script/checklist for both regions.
-   Final architecture diagram.

### Software Development

Finish:

-   UI polish, including the active-region indicator.
-   Clear status messages.
-   Failure-demo labels on both test endpoints.
-   Final deployment instructions for both regions.

### AI Engineering

Finish:

-   Prompt catalogue.
-   AI evaluation results for both scenarios.
-   AI safety evidence, including confirmation that AI never triggers
    failover or promotion.
-   Final example outputs.
-   Limitations, including that the automated-promotion-trigger stretch
    goal was intentionally not built.

### Testing & Dashboard

Finish:

-   Dashboard.
-   Test report for both scenarios.
-   RTO/RPO evidence for both scenarios.
-   Screenshots.
-   Defect list.
-   Final evidence pack.

### Final demonstration

Use this sequence:

1.  Show architecture (both regions, Traffic Manager, primary +
    replica database).
2.  Show healthy application in the primary region.
3.  Show dashboard, including active-region indicator.
4.  Show current readiness score.
5.  Trigger the app-only failure — show Traffic Manager automatically
    redirecting traffic.
6.  Trigger the database-unreachable failure — show automatic traffic
    failover plus the database's read-only state.
7.  Show the AI incident summary and diagnosis.
8.  Show the AI recommendation (promotion step).
9.  Show the human executing the promotion.
10. Show the recovery completing and the application returning to full
    service.
11. Show recovered data and RPO evidence.
12. Show RTO for both scenarios.
13. Show final dashboard.
14. Explain security and AI safeguards — especially what is automatic
    vs. what requires a human.
15. Explain business value.
16. Explain what would be required before production (e.g. per-region
    load balancer instead of exposing VMs directly, automated
    promotion trigger with additional safeguards, failback automation).

------------------------------------------------------------------------

# Daily stand-up format

Every team should report:

1.  What we completed.
2.  What we are doing next.
3.  What is blocking us.
4.  What another team needs from us.

Keep blockers visible immediately. Do not allow a team to spend half a
day stuck on one Azure configuration problem — this is especially
important now that Traffic Manager and cross-region replication add new
surfaces to debug.

# Integration checkpoints

## Checkpoint 1 --- End of Day 1

Application reachable in the primary region; idle copy exists in the
secondary region; runbook skeleton complete.

## Checkpoint 2 --- End of Day 2

Automatic Traffic Manager failover verified for at least one scenario +
AI diagnosis generated + runbook v1 finalized.

## Checkpoint 3 --- End of Day 3

Full DR drill completed for **both** scenarios.

## Checkpoint 4 --- End of Day 4

Evidence pack + presentation ready.

# If the team falls behind

Use this priority order:

### Must have

1.  Application in the primary region.
2.  Idle application in the secondary region.
3.  Azure Traffic Manager with automatic Priority-routing failover.
4.  Primary database.
5.  Cross-region read replica.
6.  Azure Monitor (both regions).
7.  Controlled app-only failure + automatic traffic failover
    (Scenario A) — prioritize this scenario if only one can be fully
    demonstrated.
8.  Azure AI Foundry post-failover diagnosis.
9.  Human-executed database promotion (Scenario B) — if time is short,
    this can be demonstrated as a scripted manual step without the full
    AI loop, and finished as a stretch item.
10. Dashboard.
11. RTO evidence for at least Scenario A.

### Nice to have

-   Full Scenario B drill with RPO measurement.
-   Azure AI Search grounding.
-   Automated event trigger into AI.
-   Fancy dashboard.
-   Predictive trend analysis (replica lag early warning).
-   More sophisticated readiness scoring.
-   Automated report generation.
-   A documented (not built) failback plan walkthrough.

If necessary, remove the nice-to-have items — and if Day 3 is at risk,
it is acceptable to fully demonstrate only Scenario A live and present
Scenario B as a tested-but-not-live-demoed capability, as long as the
runbook and evidence clearly show it was exercised at least once.
