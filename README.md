# Advanced Kubernetes — Day 1 & Day 2

Ten hands-on labs across two days: multi-cluster provisioning and service mesh (Day 1), then security and scaling/optimization (Day 2). Every command in every lab was executed end-to-end against real infrastructure — local `kind` clusters for the fully-local labs, live GKE, EKS, and AKS clusters for the cloud-dependent ones — with real screenshots embedded directly in each lab at the step they belong to (not a separate document), plus raw captured output in [`evidence/`](evidence/).

## Start here

**[00 — Setup Environment Guide](00-setup-environment-guide.md)** — do this before anything else. Tooling install, cloud account requirements, and a pre-flight checklist for both days.

## Day 1 — Multi-Cluster & Service Mesh

| # | Lab | Runs against | Cost |
|---|---|---|---|
| 1 | [Provisioning clusters with Cluster API / KubeFed federation](lab-01-cluster-api-kubefed-federation.md) | Local (`kind` + Docker) | $0 |
| 2 | [GKE fleet management with attached AWS and Azure clusters](lab-02-gke-fleet-attached-clusters.md) | Real GKE + EKS + AKS | ~a few $ |
| 3 | [Connecting and managing multiple clusters across cloud providers](lab-03-multi-cluster-management.md) | Same clusters as Lab 2 | included above |
| 4 | [Deploying Istio and configuring traffic shaping, retries, and circuit breaking](lab-04-istio-traffic-shaping.md) | Local (`kind` + Docker) | $0 |
| 5 | [Deploying a service mesh and configuring traffic policies, including mTLS](lab-05-service-mesh-mtls.md) | Local, builds on Lab 4 | $0 |

Labs 2+3 are one continuous exercise (same clusters, teardown at the end of Lab 3). Labs 4+5 are likewise meant to be run back to back (same Istio install, same cluster).

## Day 2 — Security & Scaling/Optimization

| # | Lab | Runs against | Cost |
|---|---|---|---|
| 6 | [Setting up image scanning and admission control (Kyverno)](lab-06-image-scanning-admission-control.md) | Local (`kind` + Docker) | $0 |
| 7 | [Configuring GKE Workload Identity Federation and Binary Authorization](lab-07-gke-workload-identity-binary-authorization.md) | Real GKE | ~a few $ |
| 8 | [Implementing supply chain and runtime security controls (Falco)](lab-08-falco-runtime-security.md) | Local (`kind` + Docker) | $0 |
| 9 | [Configuring advanced HPA/VPA autoscaling patterns](lab-09-hpa-vpa-autoscaling.md) | Local (`kind` + Docker) | $0 |
| 10 | [Setting up Cluster Autoscaler / Node Auto-Provisioning with GPU node pools](lab-10-cluster-autoscaler-gpu-nodepools.md) | Real GKE + real AWS EKS | the priciest single lab — a real GPU instance, briefly |

Day 2's labs are independent of each other and of Day 1 — do them in any order, though 6→7→8→9→10 is the intended sequence.

## What makes these labs different from "run this YAML and hope"

Every lab documents what actually happened when we ran it, not just what the tool's documentation says should happen. Where that included things going wrong, we kept it in — with the actual error, the root cause, and the fix:

- **Lab 1** — Cluster API provisioning is fully live-tested (a real 3-node cluster, created, scaled, torn down). KubeFed is honestly reported as reproducibly broken on current infrastructure, with root-caused evidence rather than a "should work" claim, plus pointers to its maintained successors (Karmada, Open Cluster Management).
- **Lab 2** — includes the real gotcha where an EKS cluster's Kubernetes version aged out of GKE attached-clusters support mid-lab, and how to check for that before it happens to you.
- **Lab 3** — includes the real 403 you get from Connect Gateway before you grant Kubernetes RBAC, and why fleet attachment alone doesn't grant it.
- **Lab 4** — includes a common false-positive "retries work!" demo (pairing fault injection with retries) alongside the data showing it doesn't actually prove what people think it proves, plus the correct demonstration against a real upstream failure.
- **Lab 6** — uses Kyverno's current CEL-based policy API (the legacy one used in most tutorials is deprecated and being removed), and shows a policy-compliant Pod that still crashes at runtime because the *image*, not just the spec, needs to support the constraint.
- **Lab 7** — the full deny → sign → allow cycle for Binary Authorization against a real, pushed, digest-pinned image, including the actual `exec format error` you get if you push from Apple Silicon without checking target architecture.
- **Lab 9** — shows VPA's `Auto` mode deprecation and the newer `InPlaceOrRecreate` mode actually resizing a live pod's resources with zero restarts — a real capability upgrade, not just an API rename.
- **Lab 10** — includes a real, live GPU quota wall on GCP (`GPUS_ALL_REGIONS` limit of 0, hidden behind misleadingly nonzero per-GPU-type regional quotas) and, rather than stopping there, a genuine live GPU autoscaling test on AWS where quota was actually available.

## Directory contents

```
00-setup-environment-guide.md
lab-01-cluster-api-kubefed-federation.md
lab-02-gke-fleet-attached-clusters.md
lab-03-multi-cluster-management.md
lab-04-istio-traffic-shaping.md
lab-05-service-mesh-mtls.md
lab-06-image-scanning-admission-control.md
lab-07-gke-workload-identity-binary-authorization.md
lab-08-falco-runtime-security.md
lab-09-hpa-vpa-autoscaling.md
lab-10-cluster-autoscaler-gpu-nodepools.md
evidence/            # captured, real command output referenced from each lab
screenshots/          # real screenshots referenced inline from labs 1, 2, 3, 4, 5, 6, 8, 9
```
