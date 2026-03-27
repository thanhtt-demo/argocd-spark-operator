# ArgoCD + Spark Operator - Production-like Setup

Deploy and manage Apache Spark batch & streaming applications on Kubernetes using **ArgoCD** (GitOps) and **Kubeflow Spark Operator**.

Follows the [argocd-example-apps](https://github.com/argoproj/argocd-example-apps) Helm-based App-of-Apps pattern.

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                        ArgoCD (GitOps)                       │
│                                                              │
│  ┌─────────────┐   Helm App-of-Apps (apps/ chart)            │
│  │ spark-       │──► spark-namespaces    (sync-wave: 1)      │
│  │ platform     │──► spark-operator      (sync-wave: 2, Helm)│
│  │ (root app)   │──► spark-rbac          (sync-wave: 3)      │
│  │              │──► spark-batch-apps    (sync-wave: 4)      │
│  │              │──► spark-streaming-apps (sync-wave: 4)      │
│  └─────────────┘                                             │
└──────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
   ┌──────────────┐  ┌──────────────┐  ┌─────────────┐
   │ spark-operator│  │  spark-apps  │  │  spark-apps  │
   │  namespace    │  │  (batch)     │  │  (streaming) │
   │              │  │              │  │              │
   │ Spark        │  │ SparkPi      │  │ Rate         │
   │ Operator     │  │ WordCount    │  │ Structured   │
   │ (controller) │  │ Scheduled    │  │ Streaming    │
   └──────────────┘  └──────────────┘  └─────────────┘
```

## Project Structure

```
argocd-spark-operator/
├── app-of-apps.yaml                       # Bootstrap: apply once to deploy everything
├── apps/                                  # Helm chart (App-of-Apps) - generates Application CRDs
│   ├── Chart.yaml                         # Helm chart metadata
│   ├── values.yaml                        # All applications defined here (single source of truth)
│   └── templates/
│       ├── project.yaml                   # Generates AppProject
│       └── applications.yaml              # Loops over values to generate Applications
├── namespaces/                            # Wave 1: Namespace manifests
│   ├── spark-operator-ns.yaml
│   └── spark-apps-ns.yaml
├── rbac/                                  # Wave 3: RBAC manifests
│   ├── spark-service-account.yaml
│   ├── spark-role.yaml
│   └── spark-role-binding.yaml
├── spark-batch/                           # Wave 4: Batch SparkApplications
│   ├── spark-pi-batch.yaml
│   ├── word-count-batch.yaml
│   └── spark-pi-scheduled.yaml
├── spark-streaming/                       # Wave 4: Streaming SparkApplications
│   ├── configmap-streaming-code.yaml
│   └── rate-structured-streaming.yaml
└── README.md
```

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Helm App-of-Apps** | Helm chart in `apps/` auto-generates all child Applications from `values.yaml` |
| **Sync Waves** | Ordered deployment: namespaces → operator → RBAC → apps |
| **Spark Operator** | Kubeflow controller that manages SparkApplication CRDs |
| **SparkApplication** | CRD for one-time batch Spark jobs |
| **ScheduledSparkApplication** | CRD for cron-based recurring Spark jobs |
| **Structured Streaming** | Long-running SparkApplication with `restartPolicy: Always` |

---

## Prerequisites

### Install Chocolatey (Windows)

Open PowerShell as **Administrator**:

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

### Install Tools

```powershell
choco install k9s
```

- [Minikube Installation](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download)
- [ArgoCD CLI](https://github.com/argoproj/argo-cd/releases/tag/v2.13.2) - download `argocd-windows-amd64.exe` and add to PATH

### Start Minikube

```powershell
minikube start --driver=docker --cpus=4 --memory=8192
```

### Create kubectl alias

```powershell
Set-Alias k kubectl.exe
```

---

## Step 1: Install ArgoCD

```powershell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Wait for ArgoCD to be ready

```powershell
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s
```

### Get ArgoCD admin password

```powershell
argocd admin initial-password -n argocd
```

### Access ArgoCD UI

```powershell
kubectl port-forward svc/argocd-server 8080:443 -n argocd
```

Open browser: `https://localhost:8080` (user: `admin`, password from above)

### Login via CLI

```powershell
argocd login localhost:8080
```

---

## Step 2: Configure Your Git Repository

> **IMPORTANT**: Replace the `repoURL` in `apps/values.yaml` and `app-of-apps.yaml` with your actual GitHub repo URL.

Files that need updating:
- `app-of-apps.yaml` → `spec.source.repoURL`
- `apps/values.yaml` → `config.source.repoURL`

### Push to Git

```powershell
git init
git add .
git commit -m "feat: initial spark operator + argocd setup"
git remote add origin https://github.com/thanhtt-demo/argocd-spark-operator.git
git push -u origin main
```

---

## Step 3: Deploy Everything via App-of-Apps

```powershell
kubectl apply -f app-of-apps.yaml
```

This single command triggers the entire deployment chain:

1. **Wave 1** - Creates `spark-operator` and `spark-apps` namespaces
2. **Wave 2** - Installs Spark Operator via Helm chart (with webhook, metrics)
3. **Wave 3** - Sets up RBAC (ServiceAccount, Role, RoleBinding) in `spark-apps`
4. **Wave 4** - Deploys batch and streaming SparkApplications

---

## Step 4: Verify Deployment

### Check ArgoCD applications

```powershell
argocd app list
# or
kubectl get applications -n argocd
```

### Check Spark Operator

```powershell
kubectl get pods -n spark-operator
kubectl get crd sparkapplications.sparkoperator.k8s.io
```

### Check Spark Applications

```powershell
kubectl get sparkapplication -n spark-apps
kubectl get scheduledsparkapplication -n spark-apps
kubectl get pods -n spark-apps
```

### Check batch job status

```powershell
kubectl describe sparkapplication spark-pi-batch -n spark-apps
kubectl describe sparkapplication word-count-batch -n spark-apps
```

### Check streaming job status

```powershell
kubectl describe sparkapplication rate-structured-streaming -n spark-apps
```

### View driver logs

```powershell
# Batch job logs
kubectl logs spark-pi-batch-driver -n spark-apps

# Streaming job logs
kubectl logs rate-structured-streaming-driver -n spark-apps
```

---

## Step 5: Access Spark UI

```powershell
# Forward Spark UI for a running application
kubectl port-forward svc/spark-pi-batch-ui-svc 4040:4040 -n spark-apps
```

Open browser: `http://localhost:4040`

---

## Spark Applications Included

### Batch Applications

| Application | Type | Description |
|------------|------|-------------|
| `spark-pi-batch` | Scala | Computes Pi using Monte Carlo method (2 executors) |
| `word-count-batch` | Python | Classic word count on a local text file |
| `spark-pi-scheduled` | Scala (Cron) | SparkPi running every 6 hours via ScheduledSparkApplication |

### Streaming Applications

| Application | Type | Description |
|------------|------|-------------|
| `rate-structured-streaming` | Python | Structured Streaming with rate source, windowed aggregation |

---

## Production Best Practices Applied

- **GitOps**: All configuration in Git, deployed via ArgoCD
- **App-of-Apps**: Single root application manages entire platform
- **Sync Waves**: Ordered deployment prevents race conditions
- **Auto Sync + Self-Heal**: ArgoCD automatically reconciles drift
- **Webhook**: Pod customization via mutating admission webhook
- **Security Context**: Non-root, dropped capabilities, seccomp profile
- **Resource Limits**: CPU/memory requests and limits on operator and apps
- **Restart Policy**: Configurable retry logic for failed jobs
- **Metrics**: Prometheus-compatible metrics from Spark Operator
- **RBAC**: Least-privilege service accounts for Spark drivers
- **Namespace Isolation**: Separate namespaces for operator and applications

---

## Customization

### Change Spark version

Update `image` and `sparkVersion` in SparkApplication manifests:
```yaml
spec:
  image: docker.io/library/spark:4.0.1
  sparkVersion: "4.0.1"
```

### Add your own Spark application

1. Create a new `SparkApplication` YAML in `spark-batch/` or `spark-streaming/`
2. Add the app folder to `apps/values.yaml` if it's a new folder
3. Commit and push to Git → ArgoCD auto-deploys

### Scale executors

```yaml
spec:
  executor:
    instances: 5      # number of executors
    cores: 2          # cores per executor
    memory: "2g"      # memory per executor
```

### Use custom Docker image

```yaml
spec:
  image: your-registry.com/your-spark-app:v1.0
  mainApplicationFile: local:///opt/spark/app/your-app.py
```

---

## Troubleshooting

### Spark Operator not starting

```powershell
kubectl get pods -n spark-operator
kubectl logs -l app.kubernetes.io/name=spark-operator -n spark-operator
```

### SparkApplication stuck in PENDING

```powershell
kubectl describe sparkapplication <app-name> -n spark-apps
kubectl get events -n spark-apps --sort-by='.lastTimestamp'
```

### Driver pod failing

```powershell
kubectl logs <app-name>-driver -n spark-apps
kubectl describe pod <app-name>-driver -n spark-apps
```

### ArgoCD sync issues

```powershell
argocd app get spark-platform
argocd app sync spark-platform
```

---

## Cleanup

### Delete all Spark applications

```powershell
argocd app delete spark-platform --cascade
```

### Full cleanup

```powershell
kubectl delete namespace spark-apps
kubectl delete namespace spark-operator
minikube stop
```

---

## References

- [Kubeflow Spark Operator](https://github.com/kubeflow/spark-operator)
- [Spark Operator User Guide](https://www.kubeflow.org/docs/components/spark-operator/user-guide/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ArgoCD App-of-Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [SMACAcademy ArgoCD Course](https://github.com/SMACAcademy/ArgoCD-Complete-Master-Course/tree/main)
- [Udemy ArgoCD Master Course](https://ascend.udemy.com/course/argo-cd-master-course-expert-techniques-in-gitops-devops/learn/lecture/41742588#overview)
