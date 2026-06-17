---
title: Hands-on
weight: 2
---

## affinity / topology spread 실전 매니페스트

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 6
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels: { app: web }
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels: { app: web }
                topologyKey: kubernetes.io/hostname
      containers:
        - name: web
          image: nginx
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits: { cpu: "200m", memory: "128Mi" }
```

## HPA 구성

```bash
kubectl autoscale deployment web --cpu-percent=70 --min=3 --max=10

# 또는 YAML로 (커스텀 메트릭 포함 시 더 명확)
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
EOF

kubectl get hpa web --watch
```

## 트러블슈팅 1: Pod가 계속 Pending

**현상**: `kubectl get pods`에서 Pod가 `Pending`에서 멈춰 있음.

**확인 순서** (현상 → 원인 후보 → 검증):
```bash
# 1) 스케줄러가 왜 못 배치했는지 이벤트 확인 — 가장 먼저 볼 것
kubectl describe pod <pod-name> | grep -A10 Events

# 흔한 메시지와 원인:
# "Insufficient cpu/memory"        → 노드 리소스 부족, requests 재검토 or 노드 추가
# "node(s) had taint ... not tolerated" → toleration 누락
# "didn't match Pod's node affinity"    → nodeAffinity 조건 확인
# "0/N nodes are available: N node(s) had volume node affinity conflict" → PVC가 다른 zone에 묶여있음

# 2) 클러스터 전체 가용 리소스 확인
kubectl describe nodes | grep -A5 "Allocated resources"

# 3) Cluster Autoscaler/Karpenter 로그 확인 (노드가 늘어나야 하는데 안 늘어나는 경우)
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=50
```

## 트러블슈팅 2: CrashLoopBackOff

**현상**: Pod가 계속 재시작됨.

**확인 순서**:
```bash
# 1) 직전 컨테이너의 종료 코드와 마지막 로그 확인
kubectl describe pod <pod-name> | grep -A5 "Last State"
kubectl logs <pod-name> --previous

# 종료 코드 해석:
# Exit Code 0   → 정상 종료인데 재시작됨 (livenessProbe 설정 또는 restartPolicy 확인)
# Exit Code 1   → 애플리케이션 에러 (로그 확인)
# Exit Code 137 → OOMKilled (메모리 limit 부족, kubectl describe pod 에서 "OOMKilled" 확인)
# Exit Code 143 → SIGTERM (정상 종료 신호, graceful shutdown 시간 부족 가능성)

# 2) 메모리 문제인지 빠르게 확인
kubectl describe pod <pod-name> | grep -i oom
kubectl top pod <pod-name>  # metrics-server 필요
```

**조치**: OOMKilled면 `limits.memory`를 올리거나 애플리케이션 메모리 누수를 확인합니다. livenessProbe가 너무 빡빡하면(initialDelaySeconds 부족) 정상 기동 중에도 죽는 경우가 많습니다 — 앱 기동 시간을 측정하고 `initialDelaySeconds`를 그보다 넉넉하게 설정하세요.

## 운영 체크리스트

- [ ] 모든 워크로드에 requests/limits가 설정되어 있는가 (BestEffort Pod가 없는가)
- [ ] StatefulSet의 PVC reclaim policy가 의도와 맞는가 (실수로 데이터 삭제 방지)
- [ ] critical 워크로드에 PriorityClass가 지정되어 있는가
- [ ] HPA의 min/max replica가 실제 트래픽 패턴과 맞는가
- [ ] anti-affinity가 `required`로 걸려 Pending을 유발하지 않는가
