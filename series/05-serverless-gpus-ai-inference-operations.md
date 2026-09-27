# Running Your Own Model on Azure Container Apps: Serverless GPUs and the Production Tradeoffs

**Series:** Azure Container Apps for production AI · **Part 5 of 5**  
**Suggested slug:** `azure-container-apps-serverless-gpu-ai-inference`  
**SEO description:** Decide when Azure Container Apps serverless GPUs fit private model inference, and plan cold starts, scaling, observability, and cost.  
**Technical status checked:** 22 September 2026

The incident assistant does not need to host its own model. A managed model API may be simpler. But suppose our team has a specialized open model, needs control over its inference server, or wants a container boundary for data processing. Azure Container Apps serverless GPUs make that possible without running a GPU cluster ourselves.

The placement detail is easy to miss: **serverless GPUs run in a Consumption workload profile inside a workload-profiles environment**, not in a Consumption-only environment and not in Container Apps Express. Azure currently documents NVIDIA T4 and A100 options, scale to zero, and per-second GPU billing. Access requires a GPU quota request, and availability depends on region and capacity. [Serverless GPU overview](https://learn.microsoft.com/en-us/azure/container-apps/gpu-serverless-overview)

## Select compute for the actual demand curve

For a maintenance assistant used a few times each day, an elastic GPU profile may be attractive. For sustained inference with a strict response-time target, a dedicated GPU workload profile can be more predictable. Compare both against a managed model API on the same workload: input/output volumes, model size, time to first token, concurrent requests, daily idle hours, and operational work all affect the result. Azure's Well-Architected guidance recommends serverless GPUs for bursty workloads and dedicated profiles for sustained, latency-sensitive inference. [Architecture guidance](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-container-apps)

| Decision | Serverless GPU | Dedicated GPU or managed API |
| --- | --- | --- |
| Unpredictable, intermittent inference | Scale to zero can reduce idle compute | API may still be simpler to operate |
| Continuous, latency-sensitive inference | Cold starts and model loading need measurement | Compare a dedicated GPU profile |
| Custom weights, runtime, and dependencies | Bring a containerized serving stack | Managed API may restrict these choices |
| Operational burden | Own image, model packaging, updates, and capacity testing | Managed API shifts more of that burden |

## Cold start includes the model

A container can start while the service is still downloading weights, filling GPU memory, or initializing kernels. Probe readiness only when inference can actually serve a request. Benchmark after a real idle period and after a new deployment, with the model image and artifacts you intend to ship. Lean images and artifact streaming may help; a minimum replica can protect latency at the expense of idle cost. An app-level HTTP scaling rule does not know the GPU model's queueing behavior unless you design and test for it.

GPU support has current constraints: one container in an app gets GPU access, and multi-GPU and fractional-GPU replicas are not supported. Pin and validate the serving image against the platform's documented GPU software stack; Microsoft updates its GPU drivers and runtime over time. These limits affect how you package sidecars and how large a model you can serve on one replica. [GPU limitations and software stack](https://learn.microsoft.com/en-us/azure/container-apps/gpu-serverless-overview#considerations)

## Separate inference from orchestration

Keep the orchestrator API CPU-based. It can call a GPU-backed inference app, a managed model API, or both behind the same application-level abstraction. Queue long-running document enrichment as a job, and keep the interactive path bounded. This lets the agent API and inference tier scale for different signals and budgets.

Instrument the full path: request duration, time to first token, GPU app cold starts, in-flight inference, queue wait, tool latency, error rate, and per-request model usage. Container Apps provides logs, Azure Monitor metrics, alerts, and revision information. In a standard environment, its managed OpenTelemetry agent can export instrumented application telemetry to Application Insights or another OTLP-compatible destination. Your code and inference runtime must still emit the useful traces and model-specific metrics. The platform does not automatically explain a model's slow output. [Observability](https://learn.microsoft.com/en-us/azure/container-apps/observability) and [managed OpenTelemetry](https://learn.microsoft.com/en-us/azure/container-apps/opentelemetry-agents)

## A production evaluation, not a default answer

Start with three load shapes: one request after idle, a short burst of concurrent requests, and steady traffic for an hour. Measure latency and total spend for a serverless GPU app, a dedicated option, and a managed model API. Add quota availability and failure recovery to the decision. Only then choose where inference belongs.

Together, these five articles describe the full agent path: the API receives a request, an authorized tool supplies context, an isolated environment handles generated code, and a model endpoint provides reasoning. Container Apps can host those pieces; the architecture's quality comes from the boundaries and operating choices between them.

### Sources and further reading

- [Serverless GPUs in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/gpu-serverless-overview)
- [GPU types and regions](https://learn.microsoft.com/en-us/azure/container-apps/gpu-types)
- [Azure Well-Architected guidance](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-container-apps)
- [Azure Container Apps observability](https://learn.microsoft.com/en-us/azure/container-apps/observability)
