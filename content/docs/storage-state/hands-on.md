---
title: Hands-on
weight: 2
---

## 1. StorageClass 확인과 동적 프로비저닝

```bash
# 클러스터에 어떤 StorageClass가 있는지, 기본값이 뭔지 확인
kubectl get storageclass
kubectl get sc -o jsonpath='{range .items[*]}{.metadata.name}{" default="}{.metadata.annotations."storageclass.kubernetes.io/is-default-class"}{"\n"}{end}'
```

```yaml
# pvc-data.yaml — PVC만 만들면 StorageClass가 알아서 PV를 만들어준다
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard   # 클러스터 환경에 맞게 변경
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f pvc-data.yaml
kubectl get pvc data-pvc -w   # Pending -> Bound 전환을 지켜본다
```

## 2. StatefulSet + volumeClaimTemplates

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web
  replicas: 3
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: web
          image: nginx:1.27
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 5Gi
```

```bash
kubectl apply -f statefulset-web.yaml
kubectl get pvc -l app=web
# data-web-0, data-web-1, data-web-2 가 각각 생성되는 것을 확인
```

## 3. 트러블슈팅 시나리오 — PVC가 계속 Pending

**현상**: `kubectl get pvc`에서 `data-pvc`가 몇 분째 `Pending` 상태로 멈춰 있다.

**가설 1 — 적합한 PV/StorageClass가 없다**

```bash
kubectl describe pvc data-pvc
# Events 섹션에 "no persistent volumes available for this claim" 또는
# "waiting for a volume to be created, either by external provisioner..." 확인
```

```bash
kubectl get storageclass
# storageClassName으로 지정한 이름이 실제로 존재하는지, provisioner가 정상인지 확인
```

**가설 2 — CSI 컨트롤러가 죽어 있다**

```bash
kubectl get pods -n kube-system -l app=csi-provisioner   # 환경에 따라 네임스페이스/레이블 다름
kubectl logs -n kube-system <csi-controller-pod> -c csi-provisioner --tail=100
```

**가설 3 — 클라우드 쿼터/권한 문제**

```bash
kubectl describe pvc data-pvc
# "failed to provision volume ... rpc error: code = ResourceExhausted" 등의 메시지가
# 클라우드 쪽 쿼터/IAM 권한 문제임을 시사
```

**조치**: 원인이 StorageClass 이름 오타인 경우가 실무에서 가장 흔합니다. 그다음으로 흔한 것은 클라우드 쿼터 초과입니다. CSI 컨트롤러 자체가 죽어 있는 경우는 드물지만 가장 늦게 발견됩니다 — 위 순서(설정 → 쿼터 → 컨트롤러)로 좁혀가는 것이 효율적입니다.

## 4. 트러블슈팅 시나리오 — Pod가 ContainerCreating에 멈춤 (마운트 실패)

**현상**: PVC는 `Bound`인데 Pod가 `ContainerCreating`에서 진행되지 않는다.

```bash
kubectl describe pod <pod-name>
# Events에 "FailedMount", "MountVolume.SetUp failed", "timeout expired waiting for volumes to attach" 등 확인
```

```bash
# 해당 Pod가 스케줄된 노드에서 CSI 노드 플러그인이 살아있는지 확인
kubectl get pods -n kube-system -o wide | grep csi-node
kubectl logs -n kube-system <csi-node-pod-on-that-node> --tail=100
```

**흔한 원인**: RWO 볼륨인데 이전 Pod가 다른 노드에서 아직 볼륨을 붙잡고 있는 경우(`multi-attach error`). StatefulSet 롤링 업데이트 중 이전 Pod 종료가 늦어지면 자주 발생합니다.

```bash
kubectl get events --field-selector reason=FailedAttachVolume
```

**조치**: 이전 Pod가 정상적으로 Terminating 됐는지 확인하고, 강제 종료가 필요하면 `kubectl delete pod <old-pod> --grace-period=0 --force`를 신중하게 사용합니다(데이터 정합성 리스크가 있으므로 애플리케이션이 안전하게 죽었는지 먼저 확인).

## 5. Velero 백업 / 복구

```bash
# 네임스페이스 단위 백업
velero backup create app-backup-$(date +%Y%m%d) --include-namespaces=production

# 백업 상태 확인
velero backup describe app-backup-20260617 --details

# 복구 (다른 클러스터나 새 네임스페이스로도 가능)
velero restore create --from-backup app-backup-20260617
```

```bash
# 정기 백업 스케줄 (매일 새벽 2시, 30일 보관)
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces=production \
  --ttl=720h0m0s
```

{{< callout type="warning" >}}
Velero 백업이 "성공"으로 표시되어도 실제로 복구가 되는지는 별개입니다. 최소 분기 1회는 별도 네임스페이스/클러스터에 실제 restore를 해보고 애플리케이션이 정상 기동하는지 확인하세요.
{{< /callout >}}

## 운영 체크리스트

- [ ] 프로덕션 StorageClass의 `reclaimPolicy`가 `Retain`인지 확인했는가
- [ ] StatefulSet의 `volumeClaimTemplates` accessMode가 워크로드 특성(단일 vs 멀티 attach)에 맞는가
- [ ] CSI 컨트롤러/노드 플러그인에 대한 모니터링(재시작, 로그 에러율)이 있는가
- [ ] Velero 백업 스케줄과 TTL이 RPO 요구사항을 만족하는가
- [ ] 최근 90일 내 실제 restore 리허설을 해봤는가
- [ ] PVC 삭제는 별도 승인 절차(또는 admission 정책)로 보호되는가
