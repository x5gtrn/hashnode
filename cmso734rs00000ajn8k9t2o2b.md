---
title: "Factory AI in 2026: From Specialized Droids to an Agent-Native Engineering System"
datePublished: 2026-08-11T05:03:27.237Z
cuid: cmso734rs00000ajn8k9t2o2b
slug: article-2026-08-11-1401
cover: https://cdn.hashnode.com/uploads/covers/62d5556b2f40e31decd90345/37f1a721-8731-4aab-810b-b672813a57b0.jpg
tags: ai, software-development, automation, devops, developer-tools

---

In November 2025, I published [“Factory AI: A Comprehensive Look at Agent-Native Software Development”](https://daisuke.masuda.tokyo/article-2025-11-03-0139). That article described Factory through four specialized agents—Code, Reliability, Knowledge, and Tutorial Droids—and concepts such as HyperCode and ByteRank. It captured an interesting moment, but the product has moved quickly.

This is the 2026 update.

%[https://speakerdeck.com/x5gtrn/factory-ai-the-complete-guide-2026] 

The most important change is not a longer feature list. Factory now makes more sense as an **engineering execution system**: one Droid runtime that can work interactively, launch parallel Missions, delegate to custom subagents, reuse team procedures as Skills, and run headlessly in CI. Around that runtime sits an increasingly serious control plane for permissions, sandboxing, governance, observability, and measuring whether a repository is actually ready for autonomous work.

That shift also changes the right question. Instead of asking, “How good is Factory at generating code?”, ask:

> Can my repository turn an explicit engineering intent into a small, reviewable, verified change—with enough evidence that a human can safely approve it?

This guide explains the current architecture, shows concrete workflows, and separates useful engineering practice from agent hype.

## The 2025 mental model vs. the 2026 product

The earlier article organized Factory around four named specialists. That was easy to understand, but the current documentation emphasizes a more composable system.

| 2025 framing | 2026 framing | Why it matters |
| --- | --- | --- |
| Four fixed specialist Droids | A coordinator, five built-in roles, plus Custom Droids | Factory adds Review and separates orchestration from execution; teams can still define their own specialists |
| Interactive assistance | Interactive CLI, app, IDE, and headless `droid exec` | The same runtime can move from pairing to automation |
| Agent capability as the headline | Repository readiness and verifiable workflows | Reliability depends as much on the environment as on the model |
| One agent completing a task | Missions coordinating parallel agents | Larger objectives can be decomposed without sharing one mutable workspace blindly |
| Prompting as configuration | `AGENTS.md`, Rules, Memories, Skills, hooks, and MCP | Operating knowledge becomes versionable infrastructure |
| Broad security claims | Concrete command lists, sandboxing, hooks, and Droid Shield | Teams can enforce boundaries independently of model behavior |

The 2026 deck depicts a coordinator that decomposes work and delegates to five roles: Code, Review, Test, Docs, and Knowledge Droids. The old categories have not become useless; they have become part of a broader orchestration model. In the product, teams can also encode their own specialists as Custom Droids and their repeatable procedures as Skills.

## What “agent-native” should mean to an engineer

Autocomplete predicts the next tokens. Chat answers questions. An engineering agent participates in a feedback loop:

1.  inspect the repository and its constraints;
    
2.  propose a plan;
    
3.  use tools to change the environment;
    
4.  run tests, linters, and other checks;
    
5.  interpret failures and revise;
    
6.  produce a diff and evidence for review.
    

Factory's current [Droid CLI overview](https://docs.factory.ai/droid-cli/overview) describes a runtime with project context, approvals, MCP tools, Missions, custom Droids, Skills, and headless execution. The “agent-native” part is therefore not a special model. It is the harness surrounding whichever model you select: context acquisition, tool execution, state, policy, and verification.

This distinction matters. A stronger model can improve planning or code generation, but it cannot compensate for a repository with no deterministic build, slow tests, undocumented conventions, or unrestricted credentials. Agent performance is a systems property.

## The current Factory stack

### 1\. Droid CLI: the interactive control surface

Installation remains intentionally simple:

```bash
curl -fsSL https://app.factory.ai/cli | sh
cd /path/to/repository
droid
```

From there, Droid works next to Git, tests, and the editor. Factory's [CLI reference](https://docs.factory.ai/reference/cli-reference) exposes interactive operations for model selection, review, sessions, MCP servers, hooks, Skills, Custom Droids, Missions, and readiness reports. Git worktree support is particularly useful: separate tasks can operate on separate branches and directories instead of racing over the same files.

For non-trivial changes, begin in Spec Mode. Ask the agent to explore before editing, identify uncertainty, and write acceptance criteria. A good implementation request looks like an issue a senior engineer would be comfortable handing to a new teammate:

```text
Add idempotency to POST /payments.

Scope:
- API and persistence layers only; do not change the public response schema.
- Use the Idempotency-Key header and retain results for 24 hours.
- Concurrent requests with the same key must create at most one charge.

Acceptance criteria:
- Unit tests cover replay, conflicting payloads, expiry, and concurrency.
- Existing integration tests pass.
- Add a short ADR explaining the storage and locking decision.

Before editing, inspect the payment flow and propose a plan with risks.
```

Notice what is absent: “use clean code,” “be production-ready,” or “do it perfectly.” Those phrases do not give the agent a testable target.

### 2\. `AGENTS.md`, Rules, and Memories: context as infrastructure

Repeatedly explaining the same build commands is both expensive and unreliable. Factory supports repository instructions in `AGENTS.md`, persistent Rules, and Memories. The [power-user setup checklist](https://docs.factory.ai/guides/power-user/setup-checklist) documents project and personal locations for these resources.

A minimal `AGENTS.md` might be:

```markdown
# Repository guide

## Architecture
- `apps/api`: Fastify HTTP API
- `packages/domain`: framework-free business logic
- `packages/db`: migrations and query layer

## Verification
- Format: `pnpm format:check`
- Types: `pnpm typecheck`
- Unit tests: `pnpm test`
- API integration tests: `pnpm test:integration`

## Constraints
- Never edit generated clients under `packages/sdk/generated`.
- Database changes require a backward-compatible migration.
- Do not merge or push; leave a reviewed local commit.
```

Treat this file like code. Keep it short, review changes, and prefer executable commands over prose. A stale instruction file is worse than no instruction file because it produces confident failures.

### 3\. Skills: versioned engineering procedures

A Skill turns a repeatable procedure into a reusable capability. Factory's [Skills guide](https://docs.factory.ai/cli/configuration/skills) recommends explicit scope, verification, proof artifacts, safe failure behavior, and composability.

Good Skill candidates include:

*   adding an API endpoint using your service template;
    
*   upgrading a dependency across a monorepo;
    
*   producing an incident evidence bundle;
    
*   validating a database migration;
    
*   generating a release note from a semantic diff.
    

The best Skills are not elaborate prompts. They are compact runbooks with preconditions, boundaries, commands, expected artifacts, and stop conditions. For production-adjacent work, require a pull request and forbid autonomous merging.

### 4\. Custom Droids: organization-specific specialists

Custom Droids let you encode roles that match the real ownership map of your system: `payments-reviewer`, `terraform-auditor`, `mobile-release-engineer`, or `postgres-migration-reviewer`. This is more useful than a universal “testing agent” because a specialist can carry the relevant tools, instructions, and domain constraints.

A useful rule is: create a Custom Droid when a task needs a distinct role or tool boundary; create a Skill when it needs a repeatable procedure. The two compose naturally—a release Droid can invoke release-validation Skills.

### 5\. Missions: parallel work with explicit decomposition

Missions coordinate multiple agents for objectives that can be divided into independent workstreams. In practice, parallelism is valuable only when ownership is clear.

For a framework migration, reasonable branches might be:

*   inventory deprecated APIs;
    
*   migrate the core library;
    
*   update application integrations;
    
*   expand regression tests;
    
*   update developer documentation.
    

Do not create five agents to edit the same central module. The coordination cost and merge risk will dominate. Prefer bounded tasks, separate worktrees, declared dependencies, and a final integration phase. Parallel agents amplify both good decomposition and bad decomposition.

### 6\. Droid Computers: persistent execution environments

The slide deck gives Droid Computers unusual prominence, and rightly so. Autonomous work lasting hours or days does not fit an ephemeral chat sandbox that must reinstall dependencies and reconstruct state on every run.

[Droid Computers](https://docs.factory.ai/cli/features/droid-computers) are long-lived environments that retain packages, files, services, credentials, and configuration across sessions. Factory offers both managed computers and Bring Your Own Machine (BYOM). Factory provisions a managed instance; BYOM registers infrastructure you already control, such as a workstation, VPS, or on-premises server.

Persistence removes setup cost, but it also changes the threat model. State can drift, credentials can outlive a task, and a compromised dependency can persist. Treat a Droid Computer like a developer workstation or CI runner: patch it, isolate it, monitor it, rotate credentials, constrain network access, and periodically rebuild it from a known configuration. “Persistent” should not mean “pet server nobody can reproduce.”

### 7\. Factory Router: model selection becomes infrastructure

The 2026 deck introduces Factory Router as a cost-optimization layer. Instead of asking every engineer to choose among fast, affordable, and frontier models, the router classifies a session and selects an appropriate model, with escalation or provider failover when needed.

Factory's [Router product page](https://factory.ai/product/router) reports aggregate production savings and says the router is available in CLI and Desktop, including Mission workers. Its [June 2026 announcement](https://factory.ai/news/factory-router) also publishes comparisons against an Opus baseline. Those numbers are vendor-reported and workload-dependent, so they should be a hypothesis for your own evaluation—not a procurement guarantee.

The architectural idea is sound: documentation edits, repository search, and mechanical refactors should not automatically consume the same inference budget as an ambiguous cross-service design. But routing needs policy. Authentication and payments code may deserve a frontier model even for a small diff; sensitive repositories may restrict eligible providers; and fallback behavior must preserve data-residency requirements.

### 8\. Factory Analytics: measure outcomes, not activity

Factory Analytics closes the economic loop. The official [Analytics API](https://docs.factory.ai/reference/analytics-api) exposes token consumption, tool use, user activity, productivity signals, and per-user metrics; enterprise deployments can also use [OpenTelemetry-native usage and cost telemetry](https://docs.factory.ai/enterprise/usage-cost-and-analytics).

Token savings are useful operational data, but “more tool calls” and “more generated files” are not productivity. Join Factory telemetry with delivery metrics such as accepted PR lead time, change failure rate, review time, escaped defects, and rollback rate. The unit of value is a verified change that the team wanted—not an active session.

### 9\. Droid Exec: from conversation to automation

[`droid exec`](https://docs.factory.ai/cli/droid-exec/overview) is the bridge from interactive work to CI/CD. It runs a one-shot task, exits with a success or failure status, and supports human-readable or structured output. Importantly, it is read-only by default; mutations require an explicit autonomy level.

For read-only PR analysis:

```bash
droid exec \
  --output-format json \
  --cwd "$CHECKOUT" \
  "Review the diff against origin/main. Return JSON with severity, file, line, evidence, and suggested test. Do not edit files."
```

For a controlled documentation workflow:

```bash
droid exec \
  --auto low \
  --cwd "$CHECKOUT" \
  "Update only docs/ for the API changes in the current diff. Run the docs link checker and write a summary to artifacts/docs-update.md."
```

The Factory documentation explicitly warns that `--skip-permissions-unsafe` bypasses all checks and belongs only in disposable, isolated environments. In normal CI, prefer the lowest autonomy that works, constrain the working directory, restrict tools, and ask for artifacts your pipeline can validate.

For deeper integrations, Droid Exec also provides streaming JSON-RPC and official TypeScript and Python SDK paths. That makes the runtime usable behind an internal portal or policy layer without scraping terminal text.

## A practical end-to-end workflow

Suppose you need to upgrade an authentication library with a breaking API change.

### Phase 1: establish a baseline

Before delegating anything, make the repository reproducible:

```bash
pnpm install --frozen-lockfile
pnpm typecheck
pnpm test
```

Record failures that already exist. Otherwise the agent may “fix” unrelated problems or claim responsibility for a pre-existing failure.

### Phase 2: explore in Spec Mode

Ask Droid to locate imports, configuration, wrappers, tests, and security-sensitive paths. Require a migration plan that distinguishes mechanical edits from semantic changes. Make it identify what cannot be verified locally.

### Phase 3: implement a narrow slice

Start with one package or service. Do not begin with the entire monorepo. Require the smallest diff that proves the migration pattern, including tests. Review the pattern before scaling it.

### Phase 4: scale safely

Once the pattern is accepted, encode it as a Skill or delegate independent packages through a Mission. Use worktrees so each branch has an isolated filesystem. Keep a deterministic integration order.

### Phase 5: demand evidence

The final response should not merely say “all tests pass.” Require:

*   exact commands and exit status;
    
*   changed-file summary;
    
*   test additions and what behavior they cover;
    
*   unresolved risks or skipped checks;
    
*   a diff or pull request for human review.
    

The agent's prose is not evidence. Logs, tests, generated reports, and reviewable diffs are evidence.

## Agent Readiness: the underappreciated feature

Factory's [Agent Readiness Model](https://docs.factory.ai/web/agent-readiness/overview) evaluates repositories across nine technical pillars and gates progression: a repository must pass 80% of one level's criteria to unlock the next. It can be invoked with `/readiness-report`, viewed in a dashboard, or accessed programmatically.

Even if you never accept the scoring model as objective truth, the framing is correct. Agent throughput is constrained by machine-readable feedback:

*   Can a fresh checkout build deterministically?
    
*   Are formatting, types, tests, and security checks fast enough to run repeatedly?
    
*   Are module boundaries and ownership visible?
    
*   Are secrets and production systems isolated?
    
*   Can success be evaluated without subjective interpretation?
    

This leads to a useful inversion: improving your repository for agents often improves it for humans. Faster tests, clearer ownership, stable commands, and documented architecture reduce onboarding and review costs regardless of who writes the patch.

## Security: treat the model as untrusted

Factory's current [Agent Safety & Controls documentation](https://docs.factory.ai/enterprise/llm-safety-and-agent-controls) says this directly: build on deterministic controls rather than trusting model behavior.

The available layers include:

*   command allow, deny, and blocklists;
    
*   programmable hooks around lifecycle events;
    
*   filesystem and network sandboxing;
    
*   network egress restrictions;
    
*   Droid Shield scanning around Git commit and push;
    
*   managed organization and project policies.
    

The distinction between a denylist and a blocklist is important. A denied command can still run after explicit approval. A blocked command has no approval path, including under high autonomy. Use a blocklist for actions the organization has decided must never occur.

A sensible production policy is:

1.  no long-lived production credentials in the agent environment;
    
2.  no direct merge or deployment authority;
    
3.  write access limited to the task's repository or worktree;
    
4.  network egress limited to required package registries and APIs;
    
5.  every change goes through normal CI and code review;
    
6.  tool calls and policy decisions are logged;
    
7.  prompts and retrieved content are treated as potentially hostile input.
    

MCP expands capability and attacks surface at the same time. An MCP server should be reviewed like any dependency with access to internal systems: authenticate it, minimize its scopes, pin and audit it where possible, and decide whether it belongs at user, project, or organization level.

### Security Review is a workflow, not a magic shield

The deck highlights automated security review on every pull request. Factory's current [Security Review documentation](https://docs.factory.ai/enterprise/security-review) describes a two-pass process: generate candidate issues, then validate reachability, exploitability, and existing controls before reporting. Its methodology combines STRIDE, OWASP Top 10, the OWASP Top 10 for LLM applications, supply-chain checks, and an optional repository threat model in `.factory/threat-model.md`.

You can invoke `/security-review` locally or enable a dedicated review in Droid Action. A full-repository scan writes a report on a separate branch and opens a PR. That is a useful review layer, especially for tracing changed data flows, but it is not a substitute for SAST, dependency scanning, secret detection, fuzzing, penetration testing, or a security engineer. Treat agent findings as hypotheses with evidence and track false positives and missed vulnerabilities over time.

## Model choice and benchmarks

The 2025 article cited SWE-bench and TerminalBench numbers as if they described Factory as a stable unit. That interpretation is now less useful. Factory is a multi-model harness, and results depend on model, reasoning budget, tools, repository context, and the evaluation scaffold.

Factory now publishes a [code review benchmark](https://docs.factory.ai/benchmarks/review-benchmark) based on 50 pull requests from five large open-source projects and a manually curated set of validated bugs. That is useful evidence for the narrower question “which model is cost-effective for bug-finding in this harness?” It is not proof that the same model will excel at your migrations, UI work, or incident response.

Build a small internal evaluation instead:

| Metric | Example definition |
| --- | --- |
| Task success | Acceptance tests pass without relaxing requirements |
| Review burden | Human minutes from first diff to approval |
| Rework | Number of agent revisions after review |
| Escaped defects | Regressions attributable to accepted agent changes |
| Cost | Model plus infrastructure cost per accepted task |
| Lead time | Time from assigned tasks to reviewable evidence |

Use representative tasks, keep a fixed baseline, and record failures—not just wins. Optimize for accepted engineering outcomes per dollar, not tokens generated or lines changed.

## Pricing and rollout

The old article described a free BYOK tier plus Pro and Max. Current [Factory pricing](https://docs.factory.ai/pricing) lists individual Pro, Plus, and Max plans, along with Teams and Enterprise offerings, rolling rate limits, optional Extra Usage, and a Droid Core pool. BYOK is described as an allowance within plans rather than an unlimited free tier. Pricing changes quickly, so check the official page before budgeting.

A realistic rollout is deliberately boring:

1.  **Week 1: read-only pilot.** Repository Q&A, change-impact analysis, and PR review.
    
2.  **Weeks 2–3: low-risk edits.** Tests, docs, and mechanical refactors on isolated branches.
    
3.  **Weeks 4–6: codify success.** Add `AGENTS.md`, Skills, hooks, metrics, and a small internal evaluation set.
    
4.  **After evidence: automate.** Move stable workflows to Droid Exec; use Missions only where parallel decomposition is natural.
    

Choose one team and two or three repeatable tasks. Compare against a baseline. If review time rises or defects increase, stop and improve the environment before increasing autonomy.

## Where Factory fits—and where it does not

Factory is compelling when you want the same agent runtime across terminal, IDE, scripts, CI, and team workflows; when you value model choice; and when you are willing to encode engineering knowledge and controls around it.

It may be excessive for a small repository that only needs autocomplete and occasional chat. It is also a poor fit for teams that cannot make builds reproducible, expose verification commands, or keep humans accountable for production changes. More autonomy does not repair weak engineering systems; it makes their weaknesses operate faster.

The defensible advantage is not “Droid writes more code.” It is the ability to turn team knowledge into versioned, executable operating procedures and run them through a governed agent harness.

## Factory does not have to replace your other coding agents

Two slides show Factory working alongside Claude Code, Cursor, and Sentry. That is a more realistic architecture than declaring one universal winner.

*   Use an editor-native tool for rapid, synchronous exploration and small local edits.
    
*   Hand a well-specified, long-running migration or multi-repository task to a Mission.
    
*   Use Factory's review and security workflows as an independent background check.
    
*   Connect operational systems through narrowly scoped MCP servers—for example, allowing an incident workflow to read a Sentry issue and repository context, then prepare a fix for review.
    

The handoff contract matters more than the brand combination. Pass the ticket, acceptance criteria, relevant paths, current diff, and verification status. Avoid letting two agents concurrently mutate the same branch. For incident automation, keep the progression explicit: alert → evidence collection → candidate root cause → patch and tests → human approval → deployment through the normal pipeline.

This is where the “software factory” metaphor earns its keep. A factory is not one machine. It is a controlled production system with specialized stations, quality gates, feedback, maintenance, and accountable operators.

## Final take

Factory AI in 2026 is materially different from the platform I described in November 2025. The earlier story was about four specialized Droids and impressive autonomous capabilities. The current story is more mature: a general Droid runtime, composable specialists and Skills, parallel Missions, headless execution, measurable repository readiness, and deterministic safety controls.

That evolution makes Factory more interesting—but also removes excuses for careless adoption.

Start with a well-scoped task. Give the agent executable context. Keep mutations narrow. Make verification automatic. Treat the model as untrusted. Require evidence. Measure accepted outcomes. Only then increase autonomy.

Agent-native engineering is not the removal of engineers from the loop. It is the redesign of the loop so human intent, machine execution, deterministic checks, and accountable review fit together.

## Further reading

*   [Factory Droid CLI overview](https://docs.factory.ai/droid-cli/overview)
    
*   [Droid Exec: headless automation](https://docs.factory.ai/cli/droid-exec/overview)
    
*   [Factory Agent Readiness Model](https://docs.factory.ai/web/agent-readiness/overview)
    
*   [Droid Computers](https://docs.factory.ai/cli/features/droid-computers)
    
*   [Factory Router](https://factory.ai/product/router)
    
*   [Factory Analytics API](https://docs.factory.ai/reference/analytics-api)
    
*   [Factory Agent Safety & Controls](https://docs.factory.ai/enterprise/llm-safety-and-agent-controls)
    
*   [Factory Security Review](https://docs.factory.ai/enterprise/security-review)
    
*   [Factory Skills guide](https://docs.factory.ai/cli/configuration/skills)
    
*   [Factory plans and pricing](https://docs.factory.ai/pricing)
    
*   [The November 2025 version of this article](https://daisuke.masuda.tokyo/article-2025-11-03-0139)
    
*   [The original Factory AI slide deck on Speaker Deck](https://speakerdeck.com/x5gtrn/factory-ai-the-complete-guide-to-agent-native-software-development)