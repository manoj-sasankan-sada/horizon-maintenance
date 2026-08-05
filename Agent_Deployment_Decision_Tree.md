# Agent Deployment Decision Tree
## GEAP Agent Runtime vs. Cloud Run vs. GKE


## Decision Dimensions (10 total)

### Dimension 1 — Agent Nature: Chat vs. API vs. Event-Driven/Batch

#### Interactive Chat Agent

| Platform | Fit | Why / Why Not |
|----------|-----|---------------|
| **Agent Runtime** | ✅ **Best fit** | Purpose-built. Native sessions + Memory Bank. Built-in Playground. Managed OAuth 2.0. Bidi streaming. Agent Gateway governance. Agent Identity (SPIFFE). |
| Cloud Run | ⚠️ Compatible | Must wire own sessions, UI, OAuth. Can consume Sessions/Memory Bank as client API calls. More ops overhead. |
| GKE | ⚠️ Compatible | Same gaps as Cloud Run + cluster management. Only for custom networking, sidecars, or multi-GPU inference. |

#### Public/Internal API Agent

| Platform | Fit | Why / Why Not |
|----------|-----|---------------|
| **Cloud Run** | ✅ **Best fit** | Native HTTP endpoint, custom domain, IAP, `--allow-unauthenticated`, revision traffic splitting, `gcloud` CLI. |
| Agent Runtime | ⚠️ Compatible | Vertex AI API surface only. No custom domain. No `gcloud` CLI. Awkward for external consumers. |
| GKE | ⚠️ Compatible | Full ingress control but overkill unless K8s-shaped requirements exist. |

#### Event-Driven / Batch Agent

| Platform | Fit | Why / Why Not |
|----------|-----|---------------|
| **Cloud Run** | ✅ **Best fit** | First-class Pub/Sub, Eventarc, Cloud Scheduler. ADK `trigger_sources` pattern. Concurrency control, retry, DLQ. Google's recommended platform for ambient agents. |
| Agent Runtime | ⚠️ Limited | `/api` passthrough exists but indirect. Older `agents-cli` v0.5.0: *"does not support Pub/Sub, Eventarc, or Cloud Scheduler triggers."* |
| GKE | ⚠️ Compatible | K8s Jobs, CronJobs. Good for complex orchestration (DaemonSets, MPI). More setup. |

---

### Dimension 2 — Framework & Language

| Question | Agent Runtime | Cloud Run | GKE |
|----------|--------------|-----------|-----|
| Python + ADK | ✅ Full integration | ✅ Full | ✅ Full |
| Python + other framework | ✅ SDK integration | ✅ Full | ✅ Full |
| Non-Python | ⚠️ BYOC — implement [runtime contract](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/runtime-contract). Less battle-tested. | ✅ **Recommended** | ✅ Full |

---

### Dimension 3 — Operational Complexity

| Tolerance | Recommendation | Why not the others |
|-----------|---------------|-------------------|
| **Zero infra** | → **Agent Runtime** | Cloud Run needs Dockerfile. GKE needs cluster. |
| **Containers OK** | → **Cloud Run** | Agent Runtime too opinionated. GKE unnecessary. |
| **K8s expertise + K8s problems** | → **GKE** *(prerequisite)* | Don't use GKE just because you can. |

---

### Dimension 4 — Public API Exposure

| Requirement | Best Fit | Why not the others |
|-------------|----------|-------------------|
| Standard public REST API + custom domain | → **Cloud Run** | Agent Runtime: no public URL, no custom domain. |
| Internal service-to-service | → **Agent Runtime** or **Cloud Run** | GKE adds complexity. |
| OAuth 2.0 user consent flows | → **Agent Runtime** | Cloud Run/GKE: no managed OAuth. |

---

### Dimension 5 — Scaling & Cost

| Pattern | Best Fit | Why not the others |
|---------|----------|-------------------|
| Variable/spiky, cost-sensitive | → **Cloud Run** or **Agent Runtime** | Both scale to zero. GKE has always-on costs. |
| GPU-accelerated inference | → **Cloud Run** or **GKE** | Cloud Run: NVIDIA L4 (GA), RTX PRO 6000 Blackwell (~5s cold start). GKE: broader GPU selection, multi-GPU. Agent Runtime: no GPU. |
| Very high scale (10K+ QPS) | → **GKE** | Agent Runtime caps at 1000 instances (100 w/ VPC-SC). |

#### Cost Shape Comparison *(imported from PDF)*

| Model | Agent Runtime | Cloud Run | GKE |
|-------|--------------|-----------|-----|
| **Billing type** | Compute-based (vCPU-hr + GiB-hr) | Request-based (vCPU-sec) | Cluster-based (mgmt fee + node costs) |
| **Compute rate** | $0.085/vCPU-hr | ~$0.0000240/vCPU-sec | $0.10/hr mgmt + Compute Engine |
| **Free tier** | 50 vCPU-hr + 100 GiB-hr/mo | 180K vCPU-sec + 2M requests/mo | $74.40/mo cluster credit |
| **Scale to zero** | ✅ (`min_instances=0`) | ✅ (default) | ⚠️ Autopilot only |
| **CUDs** | N/A | ~17–46% | ~46% (3yr); Spot 60–91% off |
| **Idle billing** | Not billed between turns | Not billed at zero | Cluster fee always on |

> **⚠️ Billing deadline:** Sessions and Memory Bank switch from free to billed on **September 1, 2026** — $0.30/GiB-month storage, $0.085 per 3M reads, $0.085 per 1M writes. Factor this into cost models built before that date.

> **Crossover (directional):** Cloud Run is materially cheapest for low-traffic pilots (true zero idle). At sustained high throughput, GKE wins on bin-packing + commitments. Crossover figures (~1M req/day for Cloud Run, ~5M for GKE) come from practitioner blogs, not Google benchmarks.

---

### Dimension 6 — Security & Compliance

| Requirement | Best Fit | Why not the others |
|-------------|----------|-------------------|
| VPC-SC, CMEK, HIPAA, Data Residency, Access Transparency | → **Agent Runtime** | Managed natively. Cloud Run: more manual. GKE: DIY. |
| Binary Authorization | → **Cloud Run** or **GKE** | Both support it. Agent Runtime does not. |
| Pod-level network policies, zero-trust | → **GKE** | K8s-native controls. Cloud Run/Agent Runtime lack pod-level policies. |

---

### Dimension 7 — State Management

| Requirement | Best Fit | Why not the others |
|-------------|----------|-------------------|
| Managed sessions + Memory Bank, zero config | → **Agent Runtime** | Built-in. Cloud Run/GKE: external client config. |
| Custom session backend | → **Any** | ADK session service is pluggable everywhere. |
| StatefulSets, persistent local storage | → **GKE** | Agent Runtime/Cloud Run: stateless by design. |

---

### Dimension 8 — Rollback & CI/CD

| Requirement | Best Fit | Why not the others |
|-------------|----------|-------------------|
| Revision-based traffic splitting, canary | → **Cloud Run** or **Agent Runtime** | Cloud Run: native, GA. Agent Runtime: immutable revisions + percentage-based traffic splitting (Preview, `v1beta1`, May 2026). **Caveats:** `v1beta1` API; incompatible with Agent Gateway; `min_instances` must accommodate all revisions. |
| GitOps with ArgoCD/Flux | → **GKE** | Native K8s GitOps ecosystem. |
| Simplest deploy | → **Agent Runtime** | `agents-cli deploy` one-liner. |

---

### Dimension 9 — Observability

| Requirement | Best Fit | Why not the others |
|-------------|----------|-------------------|
| Built-in tracing + logging, zero config | → **Agent Runtime** | `GOOGLE_CLOUD_AGENT_ENGINE_ENABLE_TELEMETRY=true` → Traces tab with DAG visualization. Feedback Service. | Cloud Run/GKE: manual OTel setup. |
| Prompt-response logging to BigQuery | → **Any** | ADK-level feature via `agents-cli` observability skill. |
| Third-party observability | → **Cloud Run** or **GKE** | Full OTel Collector control. Agent Runtime: limited custom exporters. |
| Custom business logic spans | → **Any** | ADK `@do_trace` + OTel work everywhere. |

> **Important:** Developers must use `@do_trace` decorators on custom functions to fully leverage DAG visualizations. Auto-instrumentation covers LLM/tool calls only.

---

### Dimension 10 — Evaluation

| Requirement | Best Fit | Why not the others |
|-------------|----------|-------------------|
| Gen AI Evals (managed rubric metrics) | → **Agent Runtime** | Native `run_inference(agent=...)`. Supports `FINAL_RESPONSE_QUALITY`, `TOOL_USE_QUALITY`, `HALLUCINATION`, `SAFETY`. |
| ADK local eval (`adk eval`) | → **Any** | Pre-deployment, target-agnostic. |
| CI/CD eval gates (pytest) | → **Any** | `AgentEvaluator.evaluate()` works everywhere. |
| Continuous production eval loop | → **Agent Runtime** | Feedback Service + Example Store + Evals = closed-loop flywheel. |

---

## Agent Gateway & Agent Identity *(imported from PDF)*

> **This is the decisive governance argument for Agent Runtime.** These primitives are Agent Runtime-exclusive and do not span Cloud Run or GKE.

| Capability | Agent Runtime | Cloud Run | GKE |
|-----------|:------------:|:---------:|:---:|
| Agent Gateway (ingress) | ✅ | ❌ | ❌ |
| Agent Gateway (egress) | ✅ | ❌ | ❌ |
| Model Armor at gateway | ✅ (GA June 24, 2026) | ❌ (not via gateway) | Via Inference Gateway |
| Agent Identity (SPIFFE) | ✅ (per-agent, non-impersonable, 24h certs, IAM principal) | ❌ (service accounts + OIDC) | ❌ (Workload Identity) |
| Agent Registry (auto) | ✅ (automatic on deploy) | ⚠️ (manual API only) | ✅ (via labels/annotations) |

> **Key insight from PDF:** *"SEI ADR-0004 makes Agent Gateway a foundational dependency and commits to per-agent Agent Identity. Both only pay off on Agent Runtime. Choosing Cloud Run or GKE as the default would strand that design decision."*

---

## Cross-Runtime Interoperability *(imported from PDF)*

Agents in different runtimes **can** talk to each other via A2A protocol. Google's codelab demonstrates an Agent Runtime-hosted ADK orchestrator calling two Cloud Run-hosted A2A agents.

### What works

- **A2A protocol** (v1.0) — JSON-RPC 2.0, gRPC, HTTP+JSON. SSE streaming + webhook push.
- **Agent Registry** — runtime-agnostic directory. Auto on Agent Runtime; auto on GKE via labels; manual API for Cloud Run.
- **Private connectivity** — PSC interface + network attachment for Agent Runtime ↔ VPC; Direct VPC egress for Cloud Run; native VPC routing for GKE.

### 6 gaps that are integration work, not platform features

| Gap | Impact |
|-----|--------|
| **Governance split** | Agent Gateway fronts Agent Runtime only. Mixed estate = two governance regimes. |
| **Identity boundary** | Agent Identity (SPIFFE) is Agent Runtime-only. Cross-runtime calls drop to service accounts + OIDC at the boundary. |
| **End-user identity** | Multi-hop propagation (A→B→C retaining original user) is not documented. |
| **Distributed tracing** | No first-party mechanism stitches Cloud Trace spans across A2A hops between runtimes. |
| **Version skew** | A2A moved 0.1→1.0 rapidly. Caller on older SDK can mismatch newer Agent Card schema. |
| **VPC-SC inconsistency** | VPC-SC applies to Agent Runtime API calls but Agent Gateway itself doesn't support VPC-SC. |

---

## Ops Burden Over 3 Years *(imported from PDF)*

| Dimension | Agent Runtime | Cloud Run | GKE |
|-----------|--------------|-----------|-----|
| **Who patches** | Google, entirely | Google (buildpacks) or you (custom Dockerfile) | Google (control plane + Autopilot nodes); you own images, CRDs, operators |
| **Upgrade exposure** | Preview churn, SDK breaking changes | Very low (long-GA) | Release channels on Google's clock; 90-day exclusion cap |
| **What team builds** | Agent code + revision hygiene | Session storage, observability, eval, anything >60 min | All of the above + the cluster platform itself |
| **Skills implied** | Python + GCP IAM | Containers + GCP | Kubernetes platform engineering as standing capability |
| **SLA** | General platform SLA (no separate runtime SLA found) | 99.95% (non-GPU); 99.95% GPU w/ zonal redundancy | Control plane only: 99.95% regional |

> *"Ops burden is not a one-time cost. It recurs every quarter, and it is the cost most often omitted from runtime comparisons."* — PDF Slide 10

---

## When You CANNOT Use Agent Runtime (Hard Blockers)

> **For SEI teams:** Default to Agent Runtime. The following are **hard blockers** where it's architecturally incompatible.

### Architectural Blockers

| # | Blocker | Alternative |
|---|---------|-------------|
| 1 | **Native event-driven triggers** (Pub/Sub, Eventarc, Scheduler) | → **Cloud Run** |
| 2 | **Standard public REST API + custom domain** | → **Cloud Run** |
| 3 | **No `gcloud` CLI** | → **Cloud Run** |
| 4 | **GPU / accelerators** | → **Cloud Run** (L4, RTX PRO 6000) or **GKE** (multi-GPU) |
| 5 | **Long-running batch (>10 min)** | → **Cloud Run Jobs** (up to 7 days) or **GKE** |
| 6 | **Sidecars, service mesh, pod-level network policies** | → **GKE** (K8s-first mandate, Istio, pod-level policies only) |
| 7 | **Binary authorization** | → **Cloud Run** (GA since Sept 2021) or **GKE** |

### Operational Constraints

| # | Constraint | Mitigation |
|---|-----------|------------|
| 8 | Max 100 agents per project per region | Multiple projects or consolidate. |
| 9 | Default 90 QPM query quota | Request increase early. |
| 10 | Max 1000 instances (100 w/ VPC-SC) | → GKE for unlimited scaling. |
| 11 | CPU limited to 1, 2, 4, 6, 8 | Sufficient for most agents. |
| 12 | Deployment takes 5–10 minutes | `--no-wait` + `--status`. Develop locally with `adk web`. |
| 13 | Limited region availability | → Cloud Run (40+ regions). |

### "Can I use Agent Runtime?"

```
├─ Need native Pub/Sub/Eventarc triggers? ──YES──→ ❌ Use Cloud Run
├─ Need standard public REST API + custom domain? ──YES──→ ❌ Use Cloud Run
├─ Need GPU/accelerator support? ──YES──→ ❌ Use Cloud Run or GKE
├─ Need sidecars or service mesh? ──YES──→ ❌ Use GKE
├─ Need batch jobs >10 minutes? ──YES──→ ❌ Use Cloud Run Jobs
├─ Required region not supported? ──YES──→ ❌ Use Cloud Run
├─ CI/CD requires `gcloud` CLI exclusively? ──YES──→ ❌ Use Cloud Run
│
└─ None of the above? ──→ ✅ Agent Runtime is the recommended default
```

---

## When to Choose GKE (Refined)

GKE should **only** be chosen when:
- Strict **Kubernetes-first mandate** with existing clusters
- **Istio service mesh** or complex sidecar patterns
- **Pod-level network policies** (K8s NetworkPolicy)
- **Multi-GPU** configurations (8x A100, H100 clusters)
- **Custom K8s operators** or CRDs
- **StatefulSets** with persistent volume claims

---

## Quick-Reference Matrix

| Criteria | Agent Runtime | Cloud Run | GKE |
|----------|:------------:|:---------:|:---:|
| Best for chat agents | ✅ | ⚠️ | ⚠️ |
| Best for public APIs | ⚠️ | ✅ | ⚠️ |
| Best for event-driven | ⚠️ | ✅ | ⚠️ |
| Python + ADK | ✅ | ✅ | ✅ |
| Non-Python | ⚠️ BYOC | ✅ Native | ✅ Native |
| Scale to zero | ✅ | ✅ | ⚠️ |
| Managed sessions | ✅ Built-in | ⚠️ Client API | ⚠️ Client API |
| Memory Bank | ✅ Built-in | ⚠️ Client API | ⚠️ Client API |
| Custom domains | ❌ | ✅ | ✅ |
| OAuth user consent | ✅ | ❌ | ❌ |
| Agent Gateway | ✅ | ❌ | ❌ |
| Agent Identity (SPIFFE) | ✅ | ❌ | ❌ |
| Pub/Sub triggers | ⚠️ Passthrough | ✅ Native | ⚠️ Custom |
| VPC-SC / CMEK / HIPAA | ✅ | ⚠️ Partial | ✅ Manual |
| Binary authorization | ❌ | ✅ | ✅ |
| Built-in Traces tab | ✅ | ❌ | ❌ |
| Third-party observability | ⚠️ Limited | ✅ | ✅ |
| Gen AI Evals integration | ✅ Native | ⚠️ Manual | ⚠️ Manual |
| Feedback Service | ✅ | ❌ | ❌ |
| Revision rollback | ✅ (Preview) | ✅ (GA) | ✅ kubectl |
| `gcloud` CLI | ❌ | ✅ | ✅ |
| GPU support | ❌ | ✅ (L4, RTX PRO 6000) | ✅ (full range) |
| Long-running jobs | ❌ | ✅ (60 min / 7 days) | ✅ (unlimited) |
| Setup complexity | Low | Medium | High |
| Cold start | 100–500ms | 100–2000ms | 5–30s |

---

## Open Questions for Google PS *(imported from PDF)*

1. ~~Agent Gateway GA status~~ — **Resolved:** GA June 18, 2026.
2. **Roadmap to extend Agent Identity (SPIFFE) and Agent Gateway to Cloud Run/GKE?** — If this closes within 12 months, mixed estates become much cheaper.
3. **Current Agent Runtime quotas** — max agents per project/region, max revisions, request timeout, max payload. Public quota pages are hard to read programmatically.
4. **Calling managed Sessions/Memory Bank from Cloud Run/GKE** — confirm quotas, SLAs, and reference customers.
5. **Realistic monthly floor cost for 5–10 low-traffic agents** across dev and prod (given `min_instances` defaults to 1, idle-not-billed).
6. **Session Event Append quota limits** in multi-agent RAG workloads — practical concurrency ceilings.
7. **End-user identity propagation across multi-hop A2A calls** — recommended pattern if not natively supported.
8. **Cross-runtime trace stitching** — supported way to stitch Cloud Trace spans across A2A calls between runtimes, or is it entirely application work?

---

## Sources

- [GEAP Agent Runtime Overview](https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/runtime)
- [Deploy an Agent](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/deploy-an-agent)
- [Runtime Contract](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/runtime-contract)
- [Manage Revisions and Traffic](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/manage-revisions-and-traffic)
- [GEAP Release Notes](https://docs.cloud.google.com/gemini-enterprise-agent-platform/release-notes)
- [GEAP Pricing](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing)
- [Agent Platform Quotas](https://docs.cloud.google.com/gemini-enterprise-agent-platform/resources/agent-quotas)
- [Cloud Run GPU (Services)](https://cloud.google.com/run/docs/configuring/services/gpu)
- [Cloud Run GPU (Jobs)](https://cloud.google.com/run/docs/configuring/jobs/gpu)
- [Cloud Run GPUs GA Blog](https://cloud.google.com/blog/products/serverless/cloud-run-gpus-are-now-generally-available)
- [Cloud Run RTX PRO 6000 Blog](https://cloud.google.com/blog/products/serverless/cloud-run-supports-nvidia-rtx-6000-pro-gpus-for-ai-workloads)
- [Binary Authorization for Cloud Run](https://cloud.google.com/binary-authorization/docs/run/enabling-binauthz-cloud-run)
- [Cloud Run Request Timeout](https://cloud.google.com/run/docs/configuring/request-timeout)
- [Cloud Run Jobs](https://docs.cloud.google.com/run/docs/create-jobs)
- [Set up Tracing](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/tracing)
- [Evaluate Agents](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/evaluation-agents-client)
- [ADK Deployment Guide (agents-cli)](https://github.com/google/agents-cli/blob/main/skills/google-agents-cli-deploy/SKILL.md)
- [ADK Deploy Docs](https://github.com/google/adk-docs/blob/main/docs/deploy/index.md)
- [Ambient Agents](https://github.com/google/adk-docs/blob/main/docs/runtime/ambient-agents.md)
- [Choosing the Right Deployment Path (Ayo Adedeji)](https://medium.com/google-cloud/choosing-the-right-deployment-path-for-your-google-adk-agents-86c89c251ab5)
- [Go ADK Friction Log](https://github.com/go-steer/core-agent/blob/main/docs/agent-runtime-go-friction-log.md)
- [BYOC Codelab](https://codelabs.developers.google.com/codelabs/agent-runtime-deploy-containerized-agent)
- [Optimize and Scale Agent Runtime](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/optimize-and-scale)
- [ADK Eval Codelab](https://codelabs.developers.google.com/adk-eval/instructions)
- [Revisions Walkthrough (Daniel Zamora)](https://discuss.google.dev/t/revisions-and-traffic-splitting-on-agent-runtime/378489)
- [SDK Commit: Runtime Versioning](https://github.com/googleapis/python-aiplatform/commit/b8eaefb5236669953865a770ba5fddfaf2dbe2b3)
- [Harness: AI Agent Deployment](https://www.harness.io/blog/introducing-ai-agent-deployment)
- [Choose Instrumentation Approach (GCP)](https://cloud.google.com/stackdriver/docs/instrumentation/choose-approach)
