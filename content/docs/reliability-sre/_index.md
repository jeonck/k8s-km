---
title: "신뢰성 / SRE"
weight: 80
---

신뢰성은 "장애가 일어나지 않게 한다"가 아니라 **"장애를 전제로 설계한다"**는 관점입니다. 노드는 죽고, 배포는 실패하고, 트래픽은 예측을 벗어납니다. 이 카테고리는 그런 상황에서도 사용자가 영향을 덜 받게 만드는 설계와 운영 관행을 다룹니다.

## 핵심 어젠더

- 고가용성과 장애 도메인 분산 (zone/node spread, PodDisruptionBudget)
- 점진적 배포 전략 (rolling / canary / blue-green)
- 장애 복구와 DR(재해 복구) 전략
- 카오스 엔지니어링 (의도적 장애 주입으로 가정 검증)
- 용량 계획과 부하 대응

{{< cards >}}
  {{< card link="concept" title="Concept" subtitle="장애 도메인, PDB, 배포 전략별 트레이드오프, 카오스 엔지니어링의 목적" >}}
  {{< card link="hands-on" title="Hands-on" subtitle="PDB/anti-affinity 설정, 카나리 배포, 장애 주입 실습과 사고 대응 시나리오" >}}
{{< /cards >}}
