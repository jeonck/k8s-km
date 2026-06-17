---
title: Hands-on
weight: 2
---

## ArgoCD 설치 및 첫 Application 등록

```bash
# ArgoCD 설치
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 초기 admin 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# CLI 로그인 (포트포워딩 후)
kubectl -n argocd port-forward svc/argocd-server 8080:443 &
argocd login localhost:8080
```

Application 정의:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/payment-service-manifests.git
    targetRevision: main
    path: environments/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: payment
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f payment-service-app.yaml
argocd app sync payment-service
argocd app get payment-service
```

## ApplicationSet으로 멀티클러스터 배포

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: payment-service-fleet
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            env: prod
  template:
    metadata:
      name: '{{name}}-payment-service'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/payment-service-manifests.git
        targetRevision: main
        path: environments/prod
      destination:
        server: '{{server}}'
        namespace: payment
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

`env: prod` 라벨이 붙은 모든 등록된 클러스터에 동일한 Application이 자동으로 생성된다. 클러스터를 새로 추가해도 ApplicationSet 정의를 바꿀 필요가 없다.

## 프로모션 PR 자동화 (이미지 태그 갱신)

```bash
# CI가 새 이미지를 빌드한 뒤, 매니페스트 레포의 dev 환경 kustomization을 갱신
cd manifests-repo/environments/dev
kustomize edit set image payment-service=registry.example.com/payment-service:v1.42.0
git checkout -b bump-dev-v1.42.0
git commit -am "chore(dev): bump payment-service to v1.42.0"
git push origin bump-dev-v1.42.0
gh pr create --title "deploy: payment-service v1.42.0 to dev" --body "automated by CI"
```

prod로의 승격은 dev/staging 검증 후 동일한 방식으로 `environments/prod` 디렉터리를 대상으로 별도 PR을 올리되, 승인자 지정(`CODEOWNERS`)으로 사람의 검토를 강제한다.

## 트러블슈팅: Application이 계속 OutOfSync로 표시됨

**현상**: `argocd app get payment-service`에서 동기화를 실행해도 몇 분 뒤 다시 `OutOfSync`로 바뀐다.

**의심되는 원인(가설)**: 클러스터 내부의 어떤 컨트롤러(HPA, admission webhook의 mutating 패치, 또는 운영자가 직접 실행한 `kubectl edit`)가 ArgoCD가 적용한 desired state를 지속적으로 변경하고 있을 가능성.

**확인할 로그/명령어**:
```bash
# 어떤 필드가 실제로 다른지 diff 확인
argocd app diff payment-service

# 리소스의 변경 이벤트 확인 (managedFields로 누가 수정했는지 추적)
kubectl get deployment payment-service -n payment -o yaml | grep -A 5 managedFields

# ArgoCD 자체의 동기화 이력
argocd app history payment-service
```

**조치**: HPA가 `spec.replicas`를 변경하는 경우라면 ArgoCD `Application`의 `spec.ignoreDifferences`에 해당 필드를 등록해 정상적인 자동 스케일링을 드리프트로 오인하지 않도록 한다.

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

운영자의 수동 `kubectl edit`이 원인이라면 `selfHeal: true`가 의도대로 되돌리고 있는 것이므로, 수동 변경 자체를 막아야 한다 (RBAC으로 직접 수정 권한 제한).

## 운영 체크리스트

- [ ] CI가 클러스터 자격증명을 직접 보유하지 않고 Git 푸시까지만 책임지는가
- [ ] 동일한 이미지 태그가 dev→staging→prod를 그대로 통과하는 구조인가 (환경마다 재빌드하지 않는가)
- [ ] `selfHeal`/`prune` 정책이 의도와 일치하는가 (HPA처럼 정상적으로 변하는 필드는 `ignoreDifferences`로 제외했는가)
- [ ] prod 승격 PR에 `CODEOWNERS` 등 사람의 승인 게이트가 있는가
- [ ] Git 레포 자체에 대한 접근 권한이 RBAC만큼 엄격하게 관리되는가 (Git = 배포 권한이므로)
