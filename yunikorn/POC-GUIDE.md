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
# EKS cluster đã running
# kubectl + helm configured
# Spark Operator đã install
# (Optional) eks-node-viewer để visualize
go install github.com/awslabs/eks-node-viewer/cmd/eks-node-viewer@latest
```

## Bước 1: Deploy YuniKorn

```bash
# Option A: Via ArgoCD (recommended - push to git)
git add yunikorn/ namespaces/yunikorn-ns.yaml apps/
git commit -m "feat: add YuniKorn scheduler for bin-packing POC"
git push

# Option B: Manual deploy (for quick testing)
helm repo add yunikorn https://apache.github.io/yunikorn-release
helm repo update
kubectl create namespace yunikorn
helm install yunikorn yunikorn/yunikorn \
  --namespace yunikorn \
  --values yunikorn/values.yaml
```

## Bước 2: Verify YuniKorn Running

```bash
# Check pods
kubectl get pods -n yunikorn
# Expected: scheduler, admission-controller, web running

# Port-forward web UI
kubectl port-forward svc/yunikorn-service 9889:9889 -n yunikorn
# Open: http://localhost:9889
```

## Bước 3: Baseline Test (Default Scheduler)

```bash
# Chạy spark-pi-batch (default scheduler) với 5 executors
# Sửa tạm instances: 5 trong spark-pi-batch.yaml rồi apply
kubectl apply -f spark-batch/spark-pi-batch.yaml

# Quan sát pod distribution
kubectl get pods -n spark-apps -o wide
# Ghi lại: pods nằm trên bao nhiêu NODE khác nhau

# Quan sát với eks-node-viewer
eks-node-viewer --node-selector "purpose=spark"
```

## Bước 4: YuniKorn Test (Bin-packing)

```bash
# Chạy spark-pi-yunikorn-stress (YuniKorn scheduler, 5 executors)
kubectl apply -f spark-batch/spark-pi-yunikorn-stress.yaml

# Quan sát pod distribution
kubectl get pods -n spark-apps -o wide
# Expected: pods tập trung trên 1-2 nodes thay vì 4-5

# Quan sát với eks-node-viewer
eks-node-viewer --node-selector "purpose=spark"
```

## Bước 5: So sánh kết quả

| Metric | Default Scheduler | YuniKorn |
|--------|-------------------|----------|
| Số node sử dụng | ? | ? |
| Avg CPU utilization/node | ? | ? |
| Avg Memory utilization/node | ? | ? |
| Job completion time | ? | ? |
| Thời gian scale-in sau job | ? | ? |

## Bước 6: Multi-job Test

```bash
# Chạy đồng thời 3 jobs để thấy rõ sự khác biệt
kubectl apply -f spark-batch/spark-pi-yunikorn.yaml
kubectl apply -f spark-batch/spark-pi-yunikorn-stress.yaml
# (Tạo thêm 1 copy với tên khác nếu cần)

# Gang scheduling sẽ đảm bảo:
# - Job chỉ start khi đủ resource cho toàn bộ driver + executors
# - Không xảy ra deadlock giữa các jobs
# - Pods pack chặt vào ít nodes
```

---

## YuniKorn Web UI

Access tại `http://localhost:9889` sau khi port-forward.

Các tab quan trọng:
- **Queues**: Xem resource usage per queue (spark-batch, spark-streaming)
- **Applications**: Xem trạng thái các Spark app (running, pending, completed)
- **Nodes**: Xem pod distribution trên từng node (verify bin-packing)

---

## Kết hợp với Karpenter

Nếu đang dùng Karpenter trên EKS, YuniKorn bin-packing giúp:

1. **Node consolidation tự nhiên**: Pod pack chặt → node trống → Karpenter remove
2. **Không cần aggressive consolidation**: Giảm pod disruption (eviction)
3. **Provisioner config**: Karpenter vẫn provision node khi cần, YuniKorn quyết định đặt pod ở đâu

```yaml
# Karpenter NodePool - không cần thay đổi gì đặc biệt
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
          values: ["m5.xlarge", "m5.2xlarge"]  # Nodes đủ lớn cho bin-packing
```

---

## Troubleshooting

```bash
# YuniKorn scheduler logs
kubectl logs -l app=yunikorn -n yunikorn --tail=100

# Check if admission controller is routing pods
kubectl get pods -n spark-apps -o yaml | grep schedulerName

# Gang scheduling pending (waiting for resources)
kubectl get events -n spark-apps --sort-by='.lastTimestamp'

# Queue status via API
kubectl port-forward svc/yunikorn-service 9889:9889 -n yunikorn
curl http://localhost:9889/ws/v1/queues
```

---

## Kết quả mong đợi

Dựa trên case studies:
- **Salesforce Data 360**: 13% cost reduction, 50% less node disruption
- **AWS EMR on EKS best practices**: 15-30% improvement in node utilization
- **Minimum expected**: Giảm 20-40% số node cho cùng workload
