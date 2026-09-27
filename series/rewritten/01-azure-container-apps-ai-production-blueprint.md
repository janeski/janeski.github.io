---
title: "What Makes Azure Container Apps Ready for AI Agents in Production — and What Is Still Your Job"
description: "Where an AI agent's API, tools, generated code, background work and model inference should run on Azure Container Apps — and which boundaries the platform leaves to you."
tags: [azure, azure-container-apps, ai, ai-agents, cloud-native, dotnet, microservices, serverless, devops, security, iot, observability]
---

**Series:** Azure Container Apps for production AI · **Part 1 of 5**
**Suggested slug:** `azure-container-apps-ai-production-architecture`
**Technical status checked:** 22 September 2026

[image: hero photo — Photo by <name> on Pexels]

## TL;DR

An AI agent demo fits in one process. A production agent does not — its API, its tools, the code it generates, its background work and its model inference all have different scaling patterns and, more importantly, different levels of trust. Azure Container Apps is interesting for this problem because **apps, jobs, dynamic sessions, Sandboxes and serverless GPUs give you distinct execution boundaries on one platform**. What the platform does *not* give you is the agent's authorization and decision policy. That part is still your job, and it is the part that decides whether the agent is production-ready.

## 1. Introduction

Every agent demo I have seen — including my own — has the same shape. A request comes in, the model is called, a tool is invoked, an answer goes back. One process, one identity, one deployment. It works, and it is genuinely useful for learning.

The simplicity disappears the moment a tool can read operational data, generate code, or change a business system. In this article, I want to walk through where each piece of an agent should run on Azure Container Apps and why. The next four articles in the series make each choice concrete: Express for the HTTP surface, Sandboxes for untrusted code, MCP and policy for tools, and serverless GPUs for inference.

Before I dive into the architecture, I want to set the scenario I will use across the whole series.

## 2. The scenario: an industrial incident assistant

An engineer asks: *"Why does the pump alarm keep recurring?"* The assistant reads the recent alarms, retrieves the maintenance procedure, runs a short diagnostic over exported telemetry, and proposes the next action.

Four steps, four different trust levels. Reading an alarm API is sensitive. Running a generated script is untrusted by definition. Proposing an action that touches equipment needs a human in the loop. **The assistant must never inherit broad production credentials merely because one tool needs to read an alarm API.** If you take one sentence from this article, take that one.

## 3. Being production-ready in 2026 means splitting at the trust boundaries

Cloud-native for me has always been three things: technology, architecture and organization. For AI agents I would add a fourth aspect — **trust boundaries as an architectural primitive**. Where code runs is a security decision, not a hosting detail.

Here is how I split the incident assistant:

| Component | Job in this example | Suitable hosting choice |
| --- | --- | --- |
| Agent API | Authenticate the engineer, coordinate model calls, stream an answer | Container app; Express only if its current feature set fits |
| Alarm and documentation tools | Return narrow, authorized data | Separate container app or MCP server |
| Generated diagnostic code | Process a bounded telemetry export | Dynamic session for disposable work; Sandbox for a workspace that must resume |
| Indexing and enrichment | Consume new manuals and alarm events | Event-driven Container Apps job or continuously running worker |
| Private model endpoint | Run a model whose weights and serving stack we control | Consumption GPU workload profile for variable demand; dedicated GPU for steadier load |

This division is an intentional architecture choice, not a product requirement. Nothing in Container Apps forces you to do it. Its value is simple: a compromised tool or a bad generated script does not automatically receive the agent API's identity, network reachability, or data.

[image: architecture diagram — the five components and their boundaries on one Container Apps environment]

## 4. The four decisions

### 4.1. Put the agent behind an ordinary production API

**Why it matters:** The orchestrator is the only component the engineer talks to, and it is the one that would be most damaging if it carried every permission.

Start with a standard container app. Give it a managed identity with only the permissions it needs to call its dependencies. Keep the alarm tool in a separate app with a different identity and, where appropriate, internal ingress. Set a minimum replica count if interactive latency matters — scale to zero trades idle cost for cold starts, and for an incident path that trade is not always worth it.

→ The API becomes a thin, replaceable front — not the place where all the power lives.

### 4.2. Scale user traffic and background work separately

**Why it matters:** A burst of engineer questions and a backlog of document-indexing messages have nothing in common, and one scaling policy cannot serve both.

Container Apps scales on HTTP traffic and on event sources through KEDA. A [job](https://learn.microsoft.com/en-us/azure/container-apps/jobs?wt.mc_id=MVP_395548) fits when one indexing execution has a clear end; a worker app fits a continuous consumer. The queue between them is also where you absorb retries and cap the rate at which expensive model calls or downstream APIs are made. The [scaling documentation](https://learn.microsoft.com/en-us/azure/container-apps/scale-app?wt.mc_id=MVP_395548) covers both models.

→ Interactive latency and batch throughput get their own budgets.

### 4.3. Treat every tool call as an authorization decision

**Why it matters:** The model can propose `getAlarmHistory(assetId)`. It cannot decide which assets the caller is allowed to see.

The tool has to check the user's authorization, validate the asset identifier, limit the result set, and record the call. A proposed `acknowledgeAlarm` is a different privilege altogether and may require a human approval step. Read access to operational technology is already sensitive; write access deserves a separate path.

For generated code, pass an export containing only the fields the analysis needs into a session or Sandbox. Do not mount production secrets into its filesystem. Apply a deny-first egress policy when the code could make outbound requests. That boundary reduces the damage from an unexpected script — it does not establish that the script's conclusion is correct. Isolation and correctness are two different problems.

→ Authorization lives at the tool boundary, where it survives changes to the agent code.

### 4.4. Operate the agent as a distributed system

**Why it matters:** An agent request crosses the API, retrieval, the model, a tool and possibly a background job within seconds. If you cannot trace that path, you cannot explain a wrong answer.

Record the selected tool, the policy decision, the model version, token usage and the outcome — and exclude credentials and sensitive prompt contents. A standard Container Apps environment can use its [managed OpenTelemetry agent](https://learn.microsoft.com/en-us/azure/container-apps/opentelemetry-agents?wt.mc_id=MVP_395548) to route telemetry to Application Insights or an OTLP destination. Note that the agent does not instrument your code for you; your application must emit the signals.

Add timeouts, idempotency keys and a budget per request. A retry loop around an expensive tool is both an outage and a bill.

Use [revisions](https://learn.microsoft.com/en-us/azure/container-apps/revisions?wt.mc_id=MVP_395548) to release new agent code progressively and shift traffic when the result is healthy. Test a candidate against realistic tool errors, slow models and denied actions. Keep in mind that Express currently runs a single active revision without traffic splitting — I come back to that in part 2.

→ The agent gets the same operational discipline as any other distributed system, plus model-specific signals.

## 5. Limitations — what the platform does not decide for you

### 5.1. No built-in agent policy layer

Container Apps gives you identities, ingress control and isolation primitives. It does not know what a "tool" is.

- Which tools an agent may call is your policy
- Which assets a caller may see is your authorization
- Human approval for write actions is your workflow

**Impact:** The security property of the agent comes from your code and configuration, not from the hosting choice.

### 5.2. Isolation is not validation

A Sandbox stops a script from reaching your network. It does not check that the script computed the right correlation.

- Validate output schemas and ranges
- Keep the source dataset traceable
- Show the engineer which window and calculation produced the conclusion

**Impact:** You still need a validated result channel between the isolated environment and the orchestrator.

## 6. The production test

Before I would call this agent production-ready, I want answers to five questions:

1. Who can invoke each tool?
2. Where does generated code execute?
3. Which destinations can it reach?
4. How do we stop a runaway workflow?
5. What would we see in the traces if it took a wrong turn?

If any of those answers is "the agent decides", the architecture is not finished.

## 7. What's next in the series

- **Part 2** — Azure Container Apps Express as the home for the agent's HTTP surface
- **Part 3** — Dynamic sessions and Sandboxes for AI-generated code
- **Part 4** — MCP tools and where the authorization decision actually lives
- **Part 5** — Serverless GPUs when an external model API is not the right fit

The complete incident assistant — all five parts wired together in one Aspire solution — is published as a single demo for the series: [TODO: link to series demo repository].

## 8. Useful Links

1. [Azure Container Apps overview](https://learn.microsoft.com/en-us/azure/container-apps/overview?wt.mc_id=MVP_395548) — Microsoft Docs
2. [Scaling rules](https://learn.microsoft.com/en-us/azure/container-apps/scale-app?wt.mc_id=MVP_395548)
3. [Container Apps jobs](https://learn.microsoft.com/en-us/azure/container-apps/jobs?wt.mc_id=MVP_395548)
4. [Managed OpenTelemetry agent](https://learn.microsoft.com/en-us/azure/container-apps/opentelemetry-agents?wt.mc_id=MVP_395548)
5. [Azure Well-Architected guidance for Container Apps](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-container-apps?wt.mc_id=MVP_395548) — Well-Architected Framework
