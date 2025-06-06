# 6장: 레디스를 메시지 브로커로 사용하기

---

## 메시지 브로커란?
- 메시지 브로커는 송신자(Publisher)와 수신자(Subscriber) 사이에서 메시지를 중개하고 전달하는 시스템입니다
- 쉽게 말해, 서로 직접 통신하지 않아도 중간에서 데이터를 안전하게 전달하고 처리 흐름을 비동기로 분리해주는 역할을 합니다

### 메시지 브로커의 필요성
- 마이크로서비스 환경에서 서비스 간 비동기 통신을 가능하게 함
  - 메시지 브로커를 활용하면 요청-응답을 기다리지 않고 다음 작업 수행 가능 → 시스템 성능 향상
- 장애로 인해 즉시 처리 불가능한 요청을 저장해두었다가 후처리 가능
  - Stream 사용 시 메시지 누락 방지 및 재처리 가능
- 로깅, 알림, 이메일 발송 등의 작업에서 동기식 트랜잭션 부하를 줄일 수 있음
- Redis는 인메모리 기반이라 빠르고 간단한 메시징 시스템을 구현할 수 있음

### Redis에서의 메시지 브로커 기능
| 방식 | 특징 | 주요 명령어 |
| --- | --- | --- |
| **Pub/Sub** | - 실시간 메시지 전송<br/>- 구독자가 없으면 메시지 유실 | `PUBLISH`, `SUBSCRIBE`, `UNSUBSCRIBE` |
| **List** 기반 큐 | - 단순한 FIFO 큐<br/>- 블로킹 대기 가능<br/>- 소비 후 데이터 제거 | `LPUSH`, `RPUSH`, `LPOP`, `BRPOP` |
| **Stream** | - Kafka 유사 구조<br/>- 메시지 ID 기반<br/>- 소비자 그룹 관리<br/>- 메시지 지속성 | `XADD`, `XREAD`, `XGROUP`, `XREADGROUP` |

### 메시징 큐 vs 이벤트 스트림
| 항목 | 메시징 큐 (List 기반) | 이벤트 스트림 (Stream 기반) |
| --- | --- | --- |
| 소비자 수 | 1개 (1:1) | 여러 개 (N:N) |
| 메시지 삭제 | 소비 후 즉시 삭제 | 읽고 나서도 일정 기간 보존 |
| 순서 보장 | 가능 | 소비자 그룹 내 순서 보장 |
| 저장 구조 | List 자료구조 | Stream 자료구조 |
| 실무 예시 | 간단한 대기열, 작업큐 | 채팅, 로그, 주문 이벤트 처리 |

## 레디스의 pub/sub
- 발행(Publish)과 구독(Subscribe) 구조를 따르는 메시지 브로커 기능
- 가벼운 pub/sub 기능을 제공함
- 생산자가 PUBLISH 로 채널에 메시지를 보냄
- 구독자는 SUBSCRIBE 로 채널 메시지를 실시간 수신
- 주의 사항 : 
  - 메시지 휘발성: 구독하지 않은 상태면 메시지를 받지 못함
  - 로그 저장이나 보류 재처리 불가
- 실시간 채팅, 이벤트 알림 등 즉시성이 중요하지만 손실돼도 되는 데이터에 적합
- 명령어
```
# 일반 채널 구독
SUBSCRIBE news updates

# 패턴 구독 (예: news.* 형태의 채널)
PSUBSCRIBE news.*

# ping 테스트
PING

# 구독 해제
UNSUBSCRIBE news
PUNSUBSCRIBE news.*

# 구독 상태 초기화
RESET

# 종료
QUIT
```

### 클러스터 구조에서의 pub/sub
- PUBLISH 시 모든 노드로 메시지가 전파됨
- 채널은 모든 노드에 존재하는 논리적 개념
- Redis Cluster에서 PUBLISH는 내부적으로 브로드캐스트
- 단점: 모든 노드에 전파되므로 트래픽 부담 증가

### Sharded Pub/Sub
- 클러스터 구조에서의 pub/sub에서 비효율을 해결하기 위해 도입
- 기존 Pub/Sub의 브로드캐스트 한계를 해결하기 위함
- 채널을 해시 슬롯에 매핑하여, 특정 슬롯 담당 노드에서만 메시지를 처리하게 만든 구조
- ❓해시 슬롯이란?
  - Redis Cluster에서 **키의 저장 위치를 정하기 위해 사용하는 범위값(0~16383)**입니다.
  - 키는 CRC16(key) % 16384 계산으로 슬롯 번호를 결정하고, 해당 슬롯을 가진 노드에 저장됩니다.
  - 채널이 특정 노드의 슬롯에 매핑됨
  - 해당 노드에서만 구독자에게 메시지 전송
- 명령어
```
# 셰어드 채널 구독
SSUBSCRIBE shard:alert shard:event

# 셰어드 채널에 메시지 발행
SPUBLISH shard:alert "중요 알림입니다!"
SPUBLISH shard:event "이벤트 발생!"

# SSUBSCRIBE/SPUBLISH는 반드시 Redis 클러스터 모드에서만 동작합니다.

```
## 레디스의 list를 메시징 큐로 사용하기
- Redis의 List 자료구조는 FIFO 큐처럼 사용할 수 있어 간단한 메시지 큐 시스템을 구현할 때 매우 유용합니다.
```
# 왼쪽(앞)에 작업 추가
LPUSH job_queue "task1"
LPUSH job_queue "task2"

# 오른쪽(뒤)에 작업 추가
RPUSH job_queue "task3"
RPUSH job_queue "task4"

# 왼쪽(앞)부터 제거
LPOP job_queue
# → "task2"

# 오른쪽(뒤)부터 제거
RPOP job_queue
# → "task4"

# job_queue가 있을 때만 뒤에 삽입
RPUSHX job_queue "task5"

# job_queue의 마지막 요소를 꺼내서 같은 큐 앞에 넣기
RPOPLPUSH job_queue job_queue

# 리스트의 마지막 값 Pop, 없으면 대기
BRPOP list timeout

# 리스트의 앞쪽 값 Pop, 없으면 대기
BLPOP list timeout

# timeout이 0이면 무한 대기
# 큐가 비어 있다가 값이 들어오면 즉시 pop
# 이벤트 기반 구조에 적합!
BRPOP job_queue 0

```

### 트위터 타임라인 캐시 구조
- 유저의 팔로우 리스트를 기준으로 새 트윗이 올라오면, 각 팔로워의 타임라인 리스트에 LPUSH로 트윗 ID 삽입
- 일정 개수 초과 시 LTRIM으로 오래된 트윗 제거
```
LPUSH timeline:user123 tweet789
LTRIM timeline:user123 0 99   # 최근 100개만 유지
```

## Redis Stream 이란?
- Append-only log 구조: 메시지를 시간 순서대로 저장하는 비휘발성 큐
- 메시지는 ID 기반으로 관리 (1685968917623-0 등)
- 소비자 그룹 기반의 분산 처리
- 메시지는 삭제되지 않고 명시적 ack 또는 TTL 삭제 필요
- Kafka와 유사하지만 인메모리 기반으로 훨씬 가볍고 빠름
- 고가용 이벤트 처리, 비동기 데이터 처리, 작업 큐, 이벤트 저장소에 활용

### Redis Stream의 활용 예시
- 대용량 이벤트 로그 처리 (Kafka 대체용)
- 채팅 서버에서 메시지 JSON 처리
- 상품 주문, 결제, 배송 프로세스 분리 처리
- 예매 서비스에서 메시지 기반 처리

###  메시지 저장 구조와 식별 방식
| 구분 | Redis Stream | Kafka |
| --- | --- | --- |
| 메시지 ID | 시간 기반 ID (e.g. `1681219200000-0`) | 오프셋 기반 (0부터 시작) |
| 메시지 구조 | Key: Field-Value 해시 | Byte Array |
| 식별자 의미 | 시간 + 순번 기반 (자연 정렬 가능) | 파티션 내의 정수형 오프셋 |
| 순서 보장 | 단일 Stream 내 보장 | 파티션 내 보장 (복수 파티션은 불가) |

### 메시지 추가(생산)
```
# 자동 ID 생성
XADD Email * sender "jaebin" title "hello"

# 명시적 ID
XADD Email 1681219200000-1 sender "minji" title "hi"
```
- 메시지는 내부적으로 Hash 형태로 저장
- Redis가 자동으로 ID 생성(명시적도 가능)

### 메시지 소비 (조회)
```
# BLOCK 0: 메시지가 없을 경우 무한 대기
# 실시간 구독
XREAD BLOCK 0 STREAMS emailStream 0

# $: 지금 이후부터 메시지만 읽음
XREAD BLOCK 0 STREAMS Email $

# XRANGE - +: 범위 조회 (모든 메시지)
XRANGE Email - +

# 최신 → 오래된 순으로 조회
XREVRANGE Email + -
```

### Redis에서의 Fan-out
- 같은 데이터를 여러 소비자에게 전달하는것을 팬아웃이라고 한다.
- 하나의 메시지를 여러 처리 계층 또는 서비스가 동시에 받도록 복제하는 구조
- 같은 Stream에서 여러 소비자 그룹을 만들어 각각 동시에 읽도록 구성할 때 발생합니다.
```
 Email Stream
 ├── Group: Log 저장 용
 ├── Group: 알림 발송 용
 └── Group: 통계 수집 용
```
- 동일한 메시지를 **서로 다른 용도(로깅, 알림, 통계 등)**로 병렬 처리

### Stream에서 메시지의 순서가 보장돼야 하는 경우와 그렇지 않은 경우
- 순서가 보장돼야 하는 경우
  - 이전 메시지가 처리되기 전에는 다음 메시지를 처리하면 안 되는 경우
    - 결제 처리 순서 : 같은 사용자 결제 요청은 순차로 처리되어야 중복 결제/충돌 방지
    - 티켓 예매 시스템 : 좌석 배정/예매는 먼저 요청한 사용자에게 우선권 부여
    - 재고 감소 처리 : 상품 재고 1개 남은 상태에서 동시 주문 시 먼저 온 요청이 우선
    - 사용자 계좌 이체 : 잔고 차감 후 입금 처리가 순차로 보장되어야 함
- 순서를 보장하지 않아도 되는 경우
  - 병렬 처리 속도와 처리량이 중요하고, 순서 자체는 의미가 없는 경우
    - 로그 수집/분석 : 개별 로그는 순서 무관, 대량 수집에 초점
    - 이메일 전송 대기 큐 : 어떤 순서든 전송되기만 하면 됨
    - 푸시 알림 전송 : 같은 유형의 알림은 순서보단 속도가 중요함
    - 통계 수집용 데이터 처리 : 각 데이터는 독립적이므로 순서 필요 없음
- 레디스 Stream에서는 데이터가 저장될 때마다 고유한 ID를 부여받아 순서대로 저장된다. 따라서 소비자에게 데이터가 전달될 때, 그 순서는 항상 보장된다.

### 메시지가 전송될 때 순서를 보장할 수 없는 카프카
- Kafka는 토픽(Topic) → 파티션(Partition) 구조로 메시지를 저장하고 소비합니다.
- 여기서 핵심은 파티션 내부에서는 메시지 순서가 보장되지만,
- 토픽 전체(복수 파티션)에서는 메시지 순서가 보장되지 않는다는 점입니다.

#### Kafka에서 순서를 보장하려면?
- 동일 키는 동일 파티션으로 묶어야 함
- 하나의 파티션 = 하나의 소비자 구조로 제한해야 함
- 여러 파티션에 동일 키가 분산되지 않도록 설계 주의

### Redis Stream에서의 소비자 그룹 구조
- Redis Stream에서는 **하나의 Stream(=토픽 개념)**에 대해 여러 개의 소비자 그룹을 만들 수 있고, 각 그룹에는 **여러 개의 소비자(consumer)**가 존재합니다.
```
Email Stream
   ├── Consumer Group A
   │    ├── Consumer A1
   │    └── Consumer A2
   └── Consumer Group B
        ├── Consumer B1
        └── Consumer B2
```
- Stream은 하나지만 여러 소비자 그룹이 독립적으로 메시지를 처리할 수 있습니다.
- 각 그룹 안에서 미처리 메시지들은 내부적으로 큐잉되며, 소비자 간에 자동 분배됩니다

#### 소비자 그룹 내부 동작 방식
- 메시지가 Stream → 그룹 → 소비자 순서로 분배됩니다.
- 같은 그룹 내의 소비자들끼리는 메시지를 중복해서 받지 않습니다.
- 즉, 같은 그룹 = 메시지를 나눠서 처리 (Load Balancing)
- 반면, 다른 그룹 = 같은 메시지를 각각 처리 가능 (Fan-out)

### 소비자 그룹 생성 및 사용
```
XGROUP CREATE emailStream emailGroup $
XREADGROUP GROUP emailGroup consumer1 STREAMS emailStream >
XACK emailStream emailGroup 1685968917623-0
```
- `>`는 소비자가 아직 받지 않은 메시지를 의미
- `XACK`는 메시지 처리가 끝났음을 알려줌(메시지 처리 완료)
- consumer1은 이 소비자를 고유하게 식별

### 메시지 재처리 및 자동 재할당
```
XPENDING emailStream emailGroup     # 처리되지 않은 메시지 확인
XCLAIM emailStream emailGroup otherConsumer 0 1685968917623-0
XAUTOCLAIM emailStream emailGroup myconsumer 60000 0 COUNT 10
```
- `XPENDING`: 보류 중인 메시지 리스트
- `XCLAIM`: 수동 재할당
- `XAUTOCLAIM`: 자동 재할당 (Redis 6.2+)

### Dead Letter Queue (실패 메시지 처리)
```
XADD deadLetterStream * reason "consumer failed 3 times"
```
- N회 이상 처리 실패한 메시지는 별도 스트림으로 보냄
- XPENDING에서 delivery count를 확인하고, 기준 초과 시 Dead Letter로 넘기는 로직 작성 필요


### At most once, At-least-once, Exactly-once(최소 한번, 최대 한번, 정확히 한번)

| 보장 수준 | 설명                                                               |
| --- |------------------------------------------------------------------|
| At most once | 처리 실패 시 유실 가능성 존재 (ACK 없이 읽기)                                    |
| At least once | 기본값. ACK 전 장애 발생 시 중복 처리 가능                                      |
| Exactly once | Redis 자체는 미지원, **앱 로직에서 보완** 필요, 중복 메시지 검사 로직 (idempotent logic) |


### Kafka와 Stream 차이
| 항목 | Redis Stream | Kafka |
| --- | --- | --- |
| 메시지 ID | timestamp 기반 (string ID) | 오프셋 기반 (long) |
| 메시지 보존 | 수동 삭제 (XDEL), TTL 설정 | 로그 보존 주기 설정 |
| 소비자 그룹 | 있음 | 있음 |
| 순서 보장 | 소비자 그룹 내에서 보장 | 파티션 내에서만 보장 |
| 재처리 방식 | XPENDING, XCLAIM | offset 관리 |
| 실시간 처리 | XREAD BLOCK | consumer.poll() |
| 설치 및 구성 | 단일 인스턴스로도 가능 | 브로커, zookeeper 필요 |

#### Kafka와의 순서 보장 차이점

| 항목 | Kafka | Redis Stream |
| --- | --- | --- |
| 순서 보장 | 파티션 단위로만 보장 | 단일 Stream 내에서 완전한 순서 보장 |
| 파티션 간 순서 | ❌ 보장 안 됨 | ✅ Stream 전체 ID 기반 순차 보장 |
| 그룹 처리 | 소비자 ↔ 파티션 1:1 매핑 | Stream 내에서 각 소비자에 메시지 자동 분배 |
| 오프셋 추적 | 내부 토픽 `__consumer_offsets` | 내부 `last_delivered_id`, `XPENDING`, `XACK` 추적 |


#### ❓ 그렇다면 카프카에서는 다중 서버 컨슈머와 다중 파티션일 경우 순서를 보장하는 방법은?
- 예시 : 광고 과금 처리
- 메시지 단위 파티셔닝 전략 개선
  - "광고 주 ID, 광고 ID 단위로 파티셔닝" → 광고별로 순서 보장됨
- Kafka + Redis List (or Stream)로 처리 큐 보완
  - Kafka → Redis Queue → 과금 처리 API 호출
  - Redis List: 광고 주 ID, 광고 ID 별 리스트 큐
  - 순서를 Redis에서 직렬로 제어
- Redis Stream 사용
  - 정확한 순서 보장
  - 동일 키(예: 광고 ID)별 정렬
  - 메시지 유실 방지 및 재처리 필요
  - 단순한 메시징 구조로 처리하고 싶다면 Stream이 적합
  - 차이점
  
| 기준 | Kafka | Redis Stream |
| --- | --- | --- |
| **글로벌 순서 보장** | ❌ 파티션 간 순서 보장 불가 | ✅ 단일 Stream에 시간 기반 ID 정렬 |
| **단일 키 메시지 순서 보장** | 파티션 키 필요, 병렬 시 꼬일 수 있음 | Stream 자체가 ID 정렬되어 순서 보장 |
| **실시간 처리** | 우수하나 소비자 지연 시 오프셋 복잡 | `XREAD BLOCK`, `XGROUP`으로 간단하게 실시간 대기 |
| **중복 메시지 처리 제어** | 오프셋 관리 필요 | `XACK`, `XPENDING`, `XAUTOCLAIM` 등으로 내장 관리 |
| **구현 복잡도** | 토픽, 파티션, 오프셋 복잡 | 명확한 단일 Stream 중심 처리 |
| **재처리/장애 처리** | 별도 오프셋 시스템 필요 | `XPENDING` 기반으로 자동화 가능 |

### 상태 확인 명령어
```
XINFO STREAM Email           # Stream 정보
XINFO GROUPS Email           # 소비자 그룹 정보
XINFO CONSUMERS Email email_group   # 그룹 내 소비자 상태
```
