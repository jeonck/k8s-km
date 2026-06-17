---
title: Docs
weight: 1
sidebar:
  open: true
---

## MECE로 자른 쿠버네티스

"워크로드, 네트워킹, 보안, 관측성…"을 한 평면에 나열하면 상호배타성이 깨집니다. 보안은 네트워킹에도, 스토리지에도, 워크로드에도 걸치기 때문입니다. 이 지식베이스는 **분류 축을 분리**합니다.

- **명사 축** — 무엇을 관리하는가 (리소스 종류, 상호배타적)
- **품질 축** — 어떤 질문에 답하는가 (안전한가/보이는가/견디는가, 횡단 관심사)

```mermaid
flowchart TB
  L0["계층 0 · Foundation<br/>왜 동작하는가"]
  L1["계층 1 · 리소스 도메인<br/>무엇을 관리하는가"]
  L2["계층 2 · 횡단 관심사<br/>어떤 품질인가"]
  L3["계층 3 · 클러스터 거버넌스<br/>클러스터 자체를 어떻게 운영하는가"]
  L0 --> L1 --> L2 --> L3
```

## 계층 0 — 기반 모델

{{< cards >}}
  {{< card link="foundation" title="Foundation" subtitle="선언적 API, control/data plane, reconciliation loop" >}}
{{< /cards >}}

## 계층 1 — 리소스 도메인 (명사 축)

{{< cards >}}
  {{< card link="workloads-scheduling" title="워크로드 & 스케줄링" subtitle="무엇을 어디서 실행하는가" >}}
  {{< card link="networking" title="네트워킹" subtitle="어떻게 통신하는가" >}}
  {{< card link="storage-state" title="스토리지 & 상태" subtitle="데이터를 어떻게 유지하는가" >}}
  {{< card link="configuration" title="구성 관리" subtitle="설정을 어떻게 주입하는가" >}}
{{< /cards >}}

## 계층 2 — 횡단 관심사 (품질 축)

{{< cards >}}
  {{< card link="security" title="보안" subtitle="안전한가" >}}
  {{< card link="observability" title="관측성" subtitle="보이는가" >}}
  {{< card link="reliability-sre" title="신뢰성 / SRE" subtitle="견디는가" >}}
  {{< card link="extensibility-automation" title="확장성 & 자동화" subtitle="어떻게 확장·자동화하는가" >}}
{{< /cards >}}

## 계층 3 — 클러스터 라이프사이클 & 거버넌스

{{< cards >}}
  {{< card link="cluster-lifecycle" title="클러스터 라이프사이클" subtitle="어떻게 띄우고 유지하는가" >}}
  {{< card link="delivery-gitops" title="전달 & GitOps" subtitle="어떻게 배포·동기화하는가" >}}
  {{< card link="governance-multitenancy" title="거버넌스 & 멀티테넌시" subtitle="어떻게 통제·격리하는가" >}}
  {{< card link="cost-efficiency" title="비용 & 효율" subtitle="어떻게 최적화하는가" >}}
{{< /cards >}}
