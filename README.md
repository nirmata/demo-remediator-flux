# demo-remediator-flux

Demo repository for testing the Nirmata Remediator Agent with **FluxCD** as the GitOps platform.

This is the Flux equivalent of [demo-remediator](https://github.com/nirmata/demo-remediator) which uses ArgoCD.

---

## What This Tests

The Remediator Agent's `fluxHub` environment type. When triggered, the Remediator:

1. Lists Flux `Kustomization` CRs on the cluster and finds `nginx-app`
2. Reads `nginx-app`'s `spec.sourceRef` → resolves the `GitRepository` → gets this repo's URL and branch
3. Reads `nginx-app`'s `spec.targetNamespace` → knows to scan the `nginx` namespace
4. Since there is no `spec.kubeConfig`, the Remediator connects to the **local cluster**
5. Scans the `nginx` namespace for Kyverno `PolicyReport` violations
6. Maps the violated resource (`nginx` Deployment) back to its source file (`apps/nginx/deployment.yaml` in this repo)
7. Creates a pull request to fix the violation

---

## Repository Structure

```
demo-remediator-flux/
├── apps/
│   └── nginx/
│       ├── deployment.yaml      ← Intentionally violating nginx deployment (privileged: true)
│       ├── service.yaml         ← ClusterIP service
│       └── kustomization.yaml   ← Kustomize resource list (not the Flux Kustomization CRD)
├── flux/
│   ├── gitrepository.yaml       ← Flux GitRepository CR: points to this repo
│   └── kustomization.yaml       ← Flux Kustomization CR: applies apps/nginx to the cluster
└── remediator/
    ├── kyverno-policy.yaml      ← Kyverno policy that detects the violation
    ├── remediator.yaml          ← Remediator CR with fluxHub environment type
    └── toolconfig.yaml          ← GitHub integration for PR creation
```

**Important distinction:** There are two different files both named `kustomization.yaml`:
- `apps/nginx/kustomization.yaml` — a standard **Kustomize** config file (lists which resources to include). Understood by both `kubectl kustomize` and Flux.
- `flux/kustomization.yaml` — a **Flux Kustomization CRD** (`kustomize.toolkit.fluxcd.io/v1`). This is the object the Remediator reads.

---

## ArgoCD vs Flux: What Changed

| Concept | ArgoCD demo | This Flux demo |
|---|---|---|
| App definition | `applications/nginx-app.yaml` (ArgoCD `Application` CR) | `flux/gitrepository.yaml` + `flux/kustomization.yaml` |
| Source repo | `app.spec.source.repoURL` | `gitrepository.spec.url` |
| Branch | `app.spec.source.targetRevision` | `gitrepository.spec.ref.branch` |
| Path in repo | `app.spec.source.path` | `kustomization.spec.path` |
| Target namespace | `app.spec.destination.namespace` | `kustomization.spec.targetNamespace` |
| Target cluster | `app.spec.destination.server` | `kustomization.spec.kubeConfig.secretRef` (or absent for local cluster) |
| Remediator env type | `environment.type: argoHub` | `environment.type: fluxHub` |
| App selector | `target.argoHub.appSelector.names: [nginx-demo]` | `target.fluxHub.kustomizationSelector.names: [nginx-app]` |
| No change | `apps/nginx/deployment.yaml` | `apps/nginx/deployment.yaml` (identical) |

---

## Prerequisites

- A Kubernetes cluster (kind, minikube, or cloud-managed like EKS/GKE/AKS)
- `kubectl` configured to point at the cluster
- `flux` CLI installed: `brew install fluxcd/tap/flux`
- Kyverno operator deployed on the cluster
- Nirmata Remediator operator deployed on the cluster
- A GitHub Personal Access Token (PAT) with `repo` scope

---

## Setup Guide

### Step 1 — Push this repo to GitHub

The Flux `GitRepository` CR must point to a publicly accessible (or authenticated) git URL. Push this repo to your GitHub account or the `nirmata` org.

```bash
# From the demo-remediator-flux directory:
git remote add origin https://github.com/nirmata/demo-remediator-flux.git
git push -u origin main
```

Then update `flux/gitrepository.yaml` → `spec.url` to match your actual repo URL if different.

---

### Step 2 — Install Flux on the cluster

```bash
# Verify cluster connectivity
flux check --pre

# Install Flux controllers (source-controller, kustomize-controller, etc.)
flux install
```

Verify Flux is running:
```bash
kubectl get pods -n flux-system
# Expected: source-controller, kustomize-controller, notification-controller all Running
```

---

### Step 3 — Create the nginx namespace

Flux does not create namespaces automatically by default. Create it manually:

```bash
kubectl create namespace nginx
```

---

### Step 4 — Apply Flux CRs (GitRepository + Kustomization)

These two objects tell Flux what to deploy and where.

```bash
kubectl apply -f flux/gitrepository.yaml
kubectl apply -f flux/kustomization.yaml
```

**If your repo is private**, create a git auth Secret first:
```bash
kubectl create secret generic github-creds \
  --namespace flux-system \
  --from-literal=username=<your-github-username> \
  --from-literal=password=<your-github-pat>
```
Then uncomment the `secretRef` block in `flux/gitrepository.yaml` and re-apply.

**Verify GitRepository is syncing:**
```bash
kubectl get gitrepository -n flux-system
# NAME                    URL                                              AGE   READY   STATUS
# demo-remediator-flux    https://github.com/nirmata/demo-remediator-flux  30s   True    stored artifact...
```

**Verify Kustomization applied:**
```bash
kubectl get kustomization -n flux-system
# NAME        AGE   READY   STATUS
# nginx-app   30s   True    Applied revision: main@sha1:...
```

**Verify nginx is running:**
```bash
kubectl get pods -n nginx
# NAME                     READY   STATUS    RESTARTS   AGE
# nginx-xxxxxxxxx-xxxxx    1/1     Running   0          30s
```

---

### Step 5 — Apply the Kyverno policy

This policy detects the `privileged: true` violation on the nginx deployment.

```bash
kubectl apply -f remediator/kyverno-policy.yaml
```

Wait 30–60 seconds for Kyverno to generate a PolicyReport, then verify violations exist:

```bash
kubectl get policyreport -n nginx
# NAME                   PASS   FAIL   WARN   ERROR   SKIP   AGE
# cpol-disallow-priv...  0      1      0      0       0      45s
```

The `FAIL: 1` confirms Kyverno detected the `privileged: true` violation. This is the violation the Remediator will fix.

---

### Step 6 — Create the GitHub PAT secret

The Remediator uses this to authenticate with GitHub for PR creation.

```bash
kubectl create namespace nirmata 2>/dev/null || true

kubectl create secret generic github-pat-secret \
  --namespace nirmata \
  --from-literal=token=<your-github-pat>
```

---

### Step 7 — Apply the Remediator config

```bash
# GitHub integration (PR title, branch prefix, etc.)
kubectl apply -f remediator/toolconfig.yaml

# The Remediator CR with fluxHub environment type
kubectl apply -f remediator/remediator.yaml
```

**Verify the Remediator CR is accepted:**
```bash
kubectl get remediator -n nirmata
# NAME                    AGE
# nginx-flux-remediator   10s
```

---

### Step 8 — Watch the Remediator run

Check the Remediator controller logs. Look for the Flux discovery happening:

```bash
kubectl logs -n nirmata deploy/remediator-controller -f
```

You should see log lines like:
```
Getting scan targets from Flux Kustomizations
Found 1 total Kustomizations
Final selected Kustomizations: 1
Processing Kustomization: flux-system/nginx-app
Resolved GitRepository flux-system/demo-remediator-flux: url=https://github.com/nirmata/demo-remediator-flux
Created violations collector for Kustomization flux-system/nginx-app (cluster: remediator-host, namespace: nginx)
Fetching PolicyReports from configured namespace: nginx
Found 1 PolicyReports in namespace nginx
Creating PR...
```

The Remediator runs every 5 minutes (as configured in `remediator.yaml`). To trigger it immediately, you can update the `crontab` to `"* * * * *"` or force a reconciliation:

```bash
kubectl annotate remediator nginx-flux-remediator \
  --namespace nirmata \
  reconcile.fluxcd.io/requestedAt=$(date +%s) \
  --overwrite
```

---

### Step 9 — Verify the PR

Go to your GitHub repository. You should see a new pull request from the `remediator-flux` branch with a title like:

```
fix: remediate policy violation in apps/nginx/deployment.yaml
```

The diff should change:
```diff
       securityContext:
-        privileged: true
+        privileged: false
         capabilities:
           add:
```

---

### Step 10 — Verify the full GitOps loop

1. Merge the PR on GitHub
2. Flux detects the new commit (within 1 minute, per `spec.interval: 1m`)
3. Flux syncs the updated `deployment.yaml` to the cluster
4. The nginx pod restarts with `privileged: false`
5. Kyverno re-evaluates → the policy now passes → `PolicyReport` shows `PASS: 1, FAIL: 0`
6. On the next Remediator run (5 min later) → no violations found → no PR created

---

## How to Verify What the Remediator Reads

You can inspect exactly what the Remediator will discover before running it:

```bash
# See the Flux Kustomization the Remediator will find
kubectl get kustomization nginx-app -n flux-system -o yaml

# Key fields the Remediator reads:
# spec.sourceRef.kind: GitRepository     ← must be GitRepository (not OCIRepository)
# spec.sourceRef.name: demo-remediator-flux
# spec.targetNamespace: nginx            ← namespace to scan for violations
# spec.path: ./apps/nginx                ← path in git repo
# spec.kubeConfig: <absent>              ← absent = local cluster

# See the GitRepository the Remediator will follow to
kubectl get gitrepository demo-remediator-flux -n flux-system -o yaml

# Key fields the Remediator reads:
# spec.url: https://github.com/...       ← git repo for the PR
# spec.ref.branch: main                  ← branch for the PR
```

---

## Troubleshooting

**GitRepository not Ready:**
```bash
kubectl describe gitrepository demo-remediator-flux -n flux-system
# Check: authentication failure (add secretRef), network error, wrong URL
```

**Kustomization not Ready:**
```bash
kubectl describe kustomization nginx-app -n flux-system
# Check: "namespace nginx not found" → run: kubectl create namespace nginx
# Check: "kustomization.yaml not found" → ensure apps/nginx/kustomization.yaml exists
```

**No PolicyReport generated:**
```bash
kubectl get policyreport -n nginx
# If empty, Kyverno may still be starting up. Wait 1-2 minutes.
# If still empty, check Kyverno pods: kubectl get pods -n kyverno
```

**Remediator finds no Kustomizations:**
```
# In controller logs: "Final selected Kustomizations: 0"
# Check that spec.names matches exactly:
kubectl get kustomization -n flux-system
# NAME must be exactly "nginx-app" to match remediator.yaml's kustomizationSelector.names
```

**PR not created after Remediator run:**
```bash
# Check RemediationRecord for status
kubectl get remediationrecord -n nirmata

# Check controller logs for git auth errors
kubectl logs -n nirmata deploy/remediator-controller | grep -i "error\|pr\|github"
```

---

## Hub/Spoke Extension (Optional)

The demo above uses the local cluster (no `spec.kubeConfig` on the Kustomization). To test the hub/spoke path — where Flux on a hub cluster deploys to a remote spoke — add `spec.kubeConfig.secretRef` to `flux/kustomization.yaml`:

```yaml
spec:
  kubeConfig:
    secretRef:
      name: spoke-cluster-kubeconfig  # Secret in the same namespace as this Kustomization
```

Create the Secret with a kubeconfig for the spoke cluster:
```bash
kubectl create secret generic spoke-cluster-kubeconfig \
  --namespace flux-system \
  --from-file=value=<path-to-spoke-kubeconfig>
```

The Remediator will then:
- Read the Secret → connect to the spoke cluster
- Scan PolicyReports in the spoke cluster's `nginx` namespace
- Create the PR against this git repo (same repo, same path)

The only difference is which cluster's API server the violation scan hits. Everything else — PR creation, git source — is identical.
