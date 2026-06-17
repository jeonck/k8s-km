---
title: "비용 & 효율"
weight: 130
---

리소스 경제성의 영역이다. 클러스터가 "동작하는가"를 넘어 "비용 대비 효율적으로 동작하는가"를 따진다. 워크로드/인프라 설계가 끝난 뒤에도 지속적으로 최적화해야 하는 운영 영역이다.

| 어젠더 | 핵심 질문 |
| --- | --- |
| FinOps·비용 가시성 (Kubecost) | 어떤 팀/워크로드가 얼마를 쓰는지 보이는가 |
| 리소스 right-sizing | requests/limits가 실제 사용량과 맞는가 |
| 스팟·노드 풀 전략 | 어떤 워크로드를 어떤 노드에 배치해야 싸지는가 |
| bin-packing 효율 | 노드 사용률을 어떻게 끌어올리는가 |

{{< cards >}}
  {{< card link="concept" title="Concept" subtitle="FinOps 모델, right-sizing 원리, 스팟 인스턴스 전략" >}}
  {{< card link="hands-on" title="Hands-on" subtitle="Kubecost 설치, VPA 기반 right-sizing, 스팟 노드풀 트러블슈팅" >}}
{{< /cards >}}
