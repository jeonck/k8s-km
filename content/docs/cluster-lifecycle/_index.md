---
title: "클러스터 라이프사이클"
weight: 100
---

클러스터 *안*의 워크로드가 아니라 클러스터 *자체*를 다루는 영역이다. 프로비저닝부터 업그레이드, 노드 관리, 폐기까지 — 클러스터를 "띄우고 유지하는" 모든 운영 활동의 범위다.

| 어젠더 | 핵심 질문 |
| --- | --- |
| 부트스트랩 (kubeadm / managed EKS·GKE·AKS / Cluster API) | 어떤 방식으로 클러스터를 처음 만드는가 |
| 업그레이드 전략과 버전 스큐 | 무중단으로 버전을 올리려면 무엇을 지켜야 하는가 |
| 노드 관리와 멀티클러스터·페더레이션 | 노드와 클러스터를 어떻게 묶어 관리하는가 |
| IaC(Terraform) 연계 | 클러스터 정의 자체를 코드로 어떻게 관리하는가 |

{{< cards >}}
  {{< card link="concept" title="Concept" subtitle="부트스트랩 방식 비교, 버전 스큐 정책, 멀티클러스터 모델" >}}
  {{< card link="hands-on" title="Hands-on" subtitle="kubeadm/Cluster API 실습, 업그레이드 절차, 노드 교체 트러블슈팅" >}}
{{< /cards >}}
