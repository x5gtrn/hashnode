---
title: "Making AI Agents Safe and Fast"
datePublished: 2026-09-22T09:45:01.980Z
cuid: cmuchn0ut00010agm2sabe9gb
slug: article-2026-09-22-1843
cover: https://cdn.hashnode.com/uploads/covers/62d5556b2f40e31decd90345/08b1f246-fe59-4af5-88e7-19830e6b3c8c.jpg
tags: software-architecture, automation, developer-tools, obsidian, ai-agents, ai-safety, jev

---

An agent investigating a production incident might read logs, search Jira, inspect GitHub, patch a service, and suggest a deployment. Each step needs different evidence and different authority. A useful explanation of the outage does not establish that the proposed migration is safe to execute.

That is where agent engineering becomes systems engineering. We need to manage context, select an appropriate worker, control side effects, and preserve enough evidence to explain what happened afterward.

This article develops an architecture with three complementary components: **Obsidian for curated knowledge, Jev for narrow probabilistic judgments, and a meta-harness for shared execution policy**. Claude Code provides the main coding-agent example. The integration described here is an engineering design you can build, rather than a bundled product or a benchmarked deployment.

%[https://speakerdeck.com/x5gtrn/making-ai-agents-safe-and-fast-jev-obsidian-and-the-meta-harness] 

*Technical references checked September 22, 2026. API examples follow the published contracts; no authenticated Jev or Claude Code integration was run for this article.*

## 1\. Give each component a specific job

Four problems repeatedly slow down agent workflows: oversized context, unreliable tools, poor model selection, and excessive approval traffic. Buying more reasoning capacity does not directly resolve any of them.

Consider a billing incident. Feeding the agent an entire engineering vault increases the amount it must process and exposes unrelated material. Letting it choose any tool creates unpredictable execution paths. Sending every classification to a reasoning model adds latency. Asking an engineer to approve every harmless lookup trains that engineer to click through prompts.

A better starting point is an explicit division of responsibility:

| Component | Responsibility | Output the next component can use |
| --- | --- | --- |
| Knowledge layer | Supply current, relevant, permitted evidence | A bounded context packet with source versions |
| Generative agent | Investigate, explain, write code, propose actions | A patch or a concrete tool request |
| Decision layer | Evaluate narrow semantic questions | Categories, distributions, scores, probabilities |
| Policy and approval layer | Decide what the requester may execute | ALLOW, ASK, or DENY, with a reason |
| Execution layer | Perform an authorized operation | A result and evidence of its effects |

This separation lets you improve one part without replacing the whole system. Retrieval can change independently of the coding model. A policy update can take effect without rewriting every prompt. A classifier can recommend escalation while ordinary code retains control of authorization.

![](https://cdn.hashnode.com/uploads/covers/62d5556b2f40e31decd90345/386092d9-ded4-4f0e-9c50-bb4280b8c158.jpg align="center")

*Figure 1. Proposed integration architecture. Jev supplies signals; the policy gate combines them with deterministic controls and human approval.*

## 2\. Jev: typed judgments for software

TypeSafe introduced Jev in September 2026 as its first **System One model**. Its interface accepts application state and typed questions, then returns structured decisions instead of generated prose. TypeSafe explicitly explains that Jev cannot replace the text-generating model inside a coding agent. Use the agent to investigate and write; call Jev where your application needs a bounded judgment. See the [Jev introduction](https://docs.typesafe.ai/introduction) and [coding-agent guidance](https://docs.typesafe.ai/introduction/coding-agents).

The three primitives serve different purposes:

| Primitive | Engineering question | Response | Practical caution |
| --- | --- | --- | --- |
| [Choice](https://docs.typesafe.ai/primitives/choice) | Which team owns this incident? | Selected option, probabilities, confidence | Include an `unknown` option when ownership may be absent |
| [Score](https://docs.typesafe.ai/primitives/score) | How severe is the observed impact? | Weighted score, level probabilities, confidence | Define an ordered rubric; the score is not necessarily between 0 and 1 |
| [Noul](https://docs.typesafe.ai/primitives/noul) | Could this change remove persisted data? | Probability that the answer is yes | No separate `confidence` field |

A four-level Score uses positions 0 through 3. Its weighted value can fall between positions. Inspect the probability mass on critical outcomes when consequences matter; averaging can conceal a meaningful probability of severe harm.

For Choice and Score, `confidence` summarizes the shape of the output distribution. It is not a universal probability that the answer is correct. A sharply peaked answer can still be wrong on unfamiliar inputs. The [confidence documentation](https://docs.typesafe.ai/confidence) recommends choosing thresholds using your own domain and observations.

### Ask atomic questions and combine the answers in code

There are alternatives worth comparing before adding another service:

| Approach | Best fit | Trade-off to evaluate |
| --- | --- | --- |
| Deterministic rules | Exact permissions, arithmetic, known identifiers | Semantic variation can make rules brittle |
| Jev | Frequent, bounded judgments with explicit options | Requires domain evaluation and a hosted dependency |
| Generative model with structured output | Decisions requiring extended investigation or explanations | Measure generation latency, cost, and semantic correctness |

For each option, a valid output schema establishes shape; correctness still needs evidence.

“Is this release safe, compliant, urgent, and approved?” combines several judgments with different evidence requirements. Split them. Calculate approval validity and budget limits deterministically. Ask the model about semantic properties that are difficult to express as exact rules.

Here is a complete HTTP request using a fictional, sanitized release proposal. Set `TYPESAFE_API_KEY` in the environment before running it. This uses the [official API endpoint and request structure](https://docs.typesafe.ai/api):

```bash
curl --fail-with-body --max-time 5 \
  https://api.typesafe.ai/v1/systemone \
  -H "Authorization: Bearer $TYPESAFE_API_KEY" \
  -H "Content-Type: application/json" \
  --data-binary @- <<'JSON'
{
  "model": "jev-1.13.0",
  "state": {
    "service": "billing-api",
    "environment": "staging",
    "proposal": "Add a nullable invoice_reference column; retain all rows.",
    "observed_impact": "Duplicate invoice display for one test account."
  },
  "questions": {
    "owner": {
      "type": "choice",
      "instructions": "Which team should investigate this change?",
      "criteria": {
        "billing": "Invoice behavior and billing data models",
        "platform": "Shared infrastructure and deployment machinery",
        "unknown": "Insufficient evidence to choose an owner"
      }
    },
    "impact": {
      "type": "score",
      "instructions": "Rate the observed customer impact.",
      "criteria": [
        "No production customer impact",
        "Limited degradation with a workaround",
        "Broad disruption of a core workflow",
        "Data loss or critical service outage"
      ]
    },
    "destructive": {
      "type": "noul",
      "instructions": "Could the proposed change remove persisted data?"
    }
  }
}
JSON
```

The keys `owner`, `impact`, and `destructive` identify answers for your code. Put the actual meaning in the instructions and criteria. A real gate should receive the resolved operation and relevant diff, not rely solely on an agent-written summary that might omit the dangerous part.

TypeSafe supports evaluating multiple questions against shared state in parallel. Batch independent questions when they need the same evidence. A later question that depends on an earlier answer still requires another step. [The fan-out documentation](https://docs.typesafe.ai/patterns/fan-out) explains the pattern and its token trade-offs.

### Treat speed and price as inputs to your own measurement

TypeSafe's launch post reports 70–500 ms end-to-end latency and notes that its published evaluations generally ran from the US West Coast. Those are vendor measurements, not your production latency budget. Measure from your deployment region with realistic context and concurrency. See the [launch report and its qualifications](https://typesafe.ai/blog/introducing-system-one-models-and-jev).

The [model reference](https://docs.typesafe.ai/models) currently lists Jev 1.13 at $0.042 per million input tokens, with output tokens free. Ten thousand evaluations averaging 1,000 billed input tokens would therefore cost about **$0.42 for model input**, excluding retries, retrieval, infrastructure, and the main agent. Use observed token usage, including question overhead, for budgeting.

Latency still accumulates: twelve sequential gates averaging 200 ms add 2.4 seconds. Skip semantic evaluation when an exact policy rule already resolves the operation. Optimize total task time and review effort, rather than calls per second alone.

## 3\. Make Obsidian useful as agent memory

Obsidian stores notes as local Markdown files in a vault. That makes the knowledge accessible to ordinary tooling, version control, and retrieval pipelines. It also means your integration must supply access control, selection, and synchronization behavior. Obsidian itself does not turn a folder into trustworthy agent memory. See [how Obsidian stores data](https://help.obsidian.md/Files+and+folders/How+Obsidian+stores+data).

Separate two kinds of knowledge:

*   **Operational knowledge:** reviewed runbooks, repository conventions, release procedures, ownership, and escalation rules.
    
*   **Reference knowledge:** design discussions, vendor documentation, incident reports, meeting notes, and supporting evidence.
    

Operational knowledge has a controlled promotion process. A newly imported document or an agent's tentative conclusion belongs in reference material until reviewed. Otherwise, one mistaken generated note can become an instruction for every future run.

An illustrative vault structure is sufficient to start:

```text
engineering-vault/
  operations/billing-release.md
  architecture/billing-data-model.md
  incidents/duplicate-invoice-display.md
  references/database-migrations.md
```

Use [Obsidian properties](https://help.obsidian.md/properties) to record metadata your retrieval service can evaluate:

```yaml
---
project: billing
knowledge_kind: operational
owner: payments-platform
reviewed_on: 2026-09-20
review_after: 2026-10-20
source_revision: billing-runbook-v12
sensitivity: internal
status: approved
---
```

These are proposed application fields, not special Obsidian permissions. Your integration must validate them and ensure an untrusted writer cannot mark its own document approved.

### Retrieve a small, sufficient packet

First restrict candidates to the requesting identity's permitted projects and sensitivity levels. Search that authorized set by topic and service, expand relevant links, then rank passages for usefulness. Apply a token budget while retaining the evidence needed to answer the question. If essential information is missing, fetch more or escalate.

For the billing incident, the packet might contain the current migration runbook, the affected table definition, the last deployment diff, and a short incident timeline. Attach source identifiers and revisions so the agent can cite evidence and a reviewer can reproduce the selection.

Jev can help rank semantic relevance, and TypeSafe provides a [RAG passage classification example](https://docs.typesafe.ai/cookbooks/classifying_rag_passages). Enforce document permissions before sending candidates to any hosted model. Comparing review dates is ordinary code; judging whether a note still explains a changed API is a semantic task.

Retrieved content remains evidence. A pasted instruction inside an incident report must not gain the authority of a reviewed policy. Keep trusted instructions and retrieved passages distinguishable in the context packet, and enforce sensitive actions outside the prompt.

## 4\. Define the harness and the shared operating layer

A harness is the software around an agent: instructions, tools, memory, the execution loop, and guardrails. It determines what the model can observe and which proposed actions can actually run.

In this article, **meta-harness** means an operating layer that applies common context, policy, approval, and audit contracts across multiple harnesses. This is the architectural meaning used in the source slides. A research system also named [Meta-Harness](https://arxiv.org/abs/2603.28052) searches over harness code to optimize applications; that is a different use of the term.

Start with one workflow and ordinary interfaces. Add a second harness when you have a concrete need for shared behavior. Centralization is useful when a terminal agent and a background worker must apply the same release restrictions. It also adds a dependency that can fail, so define degraded behavior before expanding it.

The shared action envelope should identify the requester, project, tool, resolved arguments, target environment, relevant revisions, and policy version. The gate returns a decision, a stable reason code, and an audit identifier. The executor checks that the operation it is about to perform is still the operation that was evaluated.

This matters during approval delays. If the patch, deployment artifact, or destination changes, the old approval should no longer authorize execution. Bind approval to the exact action and give it an expiry.

![](https://cdn.hashnode.com/uploads/covers/62d5556b2f40e31decd90345/3a8601bc-c057-4fdb-a6a3-94b6856affda.jpg align="center")

*Figure 2. Route and select at task boundaries; guard each proposed action; review actual outcomes before declaring completion.*

The operating loop is **ROUTE → SELECT → GUARD → EXECUTE → REVIEW**. Routing chooses the project, owner, and worker. Selection supplies evidence. The guard evaluates a concrete proposal. Execution uses scoped tools. Review checks results and decides whether another bounded iteration is necessary.

Model selection belongs in routing too. Use code for exact lookups and arithmetic, Jev for narrow classification, a generative agent for investigation and edits, and human judgment where authority or unresolved consequences require it.

## 5\. Turn ALLOW, ASK, and DENY into behavior

Assign each outcome an explicit meaning:

| Outcome | Example in this design | Required behavior |
| --- | --- | --- |
| ALLOW | Authorized read of a non-sensitive runbook | Execute within the granted scope and record the result |
| ASK | Publishing externally or deploying to production | Hold the exact action for approval |
| DENY | Secret extraction, forbidden target, or absent authority | Block execution and explain the violated rule |

A low model risk estimate cannot override a missing permission. Similarly, a valid approval cannot override a categorical prohibition.

The following pure policy function illustrates precedence. `facts` must come from trusted adapters and authorization checks; it must not be accepted as an agent's self-description. The probabilities and threshold are illustrative and need workload-specific evaluation.

```python
from math import isfinite


def decide(facts, p_destructive):
    # Missing or malformed facts never enter the automatic path.
    required = (
        "authorized", "forbidden", "requires_approval",
        "bounded_operation", "evidence_complete",
    )
    if any(type(facts.get(k)) is not bool for k in required):
        return "DENY", "invalid_facts"
    if not facts["authorized"] or facts["forbidden"]:
        return "DENY", "hard_policy"
    if facts["requires_approval"]:
        return "ASK", "explicit_approval_required"
    if not facts["bounded_operation"] or not facts["evidence_complete"]:
        return "ASK", "insufficient_scope_or_evidence"
    if type(p_destructive) not in (int, float):
        return "ASK", "evaluator_unavailable"
    if not isfinite(p_destructive) or not 0 <= p_destructive <= 1:
        return "ASK", "invalid_probability"
    if p_destructive >= 0.01:
        return "ASK", "semantic_risk"
    return "ALLOW", "within_preapproved_scope"
```

`ASK` leaves the action unexecuted. An unattended worker needs a durable pending queue and an approval expiry; it cannot interpret the absence of a reviewer as consent. Approval handling is a separate executor path that rechecks the current action, authority, and immutable prohibitions.

In practice, use narrow operations such as `read_incident` or `propose_release`, validated schemas, restricted credentials, and bounded destinations. A command's name alone says little about its side effects: a test script can access the network or run arbitrary repository code. General shell access needs isolation as well as a policy decision.

### Integrating with Claude Code

Keep concise workflow guidance in `CLAUDE.md`, such as where reviewed runbooks live and which tests establish completion. Claude Code's [memory documentation](https://code.claude.com/docs/en/memory) explicitly distinguishes this context from enforced configuration. Put actual restrictions in [permissions](https://code.claude.com/docs/en/permissions), tool adapters, and the execution environment.

A `PreToolUse` integration can translate a policy result into the documented hook response:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "ask",
    "permissionDecisionReason": "Production release requires approval of this artifact."
  }
}
```

There is an important failure case: command, HTTP, and MCP-tool hook timeouts can leave the normal permission flow running. A crashed command hook is not automatically a blocking decision either. Handle evaluator errors explicitly, test timeout behavior, and enforce critical restrictions at the tool or service boundary. The [hooks reference](https://code.claude.com/docs/en/hooks) documents the event-specific behavior. A hook improves coordination, but production credentials should remain unavailable to an unauthorized execution path.

## 6\. Apply the design to three engineering workflows

The slides identify five useful coding-agent activities: codebase navigation, debugging from logs, legacy refactoring, adding tests, and operating external tools. These become much more useful when the workflow has a concrete success condition.

For legacy refactoring, first map callers and dependencies, capture current behavior in characterization tests, then change one bounded component. Review compatibility and regression results before expanding the patch. This gives navigation, code changes, and test generation distinct deliverables instead of treating a plausible rewrite as proof of success.

### Incident response across GitHub, Jira, and Slack

Start with a symptom and a service identifier. Retrieve sanitized logs, recent changes, matching issues, and the relevant runbook. The agent assembles hypotheses and proposes a reproduction. Jev supplies ownership and impact judgments; existing alert rules still escalate known critical conditions immediately.

For ambiguous ownership, route to the triage queue instead of selecting the least unlikely team. For a routine defect, let the agent prepare a patch and regression test in an isolated checkout. Review should establish whether the test fails before the fix and passes afterward.

Slack alerts are external writes. Use an approved channel, a bounded message template, and an incident-based deduplication key. If a send times out, reconcile its status before retrying to avoid repeated notifications. The [Claude Code MCP guide](https://code.claude.com/docs/en/mcp) explains how external tools connect; your chosen servers and credentials determine the available operations.

### Release operations and change control

A release agent can collect test results, prepare notes, and inspect a migration plan. Give the gate structured evidence: artifact digest, target environment, schema changes, expected downtime, rollback procedure, and approval requirements.

Jev can flag a potentially destructive change for review. Exact checks should establish whether required tests passed, an artifact matches the approved digest, and the target is permitted. In this design, production deployment always requires explicit approval; a low-risk staging operation may fit a preapproved scope.

Use the deployment platform's own controls too. [GitHub environment protection rules](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments) support reviewer approval and branch restrictions, subject to plan and repository availability. Keep deployment credentials behind those controls.

A successful command is only intermediate evidence. Review rollout health against defined metrics and a time window. Record rollback separately from success, and recognize that irreversible data changes may require a tested restore plan rather than a simple rollback command.

### Keeping design knowledge current

After a merged API change, compare the diff with linked design notes and internal wiki pages. The agent drafts updates; Jev can help rank likely contradictions and the scope of affected documentation.

For example, a retry-policy change may affect a runbook, a client integration guide, and an incident response procedure. Track source revisions, show reviewers the exact differences, and publish updates only to authorized destinations. Preserve existing human edits when the source has changed during review.

Age alone is insufficient evidence of staleness. A two-year-old invariant may still be correct, while yesterday's generated note may already contradict the merged code. Keep factual source revisions and semantic review as separate signals.

For all three workflows, MCP supplies a tool interface. It does not establish business authorization merely because a tool is discoverable. Its [tool specification](https://modelcontextprotocol.io/specification/2025-06-18/server/tools) calls for validation, access controls, and human oversight for sensitive operations.

## 7\. Roll out through evidence, then expand

Start in observation mode on a low-consequence workflow such as incident ownership suggestions. Log proposed decisions alongside the human outcome while preserving existing controls. Include ambiguous examples, missing context, misleading logs, and hostile instructions in the evaluation set.

Measure the rate of unsafe automatic approvals, unnecessary escalations, routing errors, and missing evidence. Break results down by operation and language. Also measure p95 gate latency, total task time, context size, review effort, and cost per completed task. A cheaper classifier that creates more human work may increase total cost.

Tune thresholds on a development set and report results on a separate held-out set. Then enable automatic execution only for explicitly bounded operations. Expand to more projects after observing acceptable outcomes and exercising the failure paths.

The [Jev 1.13 limitations](https://docs.typesafe.ai/model-jaggedness/jev-1.13) include numerical precision, date comparisons, indirection, irrelevant context, and adversarial content. Keep arithmetic and structural invariants in code. Pin the model version when evaluating thresholds, and rerun the evaluation when the model, question wording, or context-selection logic changes.

Store an audit record linking the context revisions, model version, question version, policy decision, approval, actual tool result, and review outcome. Redact sensitive payloads and restrict log access. For retries, distinguish re-evaluating evidence from repeating a side effect; use idempotency or reconciliation for the latter. Give operators a way to stop execution and revert to manual handling.

The result is a governed workflow with measurable responsibilities: useful evidence reaches the agent, probabilistic judgments reach ordinary code, and authorized actions reach scoped tools. Begin with one real task, define what success means, and make every completed action explainable. That is a practical foundation for agents that can become both safer and faster as the system improves.