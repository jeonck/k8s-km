---
title: "스토리지 & 상태"
weight: 40
---

## 범위

**데이터를 어떻게 유지하는가** — 영속성과 상태 관리를 다루는 카테고리입니다. Pod는 본질적으로 휘발성이지만, 데이터베이스나 메시지 큐 같은 상태 있는(stateful) 워크로드는 Pod가 재시작되거나 노드를 이동해도 데이터가 유지되어야 합니다. 이 카테고리는 "어떻게 그 영속성을 보장하는가"에만 집중합니다.

## 핵심 어젠더

| 항목 | 다루는 질문 |
| --- | --- |
| Volume / PV / PVC / StorageClass | 어떻게 동적으로 스토리지를 프로비저닝하는가 |
| CSI 드라이버 | 클러스터가 외부 스토리지 시스템과 어떻게 통합되는가 |
| Access Mode / Reclaim Policy | 누가 어떻게 볼륨을 쓸 수 있고, 삭제 후 데이터는 어떻게 처리되는가 |
| StatefulSet의 스토리지 결합 | Pod identity와 볼륨이 어떻게 1:1로 묶이는가 |
| 백업·복구(Velero)와 DR | 클러스터 자체가 사라져도 데이터를 복구할 수 있는가 |

{{< cards >}}
  {{< card link="concept" title="Concept" subtitle="PV/PVC/StorageClass 모델, CSI 아키텍처, access mode와 reclaim policy의 의미" >}}
  {{< card link="hands-on" title="Hands-on" subtitle="동적 프로비저닝 실습, StatefulSet 볼륨 트러블슈팅, Velero 백업/복구" >}}
{{< /cards >}}
