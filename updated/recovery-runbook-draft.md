# Disaster Recovery Runbook — Cloud-Native DR Platform (Azure Prototype, Multi-Region)

> **Status:** DRAFT v0.2 — updated for the multi-region architecture.
> Sections marked `TBD (Day 2)` are completed once the real
> infrastructure, alert rules and promotion script exist. This document
> is the sandbox lab runbook for a synthetic workload only. It is not a
> production DR procedure.

| Field | Value |
| --- | --- |
| Document owner | Infrastructure & DevOps (named owner: `TBD`) |
| Contributors | Software Development, Testing/QA & Dashboard, AI Engineering |
| Version | v0.2 (multi-region) |
| Last reviewed | `TBD` |
| Environment | Sandbox / synthetic data only |
| Applies to | `app-dr-demo-<unique>` in `rg-dr-ai-capstone-primary-dev` (primary region) and `rg-dr-ai-capstone-secondary-dev` (secondary region) |

---

## 1. Purpose and scope

This runbook describes how the platform detects a failure, how
Azure automatically fails application traffic over to the secondary
region, how a human evaluates and (where needed) promotes the
cross-region database replica, and how recovery is verified. It is the
source document that:

- A human operator follows during the controlled DR drill.
- The Azure AI Foundry advisory agent is grounded on — alongside the
  system architecture document — so its recommendations cite real
  steps for the correct scenario instead of inventing them.

It covers **two distinct failure scenarios**:

- **Scenario A — Application-only failure**: the primary region's
  application fails, but its database is still reachable.
- **Scenario B — Database-unreachable failure**: the primary region's
  application cannot reach its database.

It does **not** authorize any action outside the two approved sandbox
resource groups, and it does not apply to production systems.

## 2. Normal state (baseline)

Before any drill, confirm the following baseline is true and record it
as evidence:

- Azure Traffic Manager reports the **primary region** endpoint as
  healthy and actively receiving traffic (Priority 1).
- The **secondary region** endpoint is healthy but idle — warm standby,
  Priority 2, receiving no user traffic.
- `GET /health` on the primary region returns
  `{"status": "healthy", "server": "app-primary", "region": "primary", "database_reachable": true}`.
- `GET /health` on the secondary region returns the equivalent healthy
  response for `app-secondary`.
- `GET /api/accounts` returns the synthetic account records without
  error from the primary region.
- The **primary** Azure Database for PostgreSQL — Flexible Server
  (`TBD (Day 2)` instance name) is running and accepting connections
  only from the primary region's application subnet.
- The **cross-region read replica** (`TBD (Day 2)` instance name, in
  the secondary region) is in sync, with replication lag within the
  agreed threshold (baseline target: under 60 seconds).
- Azure Monitor / Log Analytics is receiving logs and metrics from
  Traffic Manager, both VMs, and both database servers.
- Readiness score (see §6) is at or near 100%, with no open critical
  defects.

Capture a "before" screenshot of the dashboard, the readiness score,
and Traffic Manager's endpoint status — this is required for the
evidence pack.

## 3. Roles and authority

| Role | Responsibility | Who |
| --- | --- | --- |
| Recovery Operator | Only person authorized to execute the database promotion and any other recovery action. Does **not** approve or trigger traffic failover — that is automatic. | `TBD` |
| AI Engineering owner | Confirms AI output before it reaches the operator; not authorized to execute recovery actions. | `TBD` |
| Infra & DevOps owner | Runs the promotion/recovery scripts; maintains this runbook. | `TBD` |
| Testing/QA owner | Confirms verification checks and records RTO/RPO for both scenarios. | `TBD` |

**Control boundary:**

- **Traffic failover (which region serves users) is automatic** —
  Azure Traffic Manager's health probes handle it. No human or AI step
  is required or possible here.
- **AI recommends** a diagnosis and next runbook step, invoked only
  *after* traffic failover has already occurred.
- **The Recovery Operator executes** the recommended step (most
  importantly, database promotion in Scenario B).
- **AI never executes anything** — it cannot call the promotion
  command, Traffic Manager, or any other Azure API.

## 4. Failure trigger — how the lab failure is created

The controlled failure is created using the application's dedicated
test endpoints, never by disabling real infrastructure controls unless
that is the specific scenario being tested.

### Scenario A — Application-only failure

1. Confirm the drill window, participants and rollback condition have
   been approved (see §11).
2. Capture the healthy baseline (§2).
3. Trigger the failure in the **primary** region:
   ```
   POST /simulate-failure
   ```
   This is a **test-only endpoint** — do not call it outside an
   approved drill window.
4. Record the exact trigger timestamp (UTC). This is the reference
   point for RTO measurement.

### Scenario B — Database-unreachable failure

1. Confirm the drill window, participants and rollback condition have
   been approved (see §11).
2. Capture the healthy baseline (§2).
3. Trigger the condition in the **primary** region:
   ```
   POST /simulate-db-failure
   ```
   This makes the application report `database_reachable: false` on
   `/health` **without** actually taking down the real database. This
   is a **test-only endpoint** — do not call it outside an approved
   drill window.
4. Record the exact trigger timestamp (UTC). This is the reference
   point for RTO measurement, and the reference point for the RPO
   calculation once promotion occurs.

Optional/secondary failure scenario (use only if the team has time and
approval, and only as a genuine regional-outage simulation, since it
uses the **Forced** promotion path described in §9):
- Stop the primary region's database server entirely to test the
  Forced promotion option and confirm the "possible data loss" warning
  behaves as documented.

## 5. Detection — what should be observed

Within a few minutes of the trigger, confirm and record each of the
following, with a timestamp and screenshot:

### Both scenarios

- Azure Traffic Manager marks the primary region's endpoint
  **Degraded**, and DNS resolution begins directing new requests to the
  **secondary** region's endpoint. This is automatic — no alert
  acknowledgement or human action is needed for this part.
- The dashboard's "active region" indicator switches to **SECONDARY**.

### Scenario A specific

- `/health` in the primary region returns an unhealthy status or stops
  responding, while `database_reachable` would have been `true` had it
  responded.
- HTTP 5xx count increases in Azure Monitor for the primary region.
- The application in the secondary region continues to serve reads and
  writes normally against the still-healthy **primary database**
  (cross-region call) — confirm this explicitly, since no database
  action is required in this scenario.

### Scenario B specific

- `/health` in the primary region reports `database_reachable: false`.
- The application in the **secondary** region, now receiving traffic,
  also cannot perform writes, because the read replica is still
  read-only.
- The configured Azure Monitor alert rule for "database unreachable"
  (`TBD (Day 2)` name) fires.
- Replica lag at the time of the incident is captured (this feeds the
  RPO calculation in §10).

If any of these do **not** occur as expected, stop the drill, record
the discrepancy as a defect, and do not proceed to the human decision
step on faulty evidence.

## 6. Evidence package supplied to AI

The Infra/monitoring layer prepares a small JSON evidence package and
passes it, along with this runbook **and the architecture document**,
to the Azure AI Foundry model. Do not send raw logs, secrets or
unfiltered data.

```json
{
  "incident": "DATABASE_UNREACHABLE",
  "time": "2026-09-13T10:15:00Z",
  "active_region": "secondary",
  "failover_occurred": true,
  "failover_timestamp": "2026-09-13T10:14:40Z",
  "primary_db_reachable": false,
  "replica_lag_seconds": 42,
  "replica_promoted": false,
  "http_5xx": 47,
  "readiness_score": 58,
  "recent_events": [
    "Traffic Manager marked primary endpoint Degraded",
    "Secondary region application returned HTTP 503 on database write"
  ],
  "rto_target_minutes": 15,
  "rpo_target_minutes": 5
}
```

For a Scenario A (app-only) evidence package, `primary_db_reachable`
would be `true` and `replica_lag_seconds` would be at baseline —
these fields are how the AI distinguishes the two scenarios.

Readiness score reference (weights, from the project specification):

| Indicator | Weight |
| --- | --- |
| Backup / replica sync status | 30% |
| Last recovery test result | 25% |
| Monitoring coverage (both regions) | 15% |
| Runbook review status | 15% |
| Open recovery defects | 15% |

The AI must respond with: Incident summary, Evidence, Likely cause
(app-layer vs. database-layer), a statement that traffic failover has
already happened automatically and needs no approval, Recommended next
step (cite the section number from this document), Risks / missing
evidence, Confidence, and `Human execution required: YES` for any
remaining recovery action.

## 7. Human decision point

The AI's output is a **diagnosis and recommendation only**, produced
*after* traffic has already failed over automatically. The Recovery
Operator reviews:

- The active alert(s).
- Traffic Manager's current routing state (already switched to
  secondary by this point).
- The evidence package (§6).
- The AI diagnosis, cited runbook section, and confidence.
- This runbook's recovery steps (§8).

The operator then makes an explicit, recorded decision:

**Scenario A:**
```
[ INVESTIGATE PRIMARY REGION ]     [ NO ACTION NEEDED YET ]
```
No database action is required — traffic is already correctly served
from the secondary region against the healthy primary database.

**Scenario B:**
```
[ EXECUTE DATABASE PROMOTION ]     [ REJECT / INVESTIGATE FURTHER ]
```

If a UI execution button is not built in time, use a documented manual
command instead, e.g.:

```bash
./execute-recovery.sh --operator "<name>" --incident "<timestamp>" --action "promote-replica"
```

Record the operator's name, decision, and timestamp. The AI must never
be able to invoke the promotion command, Traffic Manager, or any other
Azure API directly.

## 8. Recovery procedure

### Scenario A — Application-only failure

No database action is required. Recovery here means restoring the
**primary region**, not the user-facing service (which is already
being served correctly from the secondary region):

1. Investigate and repair the primary region's application (redeploy
   from the last known-good configuration via the Terraform/IaC
   pipeline, or restart the affected VM).
2. Confirm `/health` in the primary region returns healthy, including
   `database_reachable: true`.
3. Once healthy, decide whether to fail back to the primary region now
   or leave the secondary region active until a planned maintenance
   window (see §13 — failback is not required to be executed live
   during the 4-day drill, only documented).
4. Record the recovery completion timestamp (UTC). This closes the RTO
   measurement window for Scenario A.

### Scenario B — Database-unreachable failure

Execute only after explicit operator decision (§7).

1. **Choose the promotion mode.**
   - **Planned** (default): waits for the replica to fully catch up
     before promoting. No data loss. Use this for the controlled drill.
   - **Forced**: promotes immediately without waiting for full sync.
     May lose the most recent uncommitted changes. Use only to
     simulate a genuine regional outage where the primary cannot be
     waited on (see the optional scenario in §4).
2. **Promote the read replica:**
   ```bash
   az postgres flexible-server replica promote \
     --resource-group <secondary_resource_group> \
     --name <replica-server-name> \
     --promote-mode standalone \
     --promote-option planned
   ```
   Record the exact command used, the mode, and the completion
   timestamp.
3. **Point the application at the promoted database.**
   - Update the application's connection configuration (via Key Vault
     / managed identity, not a hard-coded string) in the **secondary**
     region to the newly promoted server's endpoint.
4. **Confirm the application is serving traffic with full read/write
   access.**
   - `/health` in the secondary region reports `database_reachable:
     true`.
5. Record the recovery completion timestamp (UTC). This closes the RTO
   measurement window for Scenario B.
6. Note the former primary database's role has now changed — do not
   attempt to write to it. Plan its cleanup or re-establishment as a
   new read replica per §13.

## 9. Promotion mode reference

| Mode | Data loss risk | When to use |
| --- | --- | --- |
| Planned / SwitchOver | None — waits for full sync | Controlled drill, primary is reachable enough to finish replicating |
| Forced / Standalone | Possible — loses changes not yet replicated | Genuine regional outage, primary cannot be reached to finish syncing |

Only use Forced promotion in the optional regional-outage scenario
(§4). Using it in the standard drill would understate the real RPO of
the Planned path and is not representative of the everyday failure
mode this project is built to demonstrate.

## 10. Verification

Run these checks before declaring recovery complete. All must pass:

**Both scenarios:**
- [ ] `GET /health` returns `healthy` (and `database_reachable: true`)
      from the currently active region.
- [ ] `GET /api/accounts` returns the expected synthetic records.
- [ ] Traffic Manager reports the active region's endpoint healthy.
- [ ] No new 5xx errors observed over a defined observation window
      (e.g. 5 minutes).
- [ ] Dashboard reflects a healthy/green state and the correct active
      region.

**Scenario B additional checks:**
- [ ] Record counts in the promoted database match the last known-good
      count (or the documented acceptable data-loss window based on
      replica lag at promotion time).
- [ ] The promoted server accepts write traffic successfully.
- [ ] The former primary is not receiving application writes.

If any check fails, do not declare recovery complete — document the
failure and either retry the recovery step or escalate per §11.

## 11. RTO / RPO measurement

- **RTO (Scenario A)** = Recovery completion timestamp (§8, Scenario A
  step 4) minus failure trigger timestamp (§4, Scenario A step 4).
  - Lab target: **5 minutes.** Most of this window is the automatic
    Traffic Manager failover; investigation/repair of the primary can
    continue after the RTO clock is considered closed for user impact.
- **RTO (Scenario B)** = Recovery completion timestamp (§8, Scenario B
  step 5) minus failure trigger timestamp (§4, Scenario B step 4).
  - Lab target: **15 minutes.**
- **RPO (Scenario B only)** = Timestamp of the last data point
  successfully replicated to the replica before promotion, minus the
  timestamp of the most recent data point that existed on the primary
  before the failure. Use the replica lag value captured at the time of
  the incident (§5) as the basis for this calculation.
  - Lab target: **5 minutes.**
- Scenario A has no meaningful RPO — the database is never touched.

Record RTO (and RPO where applicable) for whichever scenario was run,
compare against the lab targets above, and note any variance in the
evidence pack.

## 12. Rollback / stop conditions

Stop the drill and roll back immediately if any of the following occur:

- The failure trigger affects anything outside the two approved
  sandbox resource groups.
- Any real, non-synthetic data is observed anywhere in the flow.
- The replica promotion fails twice.
- Traffic Manager does not fail over within a reasonable multiple of
  the configured health-check interval and TTL — investigate the
  Traffic Manager configuration before proceeding, rather than forcing
  a manual DNS change.
- A participant or reviewer calls a stop.
- The approved drill time window has elapsed.

Rollback action: stop any in-progress promotion or recovery script,
restore the application to its pre-drill healthy configuration in the
primary region (or redeploy from the last known-good IaC state), and
notify the Infra & DevOps owner and executive sponsor. If a replica was
promoted during the drill, plan its cleanup/re-creation separately —
do not attempt to reverse a promotion live during the drill window.

## 13. Cleanup

After the drill (successful or stopped):

- Export the evidence pack (screenshots, logs, AI responses, operator
  decision record, RTO/RPO figures for whichever scenario ran) to the
  designated Blob Storage evidence container.
- If a database was promoted during the drill:
  - Confirm the former primary is not silently still receiving writes.
  - Either re-establish it as a new cross-region read replica of the
    promoted server (if the team wants the topology restored for
    another drill), or decommission it, depending on time remaining.
  - Document this as a **failback plan** rather than executing a live
    failback unless time remains after the primary drill (see
    01-project-specification-and-architecture.md §15).
- Remove any temporary resources no longer needed, in both regions.
- Confirm budget/cost alert status in Azure Cost Management for both
  regions.
- Confirm the resource deletion date is still correctly tagged in both
  resource groups.
- Close or transfer any open defects to the risk register.

## 14. Post-drill improvement notes

Record after each drill:

- Which scenario(s) were run.
- What went well.
- What went wrong.
- Runbook steps that were unclear, missing, or incorrect (update this
  document directly).
- Whether the Planned promotion path was sufficient, or whether the
  team needed to test Forced promotion.
- Follow-up actions and owners.

---

## Appendix A — Command reference (fill in Day 2)

```bash
# Health check (per region)
curl -s https://<primary-endpoint>/health
curl -s https://<secondary-endpoint>/health

# Trigger controlled app-only failure (test-only, primary region)
curl -X POST https://<primary-endpoint>/simulate-failure

# Trigger controlled database-unreachable failure (test-only, primary region)
curl -X POST https://<primary-endpoint>/simulate-db-failure

# Manual execution (if no UI button)
./execute-recovery.sh --operator "<name>" --incident "<timestamp>" --action "promote-replica"

# Database promotion — planned (no data loss, controlled drill)
az postgres flexible-server replica promote \
  --resource-group <secondary_resource_group> \
  --name <replica-server-name> \
  --promote-mode standalone \
  --promote-option planned

# Database promotion — forced (regional-outage simulation only)
az postgres flexible-server replica promote \
  --resource-group <secondary_resource_group> \
  --name <replica-server-name> \
  --promote-mode standalone \
  --promote-option forced

# Check Traffic Manager endpoint status
az network traffic-manager endpoint show \
  --resource-group <resource_group> \
  --profile-name <tm-profile-name> \
  --name <endpoint-name> \
  --type azureEndpoints
```

## Appendix B — Change log

| Date | Change | By |
| --- | --- | --- |
| Day 1 | Initial single-region skeleton created | Infra & DevOps + contributors |
| Day 1/2 | Rewritten for multi-region architecture: automatic Traffic Manager failover, cross-region read replica, two failure scenarios (app-only vs. database-unreachable), human-executed promotion, updated evidence JSON and roles | Infra & DevOps + contributors |
| Day 2 | `TBD` — finalize with real commands, resource names, alert rule names | Infra & DevOps |
