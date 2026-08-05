## 🔴 Inaccuracies / Stale Information

| # | Claim in PDF | Issue | Evidence |
|---|-------------|-------|----------|
| 1 | **"No built-in rollback"** (Slide 5, 15) | **Stale.** Agent Runtime shipped immutable revisions + traffic splitting on **May 19, 2026** (Preview, `v1beta1`). Rollback = shift 100% traffic to previous revision. | [Official docs](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/manage-revisions-and-traffic); [Release notes](https://docs.cloud.google.com/gemini-enterprise-agent-platform/release-notes); [SDK commit b8eaefb](https://github.com/googleapis/python-aiplatform/commit/b8eaefb5236669953865a770ba5fddfaf2dbe2b3) |
| 2 | **"Immutable revisions and traffic splitting; no built-in rollback"** (Slide 5) | **Self-contradictory.** Traffic splitting to a previous revision *is* rollback. | Same as above; [Community walkthrough](https://discuss.google.dev/t/revisions-and-traffic-splitting-on-agent-runtime/378489) |
| 3 | **Cloud Run GPU: "narrow regions"** (Slide 7) | **Partially stale.** L4 GPUs GA since **June 2, 2025** across 6 regions. "Narrow" undersells availability. | [Cloud Run GPUs GA blog](https://cloud.google.com/blog/products/serverless/cloud-run-gpus-are-now-generally-available); [GPU docs](https://cloud.google.com/run/docs/configuring/services/gpu) |
| 4 | **Agent Gateway GA uncertainty** (Slide 2) | **Resolved.** Agent Gateway went GA on **June 18, 2026**. | [Release notes June 18, 2026](https://docs.cloud.google.com/gemini-enterprise-agent-platform/release-notes) |
| 5 | **"Python only"** for Agent Runtime (Slide 5) | **Stale.** BYOC with published [runtime contract](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/runtime-contract) now documented. Any language can implement the HTTP endpoints. | [Runtime contract](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/runtime-contract); [BYOC codelab](https://codelabs.developers.google.com/codelabs/agent-runtime-deploy-containerized-agent) |
| 6 | **Cloud Run: "No managed session or memory"** (Slide 3) | **Misleading.** Cloud Run can consume Sessions and Memory Bank as client API calls via SDK. Not built-in, but available as external services. | [Go SDK](https://pkg.go.dev/cloud.google.com/go/agentplatform); [Go ADK friction log](https://github.com/go-steer/core-agent/blob/main/docs/agent-runtime-go-friction-log.md) |

---

## 🟡 Items the Markdown Covers Better

| Topic | PDF Gap | Markdown Coverage |
|-------|---------|-------------------|
| **Agent nature as primary branch** | Organizes by technical dimension; never asks "chat vs. API vs. event-driven?" | Dimension 1 makes this the primary decision node. |
| **Event-driven / batch agents** | Mentioned in passing; no ADK `trigger_sources` or ambient agents. | Explicit coverage with Cloud Run as recommended platform. |
| **Observability depth** | High-level only (Slide 6). | Covers `@do_trace`, third-party OTel limitations, Feedback Service. |
| **Evaluation** | Zero coverage. | Full Dimension 10: Gen AI Evals, `adk eval`, Example Store, eval-fix loop. |
| **Hard blockers checklist** | Pros/cons per runtime but no definitive "can't use" list. | Concrete "Can I use Agent Runtime?" flowchart. |
| **Binary Auth on Cloud Run** | Not mentioned. | Correctly shows ✅ (GA since [Sept 2021](https://docs.cloud.google.com/binary-authorization/docs/release-notes)). |
| **Cloud Run long-running jobs** | Mentions 60-min request timeout only. | Covers Cloud Run Jobs (up to [168 hours](https://docs.cloud.google.com/run/docs/create-jobs)). |

---

## 🟢 Items the PDF Covers Better

| Topic | PDF Strength | Markdown Coverage |
|-------|-------------|--------------|
| **Cross-runtime interop** (Slides 11–13) | A2A protocol, Agent Registry, PSC networking, Google codelab proof, 6 specific integration gaps (governance, identity, tracing, version skew, VPC-SC, end-user identity). | Not covered. Significant gap for multi-runtime architectures. |
| **Agent Gateway & Identity** (Slide 8) | Agent Gateway (ingress/egress) and Agent Identity (SPIFFE, per-agent, 24h certs) shown as Agent Runtime-exclusive. Model Armor at gateway. Decisive governance argument. | Mentioned briefly; not positioned as key differentiator. |
| **Networking detail** (Slides 8, 12) | PSC interface, Direct VPC egress, private DNS, PSC-disables-internet-egress constraint. | Not covered at this depth. |
| **Cost shape comparison** (Slide 9) | Three billing models side-by-side with crossover analysis. Sessions/Memory Bank billing deadline (Sept 1, 2026). | Simpler, less actionable. Missing billing deadline. |
| **Ops burden over 3 years** (Slide 10) | Patching, upgrade exposure, skills implied, SLA differences. "The dimension that decides most platform choices." | Covered but not as long-term framing. |
| **Open questions for Google PS** (Slide 21) | 8 specific actionable questions for Google engagement. | Not included. |
| **Decision rules as triggers** (Slides 19–20) | 12 "if this → then this → because" triggers. Very actionable. | Similar flowchart but less granular (7 branches vs. 12 triggers). |

---

## 🟠 Additional Gotchas

| # | Item | Risk |
|---|------|------|
| 1 | PDF's recommendation to **"Standardise on Agent Runtime with a documented exception path"** (Slide 20) | **Aligns with Markdown File.** Key insight: *"Runtime-agnosticism sounds safer but in practice means building the governance layer twice."* Our Hard Blockers checklist provides the exception path the PDF recommends but doesn't build. |
| 2 | **"Bidi streaming Preview, 10-minute cap"** (Slide 7) | Should verify against current docs — 10-min cap is for bidi streaming specifically, not standard query timeout. |
| 3 | **"ADK 1.18.0+ deploy regression"** (Slide 15) | Time-sensitive. Current ADK is 1.32.0+; may be fixed. Don't cite as current concern without verification. |
| 4 | **"Code Execution is us-central1 only"** (Slide 15) | May have expanded. Verify current regional availability. |
| 5 | **"Sessions and Memory Bank free until 1 Sep 2026"** (Slide 9, 15, 20) | **Critical deadline.** Accurate per [pricing page](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing). Markdown file doesn't flag this. |

