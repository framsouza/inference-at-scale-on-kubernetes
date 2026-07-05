# Zero-Downtime Strategy for Node Maintenance running AI Inference on Kubernetes

Serving a large language model on Kubernetes and then trying to upgrade the node underneath it is where a lot of otherwise-solid platform teams get bitten. The model is huge, it spans every GPU on the node, in-flight requests run for a long time, and the KV cache lives in GPU memory that vanishes the moment the pod dies. None of the reflexes from stateless web serving apply cleanly.

This post walks through how to do a **voluntary node maintenance** (a Kubernetes or driver upgrade) on a single GPU node (4 x H100) hosting an LLM, without dropping requests on the floor. It has two halves that matter equally:

1. **Getting the new (green) environment genuinely ready to serve**: The bootstrap automation and the validation gates that must all be green *before* it takes a single real request. This is the part people under-invest in, and it's where most "we did a blue-green and it still broke" stories actually originate.
2. **The cut**: How you shift traffic off the old environment onto the new one reliably, and exactly what you configure on the gateway to make that safe.

Skip or hand-wave the first half and the second half can't save you: a clean cut onto a node that quietly failed a GPU diagnostic or can't reach Vault just moves the outage, it doesn't prevent it.

There's one principle worth stating up front, because everything else hangs off it:

> **Availability comes from having a second replica. Blue-green, PodDisruptionBudgets, KV offload, and graceful drain don't *create* availability, they protect and warm the failover already provisioned.**

If you internalize that (please do it, it'll make your life easier), most of the design decisions below become obvious.

## The scenario

- **Model:** Llama 3 70B, quantized to FP8 (~70 GB of weights).
- **Hardware:** one node with 4× H100 (320 GB HBM total).
- **Parallelism:** the model doesn't fit on one GPU, so it's split across all four with **tensor parallelism (TP=4)**. The remaining HBM after weights holds activations (roughly ~10% of model size) and the KV cache. Because it's one replica spread across four GPUs, **if any one GPU goes, the whole replica is down.** One replica = one vLLM pod = four GPUs = one node.
- **Platform:** GKE.

Critically: this is **N=1**. There is exactly one serving replica, and there is no spare GPU capacity sitting idle. That single fact dictates the entire strategy.

## First, kill a myth: the control plane is not your problem

On GKE the control plane is fully managed. You don't run or scale master nodes — you choose **regional** (HA control plane replicated across zones) versus **zonal**, and that's the extent of it. More importantly:

**A control-plane upgrade does not take your inference down.** Your pods keep running and keep serving while the API server cycles. The downtime you're actually defending against comes entirely from the **node pool** upgrade, the moment the node holding your only replica gets drained.

## What's actually running

A realistic cluster hosting this model has, at minimum:

- **vLLM** (or llm-d, which wraps vLLM) — the inference server itself.
- **The gateway + Inference Extension** — routing (more on the terminology below).
- **Prometheus** — metrics.
- **External Secrets Operator (ESO)** — pulling secrets from Vault or a cloud secret store.
- **A policy engine like Kyverno** — enforcing that pods declare GPU resources, that images are pinned, and any vLLM-specific admission rules.

All of these need to exist and be healthy on the *new* node before it takes traffic. That's a big part of the pre-work.

## Terminology: gateway vs. Endpoint Picker vs. load balancer

People (myself included, early on) collapse three different things into one word. Separating them is what makes the cut design click:

- **Gateway API Inference Extension (GAIE)** — the *specification* that makes routing inference-aware (KV-cache-aware, load-aware) instead of naive round-robin. Its runtime component is the **Endpoint Picker (EPP)**, a Deployment you run. Run it with **≥2 replicas**.
- **The gateway proxy** — the actual data-plane Deployment (kgateway, agentgateway, or Istio) that terminates traffic and consults the EPP. Also run **≥2 replicas**. (Note: kgateway is being deprecated in llm-d in favor of agentgateway, so pick your gateway with the current direction in mind.)
- **The load balancer** — the GCLB that the `Gateway` resource provisions in front of the proxy.

When you read "run at least two of the gateway," it means: two gateway-proxy pods *and* two EPP pods, so neither is a single point of failure during the maintenance itself.

## Why N=1 forces a temporary second node

It is not that a Kubernetes upgrade "can't be done in place." It's that **you have one replica and nothing to fail over to.** Drain that node and you have zero serving capacity until a replacement finishes loading 70 GB of weights and building CUDA graphs, minutes of hard downtime.

So you need a **temporary second node** for the duration of the maintenance window. That's the whole reason blue-green enters the picture. Two things follow from this:

1. **The GPU double-spend is unavoidable.** For the window, you're paying for 8 GPUs (blue's 4 + green's 4). No strategy removes this; availability requires the second replica to physically exist somewhere.
2. **If you had 2+ nodes permanently, you wouldn't hand-roll blue-green at all**, you'd use a GKE **surge upgrade** (`maxSurge=1, maxUnavailable=0`), which brings up a new-version node *before* draining an old one. GKE's surge upgrade is itself a node-level blue-green; you only hand-roll the orchestration when you want your own validation gates (DCGM diagnostics, load tests) to control the cutover.

## The architecture that makes the cut clean: one gateway, one pool, two node pools

Here is the single most important design correction, and it's the thing that makes the cut trivial instead of painful:

**Do not build two separate gateways/load balancers joined by a DNS switch.**

DNS is the worst possible cut mechanism for this. Clients cache records, resolvers ignore your low TTLs, and long-lived connections pin to old IPs. You lose control over *when* blue actually stops receiving traffic, which is exactly the guarantee a clean drain depends on.

Instead:

- **One cluster, one gateway, one load balancer, one DNS name.**
- **Blue and green are two node pools in the same cluster.**
- **Blue and green vLLM pods are endpoints in the same routing layer** (either the same `InferencePool`, or two pools behind one weighted `HTTPRoute`, both shown below).

Now the cut is an **endpoint / weight operation inside the cluster**  instant, reversible, and with zero client-side caching in the path. DNS never moves.

(Two load balancers plus a DNS flip is *cluster* blue-green, two entirely separate clusters. That's a heavier pattern reserved for changes you can't make inside one cluster. A node pool upgrade does not need it.)

## Part 1 — Getting green ready to serve

The cut gets all the attention, but this is the half that actually determines whether your maintenance is safe. **A blue-green only protects you if green is genuinely ready**, not "the pods are Running," but "the GPUs are proven clean, every dependency is wired, and the model is serving at the same latency and quality as before the upgrade." A flawless cut onto a green node that quietly lost a GPU to an ECC fault, or whose ESO can't reach Vault, just relocates the outage.

None of this happens live during the maintenance window. It's all automated, run ahead of time, and gated. Think of green's promotion to "eligible for traffic" as a series of stages, each of which must pass before the next runs, and any failure pages the on-call SRE and halts the workflow.

### Stage 0 — Bootstrap the new node pool (automated, idempotent, version-pinned)

Automate the entire stand-up end to end so it's reproducible and reviewable, not a sequence of `kubectl` commands someone runs by hand at 2 a.m.:

- **Argo Workflows** orchestrates the run and sequences the stages below, halting on any failed gate.
- **Terraform or Crossplane** provisions the new node pool on the target Kubernetes version, with the right machine type, GPU count, node labels, and taints. Provision it **tainted** (e.g. `node.company.io/unvalidated=true:NoSchedule`) so nothing lands on it until it's proven healthy.
- **The NVIDIA GPU Operator** installs and pins the target GPU driver, container toolkit, and DCGM exporter as part of node config.
- **Helm** installs the full workload stack onto green: vLLM, the EPP, gateway config, the Prometheus scrape config / ServiceMonitor, ESO, and Kyverno policies. Everything version-pinned so green is a known artifact, not a moving target.

The output of this stage is a node that *exists* on the new version, not one that's trusted yet.

### Stage 1 — Hardware validation (prove the GPUs before anything else)

New GPUs fail. Before green is allowed to load a model, prove the hardware:

- **`dcgmi diag -r 3`** (or the appropriate level) — the deep run that catches ECC errors, thermal problems, PCIe replay/link issues, and compute faults, not just "the GPU is present."
- **NVLink / NVSwitch topology check** — for TP=4 the GPUs talk to each other constantly; a degraded NVLink lane silently tanks throughput.
- **GPU count and MIG mode** match what the model expects (4 full GPUs, MIG off for this deployment).

Only when the hardware passes do you **remove the `unvalidated` taint**, letting the model workload actually schedule onto the node. Taint-until-proven is what stops a bad GPU from ever entering rotation.

### Stage 2 — Platform and dependency validation (the plumbing around the model)

The model doesn't serve in a vacuum. Confirm each dependency on green specifically, not "it works on blue so it'll work here", make sure all the tests are automated without human intervation:

- **ESO ↔ Vault** — External Secrets Operator can authenticate and *materialize* the secrets the model needs: model-registry / Hugging Face pull credentials, TLS certs, any API keys. A green pod that can't pull weights because ESO silently failed is a classic cutover surprise.
- **Kyverno enforcing** — the admission policies are present and active on green, so it isn't running unguarded (GPU-resource requirements, image pinning, your vLLM-specific rules). You don't want green to be the one environment where policy quietly wasn't applied.
- **Weights are local and warm** — the model image is **pre-pulled** and weights are on fast local NVMe or a warmed cache, not being pulled over the network at cutover. This is also your biggest lever against the warm-up capacity gap: a 70 GB cold pull is minutes you don't want to spend during the window.
- **KV offload backend reachable** — if you're using a shared prefix-cache tier, green can connect to it (with the storage-choice caveats from the offload section below).

### Stage 3 — Observability validation (you can't cut toward what you can't see)

You are about to send real users to green and watch its SLOs. That's only possible if telemetry is live *before* the cut:

- **Prometheus is scraping** green's vLLM metrics *and* the DCGM exporter — TTFT, inter-token latency, queue depth, KV-cache utilization, GPU SM/mem utilization.
- **Logs are shipping** from green's pods to your aggregation backend.
- **Dashboards and alerts are populated for green** and firing correctly on synthetic conditions, so during the canary you're reading real signal, not a blank panel.

### Stage 4 — Model and inference validation (the promotion gate)

This is the gate that actually earns green its traffic. "vLLM responds" is not enough, an upgrade can subtly regress performance or a re-quantization can degrade quality, and both must be caught here, not by users:

- **vLLM health and readiness passing** — `/health` green, readiness probe green, weights loaded, **CUDA graphs built** (the model is warm, not cold-starting on first request).
- **Correctness / quality check** — run a **golden-prompt set** through green and compare outputs against a known-good baseline (exact-match or a scored eval). This is what catches a bad quantization or a silently different model build after the upgrade. Responding is not the same as responding *correctly*.
- **Performance check** — a **synthetic load test** at your target concurrency, asserting that TTFT, inter-token latency, and throughput are within tolerance of blue's baseline. Regression here means you do **not** promote, green may be functional but slower, and cutting to it would breach SLO.

Implement this as an explicit gate. A minimal shape in the Argo run:

- run `dcgmi diag` → gate
- run the golden-prompt eval → gate
- run the load test, assert p95 TTFT / ITL / throughput thresholds → gate
- **all green → mark green eligible for traffic; any red → page SRE, halt, leave blue untouched.**

You can also enforce this at the Kubernetes level with a **readiness gate** on the green pods so they only report Ready to the EPP once your validation controller sets the corresponding condition, meaning even a misfired cut can't route to an unvalidated pod.

### Why this half is the point

Every item above exists to make one sentence true at cut time: *green is a full, proven, observable replacement for blue, and I can watch it while I shift traffic.* Get there and the cut is a two-line weight change. Skip it and the cut is a loaded gun. **Page on any failed gate and never promote on a red, these are hard preconditions, not a checklist to skim.**

## KV cache offload: what it does, and what it absolutely does not do

This is the concept most write-ups get wrong, so let's be precise.

**What offload does:** it *tiers* the KV cache for a running instance HBM → CPU RAM → local disk → shared storage so you can hold more cached context than fits in HBM, and, crucially, **reuse prefixes**. If you point the offload/lookup tier (LMCache-style) at a backend both node pools can read, then a request arriving on green whose prompt shares a prefix with something blue already computed can **load those KV blocks instead of recomputing prefill**. That cuts time-to-first-token and saves prefill compute on green. It's important to monitor if the time to load the KV cache from network storage pays the prefill re-computation.

**What offload does not do:** it does **not** migrate a live, mid-generation request from blue to green. There is no "pause on blue, resume on green" for an in-flight decode. A sequence actively generating on blue must either **finish on blue** or get **cut and retried** by the client (landing on green, warm from the shared tier).

So the correct mental model for the cut is:

> **In-flight users are protected by graceful drain on blue, not by KV offload. Offload only helps *new or retried* requests warm up faster on green.**

Keep those two ideas in separate paragraphs whenever you explain this. Conflating "offload" with "live migration" is the single most common error in this topic.

Two practical caveats on storage choice:

- **NFS is usually too slow to be a hot KV tier.** KV blocks are large and latency-sensitive; an NFS round-trip can rival or exceed just recomputing prefill. Putting network storage in the hot path can make things *worse*, not better.
- **S3 is a cold bottom tier at best** good for capacity and longer-horizon cross-node prefix sharing, too high-latency for the request hot path.

If you want genuinely fast cross-node prefix reuse, use a purpose-built shared cache backend, not generic NFS/S3 in the critical path. And be honest in the post: for single-shot, low-prefix-reuse traffic, cross-node offload barely helps, its value is highest for **multi-turn chat**, where conversations carry long shared prefixes.

## Part 2 — The cut

With green validated and eligible for traffic (Part 1), the cut itself is small. With the single-gateway design, **you do it at the gateway/EPP layer you do not need any additional service.** The mechanism is: *stop sending new requests to blue, let blue finish what it's already running, then terminate blue.*

The ordering guarantee you must enforce:

**green Ready and in the pool → traffic shifts to green → blue goes unready (new requests stop) → blue drains in-flight → grace period expires → blue terminates → delete blue node pool.**

Green never has a capacity gap, and the only requests at risk are the longest-tail generations that exceed your drain window (which get cut and retried onto green).

### Step 1 — Green is validated and eligible *first*

This is the payoff of Part 1: green has passed every gate hardware, dependencies, observability, and the model/performance promotion gate and is now eligible for traffic. It joins the routing layer. At this moment both blue and green are Ready, so you have momentary 2× capacity. This is the only safe state to cut from; if green isn't fully green, you don't start.

### Step 2 — Shift new requests to green

Two mechanisms. The readiness-flip is the most robust; the weighted route adds a real-traffic canary on top.

**Mechanism A — readiness flip (core, rock-solid).** Blue and green pods live in the **same `InferencePool`**, selected by label. When blue's pods flip to *unready*, the EPP stops selecting them as endpoints, instantly. New requests can only land on green. Blue stays **Running but unready** so it can keep finishing in-flight work. You trigger the flip from a preStop drain file (Step 3).

**Mechanism B — weighted HTTPRoute (adds canary control).** Run blue and green as **two `InferencePool`s** behind one `HTTPRoute`, and shift the `backendRef` weights. This lets you send, say, 10% of live traffic to green first, watch SLOs, then go to 100%:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: llama-route
spec:
  parentRefs:
    - name: inference-gateway        
  rules:
    - backendRefs:
        - name: llama-pool-blue
          group: inference.networking.k8s.io
          kind: InferencePool
          weight: 100               
        - name: llama-pool-green
          group: inference.networking.k8s.io
          kind: InferencePool
          weight: 0
```

Canary, then cut, by editing the weights: `100/0` → `90/10` → `0/100`. Because this is a gateway-level weight change, it takes effect immediately and is instantly reversible, the exact properties DNS lacks.

> Field names for the GAIE CRDs (`InferencePool`, group/kind) are evolving, verify against current GAIE/llm-d docs for your version. The Gateway API core (`HTTPRoute`, weights) and the pod-lifecycle pieces below are stable.

### Step 3 — Keep blue Running while it drains

This is the piece you were circling: **marking blue unready is not the same as killing it.** The pod stays Running and finishes its in-flight generations. You implement the drain with three coordinated settings on the vLLM pod:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llama-vllm-blue
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: llama-vllm
        pool: blue
    spec:
      # Longer than: preStop sleep + longest expected generation.
      # SIGTERM to vLLM only fires after this expires, so YOU control the drain window.
      terminationGracePeriodSeconds: 600
      containers:
        - name: vllm
          # Readiness fails the moment the drain file exists,
          # so the EPP removes this pod from selectable endpoints.
          readinessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - "! test -f /tmp/drain && curl -sf http://localhost:8000/health"
            periodSeconds: 5
          lifecycle:
            preStop:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - |
                    # 1. Deregister from routing: fail readiness -> EPP stops sending new work.
                    touch /tmp/drain
                    # 2. Wait for the EPP to observe unready AND for in-flight decode to finish.
                    #    Keep this below terminationGracePeriodSeconds.
                    sleep 550
```

How these interact during the cut:

1. You initiate deletion of the blue pod (or scale blue to zero). Kubernetes runs the **preStop hook first**, before sending SIGTERM.
2. The hook `touch`es `/tmp/drain`. Within one probe interval the **readiness probe fails**, the EPP drops blue from the pool, and **no new requests route to blue.**
3. The hook then `sleep`s, holding the pod alive while its existing generations complete. vLLM has not received SIGTERM yet.
4. When the preStop sleep ends (or `terminationGracePeriodSeconds` expires, whichever comes first), Kubernetes sends **SIGTERM**, vLLM shuts down, and the pod terminates.
5. Any generation still running past the grace period is cut — the client retries and lands on green, warm from the shared prefix tier.

Set `terminationGracePeriodSeconds` above `preStop sleep + your longest expected generation`. That single number is your explicit, controllable drain window.

### Step 4 — Protect the whole thing with a PodDisruptionBudget

For a single-node replica (one pod = one replica), a simple pod-count PDB is correct and is your safety net against a node drain evicting the wrong thing at the wrong time:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: llama-pdb
spec:
  minAvailable: 1          
  selector:
    matchLabels:
      app: llama-vllm       # matches both blue and green pods
```

With blue and green both labeled `app: llama-vllm`, `minAvailable: 1` guarantees a voluntary disruption can never leave you with zero Ready replicas. 

### Step 5 — Tear down blue

Once blue's pod has terminated and green is carrying 100% of traffic within SLO, drain and delete the blue node pool. You're back to N=1 on the new version, and you release the second node's GPUs.

## Rollback

Because the cut is a gateway weight (or a readiness flip) and DNS never moved, **rollback is symmetric and instant.** If green's SLOs degrade during the canary or right after full cut, shift the `HTTPRoute` weights back toward blue, blue is still Running until you explicitly tear it down. Never delete blue until green has held SLO for a defined soak period. That soak-before-teardown is what turns "we upgraded" into "we upgraded and can prove it's healthy."

## The whole thing in one breath

1. Control plane upgrades for managed Kubernetes don't touch serving ignore them, focus on the node pool.
2. You have one replica, so you need a temporary second node; that's what blue-green buys you, and the GPU double-spend is the price.
3. Keep **one gateway, one LB, one DNS name, one cluster**  blue and green are just two node pools and two sets of endpoints. Never cut with DNS.
4. **Half the work is getting green ready before you touch traffic.** Bootstrap it automated and version-pinned (Argo + Terraform/Crossplane + GPU Operator + Helm), then gate promotion through staged validation: prove the GPUs (`dcgmi diag`, NVLink) with the node tainted until it passes; confirm dependencies (ESO↔Vault, Kyverno enforcing, weights pre-pulled, offload backend reachable); confirm observability is live; and clear the model gate golden-prompt correctness *and* a load test asserting no latency/throughput regression. Any red gate pages SRE and halts; green never gets traffic on a red.
5. KV offload to shared storage warms green's prefills, it does **not** migrate live requests. In-flight users are saved by graceful drain, not by offload. Don't put NFS in the hot path.
6. The cut is: **green eligible → shift traffic (weighted route or readiness flip) → blue unready so new requests stop → blue drains in-flight via preStop + terminationGracePeriodSeconds → blue terminates → delete blue pool.** All of it at the gateway/EPP layer, no extra service required.
7. A PDB is your safety net; rollback is a weight change because DNS never moved.

Oh, and the line to remember: **availability is a replication property, not a clever-trick property.** Every mechanism here protects or warms the failover none of them conjures it. If you ever find yourself explaining how to get zero downtime with a single replica, the honest answer is that you can't, and the real deliverable is provisioning the second replica that makes it possible.

Happy maintenance!