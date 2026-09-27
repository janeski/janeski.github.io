# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
bundle install

# Local development server (hot-reload on port 4000)
bundle exec jekyll serve

# Or using Docker
docker compose up

# Build static site
bundle exec jekyll build
```

## Architecture

This is a Jekyll-based personal blog deployed to GitHub Pages at `https://janeski.tech`. Pushing to `master` triggers automatic deployment — there is no CI/CD pipeline.

**Theme:** Custom theme derived from [jekyll-uno](https://github.com/joshgerdes/jekyll-uno) (despite `_config.yml` referencing `jekyll-theme-slate`, the actual layout/styling lives in `_layouts/`, `_includes/`, and `_sass/`).

**Key config:** `_config.yml` — pagination (10/page), permalink format `/:year/:title/`, kramdown with GFM, rouge syntax highlighting.

## Blog Posts

Posts go in `_posts/` with filename format `YYYY-MM-DD-slug.markdown`.

Required frontmatter fields:
```yaml
title: ""
date: YYYY-MM-DD HH:MM:SS +0000
description: ""      # SEO meta description
categories: []       # e.g., [cloud, dotnet]
tags: []             # e.g., [azure, aspire, dotnet]
```

Optional frontmatter:
```yaml
image: /images/filename.jpg          # featured image
image_credit: '<a href="...">...</a>' # attribution HTML
comments: false                       # disables Disqus (on by default)
canonical: "https://..."             # shows "originally published at" banner
```

Read time is calculated automatically from word count (words/200).

## In-progress: series "From Prototype to Trust: Hardening an AI Agent on Azure Container Apps"

_Status as of 27 Sep 2026._

**Concept.** Five posts based on the talk abstract: take one deliberately unsafe AI incident assistant for a simulated compressed-air plant (3 compressors + pump, MQTT → TimescaleDB, threshold alarms) and harden it layer by layer. Each post runs an attack against the previous stage, adds one layer, reruns it, and states what that layer does *not* protect. A table showing which attacks are blocked grows one column per part. Full plan: `series/00-series-plan-v2.md`. The same plan is also in the claude.ai project "MVP Blog Writer".

**Attacks.** A1 exfiltration via generated code, A2 secret theft via generated code, A3 lateral movement from the API identity, A4 cross-asset data access, A5 injected privileged action (acknowledge alarms), A6 data leaving the boundary to an external model.

**Posts (written, in `_posts/`, not committed, not published):**

| Date (placeholder) | File | Layer | Attacks stopped | Demo tag |
| --- | --- | --- | --- | --- |
| 2026-09-23 | `unsafe-ai-agent-azure-container-apps` | Unsafe agent, threat model, four terms | — | `stage-0-unsafe` |
| 2026-09-30 | `azure-container-apps-sandboxes-ai-generated-code` | Dynamic sessions + Sandboxes, deny-first egress | A1, A2 | `stage-1-sandbox` |
| 2026-10-07 | `azure-container-apps-express-agent-identity-split` | Split into apps, per-app identity, Express for thin API | A3 | `stage-2-express` |
| 2026-10-14 | `mcp-agent-governance-toolkit-azure-container-apps` | MCP tool servers + Agent Governance Toolkit sidecar | A4, A5 | `stage-3-governance` |
| 2026-10-21 | `private-inference-serverless-gpu-azure-container-apps` | Private model on serverless GPU | A6 | `stage-4-private-model` |

- Posts contain 31 `[TODO]` markers, all waiting on the demo: repo/tag links, attack traces and screenshots, the egress policy file, the governance policy file, the model/image, and the measured tables (Express cold start in Part 3, GPU vs managed API in Part 5). Never fill them with invented numbers or code.
- Post dates are placeholders. Shift them to start about one week after Stage 0 runs, spaced by build time per stage.
- Written in the author's voice using the `janeski-blog-article` skill. Microsoft Learn links carry `?wt.mc_id=MVP_395548`.
- Previous drafts: `series/01-05*.md` (original v1), `series/v1-posts/` (v1 posts moved out of `_posts`). `series` is in `_config.yml` `exclude`, so none of this is published.

**Demo (not started).** New repo, planned as `aca-agent-hardening` in `C:\Users\mja\source\repos\`. It will be one Aspire solution plus azd/Bicep, with git tags `stage-0-unsafe` … `stage-4-private-model` (`main` = final). Planned layout: `src/AppHost`, `ServiceDefaults`, `Telemetry.Simulator`, `Agent.Api` (uses `IChatClient` from the start), `Agent.Tools.Alarms`, `Agent.Tools.Docs`, `Indexer.Job`, `Inference`; plus `execution/`, `governance/`, `redteam/` (the six attacks + results table), `infra/`, `data/maintenance-notes/` (includes the poisoned notes for A1 and A5). Part 5 deploys and measures a real serverless GPU model.

**Open decisions:**
- Confirm the repo name and location.
- Plant naming: compressed-air plant vs. fish tank with a pump (continuity with the `iot_with_ai_durable_functions` demo).
- Part 4: the governance sidecar can't run on Express. Current approach: keep two `agent-api` variants (Express ungoverned, standard + sidecar).
- Final series title.

**Next steps:**
1. Scaffold Stage 0 (unsafe agent + red-team scripts). It needs no preview features and fills Part 1's TODOs.
2. In parallel: request serverless GPU quota (Part 5), check Sandboxes preview availability and the current SDK/CLI (Part 2), and check the Agent Governance Toolkit's Container Apps guide against its current release (Part 4).
3. Then build Stages 1 → 4, filling each post's TODOs as the stage is completed.

**Local build note:** run `bundle exec jekyll serve --future` so future-dated posts show.

**Writing style:** new or revised posts follow the `janeski-blog-article` skill. It should live at `.claude/skills/janeski-blog-article/SKILL.md`. If it is not there yet, a copy is at `series/janeski-blog-article-SKILL.md`: read it before writing.
