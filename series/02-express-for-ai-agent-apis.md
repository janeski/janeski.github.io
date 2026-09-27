# Azure Container Apps Express: A Faster Home for Small AI Agent APIs?

**Series:** Azure Container Apps for production AI · **Part 2 of 5**  
**Suggested slug:** `azure-container-apps-express-ai-agent-api`  
**SEO description:** Evaluate Azure Container Apps Express for an AI gateway or agent API, including scale to zero, managed identity, networking, and preview limitations.  
**Technical status checked:** 22 September 2026

The incident assistant introduced in part 1 needs an HTTP entry point. Its first version might only authenticate a request, call a hosted model, and return a response. Azure Container Apps Express is designed for this kind of small web workload: deploy a container image with a shorter setup path, let requests drive compute, and scale back to zero when the app is idle.

The useful question is whether **the entire agent API's production requirements fit inside Express's current preview feature set**. A fast deployment experience does not remove the need to choose identities, networking, rollout behavior, and latency targets.

## What Express changes

Express is a deployment model for HTTP apps on consumption CPU. It provides an optimized cold-start path and usage-based compute. The portal experience creates a lightweight environment behind the scenes; the CLI path still creates an Express environment explicitly. The application remains a standard OCI container that you build and test locally. The platform documentation lists AI frontends and gateways among its intended workloads. [Express overview](https://learn.microsoft.com/en-us/azure/container-apps/express-overview)

For our incident assistant, an Express app could receive an engineer's question, retrieve a small amount of permitted context through an external API, call a managed model endpoint, and stream back an answer. The model inference itself happens elsewhere: **Express does not provide GPU compute**.

## Where the current feature matrix matters

As of this article's check, Express supports HTTP ingress, scale to zero, multiple replicas, metrics, Log Analytics integration, IP restrictions, user-assigned managed identity for the app runtime, and HTTP/CPU/memory scaling rules. It also supports some private-network scenarios, including virtual-network egress and environment private endpoints, with constraints. These are useful changes from early preview descriptions that listed identity, networking, and scaling as unavailable.

There are important gaps. Express does not currently support Dapr, jobs, GPUs, sidecars, built-in Easy Auth, Key Vault secret references, custom domains, internal service discovery, multiple revisions with traffic splitting, or the environment's managed OpenTelemetry feature. It offers HTTP/TCP health probes but not exec probes. Networking supports specific outbound and private-endpoint paths; it is not interchangeable with every network configuration of a standard workload-profile environment. Check the [live feature matrix](https://learn.microsoft.com/en-us/azure/container-apps/express-overview#supported-features) before deployment: preview capabilities have already changed, and the [Express FAQ](https://learn.microsoft.com/en-us/azure/container-apps/express-faq) still contains older descriptions of some capabilities.

| Requirement for the agent API | Express decision |
| --- | --- |
| Public or supported internal HTTP endpoint, small CPU orchestrator | Strong candidate |
| Runtime access using a user-assigned identity | Supported; validate the target resource and permissions |
| App-level authorization in your own code or upstream gateway | Possible; built-in Easy Auth is unavailable |
| Jobs, Dapr, GPU, sidecar-based policy, or Key Vault references | Use a standard Container Apps environment for that component |
| Percentage-based rollout across revisions | Use standard Container Apps |

## Latency and cost are an engineering choice

Scale to zero is attractive when incident requests arrive sporadically. But a first request still has to start the app, load the agent runtime, establish dependencies, and perhaps wait for a model hosted somewhere else. The platform's optimized startup does not guarantee an end-to-end response time. Measure time to first byte and total response time after realistic idle periods.

If the assistant is on an operational incident path with a strict latency target, test a warm configuration in a standard Container Apps environment as the comparison. Include idle compute, model charges, logging, and downstream service costs in the bill. “No idle compute” for an Express app does not mean “no cost” for the complete AI service.

## A sensible first release

I would deploy Express for an isolated HTTP gateway with a narrow user-assigned identity and explicit application authorization. Keep the data-access tools separate. Load-test concurrency, inspect cold requests, simulate an unavailable model endpoint, and ensure retries cannot multiply model calls. If the product later needs progressive traffic splitting, built-in auth, or a policy sidecar, move that component to standard Container Apps. The container image and HTTP boundary make the change manageable, but environment configuration and operational behavior still need rework.

Express is useful because it removes setup for a particular class of app. Its value in production comes from knowing exactly which requirements that class includes.

### Sources and further reading

- [Azure Container Apps Express overview and feature matrix](https://learn.microsoft.com/en-us/azure/container-apps/express-overview)
- [Deploy an Express app with the Azure CLI](https://learn.microsoft.com/en-us/azure/container-apps/deploy-express-cli)
- [Develop and test Express apps locally](https://learn.microsoft.com/en-us/azure/container-apps/express-local-testing)
