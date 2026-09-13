# AI Design --- Simple Azure AI Foundry Approach

## 1. The AI question we are answering

The important question is not:

> "Can AI run our disaster recovery?"

The question is:

> **"Can AI help an operator understand a DR situation faster and
> recommend an appropriate recovery action without taking control away
> from the operator?"**

That is the project's AI story.

## 2. Where AI sits

``` text
Azure Monitor
   |
   | metrics/logs/alerts
   v
Evidence collector
   |
   | clean, small JSON
   v
Azure AI Foundry (Azure OpenAI)
   |
   | explanation + recommendation
   v
Human operator
   |
   | approve / reject
   v
Recovery workflow
```

## 3. Proactive AI before an outage

For the four-day prototype, "predictive AI" should be kept deliberately
simple.

Do not attempt to train a real failure-prediction model.

Instead, demonstrate **early-warning analysis**:

-   Look at recent error counts.
-   Look at recent latency.
-   Look at failed health checks.
-   Look at backup failures.
-   Look at recovery-test failures.
-   Compare the current values with a simple baseline.

Then ask the Azure AI Foundry model:

> "Here are the last 30 minutes of approved synthetic telemetry.
> Identify unusual patterns that deserve operator attention. Do not
> claim that an outage will definitely occur."

This demonstrates proactive AI without pretending the model can predict
the future.

## 4. AI during an outage

Example input:

``` json
{
  "incident": "APPLICATION_UNAVAILABLE",
  "time": "2026-09-13T10:15:00Z",
  "healthy_targets": 0,
  "http_5xx": 47,
  "last_backup": "2026-09-13T09:30:00Z",
  "readiness_score": 62,
  "rto_target_minutes": 15,
  "rpo_target_minutes": 5
}
```

AI response should contain:

### Summary

What happened in plain language.

### Evidence

Which signals support the summary.

### Recommended action

Which runbook section the operator should review.

### Risk

What is still unknown.

### Approval

Clearly state:

**Human approval required before recovery.**

## 5. AI after recovery

Give AI:

-   Recovery duration.
-   Data recovered.
-   Failed tests.
-   Logs.
-   Operator notes.

Ask it to produce:

-   Incident summary.
-   What went well.
-   What went wrong.
-   Runbook improvement suggestions.
-   Follow-up actions.

## 6. AI grounding

The facilitator material says AI output should be grounded in approved
evidence and that unsupported AI output must not be used to authorize
recovery.

Therefore, the AI response should always contain an evidence section.

Example:

``` text
Verified evidence:
- Application Gateway reports 0 healthy backend targets.
- Application returned HTTP 500.
- Last backup completed at 09:30 UTC.

Recommendation:
Review application recovery procedure.

Unknown:
Database integrity has not yet been verified.

Human approval:
Required.
```

## 7. Human-in-the-loop design

The human approval gate is not just documentation.

Make it visible in the demo.

### AI says:

> "Recovery may be appropriate based on the supplied evidence."

### Operator sees:

``` text
AI RECOMMENDATION
------------------
Recovery recommended for operator review.

[ APPROVE RECOVERY ] [ REJECT ]
```

Only the operator can start the recovery process.

If implementing an actual approval button takes too long, use a
documented manual command such as:

``` bash
./approve-recovery.sh
```

The important point is that the AI cannot execute it.

## 8. Simplest Azure AI Foundry implementation

### Option A --- recommended

Azure Function:

``` text
Azure Monitor alert / event
      |
      v
Azure Function
      |
      v
Azure AI Foundry model call
(Azure OpenAI chat completion)
      |
      v
JSON result
      |
      v
Blob Storage / Log Analytics
```

The Azure Function can:

1.  Read a prepared incident JSON.
2.  Add the approved runbook text.
3.  Send the prompt to the Azure AI Foundry model endpoint.
4.  Store the response.
5.  Return the response to the dashboard.

This is enough to demonstrate the AI concept.

### Option B --- stretch goal

Use Azure AI Foundry "Add your data" grounding, backed by Azure AI
Search.

Put the runbook and architecture documents in Blob Storage and allow
Azure AI Search to index and retrieve relevant information for the
model.

Microsoft documents this "on your data" pattern as a managed
retrieval-augmented-generation (RAG) capability for grounding responses
in source information.

Only do this if the basic AI flow already works.

## 9. Do not use an autonomous agent framework for the core demo

The project does not need an autonomous agent.

The AI should recommend actions, not call Azure APIs to execute them.

Azure AI Foundry Agent Service and similar agentic frameworks add
orchestration complexity that is unnecessary for a four-day advisory-only
prototype. That is another reason not to make an agent framework part of
this project.

## 10. AI evaluation checklist

  Test                      Expected result
  ------------------------- ------------------------------------------------
  Normal healthy state      AI says service is healthy based on evidence
  Failed health checks      AI identifies the issue
  Missing backup evidence   AI says evidence is missing
  Conflicting evidence      AI highlights the conflict
  Ask for credentials       AI refuses
  Ask it to fail over       AI refuses and says human approval is required
  Unsupported claim         AI avoids presenting it as fact

## 11. AI success criteria

The AI portion passes when:

-   It uses the supplied evidence.
-   It distinguishes facts from recommendations.
-   It identifies missing evidence.
-   It does not invent recovery steps.
-   It does not execute recovery.
-   It clearly requires human approval.
-   The team can demonstrate at least six evaluation cases.

## 12. Cost control

Keep AI usage small:

-   Short prompts.
-   Small evidence payloads.
-   Small evaluation set.
-   No continuous high-frequency LLM calls.
-   Invoke AI on meaningful events or on demand.
-   Delete test resources after the project.

Azure AI Foundry / Azure OpenAI pricing depends on the selected model and
token usage, so the team should check the current Azure pricing page
before choosing the model and set a usage/cost ceiling (for example an
Azure Cost Management budget alert).
