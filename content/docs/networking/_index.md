---
title: "네트워킹"
weight: 30
---

"어떻게 통신하는가" — Pod 간, 서비스 간, 외부와의 연결을 다루는 영역입니다.

## 핵심 어젠더

| 영역 | 내용 |
| --- | --- |
| 기반 모델 | CNI와 Pod 네트워크 모델 |
| 서비스 디스커버리 | Service 타입과 kube-proxy 모드(iptables / IPVS / eBPF), DNS(CoreDNS) |
| 외부 진입 트래픽 | Ingress → Gateway API로의 전환 |
| 내부 트래픽 | 서비스 메시(Istio·Linkerd)와 east-west 트래픽 |

{{< cards >}}
  {{< card link="concept" title="Concept" subtitle="CNI, Service 타입, DNS, Ingress/Gateway API, 서비스 메시" >}}
  {{< card link="hands-on" title="Hands-on" subtitle="DNS 해석 실패, Service 연결 안 됨, Ingress 디버깅" >}}
{{< /cards >}}
