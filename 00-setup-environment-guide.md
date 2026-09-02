# Advanced Kubernetes — Environment Setup Guide

**Day 1: Multi-Cluster & Service Mesh · Day 2: Security & Scaling/Optimization**

This guide gets your laptop ready for all ten labs across both days. Work through it **before** the training session — several labs provision real cloud infrastructure, and getting stuck on tool installation during class eats into hands-on time.

Every command in this guide, and in every lab document in this folder, was executed end-to-end on a real machine (macOS, Apple Silicon) against real clusters — local (kind/Docker) for the fully-local labs, and live GKE + EKS + AKS clusters for the cloud-dependent ones. Verified output is captured in [`evidence/`](evidence/) alongside each lab. If a command in a lab doesn't match what you see, check this guide first — version drift in fast-moving CLIs (`gcloud`, `eksctl`, `istioctl`, `kyverno`) is the most common cause.

---

## 1. Hardware and OS

| Requirement | Minimum | Recommended |
|---|---|---|
| OS | macOS 13+, or Linux (Ubuntu 22.04+) | macOS on Apple Silicon or Linux |
| CPU | 4 cores | 8+ cores |
| RAM | 8 GB free for Docker | 16 GB free for Docker |
| Disk | 20 GB free | 40 GB free |

Labs 1, 4, and 5 run multiple local Kubernetes clusters (via `kind`) on top of Docker simultaneously. Docker Desktop's default resource allocation is often too small — go to **Docker Desktop → Settings → Resources** and confirm at least 8 GB RAM / 4 CPUs is allocated before Day 1. This guide's tooling was validated with Docker Desktop given 12 CPUs / 8 GB RAM.

> Windows users: run everything inside **WSL2** (Ubuntu). Native Windows shells are not covered by these labs.

---

## 2. Cloud accounts you need

Labs 2 and 3 attach real clusters from all three major clouds into one fleet. Before Day 1, make sure you (or your organization) have:

1. **A Google Cloud project** with billing enabled and permission to enable APIs and create GKE clusters (`roles/owner` or `roles/container.admin` + `roles/gkehub.admin` + `roles/serviceusage.serviceUsageAdmin`).
2. **An AWS account** with permission to create VPCs, EKS clusters, and IAM roles (`AdministratorAccess`, or an equivalent scoped policy, is simplest for a training sandbox).
3. **An Azure subscription** with permission to create resource groups and AKS clusters (`Contributor` role on the subscription).

Use a **disposable sandbox project/account/subscription** for these labs if at all possible — Lab 2 and 3 create real billed resources (GKE, EKS, AKS clusters). Each lab document ends with a teardown section; running it promptly keeps cost to a few dollars per person.

Costs observed during testing (single small cluster per cloud, ~1–2 hours total lifetime, deleted immediately after): **under $5 total across all three clouds.** Your cost will scale with how long you leave clusters running, so don't skip the teardown steps.

---

## 3. Install the CLI tools

All tools below were installed via [Homebrew](https://brew.sh) on macOS. Linux users: swap `brew install` for your distro's package manager or the tool's official install script (linked per-tool below).

```bash
# Core Kubernetes tooling
brew install kubectl kind helm

# Cloud provider CLIs
brew install --cask google-cloud-sdk    # gcloud
brew install awscli                     # aws
brew install azure-cli                  # az

# Multi-cluster provisioning
brew install clusterctl

# Service mesh
brew install istioctl

# AWS/Azure Kubernetes auth helpers
brew install eksctl
brew install Azure/kubelogin/kubelogin

# GKE kubectl auth plugin and beta commands (via gcloud, not brew)
gcloud components install gke-gcloud-auth-plugin
gcloud components install beta

# Day 2: image scanning and policy-as-code CLIs
brew install trivy kyverno

# Optional but recommended: terminal cluster browser
brew install derailed/k9s/k9s
```

> `gcloud components install beta` is needed for [Lab 7](lab-07-gke-workload-identity-binary-authorization.md) (`gcloud beta container binauthz attestations sign-and-create` isn't in the stable command tree).

> If you already have `gcloud` installed some other way (not via the cask), just run the `gcloud components install gke-gcloud-auth-plugin` line — that's the piece people most often miss, and `kubectl` will fail against GKE with an opaque auth error without it.

### KubeFed CLI (Lab 1, legacy/reference only)

`kubefedctl` is **not** in Homebrew — the project was archived by Kubernetes SIG Multicluster in April 2023 and no longer ships current builds. Lab 1 explains why this matters and treats KubeFed as a read-along exercise rather than a live one, but if you want the CLI on your machine anyway:

```bash
mkdir -p /tmp/kubefed-install && cd /tmp/kubefed-install
curl -sL -o kubefedctl.tgz \
  https://github.com/kubernetes-retired/kubefed/releases/download/v0.9.2/kubefedctl-0.9.2-darwin-amd64.tgz
tar xzf kubefedctl.tgz
sudo mv kubefedctl /usr/local/bin/kubefedctl   # or /opt/homebrew/bin on Apple Silicon
```

Only an `amd64` build exists (last released 2021). On Apple Silicon this runs under Rosetta 2 — install Rosetta first if you haven't: `softwareupdate --install-rosetta --agree-to-license`.

### Docker Desktop

Install from [docker.com](https://www.docker.com/products/docker-desktop/) if you don't have it. **Start it and confirm the daemon is running** before Day 1:

```bash
docker info >/dev/null 2>&1 && echo "Docker daemon: OK" || echo "Docker daemon: NOT RUNNING — start Docker Desktop"
```

---

## 4. Authenticate each CLI

Run these once per machine. Each opens a browser window for you to sign in.

```bash
# Google Cloud
gcloud auth login
gcloud config set project YOUR_GCP_PROJECT_ID
gcloud auth application-default login    # needed by some Terraform/SDK-based tools

# AWS
aws configure
#   -> enter your Access Key ID, Secret Access Key, default region (e.g. us-west-2), output format (json)
# Prefer an IAM user with programmatic access over root account keys, even in a sandbox account.

# Azure
az login
az account set --subscription "YOUR_SUBSCRIPTION_NAME_OR_ID"
```

---

## 5. Verify everything before Day 1

Run this checklist top to bottom. Every line should print a version or a successful identity check — no errors.

```bash
echo "--- local tooling ---"
kubectl version --client
kind version
helm version
clusterctl version
istioctl version --remote=false
docker info >/dev/null 2>&1 && echo "docker daemon: OK"

echo "--- cloud CLIs, authenticated ---"
gcloud config list
gcloud auth list
aws sts get-caller-identity
az account show

echo "--- cloud-specific k8s tooling ---"
which gke-gcloud-auth-plugin
eksctl version
kubelogin --version
```

Reference output (captured during testing of this guide — your versions may be newer, that's fine):

```
kubectl:    v1.36.1
kind:       v0.33.0
helm:       v4.2.4
clusterctl: v1.14.0
istioctl:   1.31.0
eksctl:     0.230.0
kubelogin:  v0.2.19
gcloud:     Google Cloud SDK 574.0.0
aws-cli:    2.35.11
az-cli:     2.88.0
docker:     29.7.2
```

### Smoke test: local Kubernetes works end-to-end

This is the single fastest way to catch a broken Docker/kind setup before Day 1:

```bash
kind create cluster --name smoke-test
kubectl get nodes
kind delete cluster --name smoke-test
```

You should see one `Ready` node, then a clean deletion. If this fails, fix it before Day 1 — every local lab (1, 4, 5) depends on `kind` + Docker working.

---

## 6. What each lab needs

| Lab | Runs against | Real cost | Local resources |
|---|---|---|---|
| **Day 1** | | | |
| 1 — Cluster API / KubeFed | Local (`kind` + Docker) | $0 | ~2 GB RAM, 3 containers |
| 2 — GKE fleet + attached AWS/Azure clusters | Real GKE + EKS + AKS | Yes — see lab doc | Minimal (CLI only) |
| 3 — Connecting/managing multi-cloud clusters | Real GKE + EKS + AKS (reuses Lab 2's clusters) | Included in Lab 2 | Minimal (CLI only) |
| 4 — Istio traffic shaping | Local (`kind` + Docker) | $0 | ~2 GB RAM, 6-8 containers |
| 5 — Service mesh mTLS | Local (`kind` + Docker), builds on Lab 4 | $0 | Same cluster as Lab 4 |
| **Day 2** | | | |
| 6 — Image scanning + admission control | Local (`kind` + Docker) | $0 | ~2 GB RAM |
| 7 — Workload Identity Federation + Binary Authorization | Real GKE | Yes — see lab doc | Minimal (CLI + Docker for one image push) |
| 8 — Falco runtime security | Local (`kind` + Docker) | $0 | ~2 GB RAM |
| 9 — Advanced HPA/VPA autoscaling | Local (`kind` + Docker) | $0 | ~2 GB RAM |
| 10 — Cluster Autoscaler / Node Auto-Provisioning + GPU | Real GKE (CA/NAP mechanics) + real AWS EKS (GPU-specific proof) | Yes — see lab doc, and this is the priciest single lab (a real `g4dn.xlarge`) | Minimal (CLI only) |

Labs 2 and 3 are designed to be done back-to-back in one sitting, since Lab 3 reuses the three clusters Lab 2 creates. Read both lab documents before starting Lab 2 so you don't tear anything down early. Labs 4 and 5 likewise share one Istio installation.

---

## 7. Known rough edges (found during testing)

These aren't hypothetical — each one was hit while validating these labs and is called out again in context in the relevant lab document:

**Day 1**

- **CAPD (Cluster API's Docker provider) needs the host Docker socket mounted into the `kind` management cluster.** If you create the management cluster with plain `kind create cluster`, cluster provisioning fails with `Cannot connect to the Docker daemon`. Lab 1 has the correct `kind` config.
- **On Docker Desktop for Mac/Windows, a CAPI workload cluster's kubeconfig points at an internal container IP that your host can't reach.** You have to patch the kubeconfig's `server:` field to `127.0.0.1:<mapped-port>`. Lab 1 shows exactly how.
- **KubeFed does not currently install successfully** on any Kubernetes version we tested (both current-generation and the older version it originally targeted). Lab 1 documents the exact failure and why, and uses Cluster API for the hands-on "provision a real cluster" exercise instead.
- **Istio fault-injection aborts are not retried**, even with a `retries` policy on the same route. If you want to demo retries working, do it against a real upstream failure (Lab 4 uses a pod deletion mid-traffic), not `fault.abort`.
- **`gke-gcloud-auth-plugin` can report "installed" via `gcloud components list` and still fail with `executable ... not found`.** On a Homebrew-cask install of `gcloud` on macOS, the plugin binary lands in `google-cloud-sdk/bin/`, which isn't itself on `PATH` — only the SDK root is. Symlink it: `ln -sf "$(gcloud info --format='value(installation.sdk_root)')/bin/gke-gcloud-auth-plugin" /opt/homebrew/bin/`.
- **GKE fleet "attached clusters" only supports EKS/AKS Kubernetes versions within roughly the last 3 minors** — check `gcloud container attached get-server-config --location=<region>` and pick a compatible version *before* you create the EKS/AKS cluster, not after. We initially created EKS at a version that had already aged out and had to delete and recreate. Lab 2 has the full detail.
- **Fleet attachment and Kubernetes RBAC are separate.** Attaching a cluster (or being able to `list` it in the fleet) does not grant your identity permission to run `kubectl` commands against it via Connect Gateway — you'll hit a `Forbidden` error until you explicitly grant RBAC with `gcloud container fleet memberships generate-gateway-rbac ... --apply`. Lab 3 walks through this exact failure and fix.

**Day 2**

- **Kyverno's `kyverno.io/v1 ClusterPolicy` is deprecated as of 1.19 and gets removed in 1.20** — most tutorials you'll find online still use it. Lab 6 uses the current `policies.kyverno.io/v1 ValidatingPolicy` (CEL-based) type throughout.
- **An admission policy passing doesn't mean the workload will run.** `runAsNonRoot: true` against the stock `nginx` image gets admitted and then crashes (`mkdir() "/var/cache/nginx/client_temp" failed: Permission denied`) because the image itself was never built to run unprivileged. Lab 6 walks through the failure and the actual fix (a different base image).
- **VPA's `updateMode: Auto` is deprecated** in favor of explicit modes (`Recreate`, `Initial`, `InPlaceOrRecreate`). Use `InPlaceOrRecreate` — it resizes running pods live via Kubernetes' in-place resize feature, with zero restarts, which `Auto` never did. Lab 9 has the full before/after.
- **Binary Authorization requires images referenced by digest, never by tag** — `nginx:1.27-alpine` is rejected outright with `Expected digest with sha256 scheme, but got tag or malformed digest`, before it even checks for an attestation. Lab 7 shows the fix.
- **Pushing images from Apple Silicon to a registry a GKE amd64 node pool will pull from needs an explicit architecture check.** A plain `docker push` sends arm64 by default and the Pod crashes with `exec format error`; `docker pull --platform linux/amd64` doesn't reliably fix it once the tag is already cached locally. Lab 7 shows the `docker buildx imagetools` workaround.
- **GPU quota is very often 0 by default on a fresh GCP project**, even when individual GPU-type regional quota buckets show a nonzero limit — the project-wide `GPUS_ALL_REGIONS` bucket is the actual binding constraint, and it doesn't show up unless you specifically check it. Lab 10 shows how to check, and what NAP's own error output looks like when it hits this wall.
- **Cluster Autoscaler's IAM policy needs to go on the node group that's actually running the Autoscaler pod, not the node group you're trying to scale.** If you're scaling a group up from zero nodes, that's almost never the target group itself. Lab 10 has the exact `AccessDenied` error this produces when it's wrong.
- **Cluster Autoscaler's `latest`/default chart image isn't automatically compatible with your cluster's Kubernetes version.** We hit a permanent retry loop watching Dynamic Resource Allocation API types that didn't exist on the target EKS version, which silently prevented any real scaling decision from ever being evaluated. Pin `image.tag` to a Cluster Autoscaler release matching your cluster's Kubernetes **minor** version. Lab 10 has the detail.

---

Once every command in §5 succeeds, you're ready for both days. Start with [Lab 1](lab-01-cluster-api-kubefed-federation.md) for Day 1, or jump to [Lab 6](lab-06-image-scanning-admission-control.md) if you're doing Day 2 on its own.
