---
title: "Developing with AI Agents: A Practical Field Guide to Codex, Claude Code, and Claude Cowork"
datePublished: 2026-06-19T09:23:46.874Z
cuid: cmqkq1rmd00010bj1b8c0541q
slug: article-2026-06-19-1822
cover: https://cdn.hashnode.com/uploads/covers/62d5556b2f40e31decd90345/95b03557-3e7b-44d2-b417-c2624e1ff2a0.jpg
tags: ai, development, codex, ai-agents, claude

---

If you've shipped anything with an AI coding agent in the last few months, you already know the feeling: you write three sentences, walk away to get coffee, and come back to a pull request with a diff, a test run, and a changelog entry. That's a different relationship to your codebase than the one most of us grew up with, and it's worth being deliberate about how we use it.

This post is a field guide — not a press release. It's about the three agent tools an individual contractor or small team is most likely to reach for right now (OpenAI's Codex, Anthropic's Claude Code, and Anthropic's Claude Cowork), how to choose between them, and — more importantly — the operating discipline that determines whether delegating to an agent makes you faster or just makes your mistakes happen faster.

A quick caveat before we start: model versions, benchmark scores, and pricing in this space change on a roughly monthly cadence. I've linked sources throughout so you can check current numbers yourself; treat anything with a specific percentage or dollar figure as a snapshot, not a permanent fact.

%[https://speakerdeck.com/x5gtrn/developing-with-ai-agents-codex-claude-code-and-cowork-practical-guide] 

## Why This Shift Happened Now

For most of the last few years, "AI coding help" meant a chat window: you pasted in a function, asked a question, got an explanation or a snippet back, and did the rest yourself by hand — copy, paste, run, debug, repeat.

What changed is that models got good enough at multi-step tool use that they could be trusted to operate *inside* a development loop rather than just narrate it: clone a repo, read the actual files, run the test suite, interpret the failures, edit code, run the tests again, and open a pull request — largely without a human re-typing every intermediate step. OpenAI describes Codex as a tool that lets engineers "offload repetitive, well-scoped tasks, like refactoring, renaming, and writing tests" in its [original launch post](https://openai.com/index/introducing-codex/), and that framing — agent as coworker for bounded units of work, not oracle that answers questions — is the right mental model for all three tools covered here.

The practical consequence for us as engineers is a role shift. The bottleneck used to be typing speed and syntax recall. Now it's increasingly **the quality of the instructions you give the agent and the rigor with which you review what comes back**. That's a less glamorous skill than "knows fifteen design patterns," but it's the one that actually predicts whether your week with an agent goes well.

## Chat Tool vs. Coding Agent: What Actually Changed

It's worth being precise about the distinction, because a lot of frustration with agent tools comes from treating them like a smarter chat window.

|  | AI Chat | AI Coding Agent |
| --- | --- | --- |
| **Scope** | Questions, explanations, snippets | Investigates a real repo, edits real files |
| **Memory** | The current conversation only | The actual codebase, working tree, git history |
| **Output** | Text and code blocks you paste in | Diffs, commits, pull requests, test results |
| **Execution** | None — you run everything yourself | Runs tests, linters, builds, git commands |
| **Your involvement** | Required at every step | Delegatable per task, with checkpoints |

The shift from "generates a snippet" to "investigates, edits, tests, and opens a PR" is the entire reason this generation of tools is worth restructuring your workflow around. It's also exactly why the failure modes are different and, frankly, scarier — an agent that's wrong about a snippet wastes your time; an agent that's wrong about a destructive shell command or a schema migration can cost you a production incident. More on that below.

## Three Tools, Three Philosophies

### OpenAI Codex — the asynchronous cloud agent

Codex (the 2025 relaunch, not the original 2021 Codex model behind early GitHub Copilot) is built around **isolated, asynchronous execution**. Each task — "add test coverage for the auth module," "upgrade this dependency," "fix the bug described in this issue" — runs in its own cloud sandbox preloaded with your repository, with its own git worktree so parallel tasks never collide. You submit a task, go do something else, and come back five to thirty minutes later to a reviewable result: command logs, test output, and a diff or PR.

Two things make Codex distinctive in practice:

*   **GitHub-native review loops.** Commenting `@codex review` on a pull request triggers an automated review pass, and issue-to-PR automation means a well-written GitHub issue can become a draft PR without anyone opening an editor.
    
*   **Genuine parallelism.** Because tasks are sandboxed and isolated, you can fire off several at once — a refactor, a test-coverage task, and a dependency bump — and they won't step on each other's working copies, as OpenAI's [Codex product page](https://openai.com/codex/) and various comparison write-ups on its [subagent architecture](https://www.morphllm.com/comparisons/codex-vs-claude-code) describe.
    

That asynchronous, batch-oriented design is also its main limitation for certain work: it's a poor fit for tasks that need tight, interactive back-and-forth, like debugging something genuinely confusing where you want to redirect the agent every thirty seconds.

### Claude Code — the interactive terminal partner

Claude Code takes the opposite default: it runs **locally**, in your terminal (with IDE and browser surfaces available too), and it asks for confirmation before consequential actions. Nothing leaves your machine unless you've configured it to.

That local, confirm-before-acting posture makes it well suited to the work Codex is weaker at: deep, interactive investigation of an unfamiliar codebase, multi-turn debugging where each step changes your understanding of the problem, and step-by-step refactors where you want to review the diff after every file rather than at the end. A `CLAUDE.md` file at the root of your project gives it durable, repo-specific context — coding conventions, prohibited actions, the exact command to run tests — that persists across the session instead of being re-explained every time, which matters a great deal once your sessions run long. (See Anthropic's [Claude Code documentation](https://code.claude.com/docs/en/whats-new) for the current feature set, which has been shipping updates at an unusually fast clip this year.)

### Claude Cowork — the same idea, for non-engineers

Cowork is Anthropic's answer to a fair question: if "an agent that investigates, acts, and reports back" is this useful for code, why should it be limited to people who write code? Launched in January 2026 and built into the Claude desktop app, Cowork applies the same agentic loop — read your files, figure out a plan, execute multi-step work, report back — to research, document creation, and business workflows, with no terminal required, as described on [Anthropic's Cowork product page](https://claude.com/product/cowork).

What makes it relevant even for engineering-heavy teams is its plugin ecosystem. Anthropic shipped [11 open-source plugins](https://www.reworked.co/collaboration-productivity/anthropic-adds-plugins-to-claude-cowork/) at launch — productivity, enterprise search, sales, and others — each bundling skills, connectors, and slash commands so Cowork shows up pre-loaded with domain context instead of starting from zero. If you're a one-person shop juggling client research, spec documents, and engineering work the way a lot of independent contractors do, Cowork is the tool for the research-and-documentation half of that workload, leaving Claude Code or Codex for the half that touches a repository. [The New Stack's coverage](https://thenewstack.io/anthropic-brings-plugins-to-cowork/) of the plugin launch is a good orientation if you want the fuller picture.

## How to Choose: A Decision Table

No single tool wins every axis. Here's the rough mapping I use day to day:

| Task | Reach for | Why |
| --- | --- | --- |
| Investigating unfamiliar code | Claude Code | Deep, interactive local investigation |
| Small-to-medium implementation fix | Claude Code *or* Codex | Interactive vs. fire-and-forget — your call |
| Bug investigation / debugging | Claude Code | Narrow down hypotheses turn by turn |
| Adding test coverage | Codex | Parallelizes well, low need for interaction |
| Refactoring | Claude Code | Review the diff step by step as it goes |
| PR review | Codex | `@codex review` automates the first pass |
| Technical research & comparison docs | Claude Cowork | Research → organize → document in one task |
| Design notes / ADRs | Claude Cowork | Strong at structured document generation |
| README / spec drafting | Claude Cowork | Reads existing files, drafts consistent docs |

A useful shorthand: **Codex for async batch work you can queue and walk away from, Claude Code for interactive work where you want to stay in the loop, Cowork for anything that ends in a document rather than a diff.**

## The Workflow That Actually Works

The single most common cause of a bad outcome with any of these tools is skipping straight to "go implement this." The workflow that consistently produces good results looks like this:

1.  **Investigate.** Ask the agent to understand the codebase and the blast radius of the change *before* touching anything.
    
2.  **Propose a plan.** Have it suggest an approach, not code.
    
3.  **Review the plan.** ← human checkpoint. You approve the approach before any file gets touched.
    
4.  **Implement in small steps.** One feature or one file at a time, not a sprawling multi-file rewrite in one shot.
    
5.  **Run tests.** Confirm existing tests still pass.
    
6.  **Review the diff.** ← human checkpoint. You read every line. Always.
    
7.  **Revise if needed.** Feed back specific issues; let it re-implement.
    
8.  **Final decision.** ← human checkpoint. Merging and releasing stays a human call.
    

Notice that three of the eight steps are explicit human checkpoints. That's not friction for its own sake — it's where the actual leverage of "agent as partner, not autopilot" lives. Skipping straight from a one-line request to step 4 is the root cause of nearly every bad agent experience I've had or read about.

## Eight Principles for Mastering AI Agents

These read like common sense in retrospect, which is exactly why they're easy to forget mid-task:

1.  **Don't jump straight to implementation.** Have it investigate and propose a change plan first.
    
2.  **One task, one purpose.** Don't bundle three unrelated changes into a single request — you lose the ability to evaluate any one of them cleanly.
    
3.  **Clarify the scope of changes explicitly.** State which files or directories are in bounds and which are not.
    
4.  **State constraints upfront.** "No API spec changes," "no new dependencies," whatever applies — say it before the agent starts, not after you're reviewing a surprising diff.
    
5.  **Always instruct test execution.** Explicitly ask for "run the tests and report the results" — don't assume it happened.
    
6.  **Humans always review diffs.** Never merge on trust alone, no matter how clean the summary looks.
    
7.  **Don't leave major design decisions to the AI.** Architecture and security calls are yours to make.
    
8.  **Explicitly state what must *not* be done.** Write prohibited actions out in plain language; don't rely on the agent inferring them.
    

## Practical Prompt Templates

Principles are easy to nod along with and easy to forget under deadline pressure, so here are three prompts I actually keep on hand, adapted to whichever tool I'm using that day.

**1\. Codebase investigation request** (always the first move on unfamiliar code):

```plaintext
Investigate this repository and explain the overall structure of
the login process. Do not edit any files yet.

Please check the following in particular:
- Entry point for authentication (endpoints)
- Related APIs and middleware
- DB tables and schema
- Error handling approach
- Presence and coverage of tests

Finally, propose a safe procedure for making modifications.
```

The two load-bearing lines here are "do not edit any files yet" and the explicit checklist. Without the first, an agent that's confident in its read of the code may start "helpfully" fixing things you didn't ask it to touch. Without the second, investigation tends to stop at the first plausible-looking answer instead of actually covering the area.

**2\. Implementation request, minimal-diff principle:**

```plaintext
Based on the investigation results above, please fix this with a
minimal diff.

Constraints:
- Do not change the existing API specification
- All existing tests must pass
- Do not add new dependency libraries
- Present the list of target files before making changes
- Show the test command to run after implementation
- Only modify files under src/auth/
```

"Minimal diff" is doing a lot of work in that first line. Left unconstrained, agents — like a lot of enthusiastic junior engineers — will sometimes "improve" adjacent code while they're in there. Pinning the file scope (`src/auth/` here) is the single highest-leverage line in this whole template.

**3\. Diff review request** (have the agent review its own work before you do):

```plaintext
Please review the current diff.

Review criteria:
- Does it satisfy the specification?
- Does it break any existing functionality?
- Are there any security issues?
- Does it contain any unnecessary changes?
- Is there insufficient test coverage?

If there are issues, classify them by severity: High / Medium / Low.
If there are High severity issues, also provide a fix proposal.
```

This doesn't replace your own review — see Principle 6 above — but a structured self-review pass catches a meaningful fraction of issues before they ever reach your eyes, and the severity classification helps you triage your own attention when you do look.

## Failure Patterns You Will Eventually Hit

A few classic mistakes show up across every team I've talked to or read about that's adopted these tools, along with the countermeasure that actually works:

| Failure pattern | Countermeasure |
| --- | --- |
| Modifying unrelated files | Explicitly specify target files/directories |
| Interpreting specs arbitrarily | State constraints and success criteria upfront |
| Treating incomplete tests as done | Mandate "run tests and report results" |
| Changing DB schema or API spec without permission | List prohibited changes at the beginning |
| Adding dependency libraries without permission | Explicitly state "no new library additions" |
| Making overly large changes at once | Split into one task, one purpose |
| Executing commands with security risks | Use confirmation mode, minimize permissions |

Every one of these traces back to the same root cause: an instruction that was vaguer than it felt at the time you wrote it. "Fix the login process" feels specific until an agent with full filesystem access starts interpreting it.

## Minimizing Security Risk: Permission Hygiene

A short, non-negotiable list:

**Never:**

*   Paste API keys, tokens, or credentials into a prompt
    
*   Grant direct permission to operate on production environments
    
*   Allow destructive commands (`rm -rf`, `DROP TABLE`, etc.) without confirmation
    

**Do:**

*   Start in suggestion/confirmation mode, not auto-execute
    
*   Default to read-only permissions and expand only as trust is earned
    
*   Allow automatic execution only in staging environments
    
*   Manage secrets via environment variables, never inline in prompts
    
*   Review execution logs as a matter of habit, not just when something looks wrong
    

If you only take one line from this section: the cost of typing `y` to confirm an action you didn't really read is identical, in the moment, to the cost of typing it after you did read it — and wildly different a week later if it was wrong.

## What Must Never Be Delegated

No matter how capable these tools get, a few categories of decision stay human, full stop:

*   **Requirements and spec decisions** — what to build is a business judgment
    
*   **Architecture decisions** — overall system design direction
    
*   **Security judgments** — final evaluation of vulnerabilities and risk tolerance
    
*   **Quality standard decisions** — what test coverage and review bar is acceptable
    
*   **Final review** — a human reads the code before it merges
    
*   **Release decisions** — final approval to ship to production
    

The reasoning is straightforward: an agent optimizes within the scope of the instructions it's given, but business context, organizational constraints, and the judgment calls that trade one risk against another are not things you can fully specify in a prompt. That's not a temporary limitation to be patched in the next model release — it's a structural reason these decisions stay with the person accountable for the outcome.

## What These Tools Are Genuinely Great At

On the other side of that line, there's a long list of work that's a very good candidate for active delegation:

*   Investigating existing code and scoping the blast radius of a change
    
*   Small bug fixes and type-error cleanup
    
*   Writing and expanding test coverage
    
*   Drafting documentation — READMEs, API specs
    
*   Producing refactoring proposals for human review
    
*   Assisting with a first-pass PR review
    
*   Eliminating duplication and routine cleanup
    
*   Mechanical conversions (adding type annotations, sync-to-async migrations, etc.)
    

This is where the time savings actually materialize, and it's not subtle. OpenAI describes Codex inside its own engineering org as the default tool for "repetitive, well-scoped tasks, like refactoring, renaming, and writing tests" that would otherwise break an engineer's focus, per the [Codex launch post](https://openai.com/index/introducing-codex/), and reporting on OpenAI's internal usage has described Codex generating the overwhelming majority of its own application code in some workflows, per [LeadDev's interviews with the OpenAI team](https://leaddev.com/ai/openai-says-there-are-easily-1000x-engineers-now). Independent benchmark trackers like [vals.ai's SWE-bench Verified leaderboard](https://www.vals.ai/benchmarks/swebench) are worth bookmarking if you want current numbers rather than a stale snapshot — both Anthropic and OpenAI have been shipping model updates roughly monthly, and the leaderboard moves with them.

The honest framing for "30% automation" type stats you'll see quoted around the industry: directionally real, organization-specific, and not something to take as a guarantee for your own codebase. Your mileage genuinely will vary by language, test coverage, and how well-scoped your tickets already are.

## Tool-Specific Tips Worth Adopting

**Codex:**

*   Maintain an `AGENTS.md` documenting project rules, prohibited actions, and conventions — it loads automatically at the start of every session.
    
*   Use `@codex review` as a standing PR check rather than an occasional manual ask.
    
*   For genuinely hard bugs, the `--attempts` option generates multiple candidate solutions in parallel so you can pick the best one instead of iterating serially.
    

**Claude Code:**

*   Keep `CLAUDE.md` current — architecture overview, prohibited actions, the exact test command. This is what keeps behavior consistent across long sessions.
    
*   Use the dedicated Explore agent for "what does this file actually do?" questions before you commit to an implementation approach.
    
*   `/compact` when context grows large in long sessions — it compresses history while preserving the working memory that matters.
    
*   Headless mode (`claude -p`) integrates into CI/CD pipelines and GitHub Actions for automated, scriptable runs.
    

**Claude Cowork:**

*   Lean on plugins for repeatable research-and-documentation workflows rather than re-explaining context every session.
    
*   Point it at local folders with existing specs so its drafts inherit your team's actual conventions instead of generic boilerplate.
    
*   Use it for the research → comparison table → ADR pipeline end to end in a single task; that handoff is where it earns its keep relative to a plain chat session.
    

## Rolling This Out Across a Team

If you're past the solo-experimentation phase:

*   **Share** `AGENTS.md` **/** `CLAUDE.md` **in the repo.** It doubles as onboarding material for new hires, human or otherwise.
    
*   **Automate PR review.** Wire `@codex review` (or the Claude Code equivalent) into CI, and standardize the review criteria as a team rather than letting everyone invent their own.
    
*   **Optimize task distribution deliberately.** Routine work — tests, type fixes, doc updates — goes to agents by default. Design and architecture discussions stay human-led.
    
*   **Hold the line on code review.** AI-generated code goes through the exact same review process as human-written code. Every team member needs the skill of critically evaluating agent output; that's not optional just because the diff came from a tool instead of a person.
    

## Starting Today

If you haven't built this into your workflow yet, a reasonable pace looks like:

**Week 1 — Try it yourself.** Run Claude Code or Codex on a personal project. Start with investigation tasks only — no implementation — to build calibration for what "a good plan" looks like before you let it touch files.

**Weeks 2–4 — Delegate in small steps.** Hand off genuinely small, bounded tasks: adding tests, fixing type errors. Build prompt templates you trust. Build the habit of actually reading every diff, not skimming it.

**Month 2+ — Roll out to the team.** Maintain and share `AGENTS.md`/`CLAUDE.md`. Set up automated PR review. Share both the wins and the near-misses with your team — the failure stories are at least as instructive as the success stories.

## The Summary, If You Read Nothing Else

AI agents aren't replacements for developers. They're development partners you can delegate discrete units of work to — research, implementation, testing, and review assistance — while you retain the judgment calls that actually carry risk.

Three things determine whether that delegation goes well:

1.  **Break work into small pieces.** Large, vague requests are the root cause of most bad outcomes.
    
2.  **Follow the full loop — investigate, plan, implement, test, review.** Don't shortcut to implementation.
    
3.  **AI does the work; humans hold the judgment and the responsibility.** Design, security, and final decisions stay yours.
    

The two skills that matter most going forward aren't really new skills at all — they're the ability to write genuinely clear instructions, and the ability to critically evaluate the output you get back. Both of those were always part of being a good engineer. The agents just made them the part that's visible.

* * *

*Further reading:* [*OpenAI Codex documentation*](https://developers.openai.com/codex)*,* [*Claude Code docs*](https://code.claude.com/docs)*,* [*Claude Cowork product page*](https://claude.com/product/cowork)*.*