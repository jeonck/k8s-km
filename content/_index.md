---
title: Kubernetes 실무 가이드
layout: hextra-home
---

<div class="hx:mt-6 hx:mb-6">
{{< hextra/hero-headline >}}
  쿠버네티스를 MECE로,&nbsp;<br class="hx:sm:block hx:hidden" />실무로 익히는 지식베이스
{{< /hextra/hero-headline >}}
</div>

<div class="hx:mb-12">
{{< hextra/hero-subtitle >}}
  "워크로드·네트워킹·보안…"을 한 평면에 나열하지 않습니다.&nbsp;<br class="hx:sm:block hx:hidden" />
  명사 축(리소스)과 품질 축(횡단 관심사)을 분리한 4계층 12카테고리 구조로 정리했습니다.
{{< /hextra/hero-subtitle >}}
</div>

<div class="hx:mb-6">
{{< hextra/hero-button text="Docs 시작하기" link="docs" >}}
</div>

<div class="hx:mt-6"></div>

{{< hextra/feature-grid >}}
  {{< hextra/feature-card
    title="계층 0 · 기반 모델"
    subtitle="선언적 API, control/data plane, reconciliation loop — 모든 것이 전제하는 토대."
    link="docs/foundation"
  >}}
  {{< hextra/feature-card
    title="계층 1 · 리소스 도메인"
    subtitle="무엇을 관리하는가. 워크로드&스케줄링 · 네트워킹 · 스토리지&상태 · 구성관리 — 상호배타적인 명사 축."
    link="docs"
  >}}
  {{< hextra/feature-card
    title="계층 2 · 횡단 관심사"
    subtitle="어떤 질문에 답하는가. 보안 · 관측성 · 신뢰성/SRE · 확장성&자동화 — 리소스를 가로지르는 품질 축."
    link="docs"
  >}}
  {{< hextra/feature-card
    title="계층 3 · 클러스터 거버넌스"
    subtitle="클러스터 자체의 운영. 라이프사이클 · 전달/GitOps · 거버넌스/멀티테넌시 · 비용&효율."
    link="docs"
  >}}
  {{< hextra/feature-card
    title="Concept + Hands-on 페어"
    subtitle="모든 카테고리는 개념 설명과 실제 명령어·매니페스트·트러블슈팅이 포함된 실습으로 짝지어집니다."
  >}}
  {{< hextra/feature-card
    title="검색 / 사이드바 / 목차"
    subtitle="FlexSearch 전문 검색과 자동 생성 사이드바, 페이지별 목차로 빠르게 원하는 내용을 찾습니다."
  >}}
{{< /hextra/feature-grid >}}
