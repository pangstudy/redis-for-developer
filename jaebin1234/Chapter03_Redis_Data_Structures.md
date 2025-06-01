# 3장: 레디스 기본 개념

---

## 레디스(Redis)란?
- 데이터를 메모리에 저장하여 초고속으로 처리할 수 있는 오픈소스 인메모리 데이터 저장소
- key - value 형태의 데이터 저장소
- 캐시, 메시지 브로커, 실시간 분석 등에 사용

### 특징
- 빠른 응답(인메모리)
  - Redis는 모든 데이터를 메모리에 저장하기 때문에 디스크 접근이 필요한 RDB보다 훨씬 빠름.→ 일반적인 요청 처리 시간이 1ms 이하
  - 예시
    - 웹 애플리케이션에서 사용자 로그인 정보를 세션 저장소로 Redis에 저장.→ 로그인 여부 확인 시, DB를 거치지 않고 1ms 내로 응답 처리됨
- 단순하고 다양한 자료구조 제공
  - 단순한 문자열(String)뿐 아니라 List, Hash, Set, Sorted Set, Bitmap, HyperLogLog, Geo, Stream 등 풍부한 자료구조 제공
- 클러스터링 및 복제 기능
  - Redis는 여러 노드에 데이터를 분산 저장할 수 있는 클러스터(Cluster) 기능과 장애 발생 시 자동 복구 가능한 복제(Replication), Sentinel 기능을 제공
  - 실시간 트래픽이 많은 서비스에서 Redis Cluster를 구성하여 수평 확장 가능
  - 마스터 노드 장애 시, Redis Sentinel이 자동으로 슬레이브를 마스터로 승격 → 서비스 중단 없음
- 멀티 클라우드 지원
  - Redis는 AWS, Azure, GCP 등 대부분의 클라우드 플랫폼에서 매우 잘 통합되며,클라우드 환경에서의 운영, 배포, 자동화, 백업 기능에 최적화되어 있음
  - AWS ElastiCache for Redis를 통해 Redis를 설정 몇 번만에 자동 복제/백업/확장 가능한 환경으로 운영

### 데이터 저장 구조
- Redis는 데이터를 RAM에 저장합니다.
  - 디스크 I/O 없이 데이터에 직접 접근 가능하므로 매우 빠른 응답 속도를 제공
- Redis는 내부적으로 C 언어로 구현된 다양한 자료구조를 활용하여 메모리 효율을 극대화 합니다.
- 싱글 스레드 기반 이벤트 루프 모델
  -  Redis는 하나의 스레드로 모든 요청을 처리하지만, I/O 멀티플렉싱(epoll, select 등)을 사용하여 병렬처럼 동작합니다.

### ❓질문
1. 왜 인메모리 DB가 디스크형식의 DB 조회보다 빠른가요? 또, 얼마나 빠른가요?
   -  저장 위치 차이
     - 인메모리 DB(Redis)는 RAM (메모리)에 저장하며, 전자 회로 직접 접근하여 나노초(ns) 단위로 즉시 접근 가능합니다.
     - 디스크 기반 DB(PostgreSQL)는 디스크 (SSD/HDD)에 저장하며, 접근 시 디스크 I/O(블록 장치) 필요하고, 접근된 데이터를 ram에 올려서 버퍼 캐시를 통하여 처리합니다.

---
##  레디스의 자료구조

### String
- 최대 512MB 크기의 바이너리 안전 문자열 저장 가능
- 원자적 연산(INCR, DECR)을 통한 동시성 이슈 해결 가능
- 기본 명령어
```
SET key value            # 값 설정
GET key                  # 값 조회
INCR key                 # 값 1 증가 (정수)
INCRBY key amount        # 지정값만큼 증가
DECR key                 # 값 1 감소 (정수)
DECRBY key amount        # 지정값만큼 감소
MSET key1 val1 key2 val2 # 여러 키 값 동시 설정
MGET key1 key2           # 여러 키 값 동시 조회
```
- 주요 옵션
```
| NX   | 키가 존재하지 않을 때만 설정 |

| XX   | 키가 이미 존재할 때만 설정 |

| EX seconds | 만료 시간 초 단위 설정 |

| PX milliseconds | 만료 시간 밀리초 단위 설정 |
```
- 실제 활용 사례

```
# 전화번호 인증

SET verify:01025867126 123456 EX 180

GET verify:01025867126
```
```
# 로그인 시도 횟수 제한

# 로그인 실패 시 카운트 증가 (원자성 보장)
INCR login_fail:user:1234

# 횟수 조회
GET login_fail:user:1234

# 로그인 시도 횟수 제한 초기화
DEL login_fail:user:1234
```

```
# 분산 락 구현 (NX 옵션 활용)

# 락 설정 (key가 없을 때만 설정)
SET lock:resource1 "locked" NX EX 30

# 락 해제
DEL lock:resource1
```
- ❓주의
    - 대부분의 문자열 작업은 O(1)이므로 매우 효율적입니다. 그러나 O(n)이 될 수 있는 [**`SUBSTR`**](https://redis.io/commands/substr), [**`GETRANGE`**](https://redis.io/commands/getrange)및 명령에 주의하십시오. [**`SETRANGE`**](https://redis.io/commands/setrange)이러한 임의 액세스 문자열 명령은 큰 문자열을 처리할 때 성능 문제를 일으킬 수 있습니다.

### List
- 스택(LPUSH/LPOP), 큐(RPUSH/LPOP)로 활용
- 최근 데이터 보관 및 로그 기록 용이(LPUSH, LTRIM)
- 기본 명령어
```
LPUSH key value            # 왼쪽 삽입
RPUSH key value            # 오른쪽 삽입
LPOP key                   # 왼쪽에서 값 제거 및 반환
RPOP key                   # 오른쪽에서 값 제거 및 반환
LRANGE key start stop      # 지정 범위 조회 (0부터 시작, -1 마지막)
LTRIM key start stop       # 지정 범위만 남기고 나머지 삭제
LINDEX key index           # 특정 인덱스 값 조회
LINSERT key BEFORE|AFTER pivot value # pivot 앞뒤로 삽입
LSET key index value       # 특정 인덱스의 값 수정
```
- 사용 예시
```
LPUSH mylist "a" "b" "c"   # 결과: c b a
LRANGE mylist 0 -1         # 결과 조회: ["c", "b", "a"]
```
- 실제 활용 사례
```
# 최근 방문한 상품 목록 관리

# 최근 방문 상품 추가
LPUSH recent:products:user123 product789

# 최근 5개 상품만 유지
LTRIM recent:products:user123 0 4

# 최근 방문 상품 조회
LRANGE recent:products:user123 0 -1
```
```
# 로그데이터 실시간 관리

# 로그 추가 (애플리케이션 로그를 왼쪽에 추가)
LPUSH logs:payment-service "[2024-05-28 10:23:15] ERROR [PaymentController] - Failed to process payment for orderId=ORD12345, reason=Insufficient balance"

LPUSH logs:payment-service "[2024-05-28 10:23:16] INFO [PaymentService] - Retry payment request initiated for orderId=ORD12345"

LPUSH logs:payment-service "[2024-05-28 10:23:17] WARN [KafkaProducer]

# 최근 100개의 로그만 유지 (이전 로그는 삭제)
LTRIM logs:payment-service 0 99
```

- ❓주의
    - 헤드 또는 테일에 액세스하는 목록 작업은 O(1)이므로 매우 효율적입니다. 그러나 목록 내의 요소를 조작하는 명령은 일반적으로 O(n)입니다. 이러한 예에는 [**`LINDEX`**](https://redis.io/commands/lindex), [**`LINSERT`**](https://redis.io/commands/linsert)및 가 포함됩니다 [**`LSET`**](https://redis.io/commands/lset). 주로 큰 목록에서 작업할 때 이러한 명령을 실행할 때 주의하십시오.

### Hash
- 필드-값 쌍을 저장하는 구조로 관계형 DB의 레코드와 유사
- 메모리 효율적이며 빠른 접근 가능(HGETALL, HMGET)
- 기본 명령어
```
HSET key field value           # 필드-값 설정
HGET key field                 # 필드의 값 조회
HMSET key field1 val1 field2 val2 # 여러 필드-값 설정
HMGET key field1 field2        # 여러 필드의 값 조회
HGETALL key                    # 모든 필드-값 조회
HDEL key field                 # 필드 삭제
HEXISTS key field              # 필드 존재 여부 확인
```
- 사용 예시
```
HSET user:1 name "재빈" age 30
HGET user:1 name               # "재빈"
HGETALL user:1                 # 모든 데이터 조회
```
- 실제 활용 사례
```
# 사용자 프로필 정보 캐싱

# 프로필 정보 저장
HSET user:profile:1234 username "재빈" email "jaebin@example.com"

# 특정 필드 조회
HGET user:profile:1234 email

# 모든 프로필 정보 조회
HGETALL user:profile:1234
```
```
# 세션 데이터 관리

# 세션 데이터 저장
HSET session:token1234 userId 5678 isLoggedIn true

# 세션 데이터 조회
HGETALL session:token1234
```
### Set
- 중복 없는 데이터 저장
- 빠른 멤버십 체크 및 집합 연산(SUNION, SINTER)
- 기본 명령어
```
SADD key member [member...]    # 멤버 추가 (중복 불가)
SMEMBERS key                   # 모든 멤버 조회
SREM key member                # 멤버 삭제
SPOP key [count]               # 임의 멤버 추출 및 삭제
SCARD key                      # 멤버 개수 조회
SISMEMBER key member           # 멤버 존재 확인
SUNION key1 key2               # 합집합
SINTER key1 key2               # 교집합
SDIFF key1 key2                # 차집합
```
- 사용 예시
```
SADD tags "Redis" "Database" "Cache"
SMEMBERS tags                  # 전체 태그 조회
```
- 실제 활용 사례
```
# 좋아요 누른 사용자 목록 관리

# 좋아요 추가
SADD likes:post:456 user123

# 특정 게시글의 좋아요 유저 조회
SMEMBERS likes:post:456

# 좋아요 취소
SREM likes:post:456 user123
```
```
#태그 관

# 태그 추가
SADD tags:post:789 "redis" "database" "nosql"

# 공통 태그 찾기
SINTER tags:post:789 tags:post:456
```

- ❓주의
  - 추가, 제거 및 항목이 집합 구성원인지 여부를 확인하는 등 대부분의 집합 작업은 O(1)입니다. 이것은 그들이 매우 효율적이라는 것을 의미합니다. 그러나 구성원이 수십만 명 이상인 대규모 집합의 경우 명령을 실행할 때 주의해야 합니다 [**`SMEMBERS`**](https://redis.io/commands/smembers). 이 명령은 O(n)이며 전체 집합을 단일 응답으로 반환합니다. 대안으로 [**`SSCAN`**](https://redis.io/commands/sscan)집합의 모든 구성원을 반복적으로 검색할 수 있는 를 고려하십시오.

### Sorted Set
- 정렬된 데이터 저장 (스코어 기반)
- 빠른 랭킹 구현 가능(ZRANGE, ZADD)
- 기본 명령어
```
ZADD key score member [score member...] # 멤버와 점수 추가
ZRANGE key start stop [WITHSCORES]      # 오름차순 범위 조회
ZREVRANGE key start stop [WITHSCORES]   # 내림차순 범위 조회
ZRANGEBYSCORE key min max               # 점수로 범위 조회
ZINCRBY key increment member            # 멤버의 점수 증가
ZREM key member                         # 멤버 삭제
ZSCORE key member                       # 멤버 점수 조회
ZRANK key member                        # 멤버의 오름차순 랭킹 조회
ZREVRANK key member                     # 멤버의 내림차순 랭킹 조회
```
- 주요 옵션
```
| 옵션 | 설명 |

|------|-----|

| NX   | 멤버가 없을 때만 추가 |

| XX   | 멤버가 존재할 때만 업데이트 |

| LT   | 기존 점수보다 작을 때만 업데이트 |

| GT   | 기존 점수보다 클 때만 업데이트 |
```
- 사용 예시
```
ZADD ranking 100 userA 200 userB
ZREVRANGE ranking 0 -1 WITHSCORES
```
- 실제 활용 사례
```
# 게임 리더보드(랭킹 시스템)

# 점수 추가 및 업데이트
ZADD leaderboard 1500 user123

# 상위 5명 조회
ZREVRANGE leaderboard 0 4 WITHSCORES

# 사용자 점수 조회
ZSCORE leaderboard user123
```
```
# 실시간 인기 검색어

# 검색어 횟수 증가
ZINCRBY search:popular 1 "레디스 책 추천"

# 상위 인기 검색어 조회
ZREVRANGE search:popular 0 9 WITHSCORES
```
### 비트맵(Bitmap)
- 비트 수준 데이터 저장, 메모리 효율성 높음
- 집합 연산 및 간단한 상태 체크에 유리(BITCOUNT, SETBIT)
- 기본 명령어
```
SETBIT key offset value         # 특정 비트 설정(0 or 1)
GETBIT key offset               # 특정 비트 조회
BITCOUNT key [start end]        # 값이 1인 비트 개수 조회
BITFIELD key subcommand         # 다수 비트 설정 및 조회
```
- 사용 예시
```
SETBIT attendance 1000 1        # offset 1000에 출석체크
BITCOUNT attendance             # 총 출석 수 확인
```
- 실제 활용 사례
```
# 출석 체크 시스템

# 특정 사용자의 오늘 출석 체크 (비트 ON)
SETBIT attendance:2024-05-27 1234 1

# 사용자의 출석 여부 확인
GETBIT attendance:2024-05-27 1234

# 하루 총 출석자 수 확인
BITCOUNT attendance:2024-05-27
```

### HyperLogLog
- 카디널리티(중복을 제외한 고유 개수) 근사값 계산에 최적
- 메모리 효율적이고 빠른 속도(PFADD, PFCOUNT)
- 기본 명령어
```
PFADD key element [element...]   # 요소 추가
PFCOUNT key                      # 고유 요소 수 조회
PFMERGE destkey sourcekey [sourcekey...] # 여러 HyperLogLog 병합
```
- 사용 예시
```
PFADD visits user1 user2 user1
PFCOUNT visits                   # 결과는 2 (중복 제외)
```
- 실제 활용 사례
```
# 페이지 방문자 수 계산

# 방문자 추가 (중복 방문자 제거)
PFADD visitors:page:home user123 user456 user123 user789

# 총 고유 방문자 수 조회 (근사치)
PFCOUNT visitors:page:home
```
### Geospatial
- 위경도 데이터를 저장 및 공간 연산(GEOSEARCH, GEODIST)
- 기본 명령어
```
GEOADD key longitude latitude member [longitude latitude member...] # 위치 추가
GEOPOS key member                  # 멤버의 위치 조회
GEODIST key member1 member2 [unit] # 두 멤버 간 거리 계산
GEOSEARCH key FROMLONLAT lon lat BYRADIUS radius unit [WITHDIST] # 반경 내 멤버 조회
```
- 주요 옵션
```
| 옵션 | 설명 |

|------|-----|

| XX   | 존재하는 멤버만 업데이트 |

| NX   | 새로운 멤버만 추가 |

| unit | m, km, mi, ft (미터, 킬로미터, 마일, 피트 단위) |
```
- 사용 예시
```
GEOADD locations 127.0 37.5 "서울"
GEOSEARCH locations FROMLONLAT 127.0 37.5 BYRADIUS 10 km WITHDIST
```
- 실제 활용 사례
```
#주변 음식점 찾기

# 음식점 위치 추가
GEOADD restaurants 127.027 37.497 "식당A" 127.030 37.500 "식당B"

# 특정 위치에서 반경 1km 내 음식점 조회
GEOSEARCH restaurants FROMLONLAT 127.028 37.498 BYRADIUS 1 km WITHDIST

# 두 음식점 간 거리 측정
GEODIST restaurants "식당A" "식당B" km
```

---
### ❓번외 질문
1. 내부 API와 외부 API 의 응답속도의 차이

| 구분 | 내부 API (Internal API) | 외부 API (External API)       |
| --- | --- |-----------------------------|
| 호출 범위 | 동일 시스템 내 or 사설 네트워크 내부 | 인터넷 또는 외부 네트워크를 통한 호출       |
| 일반적인 응답 속도 | 수 ms 이내 (1~10ms) | 수십 ~ 수백(50ms ~ 500ms 이상)*   |
| 네트워크 | LAN (Intranet, VPC 내 통신) | WAN (인터넷, 공용망 경유)           |
| 보안 계층 | 사설 인증, 내부 인증 토큰 | HTTPS, OAuth2, 방화벽, VPN 등   |
| 안정성 | 고정된 환경에서 안정적 통신 | 네트워크 지연, 타임아웃, 장애 가능성 높음    |
| 예시 | A 서비스 → B 서비스 (같은 VPC 내) | 애플리케이션 → Kakao API, Naver Map 등 |