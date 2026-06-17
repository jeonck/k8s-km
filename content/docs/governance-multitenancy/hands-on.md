---
title: Hands-on
weight: 2
---

## 테넌트 네임스페이스 격리 4종 세트

```yaml
# 1. ResourceQuota - 네임스페이스 단위 리소스 한도
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
---
# 2. LimitRange - 개별 Pod/컨테이너 기본값 강제
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-a-limits
  namespace: tenant-a
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: 512Mi
      defaultRequest:
        cpu: "100m"
        memory: 128Mi
---
# 3. NetworkPolicy - 기본 거부 + 같은 네임스페이스만 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-and-allow-same-ns
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector: {}
---
# 4. RBAC - 테넌트 운영자는 자기 네임스페이스만 관리
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tenant-a-admin
  namespace: tenant-a
subjects:
  - kind: Group
    name: tenant-a-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f tenant-a-isolation.yaml
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

## Kyverno로 조직 정책 강제

```yaml
# 승인된 레지스트리 외 이미지 차단 (처음엔 audit으로 시작)
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Audit   # 충분히 점검 후 Enforce로 전환
  rules:
    - name: only-approved-registry
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "승인된 레지스트리(registry.example.com)의 이미지만 사용할 수 있습니다."
        pattern:
          spec:
            containers:
              - image: "registry.example.com/*"
---
# 모든 Pod에 리소스 요청 강제
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-requests
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-requests
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "모든 컨테이너는 CPU/메모리 requests를 명시해야 합니다."
        pattern:
          spec:
            containers:
              - resources:
                  requests:
                    cpu: "?*"
                    memory: "?*"
```

```bash
kubectl apply -f kyverno-policies.yaml

# 정책 위반 사례를 audit 모드에서 확인
kubectl get policyreport -A
```

## kube-bench로 CIS 컴플라이언스 점검

```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs -l app=kube-bench --tail=200
```

## 트러블슈팅: 네임스페이스를 나눴는데 다른 팀 서비스로 트래픽이 새는 사고

**현상**: tenant-a의 Pod가 tenant-b 네임스페이스의 내부 서비스에 직접 접근해 데이터를 조회한 흔적이 발견됨. 두 팀은 같은 클러스터의 서로 다른 네임스페이스를 쓰고 있었다.

**의심되는 원인(가설)**: 네임스페이스만 분리했을 뿐 NetworkPolicy를 설정하지 않아, 기본적으로 모든 Pod 간 통신이 허용되는 클러스터 네트워크 모델(flat network) 그대로 노출되어 있었을 가능성.

**확인할 로그/명령어**:
```bash
# 해당 네임스페이스에 NetworkPolicy가 존재하는지 확인
kubectl get networkpolicy -n tenant-b

# CNI가 NetworkPolicy를 지원하는 플러그인인지 확인 (예: flannel 단독은 미지원)
kubectl get pods -n kube-system -l k8s-app=calico-node

# 실제 트래픽 흐름 확인 (CNI가 eBPF 기반이라면 Hubble 등으로 가능)
kubectl exec -n tenant-a <pod> -- curl -s http://service.tenant-b.svc.cluster.local
```

**조치**: 모든 네임스페이스에 "기본 거부(default deny) + 명시적 허용" NetworkPolicy를 기본 템플릿으로 강제 적용한다. 신규 네임스페이스 생성 시 Kyverno의 `generate` 규칙으로 기본 NetworkPolicy를 자동 생성하도록 만들어, 사람이 깜빡해도 누락되지 않게 한다.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: auto-generate-default-deny-netpol
spec:
  rules:
    - name: generate-default-deny
      match:
        any:
          - resources:
              kinds: ["Namespace"]
      generate:
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        name: default-deny-ingress
        namespace: "{{request.object.metadata.name}}"
        synchronize: true
        data:
          spec:
            podSelector: {}
            policyTypes: ["Ingress"]
```

## 운영 체크리스트

- [ ] 모든 테넌트 네임스페이스에 ResourceQuota/LimitRange가 존재하는가
- [ ] 기본 거부 NetworkPolicy가 신규 네임스페이스 생성 시 자동으로 적용되는가
- [ ] RBAC RoleBinding이 ClusterRoleBinding으로 잘못 격상되어 전체 클러스터 권한이 새지 않았는가
- [ ] Admission 정책이 `Audit`에서 `Enforce`로 전환되기 전 충분한 점검 기간을 가졌는가
- [ ] Secret 접근에 대한 audit 로그가 최소 Metadata 레벨로 남고 있는가
- [ ] kube-bench/CIS 벤치마크 점검을 정기적으로(예: 분기별) 수행하는가
