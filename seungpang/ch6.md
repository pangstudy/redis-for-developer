# Redis를 메시지 브로커로 사용하기

## 메시징 큐 vs 이벤트 스트림

| 항목 | 메시징 큐 (Message Queue) | 이벤트 스트림 (Event Stream) |
|------|----------------------------|-------------------------------|
| 통신 방식 | Producer → Consumer (1:1) | Publisher → Subscriber (n:n) |
| 방향성 | 생산자가 각 큐에 직접 push | 구독자는 스트림에서 pull |
| 영속성 | 소비자가 읽으면 삭제됨 | 구독자가 읽어도 데이터 유지됨 |
| 용도 | 동기 처리에 적합 (명령 전달) | 비동기 처리 및 이벤트 로그에 적합 |
| 예시 | RabbitMQ, Redis List | Kafka, Redis Stream |

> **정리**: 명령 전달과 같은 단방향(1:1) 구조는 메시징 큐에, 동일 이벤트를 여러 서비스가 처리해야 하는 구조(n:n)는 이벤트 스트림이 유리하다.

---

## Redis Pub/Sub

- **장점**: 구현이 매우 단순하고 빠름.
- **단점**:
    - 메시지 수신 여부 확인 불가
    - 누가 구독하고 있는지, 메시지를 받았는지 모름
    - 정합성이 중요한 시스템에는 부적합

### 주요 명령어
```bash
PUBLISH hello world
SUBSCRIBE hello
PSUBSCRIBE mail-*
```

### 클러스터 환경
- 일반 pub/sub은 모든 노드에 메시지를 전파하여 비효율적
- Redis 7.0부터 `Sharded Pub/Sub` 도입 → 노드별 샤딩된 pub/sub 지원 (`SPUBLISH` 사용)

---

## Redis List를 이용한 메시징 큐

- **적합한 상황**: 단순한 작업 큐 처리, 정합성이 상대적으로 덜 중요한 데이터 처리
- **특징**:
    - `LPUSH` / `RPUSH`로 데이터 넣고, `LPOP` / `RPOP`으로 꺼냄
    - `BLPOP` / `BRPOP`: 블로킹 팝 지원 (데이터가 올 때까지 기다림)
- **장점**:
    - Redis 내부에서 처리되므로 빠르고, 중간 로직 줄일 수 있음
    - 폴링 기반이 아니라 블로킹 기반 처리 가능

---

## Redis Stream

> Kafka와 유사한 구조의 이벤트 로그 시스템  
> 메시지를 append-only 형태로 저장하며 소비자 그룹, offset 관리 등 제공

### 데이터 입력

```bash
XADD Email * subject "hi" body "world"
```

### 데이터 조회

```bash
XREAD BLOCK 5000 STREAMS Email 0
```

---

## 소비자와 소비자 그룹

### 팬아웃 구조
- **Kafka**: 여러 소비자가 같은 토픽의 서로 다른 파티션을 읽음
- **Redis Stream**: 하나의 stream에 여러 소비자 그룹을 생성하여 fan-out 가능

### 메시지 순서 보장
- **Kafka**: 파티션 내 순서만 보장
- **Redis**: ID 기반으로 전체 순서 보장

### 소비자 그룹 예시
```bash
XGROUP CREATE Email EmailGroup $
XREADGROUP GROUP EmailGroup Consumer1 COUNT 1 STREAMS Email >
```

---
### 보류 리스트 관리

```bash
XCLAIM Email EmailGroup Consumer2 3600000 168421932132-0
```

---

## Redis vs Kafka

| 항목 | Redis Stream | Kafka |
|------|--------------|-------|
| 사용 편의성 | 단순, 별도 브로커 필요 없음 | 설정 복잡, 별도 브로커 필요 |
| 내구성 | 메모리 기반, 디스크도 가능 | 디스크 기반, 고가용성 가능 |
| 순서 보장 | ID 기반 전체 순서 보장 | 파티션 단위 순서 보장 |
| 재처리 | ACK 및 보류 리스트 관리 | offset 기반 재처리 가능 |
| 사용 사례 | 간단한 이벤트 큐, 백엔드 이벤트 | 대규모 로그 처리, 데이터 파이프라인 |



Q. 메시지에 문제가 있어 처리되지 못할 경우 큐서비스와 redis는 어떤 방식으로 처리할까?

| 항목             | Kafka                          | RabbitMQ                      | Amazon SQS                   | **Redis Streams**                        |
| -------------- | ------------------------------ | ----------------------------- | ---------------------------- | ---------------------------------------- |
| **DLQ 기본 지원**  | ❌ (수동 구현)                      | ✅ 기본 지원                       | ✅ 기본 지원                      | ❌ (수동 구현)                                |
| **DLQ 생성 방식**  | 수동 토픽 생성 (e.g. `.DLQ`)         | DLX(Dead Letter Exchange) 바인딩 | DLQ 큐 설정 (`maxReceiveCount`) | 별도 Stream 구성 (e.g. `mystream.dlq`)       |
| **재시도 횟수 설정**  | 컨슈머 로직에서 구현 (보통 N번까지 처리 후 DLQ) | `x-death` 헤더 기반 재시도 추적        | `maxReceiveCount` 설정         | `XPENDING` + `delivery-count` 확인 후 수동 전송 |
| **DLQ 저장소 형태** | 일반 Kafka 토픽                    | RabbitMQ 큐                    | SQS 큐                        | Redis Stream                             |
| **DLQ 자동 전송**  | ❌ (직접 구현 필요)                   | ✅                             | ✅                            | ❌ (직접 구현 필요)                             |
| **재처리 방식**     | DLQ 토픽에서 수동 consume 후 재전송      | DLQ 큐에서 consume 후 재전송         | DLQ 큐에서 consume 후 재전송        | DLQ Stream에서 다시 XREADGROUP 처리            |

### Kafka
+ Kafka는 DLQ를 기본적으로 제공하지 않음
+ 일반적으로 두 가지 방식으로 DLQ를 구현: 
  + Retry + DLQ Topic 패턴: 실패 시 N번까지 재시도 후 .DLQ 토픽으로 전송.
  + Kafka Streams / Kafka Connect의 경우, error.topic.name, errors.deadletterqueue.topic.name 등의 설정을 통해 DLQ 토픽 지정 가능.
+ Kafka는 컨슈머가 메시지의 offset을 수동 관리하므로 "DLQ 전송"은 애플리케이션 책임.

### RabbitMQ
+ 기본적으로 **Dead Letter Exchange(DLX)**를 통한 DLQ 지원
+ 메시지가 TTL 만료되거나 Nack 또는 Reject될 때 지정된 DLX로 라우팅
+ x-dead-letter-exchange, x-dead-letter-routing-key, x-message-ttl 등의 속성 설정 필요
+ x-death 헤더를 통해 얼마나 재시도 되었는지 추적 가능

### Amazon SQS 
+ DLQ 지원 내장
+ 일반 큐에 DLQ를 연결할 수 있고, maxReceiveCount를 초과하면 DLQ로 자동 전송.
+ DLQ도 SQS 큐이므로 SQS의 표준 처리 방식으로 재전송 가능.

---

## 🔚 정리

- **Pub/Sub**: 즉시 알림, 빠르지만 신뢰도 낮음
- **List (Queue)**: 단일 소비자 처리에 적합, 간단한 큐 구현 가능
- **Stream**: Kafka 유사 기능 제공, 소비자 그룹·ACK·재처리 등 지원

**Redis를 메시지 브로커로 활용할 때 가장 중요한 것은 ‘메시지의 정합성 보장 수준’과 ‘확장성 필요 여부’를 기준으로 적절한 자료구조(Pub/Sub, List, Stream)를 선택하는 것이다.**
