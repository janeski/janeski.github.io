# Azure Container Apps — whiteboard talk track

One board, 13 chapters. Open `whiteboard.html` in Edge or Chrome, full screen (F11).
**→ / Space / click** = next chapter · **←** = back · **Home** = blank board · **End** = everything.
`png/01.png` … `png/13.png` are the same build-up as images, for slides.

Full version ≈ 20 min. A 6-minute cut for a talk intro is at the end.

---

## 1 · What is Container Apps? (~1 min)
**On the board:** title, "serverless containers", the hidden layer.

- You bring a container image. Azure runs it, scales it, patches it.
- Under the hood it's Kubernetes, KEDA for scaling, Envoy for ingress, Dapr for microservice APIs — but you never see a cluster, a node or a YAML manifest.
- My rule of thumb: no Kubernetes skills and no specific Kubernetes feature needed → Container Apps is probably what you need.

## 2 · The environment (~2 min)
**On the board:** the big blue box and the two workload profiles.

- Everything lives in an **environment**: a secure boundary with its own virtual network. Apps in the same environment share the network, the log destination and the Dapr configuration.
- One environment per "thing that may share": e.g. test and production get separate environments.
- The compute under your apps comes from **workload profiles**:
  - **Consumption** — serverless, scales to zero, pay per second.
  - **Dedicated** — fixed nodes, bigger sizes, GPUs; more predictable cost.
- You can mix both in one environment. The old "consumption-only" environment type is legacy.

## 3 · Container app & ingress (~1.5 min)
**On the board:** users → ingress → the container app.

- A **container app** is a long-running service: an API, a web app, a background worker.
- **Ingress** is built in (Envoy): HTTP or TCP, external (internet) or internal (only inside the environment). No load balancer or gateway to set up.

## 4 · Revisions & replicas (~2.5 min)
**On the board:** revision v1 with three replicas, revision v2 with one, 80/20 split.

- Every change to the image or configuration creates a new, immutable **revision**.
- Each revision runs **replicas** — copies of your containers. A replica is your app container plus optional sidecars and init containers.
- Single revision mode: the new one replaces the old. Multiple revision mode: run both and **split traffic** — blue/green, canary, A/B.

## 5 · Scaling (~1.5 min)
**On the board:** the green "scale 0 → N" note and the arrow next to the replicas.

- Scale rules decide the number of replicas: HTTP traffic, CPU, memory, or **any KEDA scaler** (queues, Kafka, Service Bus…).
- Most apps can scale to **zero** — and at zero replicas there's no compute bill.
- Catch: apps scaling only on CPU or memory can't go to zero.

## 6 · Jobs (~1.5 min)
**On the board:** the orange job box.

- Apps run forever; **jobs** start, do one unit of work, and stop. Each run is an **execution**.
- Three triggers: **manual** (on demand / API), **schedule** (cron, in UTC), **event** (a KEDA scaler, e.g. one execution per queue message — also self-hosted CI runners).
- Jobs have no ingress and no Dapr.

## 7 · App-to-app (~1 min)
**On the board:** App B and the dashed arrow.

- Apps in one environment call each other **by name** (`http://app-b`) — built-in DNS service discovery.
- Internal ingress keeps App B off the internet.
- Dapr adds service invocation, pub/sub, state and more — optional, apps only.

## 8 · Identity, secrets, logs (~1.5 min)
**On the board:** the three labels under the environment.

- **Managed identity**: the app gets an Entra identity and calls Key Vault, Storage or SQL without passwords.
- **Secrets**: stored on the app, or as Key Vault references.
- **Logs**: everything goes to Log Analytics by default; OpenTelemetry, live log stream and a console into the container for debugging.

## 9 · Pricing (~1 min)
**On the board:** the green `$`.

- Consumption: per second of vCPU and GiB, with a free monthly grant. Zero replicas, no compute bill.
- Dedicated: per node instance plus a fixed plan management fee.
- Express (next chapter) uses the consumption meter with no environment fee.

## 10 · Express — the harness (~2 min)
**On the board:** the pink divider and the green Express box.

- **Transition:** "That's Container Apps as we knew it. Now: what does an AI agent need?" → harness, sandbox, state, speed.
- **Express** is the fastest way to get an HTTP app running: an image becomes an HTTPS URL, no environment design.
- It's built on Sandboxes — microVMs from prewarmed pools — which is why it scales from zero in under a second.
- The trade-off: no sidecars, no Dapr, no jobs, no GPU, no custom domains; HTTP only, one revision. Perfect for an agent API or MCP server, not for a microservice mesh.

## 11 · Dynamic sessions — run code once (~2 min)
**On the board:** the session pool with prewarmed boxes.

- The LLM just wrote Python. You don't run it in your API process — you send it to a **session**.
- A **session pool** keeps sessions prewarmed. A request with `?identifier=user-42` goes to that user's session, or gets a fresh one in milliseconds.
- Hyper-V isolation, outbound network off by default, and after an idle cooldown the session is destroyed. Nothing is remembered.
- Code interpreter pools (Python, Node, Shell) need no environment; custom container pools need a workload-profiles environment.

## 12 · Sandboxes — keep the state (~2.5 min)
**On the board:** microVM, egress, lifecycle.

- A **sandbox** is your own microVM: your image as the root filesystem, its own Linux kernel.
- **Egress rules** decide what it can reach — allow or deny by domain or CIDR.
- Lifecycle: create → running → stopped → delete. Stopping takes a **snapshot of memory and disk**; resume brings everything back in under a second.
- Auto-suspend when idle, auto-delete after N days, volumes for persistent data (Blob shared across sandboxes, Disk for one). A sandbox group is the management boundary.

## 13 · Which one? (~1 min)
**On the board:** the decision line at the bottom right.

- Always-on service → **app** · finite task → **job** · fast agent API → **Express**
- LLM code, run once → **session** · agent workspace with memory → **sandbox**
- Close: "Same platform, five shapes of compute — pick by lifecycle and by what has to be remembered."

---

## 6-minute cut (talk intro)
Chapters **1 → 2 → 3 → 4** (one sentence each, ~2 min), then **10 → 11 → 12** (~1 min each), then **13**.
Skip 5–9, or show them with **End** as the full board and say: "the rest of the platform — jobs, scaling, identity, pricing — is all here, but today we focus on the right side."

## Sources (Microsoft Learn)
- [Azure Container Apps overview](https://learn.microsoft.com/en-us/azure/container-apps/overview?wt.mc_id=MVP_395548)
- [Environments](https://learn.microsoft.com/en-us/azure/container-apps/environment?wt.mc_id=MVP_395548)
- [Jobs](https://learn.microsoft.com/en-us/azure/container-apps/jobs?wt.mc_id=MVP_395548)
- [Express overview](https://learn.microsoft.com/en-us/azure/container-apps/express-overview?wt.mc_id=MVP_395548) · [Express FAQ](https://learn.microsoft.com/en-us/azure/container-apps/express-faq?wt.mc_id=MVP_395548)
- [Dynamic sessions](https://learn.microsoft.com/en-us/azure/container-apps/sessions?wt.mc_id=MVP_395548) · [Session pools](https://learn.microsoft.com/en-us/azure/container-apps/session-pool?wt.mc_id=MVP_395548)
- [Sandboxes overview](https://learn.microsoft.com/en-us/azure/container-apps/sandboxes-overview?wt.mc_id=MVP_395548)
