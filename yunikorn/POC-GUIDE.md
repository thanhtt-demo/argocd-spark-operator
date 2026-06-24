# YuniKorn POC Guide: Bin-packing for Spark on Kubernetes

## Objective

Prove that Apache YuniKorn reduces the number of nodes claimed on EKS when running
Spark workloads, compared to the default kube-scheduler.

## Problem Statement

The default kube-scheduler uses `LeastAllocated` scoring — it spreads pods evenly
across all available nodes. For bursty Spark workloads this causes:

1. **Pod scatter**: Executors land on many nodes with low utilization each
2. **Node fragmentation**: After jobs finish, nodes remain partially occupied
3. **Cost increase**: Karpenter/CA cannot scale-in fragmented nodes → you pay for idle capacity

## Solution

YuniKorn with `nodesortpolicy: binpacking` uses `MostAllocated` scoring — it packs
pods tightly onto the fewest nodes possible. Combined with gang scheduling, Spark
jobs start atomically and land on minimal nodes.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Namespace: spark-apps              (YuniKorn scheduler) │
│  → Pods bin-packed onto fewest nodes                     │
│  → Gang scheduling: all-or-nothing start                 │
├─────────────────────────────────────────────────────────┤
│  Namespace: spark-apps-default      (kube-scheduler)     │
│  → YuniKorn admission controller BYPASSES this namespace │
│  → Pods use default LeastAllocated (spread)              │
└─────────────────────────────────────────────────────────┘
```

This separation allows a fair A/B comparison on the same cluster without
disabling any components.

---

## Prerequisites

```bash
# Minikube multi-node cluster (3 nodes, 12 CPU each)
minikube start --nodes=3 --cpus=4 --memory=16384 --cni=calico --driver=docker

# Verify all nodes Ready
kubectl get nodes

# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side=true --force-conflicts
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s

# Enable metrics server
minikube addons enable metrics-server

# Bootstrap platform (YuniKorn + Spark Operator + namespaces)
kubectl apply -f app-of-apps.yaml
```

## Verify YuniKorn Deployment

```bash
# Check pods
kubectl get pods -n yunikorn
# Expected: scheduler (2/2 Running), admission-controller (1/1 Running)

# Verify bin-packing config loaded (DATA must be > 0)
kubectl get configmap yunikorn-defaults -n yunikorn
kubectl get configmap yunikorn-defaults -n yunikorn -o yaml | grep nodesortpolicy
```

---

## A/B Test: Default Scheduler vs YuniKorn

### Test Configuration

Both jobs use **identical resource specs** — only the scheduler differs:

| Config | Value |
|--------|-------|
| Driver | 1 core, 1g memory |
| Executors | 6 instances × 1.5 cores, 1g memory |
| Total footprint | 10 CPU, 7g memory |
| Target utilization | ~83% of a 12-CPU node |

### Test A: Default Scheduler (LeastAllocated — scatter)

```bash
kubectl apply -f spark-batch/spark-pi-default-demo.yaml

# Wait for pods to be Running (~30s)
kubectl get pods -n spark-apps-default -o wide -w

# Capture results
kubectl get pods -n spark-apps-default -o wide
kubectl top nodes
```

**Expected result — pods scattered across all 3 nodes:**

```
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
minikube       ~3500m       ~29%   ~3Gi            ~2%
minikube-m02   ~3500m       ~29%   ~3Gi            ~2%
minikube-m03   ~3500m       ~29%   ~3Gi            ~2%
```

```
NAME                               NODE
spark-pi-...-exec-1                minikube
spark-pi-...-exec-2                minikube-m02
spark-pi-...-exec-3                minikube-m03
spark-pi-...-exec-4                minikube
spark-pi-...-exec-5                minikube-m02
spark-pi-...-exec-6                minikube-m03
spark-pi-default-demo-driver       minikube
```

All 3 nodes are occupied → on EKS you pay for all 3.

### Test B: YuniKorn Scheduler (binpacking — pack)

```bash
kubectl apply -f spark-batch/spark-pi-yunikorn-demo.yaml

# Wait for pods to be Running (~30s)
kubectl get pods -n spark-apps -o wide -w

# Capture results
kubectl get pods -n spark-apps -o wide
kubectl top nodes
```

**Expected result — all pods packed onto 1 node:**

```
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
minikube       ~9500m       ~80%   ~8Gi            ~6%
minikube-m02   ~100m        ~0%    ~1.4Gi          ~1%
minikube-m03   ~100m        ~0%    ~1.4Gi          ~1%
```

```
NAME                                NODE
spark-pi-...-exec-1                 minikube
spark-pi-...-exec-2                 minikube
spark-pi-...-exec-3                 minikube
spark-pi-...-exec-4                 minikube
spark-pi-...-exec-5                 minikube
spark-pi-...-exec-6                 minikube
spark-pi-yunikorn-demo-driver       minikube
```

All 7 pods on 1 node → 2 nodes are empty → on EKS Karpenter scales them in → you pay for 1 node.

### Side-by-side Comparison (run both simultaneously)

```bash
# Both jobs can run at the same time on different namespaces
kubectl apply -f spark-batch/spark-pi-default-demo.yaml
kubectl apply -f spark-batch/spark-pi-yunikorn-demo.yaml

# Compare distribution
kubectl get pods -n spark-apps-default -o wide
kubectl get pods -n spark-apps -o wide
kubectl top nodes
```

---

## Results Summary

| Metric | Default Scheduler | YuniKorn Binpacking |
|--------|-------------------|---------------------|
| Pods on Node 1 | 2-3 | **7 (all)** |
| Pods on Node 2 | 2-3 | 0 |
| Pods on Node 3 | 1-2 | 0 |
| CPU utilization (max node) | ~29% | **~80%** |
| Nodes required | **3** | **1** |
| Potential cost savings | baseline | **~66%** |

---

## How It Works

### Bin-packing (nodesortpolicy)

```yaml
# yunikorn/values.yaml → yunikornDefaults → queues.yaml
nodesortpolicy:
  type: binpacking
  resourceweights:
    memory: 1
    vcore: 1
```

YuniKorn scores nodes by how full they already are (MostAllocated). Higher
utilization = higher score = pods land there first. Result: existing nodes fill
up before new nodes are provisioned.

### Gang Scheduling (task-groups)

```yaml
# SparkApplication annotations
yunikorn.apache.org/task-groups: |
  [{"name":"spark-driver","minMember":1,"minResource":{"cpu":"1","memory":"1Gi"}},
   {"name":"spark-executor","minMember":6,"minResource":{"cpu":"1500m","memory":"1Gi"}}]
yunikorn.apache.org/schedulingPolicyParameters: "gangSchedulingStyle=Hard"
```

YuniKorn reserves resources for the entire application (driver + all executors)
before starting any pod. This prevents:
- Deadlock: two jobs each get half their executors, neither can finish
- Partial scheduling: driver starts but executors can't be placed

### Namespace Bypass (A/B isolation)

```yaml
# yunikornDefaults
admissionController.filtering.bypassNamespaces: "^kube-system$,^spark-apps-default$"
```

Pods in `spark-apps-default` are NOT mutated by YuniKorn admission controller →
they use the default kube-scheduler. Pods in `spark-apps` go through YuniKorn.

---

## YuniKorn Web UI

```bash
kubectl port-forward svc/yunikorn-service 9889:9889 -n yunikorn
# Open: http://localhost:9889
```

Key tabs:
- **Queues**: Resource usage per queue (spark-batch, spark-streaming, default)
- **Applications**: Spark app state (running, pending, completed)
- **Nodes**: Pod distribution per node — visual proof of bin-packing

---

## Production Deployment on EKS

### Integration with Karpenter

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spark-workers
spec:
  template:
    metadata:
      labels:
        purpose: spark
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand", "spot"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.2xlarge", "m5.4xlarge"]
  disruption:
    # WhenEmpty works perfectly with bin-packing:
    # nodes are either fully packed or completely empty
    consolidationPolicy: WhenEmpty
    consolidateAfter: 60s
```

Why bin-packing + Karpenter works better than default scheduler + Karpenter:

1. **Clean node boundaries**: Pods packed tightly → empty nodes are truly empty → instant scale-in
2. **No aggressive eviction needed**: Reduces pod disruption by ~50%
3. **Job completion = full node freed**: All executors on same node finish together

### Production values.yaml

```yaml
yunikorn:
  resources:
    limits:
      cpu: "4"          # EKS uses cgroup v2, no quota errors
      memory: 2Gi
  yunikornDefaults:
    queues.yaml: |
      partitions:
        - name: default
          nodesortpolicy:
            type: binpacking
            resourceweights:
              memory: 1
              vcore: 1
          queues:
            - name: root
              queues:
                - name: spark-batch
                  resources:
                    max:
                      memory: 256Gi
                      vcore: 128
                - name: spark-streaming
                  resources:
                    guaranteed:
                      memory: 32Gi
                      vcore: 16
                    max:
                      memory: 128Gi
                      vcore: 64
```

---

## Cleanup

```bash
# Delete test jobs
kubectl delete sparkapplication --all -n spark-apps --ignore-not-found
kubectl delete sparkapplication --all -n spark-apps-default --ignore-not-found
```

---

## Troubleshooting

```bash
# YuniKorn scheduler logs
kubectl logs -l app=yunikorn -n yunikorn --tail=100

# Check which scheduler is assigned to pods
kubectl get pods -n spark-apps -o jsonpath="{range .items[*]}{.metadata.name}{'\t'}{.spec.schedulerName}{'\n'}{end}"

# Verify ConfigMap loaded
kubectl get configmap yunikorn-defaults -n yunikorn -o yaml

# Gang scheduling stuck (waiting for resources)
kubectl get events -n spark-apps --sort-by='.lastTimestamp'

# Queue status via REST API
kubectl port-forward svc/yunikorn-service 9889:9889 -n yunikorn
curl http://localhost:9889/ws/v1/queues
```

---

## References

- [AWS Data on EKS: Bin-packing with custom scheduler](https://awslabs.github.io/data-on-eks/docs/resources/binpacking-custom-scheduler-eks)
- [Salesforce: 13% cost savings with bin-packing (4M Spark jobs/day)](https://engineering.salesforce.com/how-data-360-optimized-kubernetes-scheduling-architecture-that-delivered-13-cost-savings/)
- [AWS: YuniKorn for EMR on EKS](https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/tutorial-yunikorn.html)
- [Kubeflow Spark Operator: YuniKorn integration](https://www.kubeflow.org/docs/components/spark-operator/user-guide/yunikorn-integration/)
- [Apache YuniKorn documentation](https://yunikorn.apache.org/docs/)
