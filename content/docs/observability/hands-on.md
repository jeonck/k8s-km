---
title: Hands-on
weight: 2
---

## 1. Probe 설정 예시

```yaml
spec:
  containers:
    - name: app
      startupProbe:
        httpGet: { path: /healthz, port: 8080 }
        failureThreshold: 30
        periodSeconds: 2   # 최대 60초 부팅 대기
      livenessProbe:
        httpGet: { path: /healthz, port: 8080 }
        periodSeconds: 10
        failureThreshold: 3
      readinessProbe:
        httpGet: { path: /ready, port: 8080 }
        periodSeconds: 5
        failureThreshold: 2
```

## 2. PromQL — 자주 쓰는 패턴

```promql
# 5분간 5xx 비율
sum(rate(http_requests_total{status=~"5.."}[5m]))
/ sum(rate(http_requests_total[5m]))

# p99 레이턴시
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# 메모리 사용량이 limit의 90%를 넘은 Pod
kube_pod_container_resource_limits{resource="memory"} * 0.9
  < on(pod, container) group_left
  container_memory_working_set_bytes

# 번 레이트 알림 (에러버짓을 1시간 내 소진할 속도)
(
  sum(rate(http_requests_total{status=~"5.."}[1h])) /
  sum(rate(http_requests_total[1h]))
) > (14.4 * 0.001)   # SLO 99.9% 기준 fast-burn 임계
```

## 3. Loki(LogQL) 쿼리

```logql
{namespace="payments", app="checkout"} |= "panic" | json | line_format "{{.msg}}"

# 5xx 응답을 낸 요청만 필터 후 카운트
sum by (status) (count_over_time({app="checkout"} | json | status>=500 [5m]))
```

## 4. 트러블슈팅 — "메트릭은 정상인데 사용자는 느리다고 한다"

**현상**: Prometheus 대시보드의 CPU/메모리/요청률 모두 정상 범위. 그런데 고객 지원팀에 "체크아웃이 느리다"는 문의가 들어옴.

**가설**: 평균 레이턴시는 정상이지만 특정 percentile(p99)만 튀거나, 특정 다운스트림 서비스 한 구간에서만 지연이 발생해 평균에 묻혔을 가능성.

**확인 순서**:

```bash
# 1) p50/p90/p99을 분리해서 비교 — 평균만 보면 꼬리 지연을 놓친다
# (PromQL) histogram_quantile(0.50/0.90/0.99, ...) 세 가지를 동시에 그려본다

# 2) 트레이싱에서 느린 트레이스만 필터링
# Jaeger UI에서 duration > 2s 필터 후 span 분포 확인 → 어느 서비스 구간이 긴지 식별

# 3) 해당 구간 서비스의 로그를 시간 윈도우로 좁혀 확인
kubectl logs -n payments deploy/inventory-svc --since=10m | grep -i timeout
```

**조치**: 트레이스에서 특정 다운스트림(예: 재고 서비스) 호출에서만 p99가 튀는 것을 확인했다면, 해당 서비스의 커넥션 풀/타임아웃 설정을 점검하거나 캐시를 도입합니다. 평균 기반 알림만 쓰던 팀은 이 사건을 계기로 p99 기반 SLO 알림을 추가해야 합니다.

## 운영 체크리스트

- [ ] 모든 운영 워크로드에 readiness와 liveness가 명확히 분리되어 있는가 (의존성 대기 ≠ 데드락)
- [ ] 알림이 단순 임계값이 아니라 번 레이트/SLO 기반인가
- [ ] 로그 보존 기간이 평균 장애 분석 소요 시간보다 충분히 긴가
- [ ] 트레이싱 샘플링 비율이 너무 낮아 드문 장애를 놓치고 있지 않은가
