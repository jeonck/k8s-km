---
title: Hands-on
weight: 2
---

## Kubecost 설치 및 네임스페이스별 비용 확인

```bash
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost --create-namespace \
  --set kubecostToken="<token>"

kubectl port-forward -n kubecost svc/kubecost-cost-analyzer 9090:9090

# CLI로 네임스페이스별 누적 비용 조회 (지난 7일)
kubectl exec -n kubecost deploy/kubecost-cost-analyzer -- \
  curl -s "http://localhost:9003/model/allocation?window=7d&aggregate=namespace" | jq
```

## VPA로 right-sizing 추천값 관찰 (Off 모드)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: payment-service-vpa
  namespace: payment
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  updatePolicy:
    updateMode: "Off"   # 추천값만 계산, 자동 적용 안 함
```

```bash
kubectl apply -f vpa-off.yaml

# 며칠 관찰 후 추천값 확인
kubectl describe vpa payment-service-vpa -n payment
```

```
Recommendation:
  Container Recommendations:
    Container Name:  payment-service
    Target:
      Cpu:     180m
      Memory:  256Mi
    Lower Bound:
      Cpu:     120m
      Memory:  200Mi
    Upper Bound:
      Cpu:     350m
      Memory:  400Mi
```

기존 requests가 `cpu: 500m, memory: 512Mi`였다면, target 추천값(180m/256Mi)에 맞춰 하향 조정해 동일한 노드에 더 많은 Pod를 띄울 여유를 만든다.

## 스팟 노드풀 분리 및 워크로드 배치

```yaml
# 스팟 노드풀에는 taint를 걸어, 명시적으로 toleration을 준 워크로드만 스케줄링되게 한다
apiVersion: v1
kind: Node
metadata:
  labels:
    node-lifecycle: spot
spec:
  taints:
    - key: node-lifecycle
      value: spot
      effect: NoSchedule
---
# 스팟에 가도 괜찮은 stateless 워크로드
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 6
  template:
    spec:
      tolerations:
        - key: node-lifecycle
          operator: Equal
          value: spot
          effect: NoSchedule
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: node-lifecycle
                    operator: In
                    values: ["spot"]
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: web-frontend
```

```bash
kubectl apply -f spot-workload.yaml

# 스팟 회수 임박 신호 확인 (클라우드별 termination notice, 예: AWS)
kubectl get events --field-selector reason=NodeNotReady -A
```

## Karpenter로 동적 노드 프로비저닝 + Consolidation

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot-general
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
      nodeClassRef:
        name: default
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 5m
```

```bash
kubectl apply -f karpenter-nodepool.yaml

# 현재 노드별 실제 사용률 확인 (bin-packing 효율 점검)
kubectl top nodes
```

## 트러블슈팅: requests를 줄였더니 OOMKilled가 급증

**현상**: VPA 추천값을 따라 메모리 requests/limits를 512Mi → 256Mi로 낮춘 직후, 해당 워크로드의 OOMKilled 재시작이 급증했다.

**의심되는 원인(가설)**: VPA의 추천값은 평상시 평균 사용량 기준이며, 트래픽 스파이크 시점의 peak 사용량을 충분히 반영하지 못했을 가능성. 특히 GC 언어(Java, Go)는 평균 사용량과 peak 사용량의 차이가 크다.

**확인할 로그/명령어**:
```bash
# 실제 메모리 사용량의 peak를 더 긴 기간으로 재확인
kubectl exec payment-service-xxxx -- cat /sys/fs/cgroup/memory.current

# OOMKilled 이벤트와 시점 확인
kubectl describe pod payment-service-xxxx | grep -A 5 "Last State"

# 트래픽 스파이크와 OOM 시점이 겹치는지 메트릭 비교 (Prometheus)
# container_memory_working_set_bytes vs http_requests_total
```

**조치**: VPA의 `Target` 대신 `Upper Bound` 추천값을 기준으로 limits를 설정하거나, peak 트래픽 시간대를 포함한 더 긴 관찰 기간(최소 1~2주, 피크 트래픽 이벤트 포함)을 확보한 뒤 재조정한다. 비용 절감과 안정성은 트레이드오프이므로, "장애 비용 > 절감되는 비용"이면 보수적인 값을 유지하는 것이 맞는 판단이다.

## 운영 체크리스트

- [ ] 네임스페이스/팀별 비용이 가시화되어 있고 각 팀이 자기 비용을 볼 수 있는가
- [ ] right-sizing은 추측이 아니라 실측 데이터(최소 1~2주, 피크 포함)를 기반으로 했는가
- [ ] 스팟 노드풀에는 중단을 견딜 수 있는 워크로드만 배치되어 있는가 (taint/toleration으로 강제되는가)
- [ ] 노드 사용률(`kubectl top nodes`)이 지속적으로 낮은 구간이 있는가 — bin-packing/consolidation 정책 점검 대상
- [ ] 비용 절감이 신뢰성 SLO를 깎는 방향으로 가지 않았는가 (장애 비용과 비교했는가)
