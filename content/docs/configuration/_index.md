---
title: "구성 관리"
weight: 50
---

## 범위

**설정을 어떻게 주입하는가** — 코드와 분리된 설정을 외부화하는 메커니즘을 다룹니다. 이 카테고리는 "주입 방식"과 "패키징/오버레이 도구"까지만 다룹니다. Secret의 암호화·접근통제·KMS 연동 같은 보안 측면은 [보안](../security) 카테고리에서 다룹니다.

## 핵심 어젠더

| 항목 | 다루는 질문 |
| --- | --- |
| ConfigMap · Secret | 설정값을 어떻게 코드와 분리해서 보관하는가 |
| 주입 방식 (env / volume / downward API) | 그 설정값을 컨테이너 안으로 어떻게 전달하는가 |
| Kustomize | 환경별(dev/stg/prod) 차이를 어떻게 오버레이로 관리하는가 |
| Helm | 재사용 가능한 패키지/템플릿으로 어떻게 배포하는가 |
| 설정 드리프트 관리 | 선언된 설정과 실제 클러스터 상태가 어떻게 벌어지는지, 어떻게 막는가 |

{{< cards >}}
  {{< card link="concept" title="Concept" subtitle="ConfigMap/Secret 모델, 주입 방식 비교, Kustomize vs Helm 의사결정 기준" >}}
  {{< card link="hands-on" title="Hands-on" subtitle="실제 매니페스트, 핫리로드 트러블슈팅, Kustomize/Helm 실습" >}}
{{< /cards >}}
