# Deploying Helm Charts Through ArgoCD – A Beginner's Guide

## What is ArgoCD?

ArgoCD is a **GitOps continuous delivery tool** for Kubernetes. It watches a Git repository for changes and automatically (or manually) syncs your desired state to the cluster.

## What is a Helm Chart?

A Helm chart is a **package of Kubernetes manifests** (Deployments, Services, ConfigMaps, etc.) templated with Go templates, allowing parameterization via `values.yaml`.

---

## Step-by-Step: Deploying a Helm Chart via ArgoCD

### Step 1: Install ArgoCD on Your Cluster

````bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
````

Then install the CLI:
````bash
# Windows (via scoop or chocolatey)
choco install argocd-cli
````

### Step 2: Access the ArgoCD UI

````bash
# Port-forward the ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get the initial admin password
argocd admin initial-password -n argocd
````

Open `https://localhost:8080` in your browser. Login with `admin` and the password retrieved above.

### Step 3: Add Your Helm Chart Repository

ArgoCD needs to know where the chart lives. There are **two sources**:

| Source | Description |
|--------|-------------|
| **Helm Repository** | A standard Helm repo (e.g., `https://charts.example.com`) |
| **Git Repository** | A Git repo containing the chart source files |

#### Option A – Add a Helm Repo

````bash
argocd repo add https://charts.example.com --type helm --name my-helm-repo
````

#### Option B – Add a Git Repo

````bash
argocd repo add https://github.com/my-org/my-charts.git \
  --username <user> --password <token>
````

#### Option C – Via the ArgoCD UI

1. In the ArgoCD dashboard, click **Settings** (gear icon) in the left sidebar
2. Click **Repositories**
3. Click **"+ Connect Repo"** button at the top
4. Fill in the form:
   - **Connection Method**: Choose `Via HTTPS` or `Via SSH`
   - **Type**: Select `helm` for a Helm repo, or `git` for a Git repo
   - **Project**: `default` (or your project)
   - **Repository URL**: Enter the repo URL (e.g., `https://charts.example.com` or `https://github.com/my-org/my-charts.git`)
   - **Username / Password**: Enter credentials if the repo is private
   - **TLS Client Certificate**: Upload if required by your repo
   - **Skip Server Verification**: Check if using self-signed certs (not recommended for production)
5. Click **"Connect"**
6. Verify the connection status shows a **green checkmark** (Successful)

> **Tip:** You can also manage SSH keys and TLS certificates under **Settings → Certificates** if your repo requires them.

### Step 4: Create an ArgoCD Application

This is the **core step**. An ArgoCD `Application` CRD links a **source** (Helm chart) to a **destination** (Kubernetes cluster/namespace).

#### Option A – Via CLI

````bash
argocd app create my-app \
  --repo https://charts.example.com \
  --helm-chart my-chart \
  --revision 1.2.0 \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace my-namespace \
  --helm-set replicaCount=3 \
  --helm-set image.tag=latest \
  --values-literal-file values-override.yaml \
  --sync-policy automated \
  --auto-prune \
  --self-heal
````

#### Option B – Via Declarative YAML (Recommended for GitOps)

````yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default

  source:
    # --- HELM REPO SOURCE ---
    repoURL: https://charts.example.com    # Helm repo URL or Git repo URL
    chart: my-chart                         # Chart name (only for Helm repos)
    targetRevision: 1.2.0                   # Chart version / Git branch / tag

    helm:
      # Option 1: Inline value overrides
      parameters:
        - name: replicaCount
          value: "3"
        - name: image.tag
          value: "latest"

      # Option 2: Values file(s) from the repo
      valueFiles:
        - values-production.yaml

      # Option 3: Inline values block
      values: |
        service:
          type: LoadBalancer
          port: 443
        ingress:
          enabled: true

      # Option 4: Release name override
      releaseName: my-custom-release

      # Option 5: Skip CRDs
      skipCrds: false

      # Option 6: Pass --set-string for string-typed values
      # (prevents Helm from interpreting "true" as boolean)

  destination:
    server: https://kubernetes.default.svc  # Target cluster API server
    namespace: my-namespace                  # Target namespace

  # --- SYNC POLICY OPTIONS ---
  syncPolicy:
    automated:
      prune: true          # Delete resources removed from Git
      selfHeal: true       # Re-apply if someone manually changes cluster state
      allowEmpty: false     # Prevent sync if app has zero resources

    syncOptions:
      - CreateNamespace=true          # Auto-create namespace if missing
      - PrunePropagationPolicy=foreground  # How deletes propagate
      - PruneLast=true                # Prune after all other syncs
      - ApplyOutOfSyncOnly=true       # Only apply changed resources
      - ServerSideApply=true          # Use server-side apply
      - RespectIgnoreDifferences=true # Honor ignoreDifferences during sync
      - Replace=false                 # Use replace instead of apply (dangerous)
      - Validate=true                 # Validate manifests before applying

    retry:
      limit: 5             # Number of sync retries
      backoff:
        duration: 5s       # Initial retry delay
        factor: 2          # Multiplier per retry
        maxDuration: 3m    # Max retry delay

  # --- IGNORE DIFFERENCES ---
  # Tells ArgoCD to ignore certain fields that change at runtime
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas    # Ignore if HPA changes replica count

  # --- HEALTH CHECKS ---
  # ArgoCD has built-in health checks, but you can add custom ones

  # --- REVISION HISTORY LIMIT ---
  revisionHistoryLimit: 10
````

Apply it:
````bash
kubectl apply -f argocd-application.yaml
````

#### Option C – Via the ArgoCD UI

1. Click **"+ New App"**
2. Fill in:
   - **Application Name**: `my-app`
   - **Project**: `default`
   - **Sync Policy**: `Manual` or `Automatic`
   - **Repository URL**: your Helm repo or Git repo
   - **Chart**: select from dropdown (Helm repos) or enter path (Git repos)
   - **Target Revision**: chart version or branch
   - **Cluster URL**: `https://kubernetes.default.svc`
   - **Namespace**: `my-namespace`
   - **Helm values**: override in the UI form
3. Click **"Create"**

---

### Step 5: Sync the Application

If you didn't set `automated` sync policy:

````bash
# Manual sync
argocd app sync my-app

# Sync with prune (delete removed resources)
argocd app sync my-app --prune

# Sync specific resources only
argocd app sync my-app --resource apps:Deployment:my-deployment

# Dry run (preview what would happen)
argocd app sync my-app --dry-run

# Force sync (re-apply even if in sync)
argocd app sync my-app --force
````

#### Via the ArgoCD UI

1. Open the ArgoCD dashboard and click on your application (e.g., `my-app`)
2. You will see the **application detail view** showing the current sync status
3. To **sync manually**:
   - Click the **"Sync"** button at the top of the application view
   - A sync panel will appear with the following options:
     - **Revision**: Choose the target revision (branch, tag, or commit) — leave as `HEAD` for latest
     - **Prune**: Check this to delete resources that no longer exist in Git
     - **Dry Run**: Check this to preview changes without actually applying them
     - **Apply Only**: Check this to skip the pre-sync/post-sync hooks
     - **Force**: Check this to force-apply resources even if they are already in sync
   - Select/deselect individual resources in the resource list if you want a **partial sync**
   - Click **"Synchronize"** to start the sync
4. Watch the sync progress in real time — resources will transition through states:
   - **OutOfSync** (yellow) → **Syncing** (blue spinner) → **Synced** (green checkmark)
5. If sync fails, click on the failed resource to see the **error details** and **events**

> **Tip:** You can also click the **three-dot menu** on any individual resource card and select **"Sync"** to sync just that one resource.

### Step 6: Monitor & Verify

````bash
# Check app status
argocd app get my-app

# Watch for real-time sync status
argocd app wait my-app --health --timeout 300

# View app logs
argocd app logs my-app

# List all apps
argocd app list

# View detailed diff (what's different between Git and cluster)
argocd app diff my-app
````

#### Via the ArgoCD UI

1. **Application List View** — After logging in, the main dashboard shows all applications with:
   - **Sync Status**: `Synced` (green), `OutOfSync` (yellow), or `Unknown` (grey)
   - **Health Status**: `Healthy` (green heart), `Progressing` (blue spinner), `Degraded` (red heart), `Missing` (yellow)
   - Click any app card to drill into its details

2. **Application Detail View** — Click on an application to see:
   - **Resource Tree**: A visual graph of all Kubernetes resources (Deployments, Services, Pods, etc.) and their relationships. Hover over any node to see its status.
   - **Resource List**: Toggle to a flat list view of all resources with their sync/health status
   - **Diff View**: Click **"App Diff"** button to see what differs between the Git desired state and the live cluster state
   - **History & Rollback**: Click **"History"** tab to see past sync operations, their revisions, and timestamps. You can rollback to any previous revision from here.
   - **Events**: Click on any resource, then check the **"Events"** tab for Kubernetes events (pull errors, crash loops, scheduling failures, etc.)
   - **Logs**: Click on any **Pod** resource, then click the **"Logs"** tab to stream live container logs directly in the UI
   - **Parameters**: Click the **"Parameters"** tab at the top to review the Helm values and parameters currently applied

3. **Refresh vs. Hard Refresh**:
   - **Refresh** (circular arrow icon): Re-fetches the Git/Helm source and re-compares with the cluster
   - **Hard Refresh**: Additionally clears the manifest cache, useful if ArgoCD shows stale data

4. **Application Health Summary**: The top banner shows an aggregated health status. If any resource is `Degraded`, the entire app shows as `Degraded` — click on the red resource to investigate.

> **Tip:** Use the **search/filter bar** at the top of the dashboard to filter apps by project, sync status, health status, or labels when you have many applications.

---

## All Available ArgoCD Application Options Explained

| Option | Description |
|--------|-------------|
| `source.repoURL` | URL of Helm repo or Git repo |
| `source.chart` | Helm chart name (Helm repo only) |
| `source.targetRevision` | Chart version, Git branch, tag, or commit SHA |
| `source.path` | Path within Git repo to the chart (Git source only) |
| `source.helm.parameters` | Key-value overrides (like `--set`) |
| `source.helm.valueFiles` | List of values files from the repo |
| `source.helm.values` | Inline YAML values block |
| `source.helm.releaseName` | Override Helm release name |
| `source.helm.skipCrds` | Skip installing CRDs |
| `source.helm.passCredentials` | Pass repo credentials to Helm for chart downloads |
| `destination.server` | Target Kubernetes cluster API |
| `destination.namespace` | Target namespace |
| `syncPolicy.automated.prune` | Auto-delete resources no longer in Git |
| `syncPolicy.automated.selfHeal` | Auto-fix manual changes on cluster |
| `syncPolicy.syncOptions` | Fine-grained sync behaviors (see table below) |
| `syncPolicy.retry` | Retry config for failed syncs |
| `ignoreDifferences` | Fields to ignore during diff comparison |
| `revisionHistoryLimit` | Max number of stored revisions |
| `project` | ArgoCD project for RBAC/policies |

### Sync Options Reference

| Sync Option | Description |
|-------------|-------------|
| `CreateNamespace=true` | Create target namespace if it doesn't exist |
| `PruneLast=true` | Run prune as the last step |
| `ApplyOutOfSyncOnly=true` | Only apply resources that are out of sync |
| `ServerSideApply=true` | Use Kubernetes server-side apply |
| `Replace=true` | Use `kubectl replace` instead of `apply` |
| `FailOnSharedResource=true` | Fail if a resource is managed by another app |
| `Validate=true` | Validate manifests before applying |

---

## Lifecycle Flow Diagram

```
Git Repo / Helm Repo
        │
        ▼
  ArgoCD detects change
        │
        ▼
  Render Helm templates (helm template)
        │
        ▼
  Compare with live cluster state (diff)
        │
        ▼
  ┌─────────────┐
  │ Out of Sync? │──No──▶ Done (Healthy ✓)
  └──────┬──────┘
         │ Yes
         ▼
  ┌──────────────────┐
  │ Automated sync?  │──No──▶ Wait for manual sync
  └──────┬───────────┘
         │ Yes
         ▼
   Apply to cluster (kubectl apply)
         │
         ▼
   Health check (Progressing → Healthy ✓)
         │
         ▼
   Prune removed resources (if prune=true)
```