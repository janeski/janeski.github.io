---
title: "Keep the Data Inside: Private Inference on Azure Container Apps Serverless GPUs"
date: 2026-12-15 08:00:00 +0000
description: "Part 5 — running the agent's model on Azure Container Apps serverless GPUs so prompts never leave the environment, what cold start really includes, and measured tradeoffs against a managed API."
categories: [cloud, azure]
post_class: post--vivid
image: /images/posts/aca-agent-hardening-part5-cover.png
tags: [azure, azure-container-apps, serverless-gpu, gpu, ai, inference, ai-agents, security, cloud-native, observability, cost, dotnet]
---

*Part 5 of 5 in the series **From Prototype to Trust: Hardening an AI Agent on Azure Container Apps**. Technical status checked: 22 September 2026.*

After [Part 4](/2026/mcp-agent-governance-toolkit-azure-container-apps/), the incident assistant looks very different from the unsafe agent in [Part 1](/2026/unsafe-ai-agent-azure-container-apps/). Generated code runs in a dynamic session or a Sandbox with no network. The API reaches nothing it doesn't need. Tools authorize every call, and privileged actions wait for a human.

One line of the attack table has not changed since Stage 0:

> **A6 — Data leaving the boundary.** Every prompt carries telemetry, alarm history and maintenance notes to an external model provider.

A6 is not an attack. It is a trust decision, and for many teams it is the right one — a managed model API is simpler, and I would start there. But for a lot of operational-technology customers I work with, "our plant data goes to a third-party model endpoint" is where the conversation ends.

**After four layers, the only thing still leaving the environment is the prompt itself. The last layer is about where the reasoning happens.**
{: .callout}

## 1. What are serverless GPUs in Container Apps?

[Serverless GPUs](https://learn.microsoft.com/en-us/azure/container-apps/gpu-serverless-overview?wt.mc_id=MVP_395548) give a container app GPU compute with scale to zero and per-second GPU billing. Azure currently documents NVIDIA T4 and A100 options; access requires a GPU quota request, and availability depends on region and capacity.

The placement detail that trips people up: **serverless GPUs run in a Consumption workload profile inside a workload-profiles environment** — not in a Consumption-only environment and not in Express. The inference app lives next to the tool apps from Part 3, in the internal plant environment, reachable only over internal ingress.

## 2. Stage 4: bring the model inside

The new component is `inference`: an open model served by an inference server in a container, on a serverless GPU profile.

![Where the prompt goes: in Stages 0 to 3, agent-api sends prompts through IChatClient to a managed model API outside the environment; in Stage 4, the same IChatClient points at an inference app with an open model on a serverless GPU, inside a workload-profiles environment with internal ingress only](/images/posts/aca-agent-hardening-stage4.svg)

[TODO: model, inference server and image used in the demo (e.g. model name/size, vLLM or Ollama, version) — Inference/Dockerfile]

`agent-api` doesn't know which backend it talks to. It has used `IChatClient` since Stage 0, and the backend is configuration:

[TODO: code — IChatClient registration switching between Azure OpenAI and the private endpoint (Agent.Api, stage-4-private-model)]

This is an intentional design choice.

**Why it matters:** Keeping the orchestrator CPU-based and inference behind an abstraction means the two tiers scale on different signals.<br>
→ Switching between private and managed inference — for a customer, a region, or a cost reason — is a configuration change, not a rewrite.
{: .why}

Locally, the AppHost runs the same abstraction against Ollama, so the full agent still starts with `dotnet run`.

## 3. Cold start includes the model

This is the part that surprises teams coming from CPU workloads. A container can be *running* while the server is still downloading weights, filling GPU memory, or initializing kernels.

**Probe readiness only when inference can actually serve a request.** A container that is up but cannot answer is worse than one that is down, because it receives traffic.
{: .callout}

The current [constraints](https://learn.microsoft.com/en-us/azure/container-apps/gpu-serverless-overview?wt.mc_id=MVP_395548#considerations) shape the packaging:

- **One GPU container per app** — only one container in an app gets GPU access.
- **No multi-GPU, no fractional GPU** — one replica, one whole GPU.
- **A platform-managed driver stack** — pin and validate the serving image against the documented GPU software stack; Microsoft updates drivers and runtime over time.

**Impact:** Lean images and artifact streaming may help cold start; a minimum replica protects latency at the price of idle cost. There is no setting that gives you both for free.
{: .impact}

## 4. The measurements

I ran three load shapes against the private endpoint and against a managed model API, on the same prompts from the incident assistant:

1. **One request after idle** — the true cold start, model load included
2. **A burst of 10 concurrent requests** — how fast the tier scales and what queues up
3. **Steady traffic for one hour** — the cost curve you actually pay

| Load shape | Metric | Serverless GPU | Managed model API |
| --- | --- | --- | --- |
| One request after idle | Time to first token | [TODO] | [TODO] |
| One request after idle | Total response time | [TODO] | [TODO] |
| Burst of 10 | p50 / p95 total response time | [TODO] | [TODO] |
| Steady, 1 hour | p95 total response time | [TODO] | [TODO] |
| Steady, 1 hour | Cost | [TODO] | [TODO] |

GPU type, region and model: [TODO].

[TODO: two or three sentences interpreting the results — where serverless GPU wins, where it doesn't]

**Why it matters:** The platform does not explain why a model is slow.<br>
→ Instrument the whole path, not only the model: request duration, time to first token, GPU cold starts, queue wait, tool latency, error rate, per-request usage. The [managed OpenTelemetry agent](https://learn.microsoft.com/en-us/azure/container-apps/opentelemetry-agents?wt.mc_id=MVP_395548) can export it in a standard environment, but your code and your inference server still have to emit the model-specific signals.
{: .why}

## 5. So where does inference belong?

It depends — on **whether the prompt may leave, and on the shape of the demand**.

**If prompts must not leave your environment and demand is bursty:** serverless GPUs are the right fit, with a readiness probe that waits for the model.

**If inference is continuous and latency-sensitive:** compare a dedicated GPU workload profile before you commit to elastic capacity. Azure's [Well-Architected guidance](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-container-apps?wt.mc_id=MVP_395548) makes the same split.

**If neither data residency nor custom weights is a hard requirement:** use the managed model API. It is the simplest thing to operate, and simplicity in production is worth a lot.

**Impact:** Private inference has its own price. A smaller open model reasons differently from a frontier model — test answer quality on your own incident questions, not only latency. And moving the model inside also moves its operation onto your team: image updates, quota, capacity tests.
{: .impact}

## 6. The series in one table

| Attack | Stage 0 | 1 Sandbox | 2 Split + Express | 3 Governance | 4 Private model |
| --- | --- | --- | --- | --- | --- |
| A1 Exfiltration via generated code | Succeeds | Blocked | Blocked | Blocked | Blocked |
| A2 Secret theft via generated code | Succeeds | Blocked | Blocked | Blocked | Blocked |
| A3 Lateral movement from the API | Succeeds | Succeeds | Blocked | Blocked | Blocked |
| A4 Cross-asset data access | Succeeds | Succeeds | Succeeds | Blocked | Blocked |
| A5 Injected privileged action | Succeeds | Succeeds | Succeeds | Held | Held |
| A6 Data leaving the boundary | Happens | Happens | Happens | Happens | **Inside** |

Read it by columns and you see how much each layer actually buys. Read it by rows and you see why there is no single "secure the agent" feature: every attack was stopped by a different layer, and every layer left others open.

## Takeaways

1. **Sandboxing protects your environment from code you didn't write** — it stopped exfiltration and secret theft. It did nothing for the agent's own identity, and nothing for the correctness of the result.
2. **Infrastructure isolation and identity limit what a compromise can reach** — splitting the agent stopped lateral movement. It did nothing for legitimate calls with illegitimate arguments.
3. **Runtime governance decides what the agent may do at the moment of action** — authorization at the tool and policy at the agent stopped cross-asset access and injected actions, at the cost of a sidecar that doesn't run on Express.
4. **Where inference runs is a trust decision, not only a cost decision** — serverless GPUs keep prompts inside the environment; measure cold start with the model loaded, and compare against the managed API on your own prompts.

**Container Apps can host every one of these pieces. The trust comes from the boundaries between them — and those remain yours to design.**
{: .callout}

## Demo and Useful Links

This article corresponds to the `stage-4-private-model` tag, which is also `main`: [TODO: link to demo repository].

1. [Serverless GPUs in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/gpu-serverless-overview?wt.mc_id=MVP_395548) — Microsoft Learn
2. [GPU types and regions](https://learn.microsoft.com/en-us/azure/container-apps/gpu-types?wt.mc_id=MVP_395548) — Microsoft Learn
3. [Managed OpenTelemetry agent](https://learn.microsoft.com/en-us/azure/container-apps/opentelemetry-agents?wt.mc_id=MVP_395548) — Microsoft Learn
4. [Azure Container Apps observability](https://learn.microsoft.com/en-us/azure/container-apps/observability?wt.mc_id=MVP_395548) — Microsoft Learn
5. [Azure Well-Architected guidance for Container Apps](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-container-apps?wt.mc_id=MVP_395548) — Well-Architected Framework

**Series:** [Part 1 — The unsafe agent](/2026/unsafe-ai-agent-azure-container-apps/) · [Part 2 — Sandboxes](/2026/azure-container-apps-sandboxes-ai-generated-code/) · [Part 3 — Express and identities](/2026/azure-container-apps-express-agent-identity-split/) · [Part 4 — MCP and governance](/2026/mcp-agent-governance-toolkit-azure-container-apps/) · Part 5 — Private inference
