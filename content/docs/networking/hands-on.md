---
title: Hands-on
weight: 2
---

## Service / Ingress 매니페스트

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector: { app: web }
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: web.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port: { number: 80 }
```

Gateway API로 같은 구성을 표현하면:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web
spec:
  parentRefs:
    - name: shared-gateway
  hostnames: ["web.example.com"]
  rules:
    - backendRefs:
        - name: web
          port: 80
```

## 트러블슈팅 1: "Service에 연결이 안 된다"

**확인 순서**:
```bash
# 1) Endpoint가 채워져 있는가 — 가장 흔한 원인은 selector 라벨 불일치
kubectl get endpoints web
# 결과가 비어 있다면 Service selector와 Pod label을 비교
kubectl get service web -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels

# 2) Pod 자체는 정상인가 (readinessProbe 실패면 Endpoint에서 빠짐)
kubectl get pods -l app=web
kubectl describe pod <pod-name> | grep -A5 Readiness

# 3) Pod 안에서 직접 연결 테스트 (네트워크 정책/방화벽 문제 격리)
kubectl run debug --rm -it --image=nicolaka/netshoot -- curl -v web.default.svc.cluster.local

# 4) NetworkPolicy가 차단하고 있는지 확인
kubectl get networkpolicy
```

**원인 분류**: Endpoint가 비어 있으면 selector/readiness 문제, Endpoint는 있는데 연결이 안 되면 NetworkPolicy나 CNI 문제로 좁혀집니다.

## 트러블슈팅 2: DNS 해석 실패 (`Name or service not known`)

**현상**: Pod 안에서 `curl my-service`가 DNS 에러로 실패.

**확인 순서**:
```bash
# 1) CoreDNS가 정상 동작 중인가
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# 2) Pod의 DNS 설정 확인 (resolv.conf, search domain)
kubectl exec <pod-name> -- cat /etc/resolv.conf

# 3) 짧은 이름 대신 FQCN으로 시도 (search domain 문제 격리)
kubectl exec <pod-name> -- nslookup my-service.default.svc.cluster.local

# 4) CoreDNS 자체에 질의가 도달하는지 (네트워크 경로 확인)
kubectl exec <pod-name> -- nslookup my-service.default.svc.cluster.local <coredns-cluster-ip>
```

**조치**: FQCN은 되는데 짧은 이름이 안 되면 `ndots`/search domain 설정 문제입니다. FQCN도 안 되면 CoreDNS Pod 자체나 CNI 네트워크 경로 문제입니다.

## 트러블슈팅 3: Ingress는 200인데 특정 경로만 404

```bash
# Ingress 컨트롤러가 실제로 받은 규칙을 어떻게 해석했는지 확인
kubectl describe ingress web
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=100 | grep web.example.com

# pathType(Prefix/Exact)과 rewrite-target 어노테이션 조합 오류가 가장 흔한 원인
```

## 운영 체크리스트

- [ ] NetworkPolicy로 기본 차단(default-deny) 후 필요한 통신만 허용했는가
- [ ] Service selector와 Pod label이 CI에서 일치 검증되는가
- [ ] kube-proxy 모드가 Service 규모에 맞는가 (수백 개 이상이면 IPVS/eBPF 검토)
- [ ] Ingress/Gateway에 TLS가 설정되어 있는가
- [ ] 서비스 메시 도입 시 사이드카 리소스 사용량을 별도로 모니터링하는가
