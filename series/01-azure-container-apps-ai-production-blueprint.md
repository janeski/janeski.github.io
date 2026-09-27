# Azure Container Apps for AI in Production: The Architecture Behind the Agent

**Series:** Azure Container Apps for production AI · **Part 1 of 5**  
**Suggested slug:** `azure-container-apps-ai-production-architecture`  
**SEO description:** Design an AI agent on Azure Container Apps with separate boundaries for its API, tools, untrusted code, background work, and model inference.  
**Technical status checked:** 22 September 2026

An agent demo often fits in one process. It receives a request, calls a model, invokes a tool, and returns an answer. That simplicity disappears when the tool can read operational data, generate code, or change a business system. Each step has a different scaling pattern and a different level of trust.

Consider an industrial incident assistant. An engineer asks why a pump alarm keeps recurring. The assistant reads recent alarms, retrieves the maintenance procedure, runs a short diagnostic over exported telemetry, and proposes the next action. It must never inherit broad production credentials merely because one tool needs to read an alarm API.

Azure Container Apps is interesting here because **apps, jobs, dynamic sessions, and the newer Sandboxes resource give us distinct execution boundaries on one platform**. The platform still leaves the agent's authorization and decision policy to us.

## Split the workload at the trust boundaries

| Component | Job in this example | Suitable hosting choice |
| --- | --- | --- |
| Agent API | Authenticate the engineer, coordinate model calls, stream an answer | Container app; Express only if its current feature set fits |
| Alarm and documentation tools | Return narrow, authorized data | Separate container app or MCP server |
| Generated diagnostic code | Process a bounded telemetry export | Dynamic session for disposable work; Sandbox for a workspace that must resume |
| Indexing and enrichment | Consume new manuals and alarm events | Event-driven Container Apps job or continuously running worker |
| Private model endpoint | Run a model whose weights and serving stack we control | Consumption GPU workload profile for variable demand; dedicated GPU for steadier load |

This division is an architecture choice, not a product requirement. Its value is that a compromised tool or generated script does not automatically receive the agent API's identity, network reachability, or data.

## Put the agent behind an ordinary production API

Start with a standard container app for the orchestrator. Give it a managed identity with only the permissions needed to call its dependencies. Keep the alarm tool in a separate app with a different identity and, where appropriate, internal ingress. Set a minimum replica count if interactive latency matters; scale to zero trades idle cost for cold starts.

Container Apps can scale on HTTP traffic and event sources through KEDA. A burst of user requests and a backlog of document-indexing messages should therefore have different scaling policies. A job is appropriate when one indexing execution has a clear end; a worker app fits a continuous consumer. A queue is also the place to absorb retries and cap the rate at which costly model calls or downstream APIs are made. [Container Apps scaling](https://learn.microsoft.com/en-us/azure/container-apps/scale-app) and [jobs](https://learn.microsoft.com/en-us/azure/container-apps/jobs) document the two models.

## Treat tool calls as authorization decisions

The model can propose `getAlarmHistory(assetId)`. It cannot decide which assets the caller may see. The tool should check the user's authorization, validate the asset identifier, limit the result set, and record the call. A proposed `acknowledgeAlarm` is a different privilege and may require a human approval step. Read access to operational technology is already sensitive; write access deserves a separate path.

For generated code, pass an export containing only the fields needed for the analysis to a session or Sandbox. Do not mount production secrets into its filesystem. Apply a deny-first egress policy to a Sandbox when code could make outbound requests. That boundary reduces the damage from an unexpected script; it does not establish that the script's conclusion is correct.

## Operate the agent as a distributed system

Trace the request across the API, retrieval, model call, tool invocation, and background job. Record the selected tool, policy decision, model version, token usage, and outcome while excluding credentials and sensitive prompt contents. A standard Container Apps environment can use its managed OpenTelemetry agent to route instrumented application telemetry to Application Insights or an OTLP destination. The agent does not instrument your code automatically; the application must emit the signals. Add timeouts, idempotency keys, and a budget per request; a retry loop around an expensive tool can become both an outage and a bill. [Managed OpenTelemetry agent](https://learn.microsoft.com/en-us/azure/container-apps/opentelemetry-agents)

Use standard Container Apps revisions to release new code progressively and shift traffic when the result is healthy. Test a candidate against realistic tool errors, slow models, and denied actions. Express currently uses a single active revision and does not support traffic splitting, which may affect this rollout strategy. [Revisions](https://learn.microsoft.com/en-us/azure/container-apps/revisions) and [observability](https://learn.microsoft.com/en-us/azure/container-apps/observability) are the platform starting points.

## The production test

Before calling this agent production ready, I would want to answer five questions: Who can invoke each tool? Where does generated code execute? Which destinations can it reach? How do we stop a runaway workflow? What would we see in traces if it took a wrong turn?

The next four articles make those choices concrete: Express for a small HTTP agent surface; Sandboxes for untrusted execution; MCP and policy for tools; and GPUs for inference when an external model API is not the right fit.

### Sources and further reading

- [Azure Container Apps overview](https://learn.microsoft.com/en-us/azure/container-apps/overview)
- [Scaling rules](https://learn.microsoft.com/en-us/azure/container-apps/scale-app)
- [Container Apps jobs](https://learn.microsoft.com/en-us/azure/container-apps/jobs)
- [Managed OpenTelemetry agent](https://learn.microsoft.com/en-us/azure/container-apps/opentelemetry-agents)
- [Azure Well-Architected guidance for Container Apps](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-container-apps)
