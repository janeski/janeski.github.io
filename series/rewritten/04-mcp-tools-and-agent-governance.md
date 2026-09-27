---
title: "Hosting MCP Tools on Azure Container Apps — and Deciding What Those Tools May Do"
description: "MCP gives an agent tools; it does not decide who may use them — the two MCP hosting patterns on Azure Container Apps, and where the authorization decision actually has to live."
tags: [azure, azure-container-apps, mcp, ai, ai-agents, security, authorization, cloud-native, dotnet, csharp, governance]
---

**Series:** Azure Container Apps for production AI · **Part 4 of 5**
**Suggested slug:** `azure-container-apps-mcp-agent-governance`
**Technical status checked:** 22 September 2026

[image: hero photo — Photo by <name> on Pexels]

Our incident assistant answers much better questions once it has tools: `getAlarmHistory`, `findProcedure`, and perhaps `createMaintenanceDraft`. The Model Context Protocol gives clients a common way to discover and invoke those capabilities. What it does not do — by itself — is establish whether the engineer may inspect a particular asset, or whether the agent may create a work order.

**MCP solves interoperability. It does not solve authorization.** In this article, I want to show you the two ways Azure Container Apps hosts MCP, why I keep the custom tools deliberately small, and where the authorization decision has to live so it survives changes to the agent code.

## Two MCP hosting patterns, not one

Azure Container Apps supports two quite different patterns, and the [MCP hosting overview](https://learn.microsoft.com/en-us/azure/container-apps/mcp-overview?wt.mc_id=MVP_395548) describes both.

**For custom business tools** — the alarm history, the procedures, the maintenance draft — you build an MCP server with an SDK and deploy it as a normal container app. A regular container app has no special MCP runtime awareness; it hosts the HTTP server you write. That is a feature, not a limitation: you control every line of the tool.

**For sandboxed Python and shell execution**, a dynamic session pool can expose a platform-managed MCP endpoint with predefined tools such as launching an environment and running code. This is a separate preview feature, and it is not a place to host your alarm tool just because it speaks MCP.

The two patterns also authenticate differently, which I come back to below.

## Make the custom tools deliberately small

An alarm MCP server should expose `getAlarmHistory(assetId, from, to)` — not arbitrary SQL, and not a generic "query the database" tool. The narrow signature is the point. The server:

- validates the time range and the asset identifier
- enforces caller authorization
- caps the number of records
- returns only the fields the agent actually needs

A draft-creation tool records the proposed operation and routes it for review. A tool that can acknowledge alarms or control equipment needs an even tighter boundary and explicit human authorization — the same rule I set in part 1: read access to operational technology is sensitive, write access gets a separate path.

For a standalone MCP server, Container Apps gives you HTTPS ingress, autoscaling, managed identity and optional [built-in authentication](https://learn.microsoft.com/en-us/azure/container-apps/authentication?wt.mc_id=MVP_395548) backed by Microsoft Entra ID. Enabling authentication establishes *who* is calling. **The tool implementation must still enforce per-asset and per-operation authorization.** Authentication answers "who are you"; authorization answers "may you see pump 42" — and only your code knows the answer to the second question.

Two practical notes: a minimum replica of one keeps an interactive tool from paying a cold start on every question, and if clients run in a browser or VS Code, configure only the CORS origins and headers you need.

## The session-pool MCP endpoint is a different security model

The dynamic sessions MCP endpoint uses an `x-ms-apikey` credential scoped to the **entire session pool**, rather than the bearer-token approach documented for standalone MCP servers with built-in auth. The [MCP authentication](https://learn.microsoft.com/en-us/azure/container-apps/mcp-authentication?wt.mc_id=MVP_395548) page covers both.

That one key runs every environment in the pool. Keep it away from untrusted clients, rotate it, constrain network egress, and set a session lifetime. And to repeat the point: this endpoint does not become a custom alarm-tool host simply because it speaks MCP.

## Add policy at the moment of action

The model can choose a syntactically valid tool call that violates a business rule. That is not a bug in the model; it is what a model does. So the check has to happen in the invocation path, not in the prompt.

I put a policy check in the agent's tool-invocation path *and* an authorization check in the tool service. Two layers, on purpose:

1. The orchestrator proposes `getAlarmHistory(pump-42, 24h)`.
2. Policy checks the agent's allowed tools, the caller's role, and the maximum requested time range.
3. The tool verifies the caller's entitlement to pump 42 and logs the decision.
4. The result is filtered before it enters the model's context.

If the agent framework changes, the middleware changes, or someone calls the tool directly with a stolen token, step 3 still holds. That is why authorization belongs at the tool boundary and not only in the orchestrator.

Microsoft's open-source [Agent Governance Toolkit](https://github.com/microsoft/agent-governance-toolkit/blob/main/docs/deployment/azure-container-apps.md) offers policy and agent-runtime components, and its repository includes a Container Apps deployment guide using a governance sidecar. It is an integration you deploy and configure — **not an automatic, built-in Container Apps policy layer**. Its sidecar pattern currently needs standard Container Apps, because Express does not support sidecar containers (see part 2). Use a policy engine where it helps; keep authorization at the tool boundary regardless.

## Test the failure paths

The tests I would run before shipping the tools:

- Ask the agent for another customer's alarm.
- Ask for an asset outside the engineer's scope.
- Ask for an action beyond the engineer's permissions.
- Call the tool directly, bypassing the orchestrator.
- Plant a prompt injection inside a retrieved maintenance note.

Verify that the call is rejected before any data is retrieved, that the tool rejects direct access too, and that the audit trail distinguishes a policy denial from a tool error. On the last test: the note is evidence for the task. It has no authority to widen the tool's permissions, and your policy layer should make that impossible rather than unlikely.

## Takeaways

1. **Two hosting patterns, two security models** — custom tools are a normal container app with bearer-token auth; the session-pool endpoint is a preview feature with a pool-wide API key.
2. **Narrow tool signatures are the first control** — `getAlarmHistory(assetId, from, to)` is easier to authorize, cap and audit than anything generic.
3. **Authentication is the platform's; authorization is yours** — Entra ID tells you who is calling; only your tool code knows whether they may see pump 42.
4. **Policy in the invocation path, authorization at the tool boundary** — two layers, so the safety property survives changes to the agent code and direct calls to the tool.

The MCP servers and the policy layer of the incident assistant are part of the series demo: [TODO: link to series demo repository].

## Links

1. [Host MCP servers on Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/mcp-overview?wt.mc_id=MVP_395548)
2. [Secure MCP servers](https://learn.microsoft.com/en-us/azure/container-apps/mcp-authentication?wt.mc_id=MVP_395548)
3. [Azure Container Apps authentication](https://learn.microsoft.com/en-us/azure/container-apps/authentication?wt.mc_id=MVP_395548)
4. [Microsoft Agent Governance Toolkit — deployment on Container Apps](https://github.com/microsoft/agent-governance-toolkit/blob/main/docs/deployment/azure-container-apps.md) — GitHub
