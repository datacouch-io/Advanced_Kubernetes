# Advanced Kubernetes — Day 1, Day 2 & Day 3

Sixteen hands-on labs across three days: multi-cluster provisioning and service mesh (Day 1), security and scaling/optimization (Day 2), then AI/ML workloads and observability (Day 3). Every command in every lab was executed end-to-end against real infrastructure — local `kind` clusters for the fully-local labs, live GKE, EKS, and AKS clusters for the cloud-dependent ones — with real screenshots embedded directly in each lab at the step they belong to (not a separate document), plus raw captured output in [`evidence/`](evidence/). The one exception, reported honestly rather than glossed over: [Lab 13](lab-13-gpu-tpu-inference-gke.md)'s GPU-attached hardware check couldn't be completed on this training project's own GCP account (a confirmed `GPUS_ALL_REGIONS: 0` quota) — see that lab's status note for the full, real evidence.

## Start here

**[00 — Setup Environment Guide](00-setup-environment-guide.md)** — do this before anything else. Tooling install, cloud account requirements, and a pre-flight checklist for all three days.

## Day 1 — Multi-Cluster & Service Mesh

| # | Lab | Runs against | Cost |
|---|---|---|---|
| 1 | [GKE cluster architecture: Autopilot vs Standard, private clusters, release channels](lab-01-gke-cluster-architecture.md) | Real GKE | ~a few $ |
| 2 | [Provisioning clusters with Cluster API / KubeFed federation](lab-02-cluster-api-kubefed-federation.md) | Local (`kind` + Docker) | $0 |
| 3 | [GKE fleet management with attached AWS and Azure clusters](lab-03-gke-fleet-attached-clusters.md) | Real GKE + EKS + AKS | ~a few $ |
| 4 | [Connecting and managing multiple clusters across cloud providers](lab-04-multi-cluster-management.md) | Same clusters as Lab 3 | included above |
| 5 | [Deploying Istio and configuring traffic shaping, retries, and circuit breaking](lab-05-istio-traffic-shaping.md) | Local (`kind` + Docker) | $0 |
| 6 | [Deploying a service mesh and configuring traffic policies, including mTLS](lab-06-service-mesh-mtls.md) | Local, builds on Lab 5 | $0 |

Labs 3+4 are one continuous exercise (same clusters, teardown at the end of Lab 4). Labs 5+6 are likewise meant to be run back to back (same Istio install, same cluster). Lab 1 stands alone — its two clusters are created and torn down within the lab itself.

## Day 2 — Security & Scaling/Optimization

| # | Lab | Runs against | Cost |
|---|---|---|---|
| 7 | [Setting up image scanning and admission control (Kyverno)](lab-07-image-scanning-admission-control.md) | Local (`kind` + Docker) | $0 |
| 8 | [Configuring GKE Workload Identity Federation and Binary Authorization](lab-08-gke-workload-identity-binary-authorization.md) | Real GKE | ~a few $ |
| 9 | [Implementing supply chain and runtime security controls (Falco)](lab-09-falco-runtime-security.md) | Local (`kind` + Docker) | $0 |
| 10 | [Configuring advanced HPA/VPA autoscaling patterns](lab-10-hpa-vpa-autoscaling.md) | Local (`kind` + Docker) | $0 |
| 11 | [Setting up Cluster Autoscaler / Node Auto-Provisioning with GPU node pools](lab-11-cluster-autoscaler-gpu-nodepools.md) | Real GKE + real AWS EKS | the priciest single lab — a real GPU instance, briefly |

Day 2's labs are independent of each other and of Day 1 — do them in any order, though 7→8→9→10→11 is the intended sequence.

## Day 3 — AI/ML & Observability

| # | Lab | Runs against | Cost |
|---|---|---|---|
| 12 | [Setting up a Kubeflow pipeline for distributed training](lab-12-kubeflow-distributed-training.md) | Local (`kind` + Docker) | $0 |
| 13 | [Deploying a GPU/TPU inference service on GKE](lab-13-gpu-tpu-inference-gke.md) | Real GKE | ~a few $ (base cluster only — see lab's status note on GPU quota) |
| 14 | [Deploying a simple ML inference pipeline](lab-14-simple-ml-inference-pipeline.md) | Local (`kind` + Docker) | $0 |
| 15 | [Instrumenting workloads with OpenTelemetry for observability](lab-15-opentelemetry-observability.md) | Local (`kind` + Docker) | $0 |
| 16 | [Running a chaos engineering experiment with Chaos Mesh](lab-16-chaos-mesh-experiment.md) | Local (`kind` + Docker) | $0 |

Day 3's labs are independent of each other and of Days 1–2 — do them in any order, though 12→13→14→15→16 is the intended sequence. Only Lab 13 costs real money; the other four run entirely on a local `kind` cluster.

## What makes these labs different from "run this YAML and hope"

Every lab documents what actually happened when we ran it, not just what the tool's documentation says should happen. Where that included things going wrong, we kept it in — with the actual error, the root cause, and the fix.

- **Lab 1** — deliberately locks itself out of its own cluster's control plane via master authorized networks, on purpose, then recovers — a failure mode that reads as "network problem," not "RBAC problem," which is exactly what trips people up the first time it happens for real. Also the actual GKE Warden rejection message when Autopilot refuses a privileged container that Standard mode admits without complaint.
- **Lab 2** — Cluster API provisioning is fully live-tested (a real 3-node cluster, created, scaled, torn down). KubeFed is honestly reported as reproducibly broken on current infrastructure, with root-caused evidence rather than a "should work" claim, plus pointers to its maintained successors (Karmada, Open Cluster Management).
- **Lab 3** — includes the real gotcha where an EKS cluster's Kubernetes version aged out of GKE attached-clusters support mid-lab, and how to check for that before it happens to you.
- **Lab 4** — includes the real 403 you get from Connect Gateway before you grant Kubernetes RBAC, and why fleet attachment alone doesn't grant it.
- **Lab 5** — includes a common false-positive "retries work!" demo (pairing fault injection with retries) alongside the data showing it doesn't actually prove what people think it proves, plus the correct demonstration against a real upstream failure.
- **Lab 7** — uses Kyverno's current CEL-based policy API (the legacy one used in most tutorials is deprecated and being removed), and shows a policy-compliant Pod that still crashes at runtime because the *image*, not just the spec, needs to support the constraint.
- **Lab 8** — the full deny → sign → allow cycle for Binary Authorization against a real, pushed, digest-pinned image, including the actual `exec format error` you get if you push from Apple Silicon without checking target architecture.
- **Lab 10** — shows VPA's `Auto` mode deprecation and the newer `InPlaceOrRecreate` mode actually resizing a live pod's resources with zero restarts — a real capability upgrade, not just an API rename.
- **Lab 11** — includes a real, live GPU quota wall on GCP (`GPUS_ALL_REGIONS` limit of 0, hidden behind misleadingly nonzero per-GPU-type regional quotas) and, rather than stopping there, a genuine live GPU autoscaling test on AWS where quota was actually available.
- **Lab 12** — a two-layer design (Training Operator for the actual distributed `PyTorchJob`, Kubeflow Pipelines for orchestrating it) rather than treating "Kubeflow" as one monolithic thing to install; genuinely proves the training layer end to end (a real 3-Pod `torch.distributed` job, `all_reduce result: 3.0 (expected 3)`), and just as honestly reports that the official KFP standalone backend itself fails to install on current infrastructure — two of its own pinned images no longer resolve on `gcr.io` (a known, already-filed upstream bug), independent of anything in this project.
- **Lab 13** — confirms the project-wide GPU quota wall from Lab 11 is still real (`GPUS_ALL_REGIONS: 0`), then goes further: retrying in a completely different zone with a completely GPU-free cluster hit the *identical* 35-minute `GCE_STOCKOUT` failure despite ample regional CPU quota — evidence that this error class can mean genuine provider-side capacity constraints, not just your own account limits. Separately, the inference-server section hit and fixed three real bugs in one sitting (Triton's model-control-mode, a `kubectl exec` stdin gotcha, a container image with no `/models` directory at boot) before landing a genuine, verified inference round-trip.
- **Lab 14** — a deliberately un-fancy three-microservice HTTP chain, verified with real chained requests (`{"label":"low","score":0.4...}`, `{"label":"high","score":0.9}`) end to end, and honest about exactly where this pattern stops being enough (GPU serving, actual training, scale-to-zero).
- **Lab 15** — a real trace, from an unmodified Flask app, through an auto-instrumentation webhook, to Jaeger — 10/10 requests traced with durations matching the app's own code, plus two separate real Helm/Operator resource-naming gotchas (`<release>-<chart>`, `<CR-name>-collector`) caught mid-lab.
- **Lab 16** — deliberately picks one tool (Chaos Mesh) over presenting Chaos Mesh/LitmusChaos as interchangeable, and ties its `NetworkChaos` experiment directly back to Lab 5's retries-against-real-failure finding rather than treating chaos engineering and service-mesh resilience as unrelated topics. The real headline finding goes further than the original design intended: `NetworkChaos` combined with a default `readinessProbe` timeout didn't just add latency, it took the whole Service to zero ready endpoints — a materially different, worse failure, caught and fixed live.

## Directory contents

```
00-setup-environment-guide.md
lab-01-gke-cluster-architecture.md
lab-02-cluster-api-kubefed-federation.md
lab-03-gke-fleet-attached-clusters.md
lab-04-multi-cluster-management.md
lab-05-istio-traffic-shaping.md
lab-06-service-mesh-mtls.md
lab-07-image-scanning-admission-control.md
lab-08-gke-workload-identity-binary-authorization.md
lab-09-falco-runtime-security.md
lab-10-hpa-vpa-autoscaling.md
lab-11-cluster-autoscaler-gpu-nodepools.md
lab-12-kubeflow-distributed-training.md
lab-13-gpu-tpu-inference-gke.md
lab-14-simple-ml-inference-pipeline.md
lab-15-opentelemetry-observability.md
lab-16-chaos-mesh-experiment.md
evidence/            # captured, real command output referenced from each tested lab
screenshots/          # real screenshots referenced inline from every tested lab, 1 through 11
```
