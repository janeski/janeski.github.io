# MCP Gives the Agent Tools. Who Decides What Those Tools May Do?

**Series:** Azure Container Apps for production AI · **Part 4 of 5**  
**Suggested slug:** `azure-container-apps-mcp-agent-governance`  
**SEO description:** Host MCP tools on Azure Container Apps, choose the right session model, and enforce authorization and policy before an AI agent acts.  
**Technical status checked:** 22 September 2026

Our incident assistant can answer better questions once it has tools: `getAlarmHistory`, `findProcedure`, and perhaps `createMaintenanceDraft`. MCP provides a common way for clients to discover and invoke those capabilities. It does not, by itself, establish whether the engineer may inspect a particular asset or whether the agent may create a work order.

Azure Container Apps supports two quite different MCP hosting patterns. For **custom business tools**, build an MCP server with an SDK and deploy it as a normal container app. For **sandboxed Python and shell execution**, a dynamic session pool can expose a platform-managed MCP endpoint with predefined tools. A regular container app has no special MCP runtime awareness; it hosts the HTTP server you write. The managed session endpoint is a separate preview feature. [MCP hosting overview](https://learn.microsoft.com/en-us/azure/container-apps/mcp-overview)

## Make the custom tools deliberately small

An alarm MCP server should expose `getAlarmHistory(assetId, from, to)` rather than arbitrary SQL. The server validates time range and asset identifier, enforces caller authorization, caps the number of records, and returns only the fields the agent needs. A draft-creation tool should record the proposed operation and route it for review. A tool that can acknowledge alarms or control equipment requires an even tighter boundary and explicit human authorization.

For a standalone MCP server, Container Apps provides HTTPS ingress, autoscaling, managed identity, and optional built-in authentication backed by Microsoft Entra ID. Enabling authentication establishes the caller; **the tool implementation must still enforce per-asset and per-operation authorization**. A minimum replica of one can help an interactive tool avoid cold-start delay. If clients run in a browser or VS Code, configure only the necessary CORS origins and headers.

Dynamic sessions MCP is different. Its managed endpoint exposes built-in commands such as launching an environment and running Python or shell code. It uses an `x-ms-apikey` credential scoped to the **entire session pool**, rather than the bearer-token approach documented for standalone MCP with built-in auth. Keep that key away from untrusted clients, rotate it, constrain network egress, and set a session lifetime. The endpoint does not become a custom alarm-tool host simply because it speaks MCP. [MCP authentication](https://learn.microsoft.com/en-us/azure/container-apps/mcp-authentication)

## Add policy at the moment of action

The model might choose a syntactically valid tool call that violates a business rule. Put a policy check in the agent's tool-invocation path and another authorization check in the tool service. For example:

1. The orchestrator proposes `getAlarmHistory(pump-42, 24h)`.
2. Policy checks the agent's allowed tools, the caller's role, and the maximum requested time range.
3. The tool verifies the caller's entitlement to pump 42 and logs the decision.
4. The result is filtered before it enters the model's context.

Microsoft's open-source **Agent Governance Toolkit** offers policy and agent-runtime components, and its repository includes a Container Apps deployment guide using a governance sidecar. This is an integration you deploy and configure; it is **not an automatic, built-in Container Apps policy layer**. Its sidecar pattern currently calls for standard Container Apps because Express does not support sidecar containers. Use a policy engine where it helps, but keep authorization at the tool boundary even if the agent code or middleware changes. [Toolkit deployment guide](https://github.com/microsoft/agent-governance-toolkit/blob/main/docs/deployment/azure-container-apps.md)

## Test the failure paths

Ask the agent for another customer's alarm, an asset outside its scope, and an action beyond its permissions. Verify that the call is rejected before data is retrieved, that the tool also rejects direct access, and that the audit trail distinguishes a policy denial from a tool error. Simulate a prompt injection inside a retrieved maintenance note. The note is evidence for the task; it has no authority to widen the tool's permissions.

MCP improves interoperability. The production safety property comes from identities, limited tool surfaces, policy decisions, and server-side authorization working together.

### Sources and further reading

- [Host MCP servers on Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/mcp-overview)
- [Secure MCP servers](https://learn.microsoft.com/en-us/azure/container-apps/mcp-authentication)
- [Azure Container Apps authentication](https://learn.microsoft.com/en-us/azure/container-apps/authentication)
- [Microsoft Agent Governance Toolkit deployment on Container Apps](https://github.com/microsoft/agent-governance-toolkit/blob/main/docs/deployment/azure-container-apps.md)
