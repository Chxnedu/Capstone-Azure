# Four-Day Implementation Guide

## Day 1 --- Foundation and first working application

### Goal

By the end of Day 1, the application should be running in Azure and the
basic architecture should exist.

### Infrastructure & DevOps

Build:

-   Azure subscription/resource group and resource tagging.
-   Virtual Network (VNet).
-   Two Availability Zones.
-   Two Virtual Machines.
-   Network security groups.
-   Azure Application Gateway (or Load Balancer).
-   Azure Database for PostgreSQL --- Flexible Server.
-   Basic managed identities / role assignments.

Start Terraform.

### Software Development

Build:

-   Simple web page.
-   `/health`.
-   `/api/accounts`.
-   Synthetic database data.
-   `/simulate-failure`.

Deploy the application manually first if needed.

### AI Engineering

Prepare:

-   AI role and boundaries.
-   Prompt draft.
-   Example incident JSON.
-   Example runbook.
-   Six AI evaluation cases.

Do not spend Day 1 building a complex RAG system.

### Testing & Dashboard

Create:

-   Test plan.
-   Dashboard wireframe.
-   RTO/RPO measurement sheet.
-   Evidence folder structure.

### End-of-day deliverable

**A user can open the application through Azure Application Gateway and
see the synthetic banking-style application.**

------------------------------------------------------------------------

# Day 2 --- Monitoring, recovery and first AI call

## Goal

Make the system observable and prove that a failure can be detected.

### Infrastructure & DevOps

Configure:

-   Azure Monitor / Log Analytics logs.
-   Azure Monitor alert rules.
-   Application Gateway health probes.
-   VM monitoring (Azure Monitor Agent).
-   Azure Database for PostgreSQL backup.
-   Recovery script.

Create the first controlled recovery test.

### Software Development

Improve:

-   Failure simulation.
-   Application logging.
-   Database connection handling.
-   Recovery startup process.

### AI Engineering

Implement:

``` text
test incident JSON
        |
        v
Azure Function
        |
        v
Azure AI Foundry model
        |
        v
AI recommendation
```

Get the AI to return a structured response.

### Testing & Dashboard

Build the first Azure Dashboard (or Workbook).

Show:

-   Healthy backend targets.
-   Error count.
-   CPU.
-   Alert state.
-   Basic readiness score.

### End-of-day deliverable

**The team can trigger a controlled failure, see Azure Monitor detect
it, and obtain an AI explanation/recommendation.**

------------------------------------------------------------------------

# Day 3 --- End-to-end DR drill

## Goal

Connect all pieces into the full story.

### Infrastructure & DevOps

Complete:

-   Recovery workflow.
-   Database restore process.
-   Application recovery.
-   Evidence collection.
-   Terraform cleanup/redeploy testing.

### Software Development

Fix:

-   Application recovery issues.
-   Configuration issues.
-   Database connection issues.
-   Failure endpoint issues.

### AI Engineering

Add:

-   Runbook grounding.
-   Evidence section.
-   Confidence/uncertainty wording.
-   Human approval wording.
-   Evaluation results.

Optional:

-   Azure AI Search "on your data" grounding.

Only do this if the core flow is already working.

### Testing & Dashboard

Run the complete drill:

1.  Healthy baseline.
2.  Trigger failure.
3.  Alert appears.
4.  AI analyzes evidence.
5.  Human reviews.
6.  Human approves recovery.
7.  Recovery runs.
8.  Application returns.
9.  Data integrity is checked.
10. RTO/RPO recorded.

### End-of-day deliverable

**A complete DR demonstration works from failure through human-approved
recovery.**

------------------------------------------------------------------------

# Day 4 --- Hardening, evidence and presentation

## Goal

Make the project understandable and demonstrable.

### Infrastructure & DevOps

Finish:

-   IaC cleanup.
-   Security review.
-   Cost review.
-   Resource tagging.
-   Cleanup script/checklist.
-   Final architecture diagram.

### Software Development

Finish:

-   UI polish.
-   Clear status messages.
-   Failure-demo labels.
-   Final deployment instructions.

### AI Engineering

Finish:

-   Prompt catalogue.
-   AI evaluation results.
-   AI safety evidence.
-   Final example outputs.
-   Limitations.

### Testing & Dashboard

Finish:

-   Dashboard.
-   Test report.
-   RTO/RPO evidence.
-   Screenshots.
-   Defect list.
-   Final evidence pack.

### Final demonstration

Use this sequence:

1.  Show architecture.
2.  Show healthy application.
3.  Show dashboard.
4.  Show current readiness score.
5.  Trigger controlled failure.
6.  Show monitoring alert.
7.  Show AI incident summary.
8.  Show AI recommendation.
9.  Show human approval.
10. Run recovery.
11. Show restored application.
12. Show recovered data.
13. Show RTO/RPO.
14. Show final dashboard.
15. Explain security and AI safeguards.
16. Explain business value.
17. Explain what would be required before production.

------------------------------------------------------------------------

# Daily stand-up format

Every team should report:

1.  What we completed.
2.  What we are doing next.
3.  What is blocking us.
4.  What another team needs from us.

Keep blockers visible immediately. Do not allow a team to spend half a
day stuck on one Azure configuration problem.

# Integration checkpoints

## Checkpoint 1 --- End of Day 1

Application reachable.

## Checkpoint 2 --- End of Day 2

Failure detected + AI recommendation generated.

## Checkpoint 3 --- End of Day 3

Full DR drill completed.

## Checkpoint 4 --- End of Day 4

Evidence pack + presentation ready.

# If the team falls behind

Use this priority order:

### Must have

1.  Application.
2.  Two Availability Zones.
3.  Application Gateway.
4.  Database.
5.  Azure Monitor.
6.  Controlled failure.
7.  Recovery process.
8.  Human approval.
9.  Azure AI Foundry recommendation.
10. Dashboard.
11. RTO/RPO evidence.

### Nice to have

-   Azure AI Search grounding.
-   Automated event trigger into AI.
-   Fancy dashboard.
-   Predictive trend analysis.
-   More sophisticated readiness scoring.
-   Automated report generation.

If necessary, remove the nice-to-have items.
