---
title: Hands-on
weight: 2
---

## 클러스터 컴포넌트 상태 점검

```bash
# control plane 컴포넌트가 모두 떠 있는지 (managed 클러스터는 노출 안 될 수 있음)
kubectl get pods -n kube-system

# API server 응답성 / 버전 확인
kubectl version --short
kubectl get --raw='/readyz?verbose'

# etcd 상태 (kubeadm 클러스터, etcd Pod에 직접 접근)
kubectl exec -n kube-system etcd-<node-name> -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

## reconciliation을 직접 눈으로 확인하기

```bash
# 1) 터미널 A: Deployment 생성
kubectl create deployment demo --image=nginx --replicas=3

# 2) 터미널 B: 변화를 실시간 관찰
kubectl get pods -l app=demo --watch

# 3) 터미널 A: Pod 하나를 강제로 삭제
kubectl delete pod -l app=demo --field-selector=status.phase=Running -o name | head -1 | xargs kubectl delete

# 터미널 B에서 즉시 새 Pod가 Pending → ContainerCreating → Running으로 올라오는 것을 확인.
# 이것이 ReplicaSet 컨트롤러의 reconcile loop가 동작하는 증거입니다.
```

## API 요청이 어디까지 가는지 추적

```bash
# -v=8 로 실제 HTTP 요청/응답을 확인 (인증 헤더는 마스킹됨)
kubectl get pod demo-xxxxx -v=8 2>&1 | grep -E "GET|POST|Response Status"

# audit log가 활성화된 클러스터라면 누가 무엇을 호출했는지 추적 가능
kubectl logs -n kube-system kube-apiserver-<node> | grep audit
```

## 트러블슈팅: "YAML을 apply했는데 반영이 안 된다"

**현상**: `kubectl apply -f deploy.yaml` 실행 후 `kubectl get deployment`로 봐도 이미지 태그가 바뀌지 않음.

**가설 후보**:
1. apply가 실제로 실패했는데 에러를 못 봤다 (필드 충돌, validation 실패)
2. apply는 성공했지만 admission webhook이 변경을 가로채서 되돌렸다
3. 실제로는 반영됐지만, 보고 있는 게 다른 네임스페이스/컨텍스트다

**확인 순서**:
```bash
# 1) apply 결과 자체를 다시 확인 (dry-run으로 diff 미리보기)
kubectl diff -f deploy.yaml

# 2) 현재 context/namespace 확인 — 가장 흔한 실수
kubectl config current-context
kubectl config view --minify | grep namespace

# 3) admission webhook이 있는지 확인
kubectl get mutatingwebhookconfigurations,validatingwebhookconfigurations

# 4) 리소스의 변경 이력(managedFields)을 확인 — 누가 마지막으로 이 필드를 썼는가
kubectl get deployment demo -o yaml | grep -A5 managedFields
```

**조치**: 대부분 (3) namespace/context 착오이거나 (2) GitOps 컨트롤러(ArgoCD/Flux)가 Git의 상태로 즉시 되돌리는 경우입니다. GitOps를 쓴다면 "kubectl로 직접 수정"은 항상 일시적입니다 — [전달 & GitOps]({{< relref "/docs/delivery-gitops" >}}) 참고.

## 운영 체크리스트

- [ ] etcd 디스크가 SSD이고 별도 디스크인가 (다른 워크로드와 I/O 경쟁하지 않는가)
- [ ] etcd 노드 수가 홀수(3 또는 5)인가
- [ ] API server `--audit-log-path`가 설정되어 있어 사후 추적이 가능한가
- [ ] `kubectl get --raw='/readyz?verbose'`를 정기적으로 모니터링하는가
- [ ] CRD/webhook 설치 시 API server 응답 latency 변화를 관찰했는가
