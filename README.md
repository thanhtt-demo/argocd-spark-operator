# ArgoCD + Spark Operator - Production-like Setup

Deploy and manage Apache Spark batch & streaming applications on Kubernetes using **ArgoCD** (GitOps) and **Kubeflow Spark Operator**.

Follows the [argocd-example-apps](https://github.com/argoproj/argocd-example-apps) Helm-based App-of-Apps pattern.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          ArgoCD (GitOps)                            │
│                                                                     │
│  ┌─────────────┐   Helm App-of-Apps (apps/ chart)                   │
│  │ spark-       │──► spark-namespaces     (sync-wave: 1)            │
│  │ platform     │──► dagster              (sync-wave: 2, Helm)      │
│  │ (root app)   │──► keycloak             (sync-wave: 2, Helm)      │
│  │              │──► oauth2-proxy         (sync-wave: 3, Helm)      │
│  └─────────────┘                                                    │
└─────────────────────────────────────────────────────────────────────┘
                                │
            ┌───────────────────┼───────────────────┐
            ▼                   ▼                   ▼
  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
  │ dagster namespace │ │keycloak namespace│ │ dagster namespace │
  │                  │ │                  │ │                  │
  │ Dagster          │ │ Keycloak         │ │ OAuth2 Proxy     │
  │ (webserver,      │ │ (IAM / OIDC      │ │ (auth gateway,   │
  │  daemon, etc.)   │ │  provider)       │ │  email whitelist) │
  └──────────────────┘ └──────────────────┘ └──────────────────┘

  Browser ──► OAuth2 Proxy (:4180) ──► Keycloak login (:9090)
         ◄── authenticated ──────────► Dagster UI (upstream)
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
├── dagster/                               # Wave 2: Dagster orchestration platform (umbrella chart)
│   ├── Chart.yaml
│   └── values.yaml
├── keycloak/                              # Wave 2: Keycloak IAM (umbrella chart)
│   ├── Chart.yaml
│   └── values.yaml
├── oauth2-proxy/                          # Wave 3: OAuth2 Proxy for Dagster authentication
│   ├── Chart.yaml                         # Umbrella chart wrapping oauth2-proxy
│   ├── values.yaml                        # Keycloak OIDC + email whitelist config
│   ├── secret-example.yaml               # Example secret (DO NOT apply directly)
│   └── templates/
│       └── configmap-allowed-emails.yaml  # Email whitelist (add/remove users here)
├── namespaces/                            # Wave 1: Namespace manifests
│   ├── spark-operator-ns.yaml
│   ├── spark-apps-ns.yaml
│   └── dagster-ns.yaml
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
| **OAuth2 Proxy** | Reverse proxy that authenticates Dagster UI access via Keycloak OIDC |
| **Email Whitelist** | ConfigMap-based access control — only listed emails can access Dagster |

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
minikube start --driver=docker
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

1. **Wave 1** - Creates namespaces (`spark-operator`, `spark-apps`, `dagster`)
2. **Wave 2** - Installs Dagster (umbrella Helm chart) and Keycloak (umbrella Helm chart)
3. **Wave 3** - Deploys OAuth2 Proxy (umbrella Helm chart) in `dagster` namespace

> **NOTE**: Spark Operator, RBAC, and Spark Applications are defined in `apps/values.yaml` but currently commented out. Uncomment them to enable Spark workloads.

---

## Step 3a: Configure Keycloak for Dagster Authentication

OAuth2 Proxy authenticates Dagster UI users via Keycloak OIDC. This requires a one-time manual setup in Keycloak.

### 3a.1: Access Keycloak Admin Console

```powershell
kubectl port-forward svc/keycloak 9090:8080 -n keycloak
```

Open browser: `http://localhost:9090` (user: `admin`, password: `admin`)

### 3a.2: Create Realm

1. Click **Keycloak** dropdown (top-left) → **Create realm**
2. Realm name: `dagster` → **Create**

### 3a.3: Create OIDC Client

1. Navigate to **Clients** → **Create client**
2. Client type: **OpenID Connect**
3. Client ID: `dagster-oauth2-proxy` → **Next**
4. Client authentication: **ON**
5. Authentication flow: **Standard flow** selected, **Direct access grants** deselected → **Next**
6. Settings / Access settings:
   - Valid redirect URIs: `http://localhost:4180/oauth2/callback`
   - Web Origins: `http://localhost:4180`
7. **Save**

### 3a.4: Create Audience Mapper

1. Navigate to **Clients** → `dagster-oauth2-proxy` → **Client scopes**
2. Click `dagster-oauth2-proxy-dedicated`
3. **Configure a new mapper** → select **Audience**
   - Name: `aud-mapper-dagster`
   - Included Client Audience: `dagster-oauth2-proxy`
   - Add to ID token: **ON**
   - Add to access token: **ON**
4. **Save**

### 3a.5: Copy Client Secret

1. Navigate to **Clients** → `dagster-oauth2-proxy` → **Credentials** tab
2. Copy the **Client secret** value (you'll need it in the next step)

### 3a.6: Create Test Users

1. Navigate to **Users** → **Add user**
2. Fill in: Username, Email, First name, Last name
3. Set **Email verified**: **ON** → **Create**
4. Go to **Credentials** tab → **Set password** → uncheck **Temporary** → **Save**
5. Repeat for additional users

> **IMPORTANT**: The user's email must match an entry in `oauth2-proxy/configmap-allowed-emails.yaml` to be allowed access.

### 3a.7: Create Kubernetes Secret for OAuth2 Proxy

```powershell
# Generate a cookie secret
python -c "import os,base64; print(base64.urlsafe_b64encode(os.urandom(32)).decode())"

# Create the secret (replace <VALUES> with actual values)
kubectl create secret generic oauth2-proxy-secret `
  --namespace dagster `
  --from-literal=client-id=dagster-oauth2-proxy `
  --from-literal=client-secret=<YOUR_KEYCLOAK_CLIENT_SECRET> `
  --from-literal=cookie-secret=<GENERATED_COOKIE_SECRET>
```

> **NOTE**: Never commit real secrets to Git. See `oauth2-proxy/secret-example.yaml` for reference. For production, use Sealed Secrets or External Secrets Operator.

### 3a.8: Manage User Access (Email Whitelist)

Edit `oauth2-proxy/templates/configmap-allowed-emails.yaml` to control who can access Dagster:

```yaml
data:
  allowed-emails.txt: |
    alice@company.com
    bob@company.com
```

| User | Email | In whitelist? | Access |
|------|-------|---------------|--------|
| Alice | alice@company.com | Yes | **Allowed** |
| Charlie | charlie@company.com | No | **Denied (403)** |

Push to Git → ArgoCD auto-syncs → access updated immediately.

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

## Step 6: Access Dagster UI (via OAuth2 Proxy)

### Port-forward OAuth2 Proxy

```powershell
kubectl port-forward svc/oauth2-proxy 4180:4180 -n dagster
```

### Port-forward Keycloak (required for login redirect)

```powershell
kubectl port-forward svc/keycloak 9090:8080 -n keycloak
```

### Open Dagster

Open browser: `http://localhost:4180`

You will be redirected to Keycloak login at `http://localhost:9090`. After authentication, you'll be proxied to Dagster UI.

> **NOTE**: Both port-forwards (OAuth2 Proxy on 4180 and Keycloak on 9090) must be running simultaneously.

### Verify OAuth2 Proxy

```powershell
# Check oauth2-proxy pod
kubectl get pods -n dagster | Select-String oauth2-proxy

# Check oauth2-proxy logs
kubectl logs -l app.kubernetes.io/name=oauth2-proxy -n dagster
```

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
- **Authentication**: Dagster UI protected by OAuth2 Proxy + Keycloak OIDC
- **Email Whitelist**: GitOps-managed access control via ConfigMap

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

### OAuth2 Proxy not starting

```powershell
kubectl get pods -n dagster | Select-String oauth2-proxy
kubectl logs -l app.kubernetes.io/name=oauth2-proxy -n dagster
kubectl describe pod -l app.kubernetes.io/name=oauth2-proxy -n dagster
```

### OAuth2 Proxy returns 403 Forbidden

User's email is not in the whitelist. Add it to `oauth2-proxy/configmap-allowed-emails.yaml` and push to Git.

### OAuth2 Proxy returns 500 / redirect loop

1. Verify Keycloak is reachable from within the cluster:
   ```powershell
   kubectl run curl-test --rm -it --image=curlimages/curl -- curl -s http://keycloak.keycloak.svc.cluster.local:8080/realms/dagster/.well-known/openid-configuration
   ```
2. Verify the secret exists and has correct values:
   ```powershell
   kubectl get secret oauth2-proxy-secret -n dagster -o jsonpath='{.data}'
   ```
3. Check oauth2-proxy logs for specific errors:
   ```powershell
   kubectl logs -l app.kubernetes.io/name=oauth2-proxy -n dagster --tail=50
   ```

### OAuth2 Proxy: issuer mismatch error

If you see `id token issued by a different provider`, this is expected in a port-forward setup. The `insecure_oidc_skip_issuer_verification = true` setting in `oauth2-proxy/values.yaml` handles this.

### OAuth2 Proxy: 401 on userinfo endpoint

If you see `failed to fetch claims from profile URL: unexpected status "401"`, the `skip_claims_from_profile_url = true` setting handles this. The email claim is read from the ID token directly.

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
- [OAuth2 Proxy Documentation](https://oauth2-proxy.github.io/oauth2-proxy/)
- [OAuth2 Proxy Keycloak OIDC Provider](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/keycloak_oidc)
- [Keycloak Documentation](https://www.keycloak.org/documentation)
- [SMACAcademy ArgoCD Course](https://github.com/SMACAcademy/ArgoCD-Complete-Master-Course/tree/main)
- [Udemy ArgoCD Master Course](https://ascend.udemy.com/course/argo-cd-master-course-expert-techniques-in-gitops-devops/learn/lecture/41742588#overview)
