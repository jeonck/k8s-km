---
title: "기반 모델 (Foundation)"
weight: 10
---

다른 모든 카테고리가 전제하는 "왜 동작하는가"의 영역입니다. 여기가 흔들리면 나머지는 암기가 됩니다.

선언적 API와 reconciliation 패러다임, 그 위에서 도는 컴포넌트(control plane / data plane)를 다룹니다.

## 핵심 어젠더

| 영역 | 내용 |
| --- | --- |
| Control plane | API server / etcd / scheduler / controller-manager |
| Data plane | kubelet / kube-proxy / CRI 런타임 |
| 패러다임 | 선언적 모델과 컨트롤러 reconciliation loop |
| API 객체 모델 | GVK(Group/Version/Kind), 그룹/버전/리소스 |
| 저장소 | etcd 데이터 모델과 일관성(linearizable read, Raft) |
| 핵심 개념 | desired state vs actual state |

{{< cards >}}
  {{< card link="concept" title="Concept" subtitle="control/data plane 구조, reconciliation loop, API 객체 모델" >}}
  {{< card link="hands-on" title="Hands-on" subtitle="etcd 점검, API 호출 추적, 컨트롤러 동작 디버깅" >}}
{{< /cards >}}
