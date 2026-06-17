---
title: "관측성"
weight: 70
---

관측성은 "시스템 내부에서 무슨 일이 일어나는지 외부에서 추론할 수 있는가"를 다루는 영역입니다. 메트릭·로그·트레이싱은 서로 다른 질문에 답합니다 — 메트릭은 "무엇이 비정상인가", 로그는 "그때 정확히 무슨 일이 있었는가", 트레이싱은 "요청이 어디서 느려졌는가". 셋 중 하나만 갖추면 장애 원인 분석에 구멍이 생깁니다.

## 핵심 어젠더

| 신호 | 답하는 질문 | 대표 도구 |
| --- | --- | --- |
| 메트릭 | 무엇이 비정상인가 (시계열) | Prometheus, metrics-server |
| 로깅 | 그 시점에 정확히 무슨 일이 있었는가 | Fluent Bit, Loki |
| 트레이싱 | 요청이 어느 구간에서 느려졌는가 | OpenTelemetry, Jaeger |
| 헬스 신호 | 이 컨테이너가 트래픽을 받을 준비가 됐는가 | liveness/readiness/startup probe |
| SLI/SLO | 사용자가 체감하는 신뢰성은 얼마인가 | 에러버짓, 알림 설계 |

{{< cards >}}
  {{< card link="concept" title="Concept" subtitle="메트릭/로그/트레이스의 역할 분리, probe 설계, SLI/SLO 산정 기준" >}}
  {{< card link="hands-on" title="Hands-on" subtitle="PromQL, Loki 쿼리, probe 설정, 알림 설계와 장애 분석 시나리오" >}}
{{< /cards >}}
