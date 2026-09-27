---
title: "Is Azure Container Apps Express the Right Home for an AI Agent API?"
date: 2026-09-30 08:00:00 +0000
description: "Azure Container Apps Express promises a faster path for small HTTP apps — here is where it fits an AI agent API, where its preview feature set stops, and what I would deploy first."
categories: [cloud, azure]
tags: [azure, azure-container-apps, express, ai, ai-agents, serverless, cloud-native, dotnet, devops, scale-to-zero, managed-identity]
---

*This is part 2 of 5 in the series **Azure Container Apps for production AI**. Technical status checked: 22 September 2026.*

The most correct answer to the question "Should the agent API run on Azure Container Apps Express?" is: *It depends!* And what it depends on is one thing — **whether the entire set of production requirements for that API fits inside Express's current preview feature set.** A faster deployment path does not remove the need to decide identities, networking, rollout behavior and latency targets. So let me elaborate on my reasoning.

In my [previous article](/2026/azure-container-apps-ai-production-architecture/) I split the incident assistant into five components with separate trust boundaries. The first of them — the agent API — needs an HTTP entry point. Its first version does very little: authenticate a request, retrieve a small amount of permitted context, call a hosted model, stream back an answer. That is exactly the class of app Express is designed for.

## What is Azure Container Apps Express?

[Express](https://learn.microsoft.com/en-us/azure/container-apps/express-overview?wt.mc_id=MVP_395548) is a deployment model for HTTP apps on consumption CPU: an optimized cold-start path, usage-based compute, and scale to zero when the app is idle. The portal creates a lightweight environment behind the scenes; the CLI path still creates an Express environment explicitly. Your application remains a standard OCI container that you build and test locally — nothing about Express leaks into the image. Microsoft lists AI frontends and gateways among its intended workloads.

One detail is easy to miss: **Express does not provide GPU compute.** For our incident assistant, the Express app receives the engineer's question and calls a model that runs somewhere else — a managed model API, or the serverless GPU app I cover in part 5.

## The current feature matrix

Preview features move fast, and Express has already moved since its early description. As of my check, Express supports:

- HTTP ingress, scale to zero, multiple replicas
- Metrics and Log Analytics integration
- IP restrictions
- User-assigned managed identity for the app runtime
- HTTP, CPU and memory scaling rules
- Some private-network scenarios — virtual-network egress and environment private endpoints, with constraints

Identity, networking and scaling were listed as unavailable in the first preview notes. They are not anymore. That is good news for a small orchestrator.

The gaps are just as important. Express does **not** currently support:

- Dapr, jobs, GPUs, sidecars
- Built-in Easy Auth
- Key Vault secret references
- Custom domains and internal service discovery
- Multiple revisions with traffic splitting
- The environment's managed OpenTelemetry feature
- Exec health probes (HTTP/TCP probes are available)

Networking supports specific outbound and private-endpoint paths; it is not interchangeable with every configuration of a standard workload-profiles environment. Check the [live feature matrix](https://learn.microsoft.com/en-us/azure/container-apps/express-overview?wt.mc_id=MVP_395548#supported-features) before you deploy — and be aware that the [Express FAQ](https://learn.microsoft.com/en-us/azure/container-apps/express-faq?wt.mc_id=MVP_395548) still carries some older descriptions.

## Mapping the agent API to Express

| Requirement for the agent API | Express decision |
| --- | --- |
| Public or supported internal HTTP endpoint, small CPU orchestrator | Strong candidate |
| Runtime access using a user-assigned identity | Supported; validate the target resource and permissions |
| App-level authorization in your own code or upstream gateway | Possible; built-in Easy Auth is unavailable |
| Jobs, Dapr, GPU, sidecar-based policy, or Key Vault references | Use a standard Container Apps environment for that component |
| Percentage-based rollout across revisions | Use standard Container Apps |

The pattern is clear: the thin orchestrator fits; anything that needs a sidecar, a job or progressive rollout does not.

## Latency and cost are an engineering choice, not a platform promise

Scale to zero is attractive when incident requests arrive a few times a day. But the first request after an idle period still has to start the app, load the agent runtime, establish dependencies and possibly wait for a model that also scaled to zero somewhere else. The optimized startup helps with the first of those steps only. **Measure time to first byte and total response time after realistic idle periods** — not after a warm deployment.

If the assistant sits on an operational incident path with a strict latency target, test a warm configuration in a standard Container Apps environment as the comparison. And when you compare cost, include idle compute, model charges, logging and downstream services. "No idle compute" for the Express app does not mean "no cost" for the complete AI service.

## What I would deploy first

**If you need Dapr, a policy sidecar, jobs or traffic splitting from day one:** skip Express for that component and use a standard Container Apps environment. There is no point in starting somewhere you know you have to leave.

**If the first release is a thin HTTP gateway:** Express is a good fit, and this is where I would start. Concretely:

1. Deploy the orchestrator on Express with a narrow user-assigned identity and explicit authorization in the application code.
2. Keep the data-access tools in separate apps — part 4 covers why.
3. Load-test concurrency, inspect cold requests, simulate an unavailable model endpoint, and make sure retries cannot multiply model calls.

If the product later needs progressive rollout, built-in auth or a sidecar, move that one component to standard Container Apps. The container image and the HTTP boundary make the move manageable; the environment configuration and operational behavior still need rework, so plan for it rather than being surprised by it.

## Takeaways

1. **Express removes setup for a specific class of app** — a small HTTP orchestrator on consumption CPU fits that class well.
2. **The value in production comes from knowing which requirements that class includes** — identity, networking and scaling are in; Dapr, jobs, sidecars, GPUs and traffic splitting are out.
3. **Cold start is end-to-end** — the platform optimizes container startup; your runtime, dependencies and model are still yours to measure.
4. **Design the exit before you enter** — keep tools and inference in separate apps so moving the orchestrator later is a redeploy, not a rewrite.

The Express deployment of the incident assistant's API is part of the series demo: [TODO: link to series demo repository].

## Links

1. [Azure Container Apps Express overview and feature matrix](https://learn.microsoft.com/en-us/azure/container-apps/express-overview?wt.mc_id=MVP_395548)
2. [Deploy an Express app with the Azure CLI](https://learn.microsoft.com/en-us/azure/container-apps/deploy-express-cli?wt.mc_id=MVP_395548)
3. [Develop and test Express apps locally](https://learn.microsoft.com/en-us/azure/container-apps/express-local-testing?wt.mc_id=MVP_395548)
