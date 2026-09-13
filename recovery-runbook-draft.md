# Disaster Recovery Runbook — Cloud-Native DR Platform (Azure Prototype)

> **Status:** DRAFT — Day 1 skeleton. Sections marked `TBD (Day 2)` are
> completed once the real infrastructure, alert rules and recovery
> script exist. This document is the sandbox lab runbook for a
> synthetic workload only. It is not a production DR procedure.

| Field | Value |
| --- | --- |
| Document owner | Infrastructure & DevOps (named owner: `TBD`) |
| Contributors | Software Development, Testing/QA & Dashboard, AI Engineering |
| Version | v0.1 (skeleton) |
| Last reviewed | `TBD` |
| Environment | Sandbox / synthetic data only |
| Applies to | `app-dr-demo-<unique>` in `rg-dr-ai-capstone-dev` |

---

## 1. Purpose and scope

This runbook describes how to detect, evaluate, approve, execute and
verify recovery of the synthetic banking-style application used in this
capstone. It is the source document that:

- A human operator follows during the controlled DR drill.
- The Azure AI Foundry advisory agent is grounded on, so its
  recommendations cite real steps instead of inventing them.

It does **not** authorize any action outside the approved sandbox
resource group, and it does not apply to production systems.

## 2. Normal state (baseline)

Before any drill, confirm the following baseline is true and record it
as evidence:

- Application reachable through Azure Application Gateway at `TBD (Day 2)`.
- Two VM backend targets healthy across **Zone 1** and **Zone 2**.
- `GET /health` returns `{"status": "healthy", "server": "<zone-id>"}` from both targets.
- `GET /api/accounts` returns the synthetic account records without error.
- Azure Database for PostgreSQL — Flexible Server (`TBD (Day 2)` instance name) is running and accepting connections only from the application subnet.
- Most recent automated backup completed successfully within the last 24 hours (check Azure Portal → Backup and restore).
- Azure Monitor / Log Analytics is receiving logs and metrics from the App Gateway, VMs and database.
- Readiness score (see §6) is at or near 100%, with no open critical defects.

Capture a "before" screenshot of the dashboard and the readiness score
— this is required for the evidence pack.

## 3. Roles and authority

| Role | Responsibility | Who |
| --- | --- | --- |
| Recovery Operator | Only person authorized to approve and execute recovery. | `TBD` |
| AI Engineering owner | Confirms AI output before it reaches the operator; not authorized to approve recovery. | `TBD` |
| Infra & DevOps owner | Runs the recovery script; maintains this runbook. | `TBD` |
| Testing/QA owner | Confirms verification checks and records RTO/RPO. | `TBD` |

**Control boundary:** AI recommends. The Recovery Operator approves.
The recovery script executes. No other sequence is permitted.

## 4. Failure trigger — how the lab failure is created

The controlled failure is created using the application's dedicated
test endpoint, never by disabling real infrastructure controls unless
that is the specific scenario being tested.

1. Confirm the drill window, participants and rollback condition have
   been approved (see §9).
2. Capture the healthy baseline (§2).
3. Trigger the failure:
   ```
   POST /simulate-failure
   ```
   This is a **test-only endpoint** — do not call it outside an
   approved drill window.
4. Record the exact trigger timestamp (UTC). This is the reference
   point for RTO measurement.

Optional/secondary failure scenarios (use only if the team has time and
approval):
- Stop one VM instance to test Application Gateway health-probe
  failover between zones.
- Simulate a failed backup job to test the "missing evidence" AI
  behavior.

## 5. Detection — what should be observed

Within a few minutes of the trigger, confirm and record each of the
following, with a timestamp and screenshot:

- Application Gateway backend health drops to `0` (or reduced) healthy
  targets.
- `/health` returns an unhealthy status or stops responding.
- HTTP 5xx count increases in Azure Monitor.
- The configured Azure Monitor alert rule (`TBD (Day 2)` name) changes
  state to `Fired`.
- The dashboard reflects the incident state (e.g. `AMBER` / `RED`).

If any of these do **not** occur as expected, stop the drill, record
the discrepancy as a defect, and do not proceed to the human decision
step on faulty evidence.

## 6. Evidence package supplied to AI

The Infra/monitoring layer prepares a small JSON evidence package and
passes it, along with this runbook, to the Azure AI Foundry model. Do
not send raw logs, secrets or unfiltered data.

```json
{
  "application_status": "UNHEALTHY",
  "healthy_targets": 0,
  "http_5xx": 47,
  "last_backup": "2026-09-13T09:30:00Z",
  "readiness_score": 62,
  "recent_events": [
    "Health check failed",
    "Application returned HTTP 500"
  ],
  "rto_target_minutes": 15,
  "rpo_target_minutes": 5
}
```

Readiness score reference (weights, from the project specification):

| Indicator | Weight |
| --- | --- |
| Backup / protection status | 30% |
| Last recovery test result | 25% |
| Monitoring coverage | 15% |
| Runbook review status | 15% |
| Open recovery defects | 15% |

The AI must respond with: Incident summary, Evidence, Likely issue,
Recommended investigation, Relevant runbook step (cite the section
number from this document), Risks / missing evidence, Confidence, and
`Human approval required: YES`.

## 7. Human decision point

The AI's output is a **recommendation only**. The Recovery Operator
reviews:

- The active alert(s).
- The application status.
- The evidence package (§6).
- The AI recommendation and cited runbook section.
- This runbook's recovery steps (§8).

The operator then makes an explicit, recorded decision:

```
[ APPROVE RECOVERY ]     [ REJECT / INVESTIGATE FURTHER ]
```

If a UI approval button is not built in time, use a documented manual
command instead, e.g.:

```bash
./approve-recovery.sh --operator "<name>" --incident "<timestamp>"
```

Record the operator's name, decision, and timestamp. The AI must never
be able to invoke the recovery script directly.

## 8. Recovery procedure

Execute only after explicit operator approval (§7).

1. **Restore the database.**
   - Identify the most recent successful backup (or on-demand backup)
     for the Flexible Server instance.
   - Restore it as a new "recovery" server instance: `TBD (Day 2)` —
     exact Azure CLI / Portal steps once the database naming and
     resource group are finalized.
   - Record the recovery point (backup timestamp) used.
2. **Point the application at the recovery database.**
   - Update the application's connection configuration (via Key Vault
     / managed identity, not a hard-coded string) to the new recovery
     database endpoint.
3. **Restore or redeploy the application.**
   - Restart or redeploy the affected VM instance(s), or redeploy from
     the last known-good configuration via the Terraform/IaC pipeline.
4. **Confirm the application is serving traffic.**
   - Application Gateway backend health returns to healthy for both
     targets.
5. Record the recovery completion timestamp (UTC). This closes the RTO
   measurement window.

## 9. Verification

Run these checks before declaring recovery complete. All must pass:

- [ ] `GET /health` returns `healthy` from both zone targets.
- [ ] `GET /api/accounts` returns the expected synthetic records.
- [ ] Record counts in the recovery database match the last known-good
      count (or the documented acceptable data-loss window).
- [ ] Application Gateway reports both backend targets healthy.
- [ ] No new 5xx errors observed over a defined observation window
      (e.g. 5 minutes).
- [ ] Dashboard reflects a healthy/green state.

If any check fails, do not declare recovery complete — document the
failure and either retry the recovery step or escalate per §11.

## 10. RTO / RPO measurement

- **RTO** = Recovery completion timestamp (§8, step 5) minus Failure
  trigger / declaration timestamp (§4, step 4).
  - Lab target: **15 minutes.**
- **RPO** = Timestamp of the last protected data point (backup used in
  §8, step 1) minus the timestamp of the most recent data point that
  existed before the failure.
  - Lab target: **5 minutes.**

Record both figures, compare against the lab targets above, and note
any variance in the evidence pack.

## 11. Rollback / stop conditions

Stop the drill and roll back immediately if any of the following occur:

- The failure trigger affects anything outside the approved sandbox
  resource group.
- Any real, non-synthetic data is observed anywhere in the flow.
- The recovery database restore fails twice.
- A participant or reviewer calls a stop.
- The approved drill time window has elapsed.

Rollback action: stop the recovery script, restore the application to
its pre-drill healthy configuration (or redeploy from the last known
good IaC state), and notify the Infra & DevOps owner and executive
sponsor.

## 12. Cleanup

After the drill (successful or stopped):

- Export the evidence pack (screenshots, logs, AI responses, approval
  record, RTO/RPO figures) to the designated Blob Storage evidence
  container.
- Remove any temporary recovery database instances no longer needed.
- Confirm budget/cost alert status in Azure Cost Management.
- Confirm the resource deletion date is still correctly tagged.
- Close or transfer any open defects to the risk register.

## 13. Post-drill improvement notes

Record after each drill:

- What went well.
- What went wrong.
- Runbook steps that were unclear, missing, or incorrect (update this
  document directly).
- Follow-up actions and owners.

---

## Appendix A — Command reference (fill in Day 2)

```bash
# Health check
curl -s https://<app-gateway-endpoint>/health

# Trigger controlled failure (test-only)
curl -X POST https://<app-gateway-endpoint>/simulate-failure

# Manual approval (if no UI button)
./approve-recovery.sh --operator "<name>" --incident "<timestamp>"

# Database restore — TBD (Day 2): exact az CLI command once
# resource group / server names are finalized
```

## Appendix B — Change log

| Date | Change | By |
| --- | --- | --- |
| Day 1 | Initial skeleton created | Infra & DevOps + contributors |
| Day 2 | `TBD` — finalize with real commands, resource names, alert rule names | Infra & DevOps |
