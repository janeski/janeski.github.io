# Let the Agent Write Code. Give That Code a Sandbox.

**Series:** Azure Container Apps for production AI · **Part 3 of 5**  
**Suggested slug:** `azure-container-apps-sandboxes-ai-code-execution`  
**SEO description:** Use Azure Container Apps Sandboxes to isolate AI-generated code, restrict its network access, and preserve agent state when necessary.  
**Technical status checked:** 22 September 2026

Suppose our incident assistant needs to calculate how often a pump alarms after a pressure spike. Letting a model generate a Python script is useful. Running that script inside the agent API process is a poor trust boundary: the script would share the API's filesystem, network path, and possibly its credentials.

Azure Container Apps now offers two related options for isolated execution. **Dynamic sessions** provide prewarmed, short-lived environments, including a built-in code interpreter or custom container sessions. **Container Apps Sandboxes**, currently in preview, add direct lifecycle control, network policy, and state that can be suspended and resumed. They are distinct resources, not two names for the same feature. [Dynamic sessions](https://learn.microsoft.com/en-us/azure/container-apps/sessions) and [Sandboxes overview](https://learn.microsoft.com/en-us/azure/container-apps/sandboxes-overview) describe their respective models.

## Choose based on the lifetime of the work

For a one-off analysis of a bounded telemetry export, a dynamic session is a natural fit. The platform allocates an isolated execution environment from a prewarmed pool; it is disposed of after its configured idle period. A built-in interpreter works for Python. A custom container session works when the analysis needs our own dependencies.

If the engineer and agent need to return to the same workspace throughout a multi-step investigation, a Sandbox is more interesting. Sandboxes are first-class `Microsoft.App/SandboxGroups` resources with individual instances controlled through SDK or CLI. They support OCI images, suspend/resume, disk or full-memory snapshots, and lifecycle policies such as autosuspend and auto-delete. They are described as isolated lightweight virtual machines. Compute fees stop when a Sandbox is stopped, but storage, networking, and other associated charges must still be checked against current pricing.

| Question | Dynamic session | Sandbox |
| --- | --- | --- |
| Is the task short and disposable? | Usually yes | Possible, but more lifecycle to manage |
| Must the agent resume its workspace later? | No; sessions are ephemeral | Yes, use suspend/resume or snapshots |
| Do we need explicit per-workspace network policy? | Session network controls | Sandbox egress rules and inspection modes |
| Do we want direct lifecycle operations? | Pool-managed | Individual Sandbox management |

## Build a narrow execution request

For the alarm example, export only a time window and the sensor columns needed to calculate the correlation. Give the generated script a CPU, memory, time, and output-size budget. Read back a result artifact, validate its shape, and delete or suspend the workspace according to policy. Store the source dataset and approved results in a durable service outside the execution environment.

The most consequential Sandbox setting is egress. The documented policy supports `Allow`, `Deny`, `Transform`, and `Rewrite` rules, with matching on destination host, path, and HTTP method. Start with default `Deny` and allow only required destinations. For untrusted scripts, use an inspection mode that actually enforces the rules: the documentation says `Full` inspects all traffic and blocks non-HTTP traffic, while `Partial` permits non-HTTP traffic that does not match a rule. That distinction matters if the threat is exfiltration rather than an accidental HTTP call. [Sandbox egress policy](https://learn.microsoft.com/en-us/azure/container-apps/sandboxes-egress-policies)

The policy can also inject a header from a secret reference or managed identity into an outbound call. This is a useful way for an approved request to reach a model or API without placing a credential in generated code. It still requires a narrow destination and authorization policy; credential injection is not a reason to allow arbitrary URLs.

## Isolation does not validate the answer

A script can run safely and still calculate the wrong correlation. Treat its output as untrusted data. Validate schemas and ranges, keep the original dataset traceable, and show the engineer which window and calculation produced the conclusion. Where an operational action is proposed, require a separate authorization step.

Sandboxes are a **public preview**. Microsoft notes that preview Sandboxes may need to be recreated after future releases and that SDK/CLI surfaces may change. I would pilot them for the resumable investigation path, while evaluating dynamic sessions for shorter analysis steps. In either case, the production contract is clear: the orchestrator asks for computation, the isolated environment receives minimum data and network access, and the result returns through a validated channel.

### Sources and further reading

- [Azure Container Apps Sandboxes overview](https://learn.microsoft.com/en-us/azure/container-apps/sandboxes-overview)
- [Sandbox egress policies](https://learn.microsoft.com/en-us/azure/container-apps/sandboxes-egress-policies)
- [Sandbox snapshots and state](https://learn.microsoft.com/en-us/azure/container-apps/sandboxes-snapshots-state-management)
- [Dynamic sessions overview](https://learn.microsoft.com/en-us/azure/container-apps/sessions)
