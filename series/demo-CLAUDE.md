# CLAUDE.md — aca-agent-hardening

Demo repository for the blog series **"From Prototype to Trust: Hardening an AI Agent on Azure Container Apps"**. The blog lives in the sibling repo `../janeski.github.io`.

## Context from the blog repo

Read these before planning any work. They are the source of truth for scope, stages and attacks:

- Series plan: @../janeski.github.io/series/00-series-plan-v2.md
- Series status, open decisions and next steps: @../janeski.github.io/CLAUDE.md

The posts that describe each stage are in `../janeski.github.io/_posts/` (files dated 2026-09-23 … 2026-10-21). Each stage built here must match what its post says. If the demo can't do what a post claims, flag the mismatch; don't quietly change the demo or the post.

## What this repo is

One Aspire solution for an AI incident assistant on a simulated compressed-air plant (3 compressors + 1 pump, MQTT → TimescaleDB, threshold alarms). It is built deliberately unsafe (Stage 0) and hardened stage by stage. Each stage is a git tag:

| Tag | Adds | Attacks it must block | Post |
| --- | --- | --- | --- |
| `stage-0-unsafe` | Unsafe agent: one app, one identity, in-process Python, tools without authorization, external model | none (all six succeed) | Part 1 |
| `stage-1-sandbox` | Dynamic sessions + Sandbox, deny-first egress, no secrets in execution | A1, A2 | Part 2 |
| `stage-2-express` | Split into agent-api / alarm-tools / docs-tools / indexer job, per-app identity, agent-api on Express | A3 | Part 3 |
| `stage-3-governance` | MCP tool servers with entitlement checks + Agent Governance Toolkit sidecar | A4, A5 (held for approval) | Part 4 |
| `stage-4-private-model` | Open model on serverless GPU behind `IChatClient` | A6 | Part 5 |

`main` = final stage.

## Layout

```
src/AppHost  src/ServiceDefaults  src/Telemetry.Simulator  src/Agent.Api
src/Agent.Tools.Alarms  src/Agent.Tools.Docs  src/Indexer.Job  src/Inference
execution/   governance/   redteam/   infra/   data/maintenance-notes/
docs/results/
```

## Rules

- **The six attacks in `redteam/` are the acceptance tests.** After each stage, rerun all six and update `redteam/results.md` with the attack × stage table.
- **Evidence for the posts goes in `docs/results/`:** traces, screenshots, policy files, and measured numbers, with date, region, SKU and model noted. The posts' `[TODO]` markers are filled only from this folder. Never invent numbers.
- `Agent.Api` uses `IChatClient` from Stage 0, so switching the inference backend is configuration only.
- Locally, Azure-only boundaries (Sandboxes, Express, serverless GPU) are replaced by plain containers or Ollama. The AppHost must say clearly which boundaries are *not* enforced locally. Attack results quoted in posts come from the Azure deployment.
- Deploy with `azd` + Bicep. Keep per-stage parameters in `infra/`.
- Don't edit the blog posts unless asked. When a stage is done, list which `[TODO]`s in which post can now be filled, and from which files in `docs/results/`.

## Commands

```
dotnet run --project src/AppHost      # full local stack + Aspire dashboard
azd up                                # deploy current stage to Azure
```
