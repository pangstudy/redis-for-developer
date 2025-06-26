# 레디스 데이터 백업 방법

## 복제 vs 백업

* **복제 (Replication)**: 고가용성(HA)을 위해 사용. Master-Replica 구조로 구성되며, 삭제 커맨드 등도 Replica에 그대로 반영됨 → 복제본만으로는 데이터 보존 불가.

* **백업 (Backup)**: 장애 상황에서 데이터 복구를 위한 수단. Redis는 RDB와 AOF 두 가지 방식을 모두 지원하며, 하나의 인스턴스에서 병행 사용 가능.

## RDB vs AOF

| 항목    | RDB (Redis Database) | AOF (Append Only File) |
| ----- | -------------------- | ---------------------- |
| 방식    | Snapshot             | Append Log             |
| 복원 속도 | 빠름                   | 느림 (전체 replay 필요)      |
| 파일 크기 | 작음                   | 큼 (로그 전체 저장)           |
| 복구 시점 | 저장 시점                | 마지막 커맨드까지 복구 가능        |
| 성능 영향 | 적음                   | 더 큼 (쓰기 성능 저하)         |
| 특징    | 장애 시점 직전까지 복원 어려움    | 장애 직전까지 복원 가능          |

## RDB 방식의 데이터 백업

### 자동 저장 설정

```conf
save <초> <변경된 키 수>
save 60 10000   # 60초 동안 10000개 이상 변경 시 저장
```

### 수동 저장 명령

* `SAVE`: 동기 방식, 전체 클라이언트 명령 차단 발생 → 사용 권장 X
* `BGSAVE`: 백그라운드 저장 (fork로 자식 프로세스 생성)
* `SCHEDULE`: 기존 백업 종료 후 예약 백업 수행 가능
* `LASTSAVE`: 마지막 저장 시점 확인 가능

### 복제 시 자동 RDB 생성

* Replica가 처음 Master에 붙을 때 자동 RDB 파일 다운로드

## AOF 방식의 데이터 백업

### 설정

```conf
appendonly yes
```

* 메모리 변경 커맨드만 저장됨 (`DEL non_existing_key`와 같은 무효 커맨드는 기록되지 않음)

### appendfsync 설정

* `everysec`: 1초마다 디스크 기록 (기본값)
* `no`: OS에게 맡김 → 최대 30초 유실 가능
* `always`: 매 커맨드마다 디스크 기록 → 성능 저하

### 자동/수동 AOF 리라이트

* 자동: `auto-aof-rewrite-percentage`, `auto-aof-rewrite-min-size`
* 수동: `BGREWRITEAOF` 명령

### AOF 재구성

#### Redis 7 이전

* 하나의 파일로 관리됨 (RDB + Append 커맨드)
* fork → 자식 프로세스가 임시 파일 생성 → 버퍼 기록 → 덮어쓰기

#### Redis 7 이후

* RDB 파일, AOF 파일, Manifest 파일로 구성
* AOF는 증분 로그, RDB는 전체 스냅샷, Manifest로 현재 바라보는 파일 정의

### AOF-timestamp

* `aof-timestamp-enabled` 옵션으로 시점 타임스탬프 기록 가능 → 시점 복구에 유리

## 백업 시 메모리 주의사항 (OOM)

* Redis는 `Copy-on-write (COW)` 방식 사용
* BGSAVE 또는 BGREWRITEAOF 시 메모리 순간적으로 2배 이상 증가 가능
* maxmemory는 서버 RAM의 33%~66% 설정 권장


## 고려 포인트

* RDB + AOF 병행 시 Redis는 복구 시 AOF 우선 사용 (`aof-use-rdb-preamble yes`)
* Redis 실행 중에는 수동 restore 불가능 → 재시작 시 파일 로드
* 자동화: cron + cp / Redis Operator / 클라우드 Snapshot 서비스 등 활용
* AOF 리라이트와 BGSAVE는 트래픽 적은 시간에 스케줄링
