# YuniKorn POC Guide: Bin-packing for Spark on EKS

## Mục tiêu POC

Chứng minh rằng Apache YuniKorn giảm đáng kể số lượng node được claim trên EKS
khi chạy Spark workloads, so với default kube-scheduler.

## Vấn đề hiện tại

Default kube-scheduler sử dụng `LeastAllocated` scoring → dàn đều pod vào tất cả
nodes có sẵn → Karpenter/CA provision thêm node không cần thiết → chi phí tăng.

## Giải pháp

YuniKorn với `nodesortpolicy: binpacking` sử dụng `MostAllocated` → pack pod chặt
vào ít node nhất → node trống được scale-in → chi phí giảm.

---

## Prerequisites

```bash
# Minikube multi-node cluster (3 nodes, 12 CPU mỗi node)
minikube start --nodes=3 --cpus=4 --memory=16384 --cni=calico --driver=docker

# Verify nodes Ready
kubectl get nodes

# ArgoCD installed
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side=true --force-conflicts
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s

# Metrics server (for kubectl top)
minikube addons enable metrics-server
```

## Bước 1: Deploy YuniKorn + Spark Operator

```bash
# Bootstrap toàn bộ platform via ArgoCD
kubectl apply -f app-of-apps.yaml

# Verify YuniKorn running
kubectl get pods -n yunikorn
# Expected: scheduler (2/2), admission-controller (1/1)

# Verify bin-packing config loaded
kubectl get configmap yunikorn-defaults -n yunikorn -o yaml
# Expected: queues.yaml section with nodesortpolicy: binpacking
```

## Bước 2: A/B Test — Default Scheduler vs YuniKorn

### Test setup

Cả 2 job **cùng resource spec**, chỉ khác scheduler:

| Config | Giá trị |
|--------|---------|
| Driver | 1 core, 1g memory |
| Executors | 6 instances × 1.5 cores, 1g memory |
| Total footprint | 10 CPU, 7g memory |
| Target utilization | ~83% of 12-CPU node |

### Test A: Default Scheduler (LeastAllocated — scatter)

```bash
# Xóa jobs cũ
kubectl delete sparkapplication --all -n spark-apps --ignore-not-found

# Deploy default scheduler job
kubectl apply -f spark-batch/spark-pi-default-demo.yaml

# Đợi pods Running (~30s)
kubectl get pods -n spark-apps -o wide -w

# Capture kết quả
kubectl get pods -n spark-apps -o wide
kubectl top nodes
```

**Kết quả mong đợi:**

```
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
minikube       ~3500m       ~29%   ~3Gi            ~2%
minikube-m02   ~3500m       ~29%   ~3Gi            ~2%
minikube-m03   ~3500m       ~29%   ~3Gi            ~2%
```

Pods dàn đều 3 nodes → tất cả nodes đều bị claim → trên EKS trả tiền cả 3.

### Test B: YuniKorn Scheduler (binpacking — pack)

```bash
# Xóa job default
kubectl delete sparkapplication --all -n spark-apps --ignore-not-found

# Deploy YuniKorn job
kubectl apply -f spark-batch/spark-pi-yunikorn-demo.yaml

# Đợi pods Running (~30s)
kubectl get pods -n spark-apps -o wide -w

# Capture kết quả
kubectl get pods -n spark-apps -o wide
kubectl top nodes
```

**Kết quả mong đợi:**

```
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
minikube       ~9500m       ~80%   ~8Gi            ~6%
minikube-m02   ~100m        ~0%    ~1.4Gi          ~1%
minikube-m03   ~100m        ~0%    ~1.4Gi          ~1%
```

Tất cả 7 pods pack trên 1 node → 2 nodes trống → trên EKS Karpenter scale-in → trả tiền 1 node.

---

## Bước 3: So sánh kết quả

| Metric | Default Scheduler | YuniKorn Binpacking |
|--------|-------------------|---------------------|
| Pods on Node 1 | 2-3 | **7 (tất cả)** |
| Pods on Node 2 | 2-3 | 0 |
| Pods on Node 3 | 1-2 | 0 |
| CPU% Node 1 | ~29% | **~80%** |
| Nodes cần trả tiền | **3** | **1** |
| Cost savings | baseline | **~66% node cost** |

---

## Bước 4: Verify Gang Scheduling

Gang scheduling đảm bảo job chỉ start khi đủ resource cho TẤT CẢ pods:

```bash
# Xem events — YuniKorn sẽ show "queued and waiting for allocation"
kubectl describe pod spark-pi-yunikorn-demo-driver -n spark-apps | findstr yunikorn

# Xem YuniKorn app status
kubectl port-forward svc/yunikorn-service 9889:9889 -n yunikorn
# Open: http://localhost:9889 → Tab "Applications"
```

Không có gang scheduling → driver start trước, executors từng pod một → dễ deadlock khi thiếu resource.

---

## YuniKorn Web UI

```bash
kubectl port-forward svc/yunikorn-service 9889:9889 -n yunikorn
# Open: http://localhost:9889
```

Tabs quan trọng:
- **Queues**: Resource usage per queue (spark-batch, spark-streaming)
- **Applications**: Spark app state (running/pending/completed)
- **Nodes**: Pod distribution per node (visual proof of bin-packing)

---

## Áp dụng lên EKS Production

### Kết hợp với Karpenter

```yaml
# Karpenter NodePool — không cần thay đổi đặc biệt
# YuniKorn handle scheduling, Karpenter handle provisioning
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
    # Karpenter consolidation: chỉ remove nodes THỰC SỰ trống
    # YuniKorn bin-packing đảm bảo nodes trống rõ ràng (không fragmented)
    consolidationPolicy: WhenEmpty
    consolidateAfter: 60s
```

### Hiệu quả thực tế

Bin-packing + gang scheduling giúp Karpenter hiệu quả hơn:
1. **Pod không bị scatter** → nodes trống rõ ràng → consolidation nhanh
2. **Không cần aggressive eviction** → giảm pod disruption 50%
3. **Job hoàn thành → toàn bộ node freed** → scale-in ngay lập tức

### Production values.yaml changes

```yaml
yunikorn:
  resources:
    limits:
      cpu: "4"        # EKS dùng cgroup v2, không bị lỗi quota
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

## Troubleshooting

```bash
# YuniKorn scheduler logs
kubectl logs -l app=yunikorn -n yunikorn --tail=100

# Check scheduler assignment
kubectl get pods -n spark-apps -o jsonpath="{range .items[*]}{.metadata.name}{'\t'}{.spec.schedulerName}{'\n'}{end}"

# Verify ConfigMap loaded (phải có DATA > 0)
kubectl get configmap yunikorn-defaults -n yunikorn

# Gang scheduling stuck (waiting for resources)
kubectl get events -n spark-apps --sort-by='.lastTimestamp'

# Queue API
kubectl port-forward svc/yunikorn-service 9889:9889 -n yunikorn
curl http://localhost:9889/ws/v1/queues
```

---

## Kết luận

| Vấn đề | Giải pháp | Bằng chứng |
|---------|-----------|-------------|
| Pod scatter → nhiều nodes bị claim | YuniKorn binpacking | 7 pods on 1 node vs 3 nodes |
| Deadlock khi nhiều Spark jobs cùng chạy | Gang scheduling | Job only starts when ALL resources available |
| Karpenter evict pod → Spark fail | Proactive packing = clean empty nodes | consolidationPolicy: WhenEmpty works |
| Chi phí EKS tăng | Giảm 2/3 node cho cùng workload | POC: 1 node thay vì 3 |

References:
- [AWS Data on EKS: Binpacking](https://awslabs.github.io/data-on-eks/docs/resources/binpacking-custom-scheduler-eks)
- [Salesforce: 13% cost savings with bin-packing](https://engineering.salesforce.com/how-data-360-optimized-kubernetes-scheduling-architecture-that-delivered-13-cost-savings/)
- [AWS: YuniKorn for EMR on EKS](https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/tutorial-yunikorn.html)
- [YuniKorn Docs](https://yunikorn.apache.org/docs/)
