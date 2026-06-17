---
title: Hands-on
weight: 2
---

## 1. 장애 도메인 분산

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels: { app: checkout }
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels: { app: checkout }
```

## 2. PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: checkout-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels: { app: checkout }
```

노드 드레인 시 PDB가 실제로 동작하는지 확인:

```bash
kubectl drain node-3 --ignore-daemonsets --delete-emptydir-data
# "Cannot evict pod as it would violate the pod's disruption budget" 가 뜨면 정상 동작 중
```

## 3. Canary 배포 (Argo Rollouts 기준)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: checkout
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
```

진행 상황 확인 및 비정상 시 즉시 중단:

```bash
kubectl argo rollouts get rollout checkout --watch
kubectl argo rollouts abort checkout   # 비정상 지표 감지 시 즉시 롤백
```

## 4. 카오스 실험 — Pod 강제 종료 (Chaos Mesh 기준)

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: kill-one-replica
spec:
  action: pod-kill
  mode: one
  duration: "30s"
  selector:
    namespaces: ["payments"]
    labelSelectors:
      app: checkout
```

실험 직후 확인할 것: 사용자 영향(에러율 변화), 재스케줄 소요 시간, 알림이 적절한 시점에 발생했는지.

## 5. 트러블슈팅 — "롤링 업데이트 중 일시적으로 5xx가 튄다"

**현상**: `kubectl rollout restart`만 했는데 배포 중 30초 정도 5xx 에러율이 튀어 알림이 발생.

**가설**: 새 Pod가 `Running`이 되었지만 애플리케이션이 실제로 요청을 받을 준비가 되기 전에 Service 엔드포인트에 추가되었을 가능성 (readinessProbe 부재 또는 너무 관대한 설정). 또는 종료되는 Pod가 in-flight 요청을 처리 중인데 즉시 SIGKILL 당했을 가능성(graceful shutdown 미흡).

**확인 순서**:

```bash
# 1) readinessProbe 설정 확인 — 없으면 Pod가 뜨는 즉시 Ready로 간주됨
kubectl get deploy checkout -o jsonpath='{.spec.template.spec.containers[0].readinessProbe}'

# 2) terminationGracePeriodSeconds와 preStop hook 확인
kubectl get deploy checkout -o jsonpath='{.spec.template.spec.terminationGracePeriodSeconds}'

# 3) 배포 중 엔드포인트 변화를 실시간 관찰
kubectl get endpoints checkout -w
```

**조치**: readinessProbe를 실제 의존성(DB 커넥션 등) 체크 기반으로 강화하고, `preStop`에 `sleep 5` 등을 추가해 종료 직전 로드밸런서가 해당 Pod로의 라우팅을 멈출 시간을 벌어줍니다.

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]
terminationGracePeriodSeconds: 30
```

## 운영 체크리스트

- [ ] 모든 critical 서비스에 PDB가 설정되어 있는가
- [ ] 레플리카가 zone/node 단위로 분산 배치되는가 (anti-affinity/topology spread)
- [ ] 배포 전략이 서비스 중요도에 맞게 선택되어 있는가 (rolling vs canary vs blue-green)
- [ ] 카오스 실험을 정기적으로 (스테이징 또는 통제된 운영) 수행하는가
- [ ] graceful shutdown(preStop, terminationGracePeriodSeconds)이 설정되어 있는가
