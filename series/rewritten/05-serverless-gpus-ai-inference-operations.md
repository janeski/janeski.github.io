---
title: "Serverless GPUs vs Dedicated GPUs vs a Managed Model API: Where Inference Belongs on Azure Container Apps"
description: "When running your own model on Azure Container Apps serverless GPUs is the right call, what cold start really includes, and the three load shapes I would measure before deciding."
tags: [azure, azure-container-apps, serverless-gpu, gpu, ai, inference, ai-agents, cloud-native, observability, cost, performance]
---

**Series:** Azure Container Apps for production AI · **Part 5 of 5**
**Suggested slug:** `azure-container-apps-serverless-gpu-ai-inference`
**Technical status checked:** 22 September 2026

[image: hero photo — Photo by <name> on Pexels]

The incident assistant does not *need* to host its own model. For most teams a managed model API is simpler, and I would start there. But suppose your team has a specialized open model, needs control over the inference server, or wants a container boundary around the data that goes into the model. Azure Container Apps serverless GPUs make that possible without running a GPU cluster yourself.

So should inference run on serverless GPUs? *It depends* — on the demand curve, on the latency target, and on how much operational burden you are willing to own. In this article, I want to lay out the three options, the one placement detail that trips people up, and how I would evaluate them before committing.

## What are serverless GPUs in Azure Container Apps?

[Serverless GPUs](https://learn.microsoft.com/en-us/azure/container-apps/gpu-serverless-overview?wt.mc_id=MVP_395548) give a container app GPU compute with scale to zero and per-second GPU billing. Azure currently documents NVIDIA T4 and A100 options; access requires a GPU quota request, and availability depends on region and capacity.

The placement detail that is easy to miss: **serverless GPUs run in a Consumption workload profile inside a workload-profiles environment** — not in a Consumption-only environment, and not in Container Apps Express. If your agent API lives on Express (part 2), inference lives in a different environment by definition.

## The three alternatives

I see the options as a spectrum, from most to least operational control:

- **Dedicated GPU workload profile** — predictable capacity, always-on, you own everything
- **Serverless GPU** — your container and model, elastic capacity, scale to zero
- **Managed model API** — no image, no weights, no capacity; the vendor owns the runtime

Azure's [Well-Architected guidance](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-container-apps?wt.mc_id=MVP_395548) recommends serverless GPUs for bursty workloads and dedicated profiles for sustained, latency-sensitive inference. I agree with that split, with one addition: the managed API should always be the baseline you compare against, not an afterthought.

| Decision | Serverless GPU | Dedicated GPU or managed API |
| --- | --- | --- |
| Unpredictable, intermittent inference | Scale to zero can reduce idle compute | API may still be simpler to operate |
| Continuous, latency-sensitive inference | Cold starts and model loading need measurement | Compare a dedicated GPU profile |
| Custom weights, runtime, and dependencies | Bring a containerized serving stack | Managed API may restrict these choices |
| Operational burden | Own image, model packaging, updates, and capacity testing | Managed API shifts more of that burden |

For a maintenance assistant used a few times a day, the elastic profile is attractive. For sustained inference with a strict response-time target, a dedicated profile is more predictable. Input and output volumes, model size, time to first token, concurrent requests, daily idle hours and operational work all move the result — which is why I measure rather than guess.

## Cold start includes the model

This is the part that surprises teams coming from CPU workloads. A container can be *running* while the service is still downloading weights, filling GPU memory, or initializing kernels. **Probe readiness only when inference can actually serve a request** — a container that is up but cannot answer is worse than one that is down, because the load balancer will send it traffic.

Benchmark after a real idle period and after a fresh deployment, with the model image and artifacts you intend to ship. Lean images and artifact streaming may help; a minimum replica protects latency at the expense of idle cost. And an HTTP scaling rule knows nothing about the GPU model's queueing behavior unless you design and test for it.

The current [constraints](https://learn.microsoft.com/en-us/azure/container-apps/gpu-serverless-overview?wt.mc_id=MVP_395548#considerations) shape how you package the app: one container in an app gets GPU access, and multi-GPU and fractional-GPU replicas are not supported. Pin and validate the serving image against the platform's documented GPU software stack — Microsoft updates its drivers and runtime over time. These limits decide how you handle sidecars and how large a model fits on one replica.

## Separate inference from orchestration

Keep the orchestrator API CPU-based. It calls a GPU-backed inference app, a managed model API, or both behind the same application-level abstraction. This is an intentional design choice: it lets the agent API and the inference tier scale on different signals and different budgets, and it means switching from serverless GPU to a managed API — or back — is a configuration change in one place.

Queue long-running document enrichment as a job, and keep the interactive path bounded — the same split from part 1.

Instrument the full path: request duration, time to first token, GPU app cold starts, in-flight inference, queue wait, tool latency, error rate and per-request model usage. Container Apps gives you logs, Azure Monitor metrics, alerts and revision information; in a standard environment the [managed OpenTelemetry agent](https://learn.microsoft.com/en-us/azure/container-apps/opentelemetry-agents?wt.mc_id=MVP_395548) exports instrumented telemetry to Application Insights or another OTLP destination. Your code and your inference runtime still have to emit the model-specific signals. The platform does not automatically explain why a model's output is slow — the [observability](https://learn.microsoft.com/en-us/azure/container-apps/observability?wt.mc_id=MVP_395548) documentation is the starting point, not the finish.

## An evaluation, not a default answer

Here is what I would actually do before choosing where inference belongs. Run three load shapes:

1. **One request after idle** — the true cold start, model loading included
2. **A short burst of concurrent requests** — how fast the tier scales and what queues up
3. **Steady traffic for an hour** — the cost curve you will actually pay

Measure latency and total spend for a serverless GPU app, a dedicated option and a managed model API on the same workload. Add quota availability and failure recovery to the decision. Only then decide.

**If your demand is bursty and you need your own weights or runtime:** serverless GPUs are the right fit, with a readiness probe that waits for the model.
**If inference is continuous and latency-sensitive:** compare a dedicated GPU profile before you commit to elastic capacity.
**If neither custom weights nor a container boundary is a hard requirement:** the managed model API is usually the simplest thing to operate, and simplicity in production is worth a lot.

## Takeaways — for the article and for the series

1. **Placement decides availability** — serverless GPUs need a workload-profiles environment; Express and Consumption-only environments are out.
2. **Cold start is model load, not container start** — probe readiness on the model, benchmark from idle, and pin your serving image to the documented GPU stack.
3. **Keep orchestration CPU-based and inference behind an abstraction** — so the tiers scale separately and swapping the inference backend stays a configuration change.
4. **Measure three load shapes before choosing** — one-after-idle, burst, and steady — against serverless, dedicated and managed alternatives.

Taken together, the five articles describe the full agent path: the API receives a request, an authorized tool supplies context, an isolated environment handles generated code, and a model endpoint provides the reasoning. Container Apps can host every one of those pieces. **The quality of the architecture comes from the boundaries and the operating choices between them** — and those remain yours.

The complete incident assistant — Express API, MCP tools, Sandbox execution and GPU inference in one Aspire solution — is the series demo: [TODO: link to series demo repository].

## Links

1. [Serverless GPUs in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/gpu-serverless-overview?wt.mc_id=MVP_395548)
2. [GPU types and regions](https://learn.microsoft.com/en-us/azure/container-apps/gpu-types?wt.mc_id=MVP_395548)
3. [Azure Well-Architected guidance for Container Apps](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-container-apps?wt.mc_id=MVP_395548) — Well-Architected Framework
4. [Azure Container Apps observability](https://learn.microsoft.com/en-us/azure/container-apps/observability?wt.mc_id=MVP_395548)
