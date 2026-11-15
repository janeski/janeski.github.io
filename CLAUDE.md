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

_Status as of 21 Oct 2026._

**Concept.** Five posts based on the talk abstract: take one deliberately unsafe AI incident assistant for a simulated biogas plant (digesters D-1…D-3 + feed pump P-1, D-9 at site B; MQTT → NATS JetStream → Postgres/pgvector, threshold alarms) and harden it layer by layer. (The plan in `series/00-series-plan-v2.md` still says compressed-air plant / TimescaleDB; the posts follow the demo.) Each post runs an attack against the previous stage, adds one layer, reruns it, and states what that layer does *not* protect. A table showing which attacks are blocked grows one column per part. Full plan: `series/00-series-plan-v2.md`. The same plan is also in the claude.ai project "MVP Blog Writer".

**Attacks.** A1 exfiltration via generated code, A2 secret theft via generated code, A3 lateral movement from the API identity, A4 cross-asset data access, A5 injected privileged action (acknowledge alarms), A6 data leaving the boundary to an external model.

**Posts (written, in `_posts/`, committed locally in 26c7f95 but not pushed, not published):**

| Date (placeholder) | File | Layer | Attacks stopped | Demo tag |
| --- | --- | --- | --- | --- |
| 2026-12-01 | `unsafe-ai-agent-azure-container-apps` | Unsafe agent, threat model, four terms | — | `stage-0-unsafe` |
| 2026-12-04 | `azure-container-apps-sandboxes-ai-generated-code` | Dynamic sessions + Sandboxes, deny-first egress | A1, A2 | `stage-1-sandbox` |
| 2026-12-08 | `azure-container-apps-express-agent-identity-split` | Split into apps, per-app identity, Express for thin API | A3 | `stage-2-express` |
| 2026-12-11 | `mcp-agent-governance-toolkit-azure-container-apps` | MCP tool servers + Agent Governance Toolkit sidecar | A4, A5 | `stage-3-governance` |
| 2026-12-15 | `private-inference-serverless-gpu-azure-container-apps` | Private model on serverless GPU | A6 | `stage-4-private-model` |

- Posts contain 45 `[TODO]` markers (3 / 5 / 17 / 5 / 15), all waiting on the demo: repo/tag links, attack traces and screenshots, the egress policy file, the governance policy file, the model/image, and the measured tables (Express cold start in Part 3, GPU vs managed API in Part 5). Never fill them with invented numbers or code.
- Post dates: twice a week (Tue/Fri), 1–15 Dec 2026 (reset 15 Nov 2026; first set was weekly Mondays from 26 Oct). Shift them again if a stage's Azure evidence is not ready in time.
- Written in the author's voice using the `janeski-blog-article` skill. Microsoft Learn links carry `?wt.mc_id=MVP_395548`.
- Design follows `2026-10-21-azure-container-apps-agentic-era`: `post_class: post--vivid`, `{: .callout}` / `{: .why}` / `{: .impact}` blocks, numbered H2s. Each part has a cover `images/posts/aca-agent-hardening-partN-cover.png` (1920×1080, rendered with headless Edge from an HTML card) and a diagram `images/posts/aca-agent-hardening-stageN.svg`.
- Previous drafts: `series/01-05*.md` (original v1), `series/v1-posts/` (v1 posts moved out of `_posts`). `series` is in `_config.yml` `exclude`, so none of this is published.

**Demo.** `C:\Users\mja\source\repos\aca-agent-in-production` (no git remote yet). Stages 0 and 1 are tagged (`stage-0-unsafe`, `stage-1-sandbox`); Stage 2 is WIP on `feature/stage-2-express`; Stages 3–4 are not built. Only local runs so far — no stage has been deployed to Azure, so no TODO can be filled yet. Its `docs/results/` is the only source for TODOs.

**Open decisions:**
- Repo name (`aca-agent-hardening` in the plan vs `aca-agent-in-production` on disk) and publishing it on GitHub.
- Final series title.

**Next steps:**
1. Publish the demo repo, then deploy Stages 0 and 1 to Azure and fill Part 1 and Part 2 TODOs from `docs/results/`.
2. Request serverless GPU quota (Part 5); check the Agent Governance Toolkit's Container Apps guide against its current release (Part 4).
3. Finish Stage 2 (cold-start measurement), then build Stages 3 → 4, filling each post's TODOs as the stage is completed.

**Local build note:** run `bundle exec jekyll serve --future` so future-dated posts show.

**Writing style:** new or revised posts follow the `janeski-blog-article` skill. It should live at `.claude/skills/janeski-blog-article/SKILL.md`. If it is not there yet, a copy is at `series/janeski-blog-article-SKILL.md`: read it before writing.
