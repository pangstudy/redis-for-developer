# 레디스를 캐시로 사용하기

## 읽기 전략 - Look Aside (Cache Aside)

### 방식
- 애플리케이션이 데이터를 읽기 전에 Redis에서 먼저 확인 (cache hit/miss 판단)
- 없으면 DB에서 조회 후 Redis에 저장

### 유의사항
- Redis 장애 시 모든 요청이 DB로 몰려 **DB 부하 증가**
- 해결: Redis **Replication** 및 **Cache Warming** 사용

---

## 쓰기 전략

### Write Through
- DB에 쓰는 동시에 캐시에도 기록
- 항상 최신 데이터 보장
- 비효율: 불필요한 데이터도 캐시에 기록됨 → TTL 설정 필요

### Cache Invalidation
- DB에 쓰고 캐시에서는 삭제
- 캐시 재생성 시 비용 적음

### Write Behind (Write Back)
- 먼저 캐시에 저장하고, 이후 비동기로 DB 반영
- Redis 장애 시 **일부 데이터 유실 가능성**

---

## 캐시 만료 및 메모리 관리

### 만료 전략 (TTL)
- Redis는 TTL이 지나도 바로 삭제하지 않음 → 메모리 점유
- Passive 방식: 사용자 접근 시 만료 확인 후 삭제
- Active 방식: 랜덤 20개 키 중 25% 이상 만료 시 반복 삭제

### maxmemory-policy 설정

| 정책 | 설명 |
|------|------|
| `noeviction` | 삭제 없이 오류 발생 (비추천) |
| `allkeys-lru` | 전체 키 중 가장 오랫동안 사용되지 않은 것 삭제 (추천) |
| `allkeys-lfu` | 전체 키 중 사용 빈도 가장 낮은 것 삭제 |
| `random` | 랜덤 삭제 (권장하지 않음) |
| `volatile-ttl` | 만료시간이 가장 짧은 키 삭제 |

> Redis의 LRU/LFU는 근사치 기반 샘플링 방식으로 정확한 순서는 아님

---

## 캐시 스탬피드 현상 (Cache Stampede)

### 원인
- 인기 키가 만료될 때 여러 서버에서 동시에 DB 조회 → 중복 읽기/쓰기 발생

### 해결책
- TTL을 너무 짧게 잡지 않기
- **Cache Warming**으로 초기 데이터 채우기
- **PER 알고리즘**: TTL 만료 전에 확률적으로 갱신 시도

---

## Redis Cache 도입 적합 조건

- 원본 데이터 조회/계산 비용이 높은 경우
- 데이터 변경이 드물고 자주 조회되는 경우
- 캐시 저장소가 원본보다 읽기 속도가 빠른 경우

---

## 세션 스토어로서의 Redis

- Sticky session 이슈 없이 세션 공유 가능
- 서버 간 상태 동기화 문제 해결


## 참고
+ https://blog.bytebytego.com/p/how-uber-uses-integrated-redis-cache
