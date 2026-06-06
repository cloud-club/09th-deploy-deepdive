# SSE 구독 부하 테스트 & Tomcat max-connections 상향

> 비회원 동시 SSE 구독 부하 테스트와, 그 결과를 반영한 운영 환경 Tomcat 커넥터 튜닝 기록.

---

## 1. 배경

축제 공지(notification)는 **SSE(Server-Sent Events)** 로 실시간 푸시된다. 축제 피크 시간대에는
수천명의 유저가 동시에 지도 화면에 머무르며 SSE 연결을 점유한다.

따라서 다음을 운영 투입 전에 검증해야 했다.

- 서버가 **10,000개 동시 SSE 연결**을 수용·유지할 수 있는가
- 5분(서버측 `SseEmitter` timeout) 동안 연결이 끊김 없이 유지되는가
- 동시 접속 폭주 시 연결 establish latency가 합리적인가

### 테스트 대상 인프라

부하 테스트는 아래의 **실제 운영 인프라**를 대상으로 진행했다.

| 구성 요소 | 사양 / 설정 |
|-----------|-------------|
| 애플리케이션 서버 | **AWS EC2 `m7i.xlarge`** (4 vCPU / 16 GiB) |
| 데이터베이스 | **AWS Aurora** (writer/reader 구성), **오토스케일링 적용** — 부하 증가 시 reader 인스턴스가 자동 확장 |

> 즉 DB 계층은 오토스케일링이 동작하는 운영 구성 그대로 두고, **애플리케이션 서버가 10,000 동시
> SSE 연결을 견디는지**를 확인하는 테스트다.

---

## 2. 대상 엔드포인트

```
GET /api/v1/festivals/{festivalId}/sse/subscribe
```

- `PermitUrlConfig`에 등록되어 **인증 불필요** (비회원 구독)
- 서버측 `SseEmitter` timeout: **300,000ms (5분)** → 5분 후 서버가 연결을 정상 종료
- **30초마다 heartbeat** 이벤트 전송 (`NotificationSseEmitterManager.sendHeartbeat`)
- SSE fan-out은 가상 스레드로 처리

---

## 3. 부하 테스트 시나리오

도구는 **k6**(`ramping-vus` executor)를 사용했다. 1 VU = 1 SSE 연결이며, 각 iteration이
5분간 연결을 점유한다.

| 단계 | 구간 | 목표 VU |
|------|------|---------|
| 램프업 1 | 0 → 30s | 2,000 |
| 램프업 2 | 30s → 60s | 5,000 |
| 램프업 3 | 60s → 90s | 10,000 |
| **유지** | 4분 | **10,000 (동시 연결 유지)** |
| 쿨다운 | 20s | 0 |

전체 실행 시간: 약 **6분 25초** (`testRunDurationMs ≈ 385,035ms`)

### Threshold (합격 기준)

| 지표 | 기준 |
|------|------|
| `status_5xx` | `count == 0` |
| `sse_connected` (연결 성공률) | `rate > 0.95` |
| `connection_timeout` | `count < 500` |
| `sse_connect_latency_ms` | `p95 < 3000ms`, `p99 < 5000ms` |

### 테스트 머신 사전 준비

k6는 10,000 VU = 10,000 TCP 소켓을 동시에 오픈한다. macOS 기본 `ulimit -n`은 256이라
도중에 `too many open files`로 테스트 자체가 죽는다. **k6 실행 PC에서만** 한도를 올린다.

```bash
ulimit -n 32768   # 이 셸 세션에만 적용. k6 실행 직전 같은 터미널에서 실행
```

> 운영 서버의 OS/JVM fd 한도는 인프라 레벨에서 별도로 충분히 잡혀 있다는 가정(별개 문제).

### 실행

```bash
# 사전 검증 — 200 + Content-Type: text/event-stream 확인
curl -sI "https://api.example.place/api/v1/festivals/exampleId/sse/subscribe"

# 부하 테스트 실행
k6 run infra/loadtest/sse-notification.js

# 환경변수 override
BASE_URL=https://api.example.place FESTIVAL_ID=exampleId TARGET_VUS=10000 \
  k6 run infra/loadtest/sse-notification.js

# Grafana(Mimir/Prometheus)로 메트릭 전송
k6 run --out experimental-prometheus-rw infra/loadtest/sse-notification.js
```

---

## 4. 측정의 한계

vanilla k6는 SSE 스트림을 chunk 단위로 못 읽어서, 이 스크립트는 **연결 establish 여부와
헤더 도착 latency**만 측정한다. 다음은 **Grafana에서 함께** 봐야 한다.

- 실시간 이벤트 전달 latency (notification 발행 → 구독자 수신까지) — *측정하려면 `xk6-sse` 확장 필요*
- JVM heap / GC pause
- Tomcat thread / 가상 스레드 활성 수
- 서버 file descriptor 사용량 (`ss -s`, `lsof`)
- `NotificationSseEmitterManager.getTotalSubscribers()` actuator 노출 권장

---

## 5. 결과

**모든 threshold 통과 ✅**

| 지표 | 결과 | 기준 | 판정 |
|------|------|------|------|
| 최대 동시 VU | **10,000** | 10,000 | ✅ |
| 총 연결 수 (iterations) | 6,200 | — | — |
| 연결 성공률 (`sse_connected`, status=200) | **100%** | ≥ 95% | ✅ |
| 5xx | **0** | 0 | ✅ |
| client timeout (`connection_timeout`) | **0** | < 500 | ✅ |
| 실패율 (`http_req_failed`) | ~0.016% | — | ✅ |

### 연결 establish latency (헤더 도착까지, ms)

| avg | med | p90 | p95 | max |
|-----|-----|-----|-----|-----|
| 265 | 228 | 420 | 555 | 1,624 |

- p95 555ms ≪ 기준 3,000ms → 동시 10,000 접속에서도 응답 시작이 빠름
- iteration_duration이 일관되게 ~300,000ms(5분)에 수렴 → **서버가 5분 timeout으로 모든 연결을
  정상 종료**했고, 클라이언트 강제 timeout(350s)에 걸린 연결은 0건

### 해석

10,000 동시 SSE 연결을 5xx·timeout 없이 수용·유지하고, 5분 후 전부 정상 종료했다.
즉 **애플리케이션 레이어(가상 스레드 fan-out, SseEmitter 관리)는 10,000 동시 연결을 문제없이 처리**한다.
병목은 그 앞단인 **Tomcat 커넥터의 커넥션 수용 한도**였다 (→ 6장).

---

## 6. Tomcat max-connections 상향

### 문제

Spring Boot 내장 Tomcat의 기본값은 **`max-connections: 8192`**, **`accept-count: 100`** 이다.
SSE 연결은 5분간 소켓을 점유하는 **장기 연결(long-lived connection)** 이라, 일반 요청과 달리
빠르게 반납되지 않는다. 따라서 동시 구독자 수가 8,192를 넘는 순간:

- 신규 연결은 OS accept 큐(`accept-count`, 기본 100)에 쌓이다가
- 큐마저 차면 **connection refused / 연결 지연**이 발생한다.

10,000 동시 구독을 목표로 하는 상황에서 기본값(8,192)은 **그대로 한계점**이었다.

### 조치

`src/main/resources/application-prod.yml`:

```yaml
server:
  shutdown: graceful
  tomcat:
    max-connections: 30000   # 기본 8192 → 30000
    accept-count: 500        # 기본 100 → 500
```

| 항목 | 기본값 | 변경값 | 의미 |
|------|--------|--------|------|
| `max-connections` | 8,192 | **30,000** | Tomcat이 동시에 수용·유지하는 최대 커넥션 수. SSE 장기 연결이 누적돼도 여유 확보 |
| `accept-count` | 100 | **500** | 모든 처리 스레드가 바쁠 때 OS accept 큐에 대기시킬 연결 수. 순간 폭주(thundering herd) 흡수 |

> `max-connections`는 **동시 수용 커넥션 수**(스레드 수 `max-threads`와 별개)다. SSE처럼 스레드를
> 오래 점유하지 않는 장기 연결에서는 이 값이 동시 구독자 상한을 결정한다.

### 효과

상향 후 동일 시나리오(10,000 VU)에서 5xx·connection refused 없이 전 구간 100% 연결 성공
(5장 결과). 30,000 한도는 10,000 목표 대비 약 3배 헤드룸을 확보한다.

---

## 7. 남은 궁금증

- **인스턴스 선택이 효율적이었는가** — 이번에는 안정적인 운영을 최우선으로 두고 인스턴스를 다소
  넉넉하게(크게) 잡았다. 덕분에 10,000 동시 연결을 무리 없이 견뎠지만, 이 선택이 *효율적*이었는지는
  전혀 확신할 수 없다. 더 작은 인스턴스로도 충분했을 수도, 혹은 오토스케일링되는 여러 대로 분산하는
  편이 비용·안정성 모두 나았을 수도 있다. **운영 환경에 맞는 서버 사양을 어떤 기준으로 선택하고,
  인프라를 어떻게 구축해야 하는지**가 궁금하고 다른 사람들의 의견이 궁금하다.
