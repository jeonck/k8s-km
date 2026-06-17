---
title: Hands-on
weight: 2
---

## 1. RBAC 최소 권한 설계

배포 전용 ServiceAccount — Deployment/ReplicaSet/Pod만 보고 고치게 하고, Secret 읽기는 막습니다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: payments
  name: deployer
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: payments
  name: ci-deployer-binding
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: payments
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

권한 점검 명령:

```bash
kubectl auth can-i update deployments --as=system:serviceaccount:payments:ci-deployer -n payments
kubectl auth can-i list secrets --as=system:serviceaccount:payments:ci-deployer -n payments
# no 가 나와야 정상
```

{{< callout type="warning" >}}
`kubectl create rolebinding ... --clusterrole=cluster-admin` 같은 임시 디버깅 명령을 운영 환경에 남겨두는 것이 가장 흔한 사고 원인입니다. `kubectl get clusterrolebindings -o json | jq '.items[] | select(.roleRef.name=="cluster-admin")'`로 정기적으로 점검하세요.
{{< /callout >}}

## 2. Pod Security Admission 적용

```bash
kubectl label namespace payments \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

`restricted`를 만족하는 Pod 스펙 예시:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
        readOnlyRootFilesystem: true
```

## 3. Kyverno로 정책 강제 — 서명된 이미지만 허용

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-signature
      match:
        resources:
          kinds: ["Pod"]
      verifyImages:
        - imageReferences: ["registry.internal.com/*"]
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      ...
                      -----END PUBLIC KEY-----
```

## 4. Secret 암호화 확인 (encryption-at-rest)

etcd에 실제로 암호화되어 저장되는지 검증:

```bash
ETCDCTL_API=3 etcdctl get /registry/secrets/payments/db-credentials | hexdump -C | head -5
# k8s:enc:aescbc:v1: 같은 prefix가 보이면 암호화 적용됨, base64 평문이 그대로 보이면 미적용
```

## 5. 트러블슈팅 — "권한이 있는데 403이 난다"

**현상**: ServiceAccount에 `edit` ClusterRole을 RoleBinding으로 바인딩했는데도 특정 리소스에서 403.

**가설**: `edit` 기본 ClusterRole은 RBAC 리소스 자체(Role, RoleBinding)나 ResourceQuota 등 일부 리소스를 의도적으로 제외한다. 또는 어드미션 단계(Kyverno/Gatekeeper)에서 별도로 막혔을 수 있다.

**확인 순서**:

```bash
# 1) RBAC 자체가 막는지 확인
kubectl auth can-i delete pods --as=system:serviceaccount:payments:app -n payments

# 2) 403 응답 바디에 어드미션 webhook 이름이 찍히는지 확인 (RBAC 거부는 메시지가 다름)
kubectl delete pod sample --as=system:serviceaccount:payments:app -n payments
# Error from server (Forbidden): admission webhook "validate.kyverno.svc" denied the request

# 3) 어드미션이 원인이면 정책 리포트 확인
kubectl get events -n payments --field-selector reason=PolicyViolation
kubectl get policyreport -n payments
```

**조치**: RBAC 거부면 Role 수정, 어드미션 거부면 해당 ClusterPolicy의 `validationFailureAction`을 `Audit`으로 잠깐 낮춰 영향 범위를 파악한 뒤 정책 또는 워크로드 스펙을 수정합니다. `Audit`으로 낮추는 것은 임시 진단 목적이며 되돌리는 것을 반드시 후속 작업으로 추적해야 합니다.

## 운영 체크리스트

- [ ] `cluster-admin` ClusterRoleBinding 목록을 정기 점검하는가
- [ ] 모든 운영 네임스페이스에 PSA `restricted` 또는 최소 `baseline`이 걸려 있는가
- [ ] 서명되지 않은 이미지의 배포를 막는 어드미션 정책이 있는가
- [ ] etcd Secret이 KMS로 encryption-at-rest 적용되어 있는가
- [ ] Falco 등 런타임 탐지 알림이 실제로 누군가에게 도달하는가 (알림 채널 점검)
