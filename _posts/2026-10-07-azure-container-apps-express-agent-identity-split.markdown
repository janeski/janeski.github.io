---
title: "Split the Agent: Container Apps Express for the Thin API, Identities for Everything Else"
date: 2026-10-07 08:00:00 +0000
description: "Part 3 — why sandboxing generated code doesn't stop lateral movement, how to split an AI agent into apps with narrow identities, and where Azure Container Apps Express fits."
categories: [cloud, azure]
tags: [azure, azure-container-apps, express, managed-identity, ai, ai-agents, security, serverless, cloud-native, aspire, dotnet, scale-to-zero]
---

*Part 3 of 5 in the series **From Prototype to Trust: Hardening an AI Agent on Azure Container Apps**. Technical status checked: 22 September 2026.*

After [Part 2](/2026/azure-container-apps-sandboxes-ai-generated-code/), generated code runs in a Sandbox with no secrets and no network. A1 and A2 are blocked. It is tempting to call the agent secure at this point.

It isn't. **Isolating the code did nothing for the agent itself.** The agent API still runs as one container app with one managed identity that can reach the telemetry database, the document store, the alarm API and the model. If anything running as the agent is compromised — a vulnerable dependency, a debugging endpoint someone forgot, a future tool with a bug — it inherits all of that.

## Attack A3: lateral movement from the API

The demo simulates a compromised dependency in the agent API with a small endpoint that runs an arbitrary SQL query using the agent's identity. Against Stage 1, it reads the full alarm history of every asset in every plant.

[TODO: A3 run against stage-1-sandbox — the query and the returned row count]

The Sandbox never enters the picture. The problem is not *what code runs* but *what the process is allowed to reach*. That is infrastructure isolation and identity — two of the four terms from [Part 1](/2026/unsafe-ai-agent-azure-container-apps/).

## Stage 2: split along the trust boundaries

The agent becomes four apps:

| App | Job | Identity can reach | Ingress |
| --- | --- | --- | --- |
| `agent-api` | Authenticate the engineer, call the model, call tools, stream the answer | Model endpoint, the two tool apps, the execution environment | External |
| `alarm-tools` | Alarm history, acknowledge | Alarm tables in the telemetry DB | Internal |
| `docs-tools` | Procedures, maintenance notes, vector search | Document store and vector index | Internal |
| `indexer` | Ingest new manuals and notes | Document store (write) | None — Container Apps job |

This is an intentional design choice: **the app the engineer talks to is the one that can reach the least.** `agent-api` has no database connection string at all. It asks `alarm-tools` for alarms, and `alarm-tools` decides what to return.

In the AppHost the split is visible in the references — this is where Aspire's "architecture is the code" pays off:

```csharp
var db = builder.AddPostgres("postgres")
    .WithImage("timescale/timescaledb-ha", "pg16")
    .AddDatabase("plant");

var alarmTools = builder.AddProject<Projects.Agent_Tools_Alarms>("alarm-tools")
    .WithReference(db);

var docsTools = builder.AddProject<Projects.Agent_Tools_Docs>("docs-tools")
    .WithReference(db);

builder.AddProject<Projects.Agent_Api>("agent-api")
    .WithReference(alarmTools)
    .WithReference(docsTools)   // no reference to db
    .WithExternalHttpEndpoints();
```

Compare it with Stage 0: `agent-api` lost its `WithReference(db)`. In Azure, each app gets its own user-assigned managed identity, and only the tool apps get database roles.

[TODO: image — Aspire Dashboard resource graph of Stage 2]

Rerun A3: the compromised endpoint in `agent-api` has nothing to connect to. The database is not reachable from its identity and not referenced in its configuration.

## What is Container Apps Express?

Once the API is thin — authenticate, call a model, call two internal tools, stream — it becomes a candidate for [Azure Container Apps Express](https://learn.microsoft.com/en-us/azure/container-apps/express-overview?wt.mc_id=MVP_395548). Express is a deployment model for HTTP apps on consumption CPU: an optimized cold-start path, usage-based compute and scale to zero. The image stays a standard OCI container; nothing about Express leaks into the code.

The question for any component is simple: **does its complete set of production requirements fit inside the current preview feature set?**

As of my check, Express supports HTTP ingress, scale to zero, multiple replicas, metrics and Log Analytics, IP restrictions, **user-assigned managed identity**, HTTP/CPU/memory scaling rules, and some private-network scenarios (virtual-network egress and environment private endpoints, with constraints).

It does **not** currently support Dapr, jobs, GPUs, sidecars, built-in Easy Auth, Key Vault secret references, custom domains, internal service discovery, multiple revisions with traffic splitting, the managed OpenTelemetry agent, or exec probes. Check the [live feature matrix](https://learn.microsoft.com/en-us/azure/container-apps/express-overview?wt.mc_id=MVP_395548#supported-features) before deploying — it has already changed during the preview, and the [FAQ](https://learn.microsoft.com/en-us/azure/container-apps/express-faq?wt.mc_id=MVP_395548) still carries some older descriptions.

## Which parts of the agent fit Express?

| Component | Fits Express? | Why |
| --- | --- | --- |
| `agent-api` (Stage 2) | Yes | Thin HTTP orchestrator, user-assigned identity, own authorization code |
| `alarm-tools`, `docs-tools` | No | Internal ingress and service discovery from other apps |
| `indexer` | No | It is a job |
| `agent-api` with a policy sidecar (Part 4) | No | Express has no sidecars |
| GPU inference (Part 5) | No | Express has no GPUs |

So in Stage 2, `agent-api` moves to Express and everything else stays in a standard workload-profiles environment. The tool apps are reached over their internal endpoints from the Express app's supported private-network path — [TODO: confirm and describe the exact networking configuration used in the demo].

I want to be upfront about the consequence: in Part 4 I add a governance sidecar next to the agent, and Express does not run sidecars. The demo therefore keeps **two variants** of `agent-api` from Part 4 on — Express without the sidecar, standard with it — so the tradeoff is visible rather than hidden.

## Cold start is an end-to-end number

Scale to zero is attractive for an assistant used a few times a day. But the first request after idle has to start the app, load the agent runtime, and wait for whatever the model is doing. Express optimizes the first of those steps only.

I measured `agent-api` on Express and on a standard consumption app, after 15 minutes of idle and warm:

| Configuration | Time to first byte — cold | Total — cold | Time to first byte — warm | Total — warm |
| --- | --- | --- | --- | --- |
| Express | [TODO] | [TODO] | [TODO] | [TODO] |
| Standard consumption, min replicas 0 | [TODO] | [TODO] | [TODO] | [TODO] |
| Standard consumption, min replicas 1 | [TODO] | [TODO] | [TODO] | [TODO] |

[TODO: one or two sentences interpreting the measured numbers once the table is filled in]

When you compare cost, include the model, logging and the tool apps. "No idle compute" for the Express app is not "no cost" for the agent.

## What this layer does not do

Identity answers *who is calling*. It does not answer *what that caller may see*. `agent-api` can legitimately call `alarm-tools`, and `alarm-tools` still returns any asset it is asked about.

| Attack | Stage 0 | Stage 1 | Stage 2 |
| --- | --- | --- | --- |
| A1 Exfiltration via generated code | Succeeds | Blocked | Blocked |
| A2 Secret theft via generated code | Succeeds | Blocked | Blocked |
| A3 Lateral movement from the API | Succeeds | Succeeds | **Blocked** |
| A4 Cross-asset data access | Succeeds | Succeeds | Succeeds |
| A5 Injected privileged action | Succeeds | Succeeds | Succeeds |
| A6 Data leaving the boundary | Happens | Happens | Happens |

A4 and A5 go straight through the new boundaries, because they use the agent's legitimate path. Stopping them needs a decision at the moment of the tool call — Part 4.

## Takeaways

1. **Sandboxing the code does not isolate the agent** — lateral movement comes from what the agent's own process can reach.
2. **The component the user talks to should reach the least** — the thin API has no database access; tools own their data.
3. **Express fits the thin API, not the whole agent** — identity and scale-to-zero are in; sidecars, jobs, GPUs and internal service discovery are out.
4. **Measure cold start end to end** — the platform optimizes container startup; the runtime and the model are still yours.

This article corresponds to the `stage-2-express` tag of the demo: [TODO: link to demo repository and tag].

## Links

1. [Azure Container Apps Express overview and feature matrix](https://learn.microsoft.com/en-us/azure/container-apps/express-overview?wt.mc_id=MVP_395548)
2. [Deploy an Express app with the Azure CLI](https://learn.microsoft.com/en-us/azure/container-apps/deploy-express-cli?wt.mc_id=MVP_395548)
3. [Develop and test Express apps locally](https://learn.microsoft.com/en-us/azure/container-apps/express-local-testing?wt.mc_id=MVP_395548)
4. [Managed identities in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/managed-identity?wt.mc_id=MVP_395548)
5. [Container Apps jobs](https://learn.microsoft.com/en-us/azure/container-apps/jobs?wt.mc_id=MVP_395548)

**Series:** [Part 1 — The unsafe agent](/2026/unsafe-ai-agent-azure-container-apps/) · [Part 2 — Sandboxes](/2026/azure-container-apps-sandboxes-ai-generated-code/) · Part 3 — Express and identities · [Part 4 — MCP and governance](/2026/mcp-agent-governance-toolkit-azure-container-apps/) · [Part 5 — Private inference](/2026/private-inference-serverless-gpu-azure-container-apps/)
