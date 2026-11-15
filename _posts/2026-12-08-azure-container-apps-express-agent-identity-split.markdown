---
title: "Split the Agent: Container Apps Express for the Thin API, Identities for Everything Else"
date: 2026-12-08 08:00:00 +0000
description: "Part 3 — why sandboxing generated code doesn't stop lateral movement, how to split an AI agent into apps with narrow identities, and where Azure Container Apps Express fits."
categories: [cloud, azure]
post_class: post--vivid
image: /images/posts/aca-agent-hardening-part3-cover.png
tags: [azure, azure-container-apps, express, managed-identity, ai, ai-agents, security, serverless, cloud-native, aspire, dotnet, scale-to-zero]
---

*Part 3 of 5 in the series **From Prototype to Trust: Hardening an AI Agent on Azure Container Apps**. Technical status checked: 28 September 2026.*

After [Part 2](/2026/azure-container-apps-sandboxes-ai-generated-code/), generated code runs in a dynamic session or a Sandbox, with no secrets and no network. A1 and A2 are blocked. It is tempting to call the agent secure at this point.

It isn't. The agent API still runs as one container app that can reach the plant database, the maintenance notes, the alarm tools and the model. If anything running as the agent is compromised — a vulnerable dependency, a debugging endpoint someone forgot, a future tool with a bug — it inherits all of that.

**Isolating the code did nothing for the agent itself.**
{: .callout}

In this article, I want to show you how to split the agent along its trust boundaries, give every part its own narrow identity, and where Azure Container Apps Express fits into that picture — and where it doesn't.

## 1. Attack A3: lateral movement from the API

The demo simulates a compromised dependency in the agent API with a small endpoint that runs an arbitrary SQL query using the agent's own database access. Against Stage 1, it reads the full alarm history of every asset on every site.

[TODO: A3 run against stage-1-sandbox — the query and the returned row count]

The Sandbox never enters the picture. The problem is not *what code runs* but *what the process is allowed to reach*. That is infrastructure isolation and identity — two of the four terms from [Part 1](/2026/unsafe-ai-agent-azure-container-apps/).

## 2. Stage 2: split along the trust boundaries

The agent becomes four apps:

| App | Job | Database role | Ingress |
| --- | --- | --- | --- |
| `agent-api` | Authenticate the engineer, call the model, call tools, stream the answer | None | External |
| `alarm-tools` | Alarm history, acknowledge | `alarm_tools` — alarm tables | Internal |
| `docs-tools` | Procedures, maintenance notes, vector search | `docs_tools` — documents, read | Internal |
| `indexer` | Embed new manuals and notes | `indexer` — documents, write | None — Container Apps job |

Every app also gets its own user-assigned managed identity, so "who is calling" is a platform fact for every call between them.

**The app the engineer talks to is the one that can reach the least.** `agent-api` has no database role and no connection string. It asks `alarm-tools` for alarms, and `alarm-tools` decides what to return.
{: .callout}

This is an intentional design choice. The orchestrator is the component that reads the model's output — the part most exposed to manipulation. It should hold the fewest privileges, not the most.

### The AppHost

In the AppHost, the split is visible in the references — this is where Aspire's "architecture is the code" pays off:

```csharp
var alarmTools = builder.AddProject<Projects.Agent_Tools_Alarms>("alarm-tools")
    .WithEnvironment("ConnectionStrings__plant", PlantConnection("alarm_tools"));

var docsTools = builder.AddProject<Projects.Agent_Tools_Docs>("docs-tools")
    .WithEnvironment("ConnectionStrings__plant", PlantConnection("docs_tools"))
    .WithReference(embed);

builder.AddProject<Projects.Agent_Api>("agent-api")
    .WithReference(alarmTools)
    .WithReference(docsTools)   // no database at all
    .WithExternalHttpEndpoints();
```

Compare it with Stage 0: `agent-api` lost its `WithReference(db)`.

You may notice the tool apps don't use `WithReference(db)` either. That one line hands an app the **admin** connection string — exactly what this stage is trying to remove. So each app gets a connection string for its own Postgres role instead, and the role only has grants on the tables that app needs.

**Why it matters:** The demo's Postgres is a container, which has no Entra integration — so the roles use per-app passwords, stored as Container Apps secrets.<br>
→ With Azure Database for PostgreSQL, the same roles map to the apps' managed identities and the passwords disappear.
{: .why}

[TODO: image — Aspire Dashboard resource graph of Stage 2]

### Two environments, one network

In Azure, the split is also a network boundary. Stage 2 uses two Container Apps environments in one virtual network:

![Stage 2 topology: a virtual network with a public front environment running agent-api with no database role, an internal plant environment running alarm-tools, docs-tools, the indexer job, the simulator with NATS and Postgres with one role per app, and an Express environment whose agent-api variant egresses into the same network to reach the tools](/images/posts/aca-agent-hardening-stage2.svg)

- **plant** — an *internal* environment, published only on an internal load balancer and resolved through a private DNS zone. It holds the data and the tools: Postgres, NATS, the simulator, `alarm-tools`, `docs-tools` and the indexer job.
- **front** — a public, VNet-integrated environment. It holds `agent-api`.

Rerun A3: the compromised endpoint in `agent-api` has nothing to connect *with* — no role, no connection string — and nothing to connect *to*, because Postgres and NATS are reachable only inside the plant environment.

## 3. What is Container Apps Express?

Once the API is thin — authenticate, call a model, call two internal tools, stream — it becomes a candidate for [Azure Container Apps Express](https://learn.microsoft.com/en-us/azure/container-apps/express-overview?wt.mc_id=MVP_395548). I covered Express in detail in [Harness, Sandbox, State, Speed](/2026/azure-container-apps-agentic-era/): it is the "just run my container" mode of Container Apps, built on Sandboxes, with subsecond scale from zero and usage-based compute. The image stays a standard OCI container; nothing about Express leaks into the code.

The question for any component is simple: **does its complete set of production requirements fit inside the current Express feature set?**

As of my check, Express supports HTTP ingress (internal and external), scale to zero, multiple replicas, metrics and Log Analytics, IP restrictions, CORS, **user-assigned managed identity**, HTTP/CPU/memory scaling rules, VNet integration for **outbound** traffic, and a private endpoint when public access is disabled.

It does **not** currently support sidecars, init containers, Dapr, jobs, GPUs, Easy Auth, Key Vault secret references, custom domains, internal service discovery, multiple revisions with traffic splitting, OpenTelemetry, Azure Monitor logging, Aspire, HTTP/2, or exec probes. Check the [live feature matrix](https://learn.microsoft.com/en-us/azure/container-apps/express-overview?wt.mc_id=MVP_395548#supported-features) before deploying — it has already changed during the preview.

## 4. Which parts of the agent fit Express?

| Component | Fits Express? | Why |
| --- | --- | --- |
| `agent-api` (Stage 2) | Yes | Thin HTTP orchestrator, user-assigned identity, own authorization code |
| `alarm-tools`, `docs-tools` | No | Internal ingress and service discovery from other apps |
| `indexer` | No | It is a job |
| `agent-api` with a policy sidecar (Part 4) | No | Express has no sidecars |
| GPU inference (Part 5) | No | Express has no GPUs |

So `agent-api` fits. But two findings changed how the demo uses Express.

**Networking.** An Express app cannot reach *internal* ingress in another environment — the documentation is explicit that clients elsewhere in the virtual network get an HTTP 404. That is why the plant environment is an internal environment with the tools published on its internal load balancer: both the front environment and the Express environment egress into the same VNet and resolve the tools through private DNS.

[TODO: confirm the networking configuration on the deployed Express variant — docs/results/stage-2/express-check.md]

**Tooling.** At the time of writing, `azd` fails on Express environments (a fix is open in the azure-dev repository), and Aspire's `AsExpress()` ships only in Aspire 13.6. And Express has no OpenTelemetry, so the Express variant doesn't show up in the traces at all.

**Impact:** In the demo, the split deploys with `azd` to the standard environments, and `agent-api` on Express is a separately deployed **variant** of the same image — for the comparison and the cold-start numbers below.
{: .impact}

I want to be upfront about one more consequence: in Part 4 I add a governance sidecar next to the agent, and Express does not run sidecars. The demo therefore keeps **both variants** of `agent-api` from Part 4 on — Express without the sidecar, standard with it — so the tradeoff is visible rather than hidden.

## 5. Cold start is an end-to-end number

Scale to zero is attractive for an assistant used a few times a day. But the first request after idle has to start the app, load the agent runtime, and wait for whatever the model is doing. Express optimizes the first of those steps only.

In my previous article, I quoted a Microsoft engineer's measurement of a plain container: about 1.5 seconds from zero on Express versus about 20 seconds on standard consumption. Here I measured the agent itself — `agent-api` on Express and on a standard consumption app, cold after idle and warm:

| Configuration | Time to first byte — cold | Total — cold | Time to first byte — warm | Total — warm |
| --- | --- | --- | --- | --- |
| Express | [TODO] | [TODO] | [TODO] | [TODO] |
| Standard consumption, min replicas 0 | [TODO] | [TODO] | [TODO] | [TODO] |
| Standard consumption, min replicas 1 | [TODO] | [TODO] | [TODO] | [TODO] |

[TODO: one or two sentences interpreting the measured numbers once the table is filled in]

**Why it matters:** "No idle compute" for the Express app is not "no cost" for the agent.<br>
→ When you compare cost, include the model, logging and the tool apps.
{: .why}

## 6. What this layer does not do

Identity answers *who is calling*. It does not answer *what that caller may see*. `agent-api` can legitimately call `alarm-tools`, and `alarm-tools` still returns any asset it is asked about.

| Attack | Stage 0 | Stage 1 | Stage 2 |
| --- | --- | --- | --- |
| A1 Exfiltration via generated code | Succeeds | Blocked | Blocked |
| A2 Secret theft via generated code | Succeeds | Blocked | Blocked |
| A3 Lateral movement from the API | Succeeds | Succeeds | **Blocked** |
| A4 Cross-asset data access | Succeeds | Succeeds | Succeeds |
| A5 Injected privileged action | Succeeds | Succeeds | Succeeds |
| A6 Data leaving the boundary | Happens | Happens | Happens |

**Impact:** A4 and A5 go straight through the new boundaries, because they use the agent's legitimate path. Stopping them needs a decision at the moment of the tool call — Part 4.
{: .impact}

## Takeaways

1. **Sandboxing the code does not isolate the agent** — lateral movement comes from what the agent's own process can reach.
2. **The component the user talks to should reach the least** — the thin API has no database role and no connection string; tools own their data.
3. **`WithReference(db)` is an admin credential** — splitting apps is not enough if every app still gets the same connection string.
4. **Express fits the thin API, not the whole agent** — identity and scale to zero are in; sidecars, jobs, GPUs, internal service discovery and OpenTelemetry are out.
5. **Measure cold start end to end** — the platform optimizes container startup; the runtime and the model are still yours.

## Demo and Useful Links

This article corresponds to the `stage-2-express` tag of the demo: [TODO: link to demo repository and tag].

1. [Azure Container Apps express overview and feature matrix](https://learn.microsoft.com/en-us/azure/container-apps/express-overview?wt.mc_id=MVP_395548) — Microsoft Learn
2. [Azure Container Apps express FAQ](https://learn.microsoft.com/en-us/azure/container-apps/express-faq?wt.mc_id=MVP_395548) — Microsoft Learn
3. [Deploy an Express app with the Azure CLI](https://learn.microsoft.com/en-us/azure/container-apps/deploy-express-cli?wt.mc_id=MVP_395548) — Microsoft Learn
4. [Ingress in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/ingress-overview?wt.mc_id=MVP_395548) — Microsoft Learn
5. [Managed identities in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/managed-identity?wt.mc_id=MVP_395548) — Microsoft Learn
6. [Container Apps jobs](https://learn.microsoft.com/en-us/azure/container-apps/jobs?wt.mc_id=MVP_395548) — Microsoft Learn

**Series:** [Part 1 — The unsafe agent](/2026/unsafe-ai-agent-azure-container-apps/) · [Part 2 — Sandboxes](/2026/azure-container-apps-sandboxes-ai-generated-code/) · Part 3 — Express and identities · [Part 4 — MCP and governance](/2026/mcp-agent-governance-toolkit-azure-container-apps/) · [Part 5 — Private inference](/2026/private-inference-serverless-gpu-azure-container-apps/)
