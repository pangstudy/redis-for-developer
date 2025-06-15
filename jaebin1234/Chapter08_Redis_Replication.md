# 8장: 레디스 복제

---

## 레디스에서의 복제 구조

### Redis에서 복제란?
- Master → Replica로 데이터를 실시간 복사
- Redis는 기본적으로 비동기 복제
- 장애 대비 + 읽기 부하 분산 + 백업 효율화 가능

### 가용성이란?
- 일정 기간 동안 서비스를 정상적으로 사용할 수 있는 시간의 비율
- 이용할수있는 사용 시간 / 총 시간

### 가용성과의 관계
- 마스터 노드 장애 시 서비스 중단 방지
- Redis Sentinel이나 클러스터와 함께 사용 시 자동 failover
- Redis에서 복제는 단순한 백업이 아닙니다. 읽기 전용 Replica 활용 + 백업 전용 Replica로 쓰면 성능과 안정성 모두 확보 가능

### Redis 복제 구조
```
# Replica 노드에서 실행
REPLICAOF <master-ip> <master-port>
```
- Redis 5 이전에는 `SLAVEOF`
- Sentinel을 쓸 경우 자동화됨
- 복제 구성 이유

| 목적 | 설명 |
| --- | --- |
| 고가용성 | Master 장애 시 자동 전환 |
| 부하 분산 | Master는 쓰기 집중, Replica는 읽기 분산 |
| 백업 | RDB, AOF 백업 작업은 Replica에서 수행하여 성능 보호 |

### 복제 매커니즘

#### 디스크 기반 복제(RDB 파일 사용)
1. `REPLICAOF` 명령으로 연결
2. Master가 `BGSAVE`로 RDB 스냅샷 생성 (fork 사용)
3. 2번 과정의 해당 시점의 데이터 변경은 **복제 버퍼**에 레디스 프로토콜 RESP로 저장
4. RDB 생성 완료 → 파일은 복제본 노드로 복사
5. Replica(복제 노드)는 로컬 메모리 초기화 후 RDB 파일 로딩
6. 복제 버퍼 데이터 순차 실행 (동기화 마무리)

- 이 과정은 네트워크보다 디스크 I/O 영향을 많이 받는다 → 디스크가 느리면 전체 복제 느림

#### 디스크 없이 복제 (socket 직접 전송)
- repl-diskless-sync yes 설정 시 사용
1. `REPLICAOF` 명령으로 연결
2. Master가 RDB 생성됨과 동시에 점진적으로 직접 복제본 소켓으로 전송
3. 2번 과정의 해당 시점의 데이터 변경은 **복제 버퍼**에 레디스 프로토콜 RESP로 저장
4. 복제본은 소켓에서 읽어온 RDB 파일을 복제본의 디스크에 저장
5. 복제본의 저장된 모든 데이터를 모두 삭제한 뒤 RDB 파일 내용을 메모리에 로딩
6. 버퍼 데이터도 실시간으로 Socket으로 전송하여 수행

### 복제 세부 매커니즘

#### 비동기 복제
- Master → Replica로 단방향
- Replica는 쓰기 불가 (`read-only yes`)
- Master → 여러 Replica 연결 가능

### 복제 ID와 오프셋
- Redis는 복제 상태를 replication ID + offset으로 추적
```
INFO REPLICATION
```
```
master_replid: 347a189...
master_repl_offset: 892334
```

### 부분 재동기화
- 복제 연결이 끊길 때마다 마스터에서 RDB 파일을 새로 내려 복제본에 전달하는 과정은 성능상 문제가 생길 수 있음.
- PSYNC 커맨드를 호출해 replication id와 오프셋을 마스터에게 전달

#### 백로그(buffer) 기반 동기화
- 복제본과의 연결이 끊겼을 때 → 전체 복제(비용 큼) 대신 백로그로 재동기화 시도
- 백로그는 circular buffer 형태 (기본 1MB)
```
repl-backlog-size 5mb
```
- 주의 : 백로그에 데이터가 없거나 ID 불일치 시 → 전체 복제 다시 수행
- 안정적인 부분 재동기화를 위해 백로그 크기 늘려두기 추천

#### Secondary ID (복제 ID 전환)
- 예: A → B → C 구성 중 A 장애 발생
- B가 새로운 Master가 되면, 새로운 replication ID 생성
- 기존 연결 노드(C)는 새로운 ID에 따라 부분 재복제 또는 전체 복제 결정

#### Replica 읽기 모드
```
replica-read-only yes   # default
```
- 복제본은 기본적으로 읽기 전용
- 복제본에서 쓰기 수행 가능은 하나, 다른 복제본에는 전파되지 않음
- 주의 : 복제본에서 임의로 SET 같은 명령 수행 시 복제 일관성 깨짐
