---
title: "보안"
weight: 60
---

쿠버네티스 보안은 단일 기능이 아니라 **방어 깊이(defense in depth)**입니다. 클러스터에 누가 들어올 수 있는지(인증·인가)부터, Pod가 무엇을 할 수 있는지(런타임 제약), 어떤 정책이 강제되는지(어드미션), 이미지가 신뢰 가능한지(공급망), 이상 행위를 탐지하는지(런타임 보안), 그리고 민감 정보가 어떻게 보호되는지(Secret 암호화)까지를 모두 포함합니다.

이 카테고리는 "구성 관리"에서 다루는 ConfigMap/Secret의 *주입 메커니즘*과는 다릅니다. 여기서는 Secret의 **암호화와 접근통제**, 즉 "그 값을 누가 읽을 수 있고 어떻게 평문 노출을 막는가"를 다룹니다.

## 핵심 어젠더

| 영역 | 다루는 질문 | 대표 도구 |
| --- | --- | --- |
| 인증·인가 | 누가 클러스터에 무엇을 할 수 있는가 | RBAC, ServiceAccount, OIDC |
| Pod 보안 | 컨테이너가 호스트에 어떤 권한을 가지는가 | Pod Security Admission, securityContext, seccomp/AppArmor |
| 정책 강제 | 어떤 리소스 생성을 막거나 변형하는가 | Admission Webhook, OPA/Gatekeeper, Kyverno |
| 공급망 보안 | 이 이미지를 신뢰할 수 있는가 | cosign, SBOM, 이미지 스캐닝 |
| 런타임 탐지 | 지금 클러스터 안에서 이상 행위가 있는가 | Falco, eBPF |
| Secret 보호 | 민감 정보가 평문으로 노출되지 않는가 | KMS encryption-at-rest, External Secrets, Vault, SOPS |

{{< callout type="warning" >}}
PodSecurityPolicy(PSP)는 Kubernetes 1.25에서 완전히 제거되었습니다. 과거 자료를 참고할 때 PSP 기반 예시는 Pod Security Admission(PSA)으로 치환해서 읽어야 합니다.
{{< /callout >}}

{{< cards >}}
  {{< card link="concept" title="Concept" subtitle="방어 깊이 모델, RBAC/PSA/어드미션 체인의 동작 원리와 선택 기준" >}}
  {{< card link="hands-on" title="Hands-on" subtitle="RBAC 최소권한 설계, Kyverno 정책, Secret 암호화, 권한 상승 트러블슈팅" >}}
  {{< card link="psa-restricted-migration" title="PSA 심화: Baseline → Restricted 전환" subtitle="레벨별 차단 항목, securityContext 모범답안, 단계적 상향 전략" >}}
  {{< card link="privilege-escalation" title="privileged vs allowPrivilegeEscalation" subtitle="내부 승진과 외부 탈취의 차이, no_new_privs, 탐지·트러블슈팅" >}}
  {{< card link="sbom-with-bom" title="공급망 보안 심화: bom으로 SBOM 만들기" subtitle="SBOM 생성·스캔·CI/CD 게이팅, Log4j급 취약점 긴급 대응" >}}
{{< /cards >}}
