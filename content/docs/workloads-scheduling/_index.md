---
title: "워크로드 & 스케줄링"
weight: 20
---

"무엇을 어디서 실행하는가" — 실행 단위의 정의와 배치를 다루는 영역입니다.

## 핵심 어젠더

| 영역 | 내용 |
| --- | --- |
| 실행 단위 | Pod와 상위 컨트롤러(Deployment / StatefulSet / DaemonSet / Job·CronJob)의 선택 기준 |
| 스케줄링 제어 | affinity·anti-affinity, taints·tolerations, topology spread, priority·preemption |
| 리소스 모델 | requests·limits, QoS class, LimitRange·ResourceQuota |
| 오토스케일링 | HPA·VPA·Cluster Autoscaler·Karpenter |

{{< cards >}}
  {{< card link="concept" title="Concept" subtitle="컨트롤러 선택 기준, 스케줄링 제어, 리소스 모델, 오토스케일링 비교" >}}
  {{< card link="hands-on" title="Hands-on" subtitle="Pending/CrashLoopBackOff 디버깅, affinity 설정, HPA 구성" >}}
{{< /cards >}}
