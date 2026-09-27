---
name: "janeski-blog-article"
description: "Write a new technical blog article (janeski.tech / LinkedIn) in Miroslav Janeski's own voice and structure — Azure, .NET, Aspire, cloud-native, IoT, process mining topics."
---

# Writing a blog article in Miroslav's style

Miroslav is a Technical Director, Microsoft MVP (Azure & .NET) and EngD with a process-mining background. His articles are practitioner writing: a working engineer explaining what he built or decided, why, and what he learned. They are published on janeski.tech and as LinkedIn articles.

## 1. Gather inputs first

Before drafting, make sure you have (ask in one short batch if missing; don't ask for what's already given):

- **Topic and the one point the article must land** (the insight, recommendation, or thing the reader can do afterwards)
- **Article type** (see section 3) — infer it if obvious
- **Material**: code, repo link, architecture, numbers, screenshots, talk/meetup context, previous posts to link
- **His opinion** on the topic, if it's a comparison or analysis — he always takes a position

Never invent benchmark numbers, dates, versions, repo URLs or personal anecdotes. Leave a clearly marked `[TODO: ...]` instead. Verify current product names/versions/features with a web search when the article describes the present state of a service (Azure moves fast; e.g. ".NET Aspire" is now just "Aspire").

## 2. Voice

- **First person, direct, calm.** "In this article, I want to show you how…", "I intentionally place…", "My preference is…", "The reason I chose SQLite here is simplicity."
- **Opinionated but grounded.** He states a position and defends it with reasoning, and admits when something is contrarian ("Quite brave and provocative, but…"). He dislikes non-answers: "It depends!" is immediately followed by *what* it depends on.
- **Honest about limits.** Every service or tool gets its weaknesses named. No hype, no marketing adjectives, no "game-changer", "revolutionary", "delve", "unlock the power of".
- **Explains design choices explicitly.** "This is an intentional design choice." Then one short paragraph of why.
- **Defines before using.** Core concepts get a short "What is X?" section before the deep dive.
- **Short punchy lines as transitions** between longer paragraphs: "Twelve years later the infrastructure caught up." / "That's the only dependency." / "It just works."
- **Relatable developer scenarios** to motivate a tool: "You have a feature branch running locally. The product owner wants to click through it…" and a before/after contrast ("The traditional answer is… With X, the answer is…").
- Plain, medium-length sentences. Em dashes (—) are used often for asides and explanations. Bold key terms and the single most important sentence in a section; italics sparingly for emphasis.
- Recurring worldview he can build on: cloud-native = **technology** (Kubernetes / portability), **architecture** (distributed, microservice-like), **organization** (agile, automated DevSecOps) — and since 2025, **AI-assisted development**. Aspire makes "architecture part of the code". Local development must be as simple as production.

## 3. Structures by article type

Pick the one that fits; H2 headings, sometimes numbered ("## 1. Introduction", "### 4.1. …") for longer analytical pieces.

**How-to / integration** (2–4 min read, e.g. "How to Use Azure Dev Tunnels with Aspire")
1. Opening: what the tool does in one sentence + the everyday scenario it solves (bold the scenario sentence)
2. The Setup — package/dependency, then "That's the only dependency."-style remark
3. Core code section — short snippet, then a paragraph explaining the key line
4. One section per notable design decision (e.g. opt-in via env var, `ExcludeFromManifest`) and why
5. Why This Works Well
6. Full Source — context (demo, talk, meetup) + GitHub link

**Comparison / "which should I choose"** (e.g. "ACA vs AKS vs Functions")
1. Opening: answer the question honestly (often "It depends!"), then promise the reasoning
2. What is <concept>? — his definition, numbered aspects with bold labels
3. The alternatives on a spectrum (High / Middle ground / Low complexity) with a diagram
4. Deep dive on the favored option + supporting tech (H3 per technology)
5. The decision: bold conditional scenarios ("**If your organization already…**") and a clear final recommendation ("In most scenarios, I recommend…")
6. Links

**Service analysis** (e.g. "What Makes Azure App Service Cloud-Native — and What Still Doesn't")
1. Hero image + TL;DR paragraph
2. Introduction — what he'll explore and a "before I dive in" framing
3. His definition / stance for the current year
4. The service: overview + timeline of features grouped by era
5. Key features, each as H3 with `**Why it matters:**` paragraph and a closing `→ one-line consequence`
6. Typical use cases (H3 each: one-line description, Tech stack:, Examples:, what the service provides)
7. Limitations (H3 each: sentence, 2–3 bullets, `**Impact:**` line)
8. Useful Links (numbered, with "— Book" / "— Microsoft Definition" labels)

**Deep technical build** (8–12 min, e.g. process mining + OpenTelemetry)
1. Personal hook where genuine (history, thesis, a talk) — only if the user supplied it
2. Concept explainer (What is X?) with bold sub-concepts
3. "The insight" — the non-obvious connection, stated in one or two crisp sentences, then "In this article, I want to show you how…"
4. The system/workflow (numbered steps)
5. Implementation sections with code, tables for data requirements, screenshots with descriptive alt text
6. What it reveals / results (bold lead-ins per finding, tables of numbers)
7. Takeaways — each a **bold thesis sentence** followed by a paragraph of reasoning
8. "View full code on GitHub" link

**Short starter / PoC post** (1–2 min): problem paragraph → "This article presents…" → Architecture (components in bold, numbered final list) → why the tool helps → Takeaways as numbered `**Bold claim** — explanation` → GitHub link.

## 4. Recurring conventions

- Takeaways lists: `1. **Aspire accelerates PoC delivery** — It enables teams to…`
- Cross-link earlier posts naturally: "In my previous article…", "In my previous post on…, I covered…"
- Microsoft Learn links carry his MVP tracking suffix: `?wt.mc_id=MVP_395548` (or `&wt.mc_id=MVP_395548` if the URL already has a query string).
- Code: short, focused C# (Aspire AppHost, ServiceDefaults, Functions), shell one-liners, csproj `PackageReference`. Use `// ... additional configuration` to trim. Always follow code with prose explaining the one line that matters.
- Tables for data requirements, comparisons, and measured numbers.
- Images: hero photo (Pexels, credited as `Photo by <name> on Pexels`) or an Aspire Dashboard / architecture screenshot. Use `[image: description]` placeholders for images the user must supply.
- Ends with the GitHub repo or a links list — no generic "Thanks for reading" or call-to-action fluff.

## 5. Output

Deliver one Markdown file named after the slug (e.g. `azure-dev-tunnels-aspire.md`) with front matter:

```
---
title: "<Title>"
description: "<one sentence, often with an em dash, ~150-160 chars>"
tags: [dotnet, aspire, azure, cloud-native, ...]   # lowercase, hyphenated, 6–15 tags
---
```

Title patterns he uses: "How to Use X with Y", "X vs Y vs Z: The Best <thing> for .NET Developers", "What Makes X <quality> — and What Still Doesn't", "<Verb>ing Your X with Y and Z", "<Adjective> and <Adjective>: Building X with Y", "From A to B: Getting X to Y with Z".

After the article, offer (one line) a short LinkedIn post to announce it: 3–5 lines, first-person, the core insight, link placeholder, 3–5 hashtags.

## 6. Self-check before delivering

- Does the opening make the problem or question concrete within the first two paragraphs?
- Is there a clear position/recommendation, with reasons and limitations?
- Is every design choice explained with a "why"?
- Are all facts, numbers, versions and links either supplied by the user, verified, or marked `[TODO]`?
- Any hype words or filler conclusions? Remove them.
- Does it end with Takeaways and/or the code/links section?