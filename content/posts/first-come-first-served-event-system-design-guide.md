---
title: "선착순 이벤트 시스템 설계 가이드"
description: "평시 대비 15~30배 트래픽이 몰리는 선착순 이벤트에서 ALB LCU 예약, nginx 커넥션 한계, WAS 스레드 풀, Bulkhead/CircuitBreaker 연동, 봇 차단까지 실제 운영 경험을 바탕으로 정리한 체크리스트 가이드."
date: 2026-03-20T16:40:00+09:00
tags: ["system-design", "aws", "nginx", "resilience4j", "bulkhead", "circuit-breaker", "traffic", "rate-limiting"]
---

# 선착순 이벤트 시스템 설계 가이드

선착순(FCFS, First-Come-First-Served) 기능은 특정 시각에 트래픽이 극단적으로 집중되는 특성을 갖는다. 평시 대비 15~30배의 트래픽이 수 초 내에 몰리며 이 과정에서 시스템의 약한 고리가 연쇄적으로 무너질 수 있다. 이 문서는 실제 운영에서 겪은 문제를 바탕으로 선착순 기능 도입 시 검토해야 할 항목을 정리한다.

---

## 1. 로드 밸런서 용량 사전 예약 — LCU / Capacity Unit Reservation

### 개념

AWS ALB(Application Load Balancer)는 트래픽에 따라 자동으로 스케일링한다. 그러나 내부적으로 용량을 확장하는 데 시간이 걸리기 때문에, **초당 50% 이상 급격히 증가하는 스파이크** 에는 스케일링이 따라오지 못해 503 오류나 응답 지연이 발생할 수 있다.

**LCU(Load Balancer Capacity Unit)** 는 ALB가 처리하는 트래픽의 양을 측정하는 단위다. 아래 4가지 차원 중 가장 높은 값으로 측정된다.

| 차원 | 1 LCU 기준 |
|---|---|
| 신규 연결 | 초당 25개 |
| 활성 연결 | 분당 3,000개 |
| 처리된 바이트 | 시간당 1 GB (EC2/컨테이너 대상) |
| 규칙 평가 | 초당 1,000개 (첫 10개 규칙 무료) |

**Capacity Unit Reservation** 은 예상 피크 트래픽에 필요한 LCU를 이벤트 전에 사전 예약하는 기능이다. 예약된 용량은 즉시 사용 가능한 상태로 유지되어 스파이크 시 스케일링 지연을 방지한다.

### 문제

ALB의 자동 스케일링은 완만하게 증가하는 트래픽에는 문제가 없지만, 정각에 트래픽이 수직으로 치솟는 선착순 이벤트에서는 용량 확장 전에 이미 요청이 버려진다.

### 해결

#### LCU 산정

예상 피크 RPS를 기준으로 아래와 같이 산정한다.

```
LCU(신규 연결) = 피크 신규 연결/초 ÷ 25
LCU(활성 연결) = 피크 활성 연결/분 ÷ 3,000
LCU(처리 바이트) = 시간당 처리 GB ÷ 1

필요 LCU = max(위 세 값)
```

선착순 이벤트는 짧은 시간에 신규 연결이 폭발적으로 발생하고 keep-alive 재사용률이 낮아 **신규 연결 기준** 이 주로 병목이 된다.

> 예시: 피크 RPS 500, HTTP keep-alive 미적용, 응답 크기 평균 10 KB 가정
> LCU(신규 연결) = 500 ÷ 25 = **20 LCU**
> LCU(활성 연결) = (500 × 60) ÷ 3,000 = **10 LCU**
> LCU(처리 바이트) = (500 × 10 KB × 3,600) ÷ (1 GB × 1,024 MB/GB × 1,024 KB/MB) ≈ **0.02 LCU**
> → **20 LCU 예약**

#### Capacity Unit Reservation 적용

AWS 콘솔(EC2 → Load Balancers → 해당 ALB → Capacity Reservation 탭)에서 예약하거나, 이벤트 규모가 매우 큰 경우 AWS Support에 Pre-warming 요청을 별도로 제출한다.

- Pre-warming 요청 시 이벤트 날짜/시각, 예상 최대 RPS, 평균 응답 크기, 트래픽 증가 패턴을 함께 제공
- 예약된 LCU는 사용 여부와 관계없이 비용이 발생하므로 이벤트 종료 후 해제

---

## 2. 리버스 프록시 커넥션 한계

### 문제

애플리케이션 앞단의 리버스 프록시(nginx 등)에 `worker_connections` 같은 커넥션 제한이 기본값으로 설정되어 있으면 스파이크 트래픽에서 커넥션이 소진되어 요청이 애플리케이션 서버까지 도달하지 못한다.

이 상황에서는 애플리케이션 로그에 아무런 에러가 남지 않기 때문에 원인 파악이 어렵다. nginx error.log에 `worker_connections are not enough` 같은 경고가 나타나는지 확인해야 한다.

```
클라이언트 → nginx (worker_connections 소진) → 요청 전달 불가 → 502/지연
```

### 해결

- nginx `worker_connections`를 서비스 특성에 맞게 상향 (기본 1024 → 최소 2048, I/O 바운드 서비스는 3072 이상)
- 이벤트 전에 `worker_connections × worker_processes`가 예상 동시 접속 수를 수용할 수 있는지 검증
- nginx 레이어의 메트릭(active connections, accepted/handled 비율)을 모니터링에 포함

#### OS 파일 디스크립터 한계 확인

`worker_connections`를 올려도 OS의 파일 디스크립터 한계가 낮으면 실제로 그만큼 열리지 않는다. nginx 설정과 함께 OS 레벨도 함께 조정해야 한다.

```bash
# 현재 한계 확인
ulimit -n
cat /proc/sys/fs/file-max

# /etc/security/limits.conf
nginx  soft  nofile  65536
nginx  hard  nofile  65536

# /etc/sysctl.conf
fs.file-max = 200000
```

nginx의 `worker_rlimit_nofile`도 `worker_connections × 2` 이상으로 설정하는 것이 권장된다 (연결 하나에 upstream/downstream 소켓 각 1개씩 소비).

---

## 3. WAS 스레드 풀 고갈

### 문제

WAS(Tomcat 등)의 스레드 풀이 기본값으로 운영되면 스파이크 트래픽을 수용하지 못한다. 스레드가 부족하면 요청이 큐에 쌓이고 큐마저 가득 차면 요청이 거절된다. 코루틴이나 비동기 디스패처를 사용하는 경우 디스패처 스레드 지연이 메인 스레드까지 무기한 블로킹할 수 있다.

### 해결

- **스레드 풀 튜닝**: `max-threads`, `min-spare-threads`를 인스턴스 스펙과 예상 트래픽에 맞게 설정
- **타임아웃 설정**: 코루틴의 `withTimeout` 등을 활용하여 디스패처 지연 시 메인 스레드의 무기한 대기를 방어
- **스케일아웃 전략**: 이벤트 시작 전에 충분한 인스턴스를 미리 확보하고, 이벤트 종료 후 scale-in하는 스케줄링 적용
- **모니터링 추가**: WAS의 busy threads, max threads 등을 실시간 모니터링에 포함 (`server.tomcat.mbeanregistry.enabled: true` 등)

#### JVM Warm-up / 콜드 스타트

이벤트 전에 스케일아웃으로 인스턴스를 미리 확보하더라도, 새 인스턴스가 로드밸런서에 등록되자마자 실제 피크 트래픽을 받으면 아래 이유로 초기 응답 시간이 평시보다 길 수 있다.

- **JVM JIT 컴파일**: 코드가 처음 실행될 때는 인터프리터 모드로 동작하다가 JIT 컴파일이 완료된 후에야 최적화된 속도가 나온다.
- **커넥션 풀 초기화**: HikariCP 등의 커넥션 풀이 아직 채워지지 않아 연결 생성 비용이 발생한다.
- **캐시 콜드 스타트**: 로컬 캐시가 비어 있어 초기 요청이 모두 캐시 미스를 낸다.

이를 방지하기 위해 새 인스턴스에는 실트래픽 투입 전 워밍 트래픽을 흘리거나, 로드밸런서의 **Slow Start** (트래픽을 서서히 증가시켜 라우팅) 기능을 활성화한다.

```
# AWS ALB Target Group - Slow start duration 설정
aws elbv2 modify-target-group-attributes \
  --target-group-arn <arn> \
  --attributes Key=slow_start.duration_seconds,Value=60
```

---

## 4. 외부 API 장애 전파 — 격벽(Bulkhead) 부재

### 문제

선착순 이벤트에서 가장 치명적인 패턴이다. 시스템 A가 외부 시스템 B를 호출하는데 B의 응답이 느려지면 A의 스레드가 B의 응답을 기다리며 점유된다. 스파이크 트래픽에서는 이것이 수백~수천 개 스레드에 동시에 발생하면서 시스템 A 전체가 마비된다.

CircuitBreaker만으로는 순간적인 스파이크에 대응이 늦다. CB는 일정 수의 실패가 누적된 후에야 열리므로 그 사이에 이미 스레드 풀이 고갈될 수 있다.

```
이벤트 API (스레드 풀 200)
 ├─→ 외부 API A (지연 1초) × 수백 건 → 스레드 점유
 ├─→ 외부 API B (정상 10ms) → 스레드 할당 불가 → 지연
 └─→ 쿠폰 발급 API (정상 30ms) → 스레드 할당 불가 → 지연
     → 외부 API A의 장애가 시스템 전체로 전파
```

외부 시스템의 DB 커넥션 풀(HikariCP 등) 고갈도 자주 동반된다. 스파이크 트래픽이 외부 시스템으로 한꺼번에 밀려들면 커넥션 풀이 포화되고 커넥션 대기 시간이 급증하면서 응답 시간이 timeout 수준까지 치솟는다.

#### Timeout 불일치 문제

**원칙**: callee 서버가 응답을 만들기 위해 소요하는 최대 시간(DB `connectionTimeout` 포함)이 caller의 `readTimeout`보다 짧아야 한다. 그렇지 않으면 caller가 먼저 연결을 끊은 후에도 callee는 이미 아무도 기다리지 않는 요청을 처리하기 위해 리소스를 계속 점유한다.

```
caller readTimeout:        1,000ms  → 먼저 끊음 (SocketTimeoutException)
callee DB connectionTimeout: 3,000ms  → 끊긴 후에도 2초간 커넥션 점유 계속
```

이 상황이 대량으로 발생하면 callee의 DB 커넥션 풀 고갈을 가속시킨다.

### 해결

**Bulkhead(격벽) 도입**: 외부 API별로 동시 호출 수를 제한하여, 특정 API의 지연이 전체 스레드 풀을 잠식하지 못하도록 격리한다.

#### SemaphoreBulkhead vs ThreadPoolBulkhead

Resilience4j는 두 가지 Bulkhead 구현을 제공한다.

| 구분 | SemaphoreBulkhead | ThreadPoolBulkhead |
|---|---|---|
| 동작 방식 | caller 스레드에서 세마포어 획득 후 실행 | 별도 스레드 풀에서 실행 |
| Tomcat 스레드 보호 | permit 초과 시 즉시 실패 → 보호 | Tomcat 스레드를 소비하지 않음 → 더 강력 |
| 오버헤드 | 낮음 | 스레드 컨텍스트 전환 비용 발생 |
| 선착순 이벤트 권장 | **maxWaitDuration=0ms 필수** | 구성이 복잡하나 Tomcat 스레드 완전 분리 가능 |

이 문서의 예시는 `SemaphoreBulkhead` 기준이다. Tomcat 스레드를 완전히 보호하려면 `ThreadPoolBulkhead`가 더 강력하지만 스레드 풀 크기, 큐 설정 등 관리 포인트가 늘어난다.

#### maxConcurrentCalls 산정 — Little's Law

```
인스턴스당 concurrent = (전체 RPS ÷ 인스턴스 수) × 평균 응답시간(초)
```

평시의 인스턴스당 concurrent를 기준으로 충분한 여유(5~10배)를 두고 설정한다. 평시에는 Bulkhead가 발동하지 않고, 스파이크에서만 초과하는 구조가 이상적이다.

외부 시스템의 수용 능력을 기준으로 산정하는 것도 가능하다:

```
maxConcurrentCalls = 외부 시스템 수용 TPS × 평균 응답시간(초) / 우리 서버 대수
```

> `maxConcurrentCalls` 산정 시에는 실패 요청도 timeout까지 slot을 점유하므로, 성공 응답시간과 실패(timeout) 응답시간의 **가중 평균** 을 사용해야 한다. 예를 들어 성공 비율 90%(50ms), 실패 비율 10%(1,000ms)라면 가중 평균 = 0.9 × 50 + 0.1 × 1,000 = 145ms.

#### maxWaitDuration — 선착순 이벤트에서는 0ms 권장

`SemaphoreBulkhead`의 `maxWaitDuration`을 0보다 크게 설정하면, slot이 꽉 찬 상태에서 들어오는 **모든 초과 요청의 스레드가 permit을 얻을 때까지 블로킹** 된다.

| maxWaitDuration | 동작 | 선착순 이벤트에서의 문제 |
|---|---|---|
| 500ms | 500ms 동안 스레드 점유 후 실패 | 스레드 낭비 + 지연 전파 |
| 100ms | 100ms 동안 스레드 점유 후 실패 | 초과 요청이 동시에 블로킹되어 스레드 풀 고갈 가능 |
| **0ms** | **즉시 BulkheadFullException → fallback** | **대기 스레드 0개. 가장 안전** |

수천 건이 한꺼번에 몰리는 선착순 이벤트 특성상 0보다 큰 값은 격벽의 본래 목적(스레드 보호)과 반대로 동작할 수 있다. 외부 API 응답이 timeout(예: 1초)까지 걸리는 상황에서 slot은 1초마다 하나씩 열리므로, 100ms를 기다려봐야 slot을 얻을 확률이 ~10%에 불과하다.

#### Bulkhead와 CircuitBreaker의 연동

Bulkhead를 적용하면 `BulkheadFullException`이 CircuitBreaker의 실패율에 합산되어 서킷이 불필요하게 열릴 수 있다. 격벽이 동시 호출을 제한하면서 서킷은 **실제 API 장애에만** 반응하도록, `ignoreExceptions`에 `BulkheadFullException`을 추가해야 한다.

```yaml
resilience4j.circuitbreaker:
  configs:
    default:
      ignoreExceptions:
        - io.github.resilience4j.bulkhead.BulkheadFullException
```

#### Timeout 정합성 확보

callee 서버의 DB `connectionTimeout`(HikariCP 기준)을 caller의 `readTimeout`보다 **짧게** 설정해야 한다. callee가 먼저 실패해야 이미 끊긴 요청이 리소스를 계속 점유하는 것을 방지할 수 있다.

```
caller readTimeout:           1,000ms
callee DB connectionTimeout:    800ms  ← caller readTimeout보다 짧게
```

---

## 5. 쓰기 경로의 보호 장치 누락

### 문제

읽기 API에는 CircuitBreaker/Bulkhead를 적용했지만 쓰기 API(발급, 결제 등)에는 적용하지 않는 경우가 흔하다. "쓰기에 서킷을 걸면 UX에 영향이 크다"는 판단이 이유인 경우가 많다.

그러나 보호 장치가 없는 쓰기 경로에서 외부 API timeout(예: 1초)이 대량 발생하면 스레드가 점유되면서 **보호 장치가 걸려 있는 읽기 경로까지 영향** 을 준다. Bulkhead는 Tomcat 스레드를 할당받은 **후에** 동작하므로, 보호되지 않은 경로의 스레드 점유가 보호된 경로의 스레드 할당을 방해한다.

```
[요청 흐름]
Tomcat 스레드 할당 → Bulkhead 체크 → downstream 호출
↑
보호되지 않은 쓰기 요청이 여기서 스레드를 고갈시킴
```

또한 `Read timed out`(서버가 응답을 안 줌)과 `Connect timed out`(TCP handshake 자체 실패)은 원인이 다르다. 후자는 대상 서버의 Accept 큐가 완전히 포화된 상태를 의미하며, 이미 서버가 새로운 연결을 수용할 수 없는 상태다.

### 해결

- 외부 API를 호출하는 **모든 경로** (읽기/쓰기)에 Bulkhead와 CircuitBreaker를 적용
- 쓰기 경로에 서킷을 적용하되 보수적 임계값을 사용하여 일시적 에러에 쉽게 열리지 않도록 설정

| 설정 항목 | 권장 범위 | 비고 |
|---|---|---|
| minimumNumberOfCalls | 20~50 | 일시적 에러에 서킷이 쉽게 열리지 않음 |
| failureRateThreshold | 50% | 실제 장애 시에만 반응 |
| waitDurationInOpenState | 5~15s | 빠른 실패 + 적절한 복구 대기 |
| slowCallDurationThreshold | readTimeout의 70~80% | timeout 직전의 slow call을 감지 |
| slowCallRateThreshold | 50~80% | slow call 비율이 임계치 초과 시 서킷 오픈 |

---

## 6. 프론트엔드 반복 호출

### 문제

이벤트 시작 시각 전후에 사용자가 새로고침을 반복하거나 이벤트 페이지 자체가 서버를 주기적으로 polling하면서 서버 호출이 급증한다. 이 트래픽은 실제 발급 요청보다 훨씬 많을 수 있다.

### 해결

- **이벤트 시간 전후 캐시**: 이벤트 시작 N분 전 ~ N분 후 동안 서버 응답을 프론트에서 캐시하여 반복 호출 차단
- **최종적 일관성 허용**: 그 시간 동안 실시간 정보가 아닌 캐시된 값이 노출될 수 있지만 최종적 일관성(eventual consistency)이 보장된다면 수용 가능한 트레이드오프

---

## 7. 재고 소진 후 불필요한 호출

### 문제

선착순 재고(쿠폰, 상품 등)가 소진된 이후에도 발급/구매 요청이 계속 서버에 인입된다. 매 요청마다 외부 시스템에 재고를 확인하면 불필요한 부하가 지속된다.

### 해결

- **로컬 캐시로 소진 상태 저장**: 재고 소진 응답을 받으면 일정 시간(예: 1분) 로컬 캐시에 저장하여, 이후 요청은 외부 시스템을 호출하지 않고 즉시 "소진" 응답 반환
- **주기적 캐시 갱신**: 1분마다 외부 시스템을 호출하여 캐시를 갱신하고, 피크 타임에는 외부 시스템 직접 호출을 방지

---

## 8. 봇/비정상 트래픽

### 문제

선착순 이벤트는 봇의 표적이 되기 쉽다. 이벤트 시작 전부터 API를 직접 호출하거나 정상 사용자의 클릭 속도를 초과하는 요청을 보내는 패턴이 관찰된다. 이벤트가 반복될수록 비정상 사용자가 증가하는 경향이 있다.

### 해결

다층 방어를 적용한다.

**CDN/WAF 레벨 (Cloudflare, AWS WAF 등)**:
- **이벤트 시작 전 차단**: 시작 전에는 UI 버튼이 비활성화되어 정상 사용자는 요청을 보내지 않으므로, 이 시간대 요청에 대해 서버에서 400을 반환하고 그 IP를 Rate Limit으로 차단. 오탐률이 0%에 가까움
- **Rate Limit**: 서비스의 정상 사용 패턴(예: 1초에 1회 클릭)을 기준으로 10초 단위로 환산. 정상 사용자에게 여유 있는 임계값(예: 10초에 5회)을 설정하되 봇에게는 치명적인 수준으로
- **Bot Score 기반 차단**: 앱 내 웹뷰 등에서 봇 스코어가 낮은 요청을 차단. 오탐 우려 시 보수적으로 적용 (예: 10점 미만만 차단)

**서버 레벨**:
- Rate Limiter(bucket4j, Guava RateLimiter 등)를 적용하여 사용자별 요청 빈도 제한
- 다중 인스턴스 환경에서 로컬 캐시 기반 Rate Limiter는 **인스턴스 수만큼 실효 한도가 늘어나는 문제** 가 있다. 예를 들어 인스턴스 10대에서 사용자별 10 req/s 제한을 로컬에서 적용하면, 사용자가 각 인스턴스에 1 req/s씩 분산하면 실제로는 10 req/s를 허용하게 된다.
- 정확한 분산 Rate Limiting이 필요하면 **bucket4j + Redis(Lettuce)** 조합으로 Sliding Window를 공유 카운터로 관리한다. Redis가 없는 환경에서는 로컬 캐시 + Sticky Session으로 대체하되, 이 한계를 인지하고 CDN/WAF 레벨 Rate Limit을 주 방어선으로 삼아야 한다.

**프론트엔드 레벨**:
- 발급 버튼에 쓰로틀링(예: 1초에 1회) 적용
- 발급 완료/소진 시 버튼 즉시 비활성화

---

## 9. 에러 처리 전략 — 과부하 vs 장애 구분

### 문제

외부 API 호출 실패 시 일괄적으로 5xx를 반환하면 프론트에서 재시도 페이지를 노출하고 사용자가 재시도를 하면서 트래픽이 더 증폭된다. 격벽/서킷이 열릴 정도의 과부하 상황에서 재시도를 유도하면 상황이 악화된다.

### 해결

실패 유형에 따라 응답을 분기한다.

| 상태 | HTTP 응답 | 프론트 동작 |
|---|---|---|
| 정상 | 200 + 데이터 | 정상 페이지 |
| **과부하** (Bulkhead Full / CB Open) | **503 Service Unavailable** | "접속이 많아 일시적으로 표시할 수 없습니다" (재시도 안내 없음) |
| 외부 서비스 장애 | 502 Bad Gateway | 재시도 페이지 |

핵심은 **과부하 상태에서는 트래픽을 줄이는 방향** 으로, **실제 장애에서는 재시도를 유도하는 방향** 으로 UX를 설계하는 것이다.

또한 외부 호출 실패 시 예외를 던져 5xx를 반환하는 것보다 graceful degradation(빈 리스트 반환, 기본값 사용 등)이 가능한지 검토한다. 페이지의 핵심이 아닌 보조 데이터(추천, 배너 등)는 외부 호출 실패 시 빈 상태로 렌더링해도 UX에 큰 영향이 없는 경우가 많다.

```kotlin
// 비권장: 보조 데이터 실패 시 전체 페이지 에러
val brands = brandApi.getBrands().getOrThrow { ServerException() }

// 권장: 보조 데이터 실패 시 빈 리스트로 대체
val brands = brandApi.getBrands().getOrElse { emptyList() }
```

---

## 체크리스트

선착순 이벤트 출시 전에 아래 항목을 확인한다.

### 인프라

- [ ] 예상 피크 RPS 기준으로 필요한 LCU를 산정하고, Capacity Unit Reservation 또는 Pre-warming 요청을 제출했는가?
- [ ] nginx `worker_connections`가 예상 동시 접속 수를 수용할 수 있는가?
- [ ] nginx의 `worker_rlimit_nofile`과 OS의 파일 디스크립터 한계(`ulimit -n`)가 충분한가?
- [ ] WAS 스레드 풀(`max-threads`)이 적절하게 설정되어 있는가?
- [ ] 이벤트 시간 전 충분한 인스턴스로 스케일아웃되는가?
- [ ] 새 인스턴스에 Slow Start 또는 Warm-up 트래픽이 적용되어 콜드 스타트 문제가 방지되는가?
- [ ] 이벤트 종료 후 스케일인되는 스케줄이 있는가?

### 장애 격리

- [ ] 외부 API를 호출하는 **모든 경로** (읽기/쓰기)에 Bulkhead가 적용되어 있는가?
- [ ] `maxConcurrentCalls`를 Little's Law로 산정하고, 성공/실패 가중 평균 응답시간을 사용했는가?
- [ ] `maxWaitDuration`이 0ms로 설정되어 있는가? (선착순 이벤트 특성상 권장)
- [ ] CircuitBreaker에 `BulkheadFullException`이 `ignoreExceptions`에 포함되어 있는가?
- [ ] callee의 DB `connectionTimeout`이 caller의 `readTimeout`보다 짧게 설정되어 있는가?

### 트래픽 제어

- [ ] 이벤트 시간 전후 프론트엔드 캐시가 적용되어 있는가?
- [ ] 재고 소진 시 로컬 캐시로 불필요한 외부 호출을 차단하는가?
- [ ] 봇 차단을 위한 Rate Limit / Bot Score 정책이 수립되어 있는가?
- [ ] 다중 인스턴스 환경에서 Rate Limiting이 로컬 캐시 기반이라면 그 한계를 인지하고 CDN/WAF를 주 방어선으로 설정했는가?
- [ ] 프론트에서 발급 버튼에 쓰로틀링이 걸려 있는가?

### 에러 처리

- [ ] 과부하(Bulkhead Full/CB Open)와 실제 장애를 구분하여 응답하는가?
- [ ] 보조 데이터(추천, 배너 등)는 외부 호출 실패 시 graceful degradation하는가?

### 모니터링

- [ ] Bulkhead available slots 모니터링이 설정되어 있는가?
- [ ] Bulkhead rejected 건수에 대한 알람이 있는가?
- [ ] DB 커넥션 풀(pending/active) 모니터링이 있는가?
- [ ] CircuitBreaker 상태(open/closed) 모니터링이 있는가?
- [ ] WAS 스레드 상태(busy/max) 모니터링이 있는가?

---

*이 글은 AI의 도움을 받아 교정 및 정리되었습니다.*
