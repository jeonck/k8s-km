---
title: "확장성 & 자동화"
weight: 90
---

쿠버네티스의 진짜 힘은 미리 정의된 리소스(Pod, Deployment...)에 있는 게 아니라, **플랫폼 자체를 프로그래밍 가능한 대상으로 다룰 수 있다는 것**에 있습니다. CRD로 새로운 리소스 타입을 정의하고, 컨트롤러로 그 리소스에 대한 reconciliation 로직을 작성하면, 클러스터는 "내가 원하는 도메인 개념"을 1급 시민으로 다루는 플랫폼이 됩니다.

## 핵심 어젠더

- CRD(CustomResourceDefinition)와 커스텀 컨트롤러
- Operator 패턴 (controller-runtime 기반)
- Admission Webhook을 통한 동작 확장 (Mutating/Validating)
- Device Plugin, Scheduler Extension
- 플랫폼 API로서의 추상화 설계

{{< cards >}}
  {{< card link="concept" title="Concept" subtitle="CRD/컨트롤러/Operator의 관계, reconciliation loop의 설계 원칙" >}}
  {{< card link="hands-on" title="Hands-on" subtitle="CRD 정의, 간단한 컨트롤러 스캐폴드, webhook 디버깅 시나리오" >}}
{{< /cards >}}
