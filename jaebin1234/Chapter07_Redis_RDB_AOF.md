# 7장: 레디스 데이터 백업 방법

---
## 레디스에서 데이터를 영구 저장하기
- Redis는 메모리 기반 저장소
- 기본적으로 모든 데이터는 메모리에 저장됨 → 전원 OFF나 장애 발생 시 모든 데이터 소실 가능성

| 종류 | 목적 | 특징 |
| --- | --- | --- |
| 복제(replication) | **가용성 향상** | 마스터의 데이터를 실시간 복제 |
| 백업(backup) | **장애 복구** | 저장된 시점이나 작업 로그 기반 복원 |

## RDB
- 스냅샷 방식으로 메모리 상태를 .rdb 바이너리 파일로 저장
- 특정 시점의 데이터를 빠르게 복구 가능

- 자동 백업 조건
```
# redis.conf 예시
save 900 1    # 900초(15분) 동안 1개 이상의 변경이 있으면 백업
save 300 10
save 60 10000
```

- 비활성화
```
save ""                       # 백업 비활성화
CONFIG SET save ""            # 실행 중 변경
CONFIG REWRITE                # 재시작 시 적용
```

- 수동 백업 명령어
```
SAVE       # 현재 스레드가 블로킹되며 저장
BGSAVE     # 포크를 사용한 비동기 저장 (실무에서 사용)
LASTSAVE   # 마지막으로 저장된 시간 확인 (Unix timestamp)
```
- 주의 : SAVE는 서비스 중지 가능성 있으므로 운영 환경에서는 사용 자제

## AOF
- Redis에서 발생한 모든 쓰기 명령을 순서대로 기록
- AOF 로그를 재실행하면 DB 상태 복원 가능
- .aof 파일은 RDB보다 훨씬 큼
```
appendonly yes
appendfsync everysec   # 1초마다 디스크에 동기화
```
- 주요 명령어들은 다른 방식으로 변경 기록 됨
| 명령 | 기록 방식 |
| --- | --- |
| `INCRBYFLOAT` | `SET`으로 기록 |
| `BRPOP` | `RPOP`으로 기록 |

### AOF 파일을 재구성하는 방법
- AOF 재구성이 필요한 이유 : AOF는 계속 커지기 때문에 주기적으로 압축 필요

#### Redis 6까지 재구성 절차
1. `BGREWRITEAOF`: 포크로 자식 프로세스 생성
2. 자식 프로세스가 새로운 AOF(임시파일) 작성
3. 부모 프로세스는 변경 내용을 버퍼, 기존 AOF에 동시 저장
4. 자식이 끝나면 버퍼 내용을 임시 파일에 병합(추가) → 이후 기존 파일 덮어쓰기
- 주의 :  RDB + AOF 텍스트가 혼재된 파일로 복잡성 증가

#### Redis 7 이후 재구성 절차
- 매니페스트 기반 AOF
- RDB 파일 + AOF 로그를 번호 기반 파일로 나눠 관리
- appendonly.aof.n.base.rdb, appendonly.aof.n.incr.aof
- appendonly.aof.manifest에서 각 파일 ID와 순서를 관리

1. 포크 생성 → 메모리에 저장된 전체 데이터 임시 RDB파일(appendonly.aof.n.base.rdb) 생성(메모리 스냅샷 기반)
2. 새로운 변경 내용을 신규 AOF 파일에 저장(appendonly.aof.n.incr.aof)
3. 자식 프로세스가 파일 생성 후 → AOF 파일의 정보를 나타내는 manifest 파일 생성(새 매니페스트 생성)
4. 기존 manifest 파일과 교체 완료 후 이전 파일(기존 버전의 AOF 파일) 삭제

#### 자동 AOF 재구성
```
auto-aof-rewrite-percentage 100       # 기존 파일의 크기 2배 이상이면 트리거
auto-aof-rewrite-min-size 64mb        # 최소 트리거 용량
```
```
INFO persistence
# 확인
aof_base_size:67108864
```

#### AOF Timestamp 복원 (Redis 7 이후)
- `aof-timestamp-enabled yes`
- 실수로 `FLUSHALL` 했을 경우 특정 시점으로 되돌리기

```
redis-check-aof --truncate-to-timestamp 1717582348 appendonly.aof
```

#### AOF 수동 복구

```
redis-check-aof --fix appendonly.aof
```
- .aof는 로그이므로 손상되더라도 대부분 일부 손실만 발생 → 부분 복원 가능

#### AOF 디스크 반영 정책
| 설정값 | 설명 |
| --- | --- |
| `appendfsync no` | OS가 디스크에 쓸 때까지 기다림 → 가장 빠르지만 불안정 |
| `appendfsync everysec` | 1초마다 `fsync()` 호출 (기본값, 추천) |
| `appendfsync always` | 매 명령마다 `fsync()` → 안정적이지만 느림 |

#### 백업 관련 주의사항

- maxmemory
- 백업 중 `fork()` 사용 시 **전체 메모리 + 자식용 메모리** 필요
- 메모리가 부족하면 저장 실패

- Copy-On-Write(COW)
- 백그라운드 저장 시 포크된 프로세스는 기존 메모리 페이지를 공유
- 메모리 변경 시에만 복사본 생성
- 주의 : COW로 인해 SAVE/BGSAVE/BGREWRITEAOF는 메모리를 순간적으로 많이 사용함 → 운영 환경에서는 메모리 모니터링 필수!



