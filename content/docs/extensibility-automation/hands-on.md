---
title: Hands-on
weight: 2
---

## 1. CRD 정의

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: redisclusters.cache.example.com
spec:
  group: cache.example.com
  names:
    kind: RedisCluster
    plural: redisclusters
    singular: rediscluster
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas: { type: integer, minimum: 1 }
                version: { type: string }
              required: ["replicas"]
            status:
              type: object
              properties:
                readyReplicas: { type: integer }
      subresources:
        status: {}
```

```bash
kubectl apply -f rediscluster-crd.yaml
kubectl get crd redisclusters.cache.example.com
```

CR 생성:

```yaml
apiVersion: cache.example.com/v1
kind: RedisCluster
metadata:
  name: sessions
spec:
  replicas: 3
  version: "7.2"
```

## 2. 컨트롤러 스캐폴드 (controller-runtime, Go)

```go
func (r *RedisClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var rc cachev1.RedisCluster
    if err := r.Get(ctx, req.NamespacedName, &rc); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    sts := desiredStatefulSet(&rc)
    if err := ctrl.SetControllerReference(&rc, sts, r.Scheme); err != nil {
        return ctrl.Result{}, err
    }
    if err := r.Patch(ctx, sts, client.Apply, client.ForceOwnership, client.FieldOwner("rediscluster-controller")); err != nil {
        return ctrl.Result{}, err
    }

    rc.Status.ReadyReplicas = sts.Status.ReadyReplicas
    if err := r.Status().Update(ctx, &rc); err != nil {
        return ctrl.Result{}, err
    }
    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}
```

`ctrl.SetControllerReference`로 owner reference를 설정하면 RedisCluster CR을 삭제할 때 생성된 StatefulSet도 가비지 컬렉션으로 함께 정리됩니다 — 이걸 빼먹으면 CR을 지워도 하위 리소스가 고아로 남습니다.

## 3. Kubebuilder로 빠르게 시작하기

```bash
kubebuilder init --domain example.com --repo github.com/org/redis-operator
kubebuilder create api --group cache --version v1 --kind RedisCluster
make manifests && make install   # CRD를 클러스터에 적용
make run                          # 컨트롤러를 로컬에서 실행해 빠르게 반복 개발
```

## 4. 트러블슈팅 — "CR을 수정해도 컨트롤러가 반응하지 않는다"

**현상**: `kubectl edit rediscluster sessions`로 `replicas`를 5로 바꿨는데 StatefulSet이 그대로 3.

**가설**: (a) 컨트롤러 Pod가 크래시 루프 중일 수 있음, (b) watch 설정에서 해당 필드 변경이 큐에 들어가지 않을 수 있음(예: `Owns()`만 설정하고 `For()` watch가 사실은 다른 타입), (c) RBAC 부족으로 컨트롤러가 업데이트 자체를 못 하고 있을 수 있음.

**확인 순서**:

```bash
# 1) 컨트롤러가 살아있고 reconcile 로그가 찍히는지 확인
kubectl logs -n redis-operator-system deploy/redis-operator-controller-manager -f | grep sessions

# 2) 컨트롤러의 RBAC 확인 — StatefulSet update 권한 누락이 흔한 원인
kubectl auth can-i update statefulsets \
  --as=system:serviceaccount:redis-operator-system:redis-operator-controller-manager

# 3) 이벤트로 에러 확인
kubectl describe rediscluster sessions
```

**조치**: 로그에 reconcile 자체가 안 찍히면 watch 설정(`SetupWithManager`의 `For`/`Owns`)을 점검합니다. RBAC 에러가 보이면 `ClusterRole`에 해당 리소스의 `update`/`patch` verb를 추가합니다.

## 운영 체크리스트

- [ ] 컨트롤러가 생성한 하위 리소스에 owner reference가 설정되어 가비지 컬렉션이 동작하는가
- [ ] reconcile 함수가 멱등적인가 (같은 입력으로 여러 번 실행해도 동일한 결과)
- [ ] 컨트롤러의 RBAC가 필요한 리소스의 verb를 빠짐없이 포함하는가
- [ ] CRD 스펙이 구현 세부사항이 아니라 사용자의 의도를 표현하는가
