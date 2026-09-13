# Cloud-Native Disaster Recovery Platform --- Project Overview

## 1. What we are building

We are building a **small, safe, Azure-based disaster recovery prototype
for a banking-style web application**.

The goal is not to build a real banking platform or an enterprise DR
system. The goal is to demonstrate one clear idea:

> **Azure keeps the application observable and recoverable, while AI
> helps people understand what is happening and recommends what to do. A
> human remains responsible for approving recovery actions.**

The project will use only synthetic data and a sandbox environment.

## 2. The simple story

The demo should be easy for someone outside the project to understand:

1.  A small banking-style web app is running normally.
2.  The app is deployed across **two Availability Zones** behind an
    Azure Application Gateway.
3.  The application produces health checks, logs and metrics.
4.  A database stores fake customer/account records.
5.  Azure monitoring detects a problem.
6.  The AI component reviews the monitoring evidence and the recovery
    runbook.
7.  AI produces a short explanation and recommended recovery steps.
8.  A human reviews and approves the recovery.
9.  The recovery process restores or redeploys the application/database.
10. The dashboard shows what happened, how long recovery took, whether
    data was preserved, and what AI recommended.

## 3. What makes this an AI + DR project

AI should **not** be the thing that decides that a disaster has
happened.

Use normal Azure rules for things that must be predictable:

-   Is the application healthy?
-   Is an instance unhealthy?
-   Did a backup succeed?
-   Did the recovery test pass?
-   What was the measured RTO?
-   What was the measured RPO?

Use AI for things that benefit from interpretation:

-   Explain why the readiness score changed.
-   Summarize an incident from logs/alerts.
-   Identify unusual patterns in recent telemetry.
-   Recommend the relevant recovery-runbook steps.
-   Point out missing information or prerequisites.
-   Produce a short technical and executive summary after the drill.

This follows the facilitator's core design: deterministic controls
first, AI as decision support, and human approval before recovery.

## 4. Four workstreams

### Infrastructure & DevOps

Own the Azure environment, networking, VM/application deployment,
Application Gateway, database protection/recovery, monitoring
infrastructure, IaC and recovery automation.

### AI Engineering

Own Azure AI Foundry (Azure OpenAI Service) integration, prompts,
grounding/evidence, AI safety rules, evaluation tests and the AI
recommendation flow.

### Software Development

Own the simple banking-style application, health endpoint,
failure-simulation endpoint, synthetic data and any small API needed to
connect the application to the DR workflow.

### Testing, QA & Dashboard

Own the test plan, DR drill evidence, RTO/RPO measurement, dashboard,
screenshots/evidence pack and final validation.

## 5. What we are deliberately NOT building

To finish in four days, do not build:

-   A real banking system.
-   A production-grade multi-region platform.
-   Kubernetes/Azure Kubernetes Service (AKS).
-   Microservices.
-   Complex event-driven orchestration.
-   Autonomous AI agents that can change infrastructure.
-   A sophisticated ML model.
-   A custom vector database.
-   A complex CI/CD platform.
-   Customer data handling.
-   Full regulatory certification.

The prototype should be small enough that the entire team can understand
the complete flow.

## 6. Recommended Azure shape

**Application path**

`User -> Azure Application Gateway -> VM in Zone 1 / VM in Zone 2 -> Azure Database for PostgreSQL`

**Operations path**

`VMs/Database/Application Gateway -> Azure Monitor metrics + logs + alerts -> AI evidence -> Azure AI Foundry (Azure OpenAI) -> Human approval -> Recovery script/workflow`

**Dashboard**

Use a simple **Azure Dashboard** (or Azure Workbook) in Azure Monitor for
operational metrics and alerts. If the team needs a more polished
executive view, the dashboard team can add a very small web page that
reads a prepared JSON summary. Do not introduce a large BI platform
unless the team already knows it.

## 7. Definition of done

The project is successful when the team can demonstrate:

-   The application works normally.
-   Traffic is distributed across two Availability Zones.
-   A controlled failure can be triggered safely.
-   Monitoring detects the problem.
-   The dashboard reflects the problem.
-   AI receives approved evidence and produces a grounded
    explanation/recommendation.
-   AI does not execute recovery.
-   A human explicitly approves recovery.
-   Recovery completes.
-   The application passes health/integrity checks.
-   RTO and RPO evidence is captured.
-   The team can explain the architecture, security controls, AI
    controls and business value.

## 8. Source basis

The facilitator documents describe a controlled sandbox prototype using
synthetic data, transparent readiness rules, AI-assisted analysis, human
approval, recovery drills, monitoring, evidence collection and cost
controls. They explicitly exclude production customer data, autonomous
disaster declaration/failover and enterprise compliance certification.

The Azure version in this project keeps that same concept while
replacing AWS-native components with Azure equivalents such as Azure
Monitor, Azure Database for PostgreSQL, Virtual Machines, Blob Storage,
Azure Functions where useful, and Azure AI Foundry (Azure OpenAI
Service).
