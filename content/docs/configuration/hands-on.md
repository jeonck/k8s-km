---
title: Hands-on
weight: 2
---

## 1. ConfigMap/Secret 생성과 3가지 주입 방식

```bash
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=FEATURE_NEW_UI=true

kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD='s3cr3t'
```

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 2
  selector:
    matchLabels: { app: app }
  template:
    metadata:
      labels: { app: app }
    spec:
      containers:
        - name: app
          image: myapp:1.0
          envFrom:
            - configMapRef: { name: app-config }   # 방식 1: env
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef: { name: db-secret, key: DB_PASSWORD }
            - name: POD_NAME                        # 방식 3: downward API
              valueFrom:
                fieldRef: { fieldPath: metadata.name }
          volumeMounts:
            - name: config-vol                       # 방식 2: volume
              mountPath: /etc/app/config
      volumes:
        - name: config-vol
          configMap:
            name: app-config
```

## 2. 트러블슈팅 — ConfigMap을 수정했는데 반영이 안 됨

**현상**: `kubectl edit configmap app-config`로 `LOG_LEVEL`을 `debug`로 바꿨는데 애플리케이션 로그 레벨이 그대로다.

```bash
# 1. 컨테이너가 env로 주입받았는지 volume으로 받았는지 먼저 확인
kubectl get deploy app -o jsonpath='{.spec.template.spec.containers[0].env}'
kubectl get deploy app -o jsonpath='{.spec.template.spec.containers[0].envFrom}'
```

- **env/envFrom으로 주입된 경우**: 설계상 절대 자동 반영되지 않습니다. Pod를 재시작해야 합니다.

```bash
kubectl rollout restart deployment app
```

- **volume으로 주입된 경우**: kubelet이 보통 1분 이내에 마운트된 파일을 갱신합니다. 애플리케이션이 그 파일을 한 번만 읽고 캐싱하고 있다면(많은 앱의 기본 동작) 역시 재시작이 필요합니다.

```bash
# 실제 마운트된 파일 내용이 갱신됐는지부터 확인 (애플리케이션 문제와 분리)
kubectl exec deploy/app -- cat /etc/app/config/LOG_LEVEL
```

**조치**: ConfigMap 변경 시 자동 롤링 재시작이 필요하다면 `checksum/config` 어노테이션 패턴(Helm 차트에서 흔히 씀)을 쓰거나 [Reloader](https://github.com/stakater/Reloader) 같은 컨트롤러를 도입합니다.

```yaml
# Helm 템플릿 패턴 예시
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

## 3. Kustomize 오버레이

```
base/
  deployment.yaml
  configmap.yaml
  kustomization.yaml
overlays/
  staging/
    kustomization.yaml
    config-patch.yaml
  production/
    kustomization.yaml
    config-patch.yaml
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: config-patch.yaml
replicas:
  - name: app
    count: 5
```

```bash
# 실제 적용 전에 렌더링 결과를 먼저 눈으로 확인하는 습관이 중요
kubectl kustomize overlays/production/ | less
kubectl apply -k overlays/production/
```

## 4. Helm Chart 배포와 드리프트 확인

```bash
helm install myapp ./charts/myapp -f values-production.yaml
helm upgrade myapp ./charts/myapp -f values-production.yaml

# 클러스터의 실제 상태와 Chart가 선언한 상태의 차이를 확인
helm diff upgrade myapp ./charts/myapp -f values-production.yaml   # helm-diff 플러그인
```

```bash
# 누군가 kubectl edit으로 직접 건드린 리소스 찾기 (helm 관리 대상 중)
helm get manifest myapp | kubectl diff -f -
```

## 운영 체크리스트

- [ ] 민감하지 않은 값과 민감한 값을 ConfigMap/Secret으로 명확히 분리했는가
- [ ] env로 주입한 설정 변경 시 재시작이 필요하다는 것을 팀이 알고 있는가
- [ ] ConfigMap 변경 시 자동 재시작(checksum 패턴 또는 Reloader)이 필요한 워크로드인지 판단했는가
- [ ] 환경별 차이가 Kustomize 오버레이/Helm values로 명확히 추적되는가 (수동 `kubectl edit` 금지)
- [ ] `helm diff` 또는 GitOps 도구의 드리프트 감지가 주기적으로 확인되는가
