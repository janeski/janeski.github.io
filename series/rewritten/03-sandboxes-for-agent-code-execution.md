---
title: "Dynamic Sessions vs Sandboxes: Where AI-Generated Code Should Run on Azure Container Apps"
description: "Letting an agent write code is useful; running that code inside the agent's process is a poor trust boundary — how I choose between dynamic sessions and Sandboxes, and how I lock down egress."
tags: [azure, azure-container-apps, sandboxes, dynamic-sessions, ai, ai-agents, security, python, cloud-native, code-execution, iot]
---

**Series:** Azure Container Apps for production AI · **Part 3 of 5**
**Suggested slug:** `azure-container-apps-sandboxes-ai-code-execution`
**Technical status checked:** 22 September 2026

[image: hero photo — Photo by <name> on Pexels]

Suppose our incident assistant needs to calculate how often a pump alarms after a pressure spike. Letting the model generate a short Python script for that is genuinely useful — it is the kind of ad-hoc analysis nobody writes a permanent feature for. **Running that script inside the agent API process is a poor trust boundary.** The script would share the API's filesystem, its network path, and quite possibly its credentials.

The traditional answer is "don't let the agent run code". With Azure Container Apps, the answer is "run it somewhere else, with less than it needs to do damage". In this article, I want to show you the two options the platform gives you for that, how I choose between them, and the one setting that matters most.

## What are dynamic sessions and Sandboxes?

Azure Container Apps offers two related but distinct resources for isolated execution. They are not two names for the same feature.

**[Dynamic sessions](https://learn.microsoft.com/en-us/azure/container-apps/sessions?wt.mc_id=MVP_395548)** provide prewarmed, short-lived environments allocated from a pool — a built-in code interpreter for Python, or custom container sessions when you need your own dependencies. The platform hands you an isolated environment, you run your work, and it is disposed of after its configured idle period.

**[Sandboxes](https://learn.microsoft.com/en-us/azure/container-apps/sandboxes-overview?wt.mc_id=MVP_395548)** — currently in public preview — are first-class `Microsoft.App/SandboxGroups` resources with individual instances you control through SDK or CLI. They support OCI images, suspend and resume, disk or full-memory snapshots, and lifecycle policies such as autosuspend and auto-delete. Microsoft describes them as isolated lightweight virtual machines. Compute charges stop when a Sandbox is stopped; storage, networking and other associated charges you still have to check against current pricing.

## Choose by the lifetime of the work

This is the whole decision, and it is simpler than the feature lists suggest.

| Question | Dynamic session | Sandbox |
| --- | --- | --- |
| Is the task short and disposable? | Usually yes | Possible, but more lifecycle to manage |
| Must the agent resume its workspace later? | No; sessions are ephemeral | Yes, use suspend/resume or snapshots |
| Do we need explicit per-workspace network policy? | Session network controls | Sandbox egress rules and inspection modes |
| Do we want direct lifecycle operations? | Pool-managed | Individual Sandbox management |

For a one-off analysis of a bounded telemetry export, a dynamic session is the natural fit. The pump correlation is exactly that: one script, one result, throw the environment away.

If the engineer and the agent need to come back to the same workspace over a multi-step investigation — load the data on Monday, refine the analysis on Tuesday, compare against another asset on Wednesday — a Sandbox is more interesting. Suspend it between steps, snapshot it before a risky change, resume it when the engineer returns.

My position: start with dynamic sessions, and introduce Sandboxes only for the resumable path. Sessions carry less lifecycle to manage, and most agent code execution is disposable.

## Build a narrow execution request

Whichever resource you choose, the request you hand it should be minimal. This is an intentional design choice: the isolated environment should never have more than the analysis needs.

For the alarm example:

1. Export only the time window and the sensor columns needed for the correlation — not the full telemetry table.
2. Give the generated script a CPU, memory, time and output-size budget.
3. Read back a result artifact and validate its shape before it goes anywhere near the model's context.
4. Delete or suspend the workspace according to policy.
5. Keep the source dataset and the approved results in a durable service outside the execution environment.

The execution environment is a calculator, not a data store.

## Egress is the setting that matters

The most consequential Sandbox setting is the [egress policy](https://learn.microsoft.com/en-us/azure/container-apps/sandboxes-egress-policies?wt.mc_id=MVP_395548). It supports `Allow`, `Deny`, `Transform` and `Rewrite` rules, matched on destination host, path and HTTP method.

**Start with default `Deny` and allow only the destinations the analysis requires.** For untrusted scripts, pick an inspection mode that actually enforces the rules: the documentation says `Full` inspects all traffic and blocks non-HTTP traffic, while `Partial` permits non-HTTP traffic that does not match a rule. That distinction matters if the threat you care about is exfiltration rather than an accidental HTTP call — and for AI-generated code, exfiltration is the threat you care about.

The policy can also inject a header from a secret reference or a managed identity into an outbound call. That is a good way for an approved request to reach a model or an API without a credential ever appearing in the generated code. It is not a reason to allow arbitrary URLs. Narrow destination first, credential injection second.

## Isolation does not validate the answer

A script can run perfectly safely and still calculate the wrong correlation. Treat its output as untrusted data:

- Validate schemas and ranges
- Keep the original dataset traceable
- Show the engineer which window and which calculation produced the conclusion
- Where an operational action is proposed, require a separate authorization step — part 4 covers that

Sandboxes are a **public preview**. Microsoft notes that preview Sandboxes may need to be recreated after future releases and that the SDK and CLI surfaces may change. I would pilot them for the resumable investigation path and keep dynamic sessions for the short analysis steps, but I would not build a production dependency on the preview surface yet.

## Takeaways

1. **Sessions and Sandboxes are different resources** — ephemeral pool-managed environments versus individually controlled, resumable lightweight VMs.
2. **Lifetime of the work decides** — disposable analysis goes to a dynamic session; a multi-step investigation that must resume goes to a Sandbox.
3. **Deny-first egress is the production contract** — the orchestrator asks for computation, the isolated environment receives minimum data and minimum network, the result returns through a validated channel.
4. **Isolation and correctness are separate problems** — the Sandbox protects your network; your validation protects the engineer.

The session and Sandbox execution paths of the incident assistant are part of the series demo: [TODO: link to series demo repository].

## Links

1. [Azure Container Apps Sandboxes overview](https://learn.microsoft.com/en-us/azure/container-apps/sandboxes-overview?wt.mc_id=MVP_395548)
2. [Sandbox egress policies](https://learn.microsoft.com/en-us/azure/container-apps/sandboxes-egress-policies?wt.mc_id=MVP_395548)
3. [Sandbox snapshots and state](https://learn.microsoft.com/en-us/azure/container-apps/sandboxes-snapshots-state-management?wt.mc_id=MVP_395548)
4. [Dynamic sessions overview](https://learn.microsoft.com/en-us/azure/container-apps/sessions?wt.mc_id=MVP_395548)
