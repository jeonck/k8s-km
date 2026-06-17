---
title: "전달 & GitOps"
weight: 110
---

선언적 상태를 클러스터로 흘려보내는 파이프라인의 영역이다. "코드를 어떻게 빌드하는가"(CI)가 아니라 "이미 빌드된 산출물을 클러스터의 desired state로 어떻게 동기화하는가"(CD/GitOps)에 집중한다.

| 어젠더 | 핵심 질문 |
| --- | --- |
| GitOps 원칙 (Git을 단일 진실 원천으로) | desired state는 어디에 있어야 하는가 |
| ArgoCD·Flux | 어떤 도구로 동기화를 자동화하는가 |
| 프로모션 전략 (환경 승격) | dev→staging→prod로 어떻게 안전하게 넘기는가 |
| CI/CD와의 경계 구분 | 빌드와 배포의 책임을 어디서 나누는가 |

{{< cards >}}
  {{< card link="concept" title="Concept" subtitle="GitOps 원칙, Push vs Pull 배포, ArgoCD/Flux 비교" >}}
  {{< card link="hands-on" title="Hands-on" subtitle="ArgoCD Application/ApplicationSet 실습, 동기화 드리프트 트러블슈팅" >}}
{{< /cards >}}
