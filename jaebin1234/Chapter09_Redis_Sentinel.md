# 9장: 레디스 센티널

---

## 센티널이란?

### 고가용성 HA 이란?
- 고가용성(High Availability, HA) 이란 시스템이나 서비스가 중단 없이 지속적으로 운영될 수 있는 능력을 말합니다.
- 서비스 연속성 보장
- 신뢰성 및 고객 신뢰도
- 비용 절감 및 효율성 향상

### Redis 환경에서 고가용성(HA)의 필요성
- 데이터 유실 최소화
  - 장애 발생 시 복제본(replica)이나 백업을 통해 데이터를 즉시 복원하거나 손실을 최소화할 수 있습니다.
- 서비스 무중단 운영
  - 마스터 노드가 다운되더라도 빠른 페일오버(failover)를 통해 레플리카 노드가 자동으로 마스터 역할을 승계하도록 합니다.

### 단일 장애 지점(SPOF)이란?
- 단일 장애 지점(Single Point of Failure, SPOF) 이란 시스템 구성요소 중 하나가 장애가 발생했을 때, 전체 시스템 또는 서비스가 중단되는 상황을 의미합니다.
- 즉, 시스템이 한 요소의 장애로 인해 전체적으로 동작하지 못하는 약점을 가진 상태를 의미합니다.

### 센티널이란?
- Redis의 고가용성(HA, High Availability)을 보장하기 위한 모니터링 및 장애 처리 솔루션입니다.
- Sentinel은 Redis 인스턴스의 상태를 실시간으로 감시하며, 장애 발생 시 자동으로 복제본(replica)을 마스터(master)로 승격하여 서비스 중단을 최소화합니다.

### 주요 센티널 기능
- 모니터링 (Monitoring)
- 알림 (Notification)
- 자동 장애 처리 (Automatic Failover)
- 클라이언트 구성 제공

## 센티널 동작과 구성 
- 센티널은 Quorum 기반의 장애 인식 프로세스
- 장애 감지 및 동의 절차 (Leader Election)

### 센티널에서 Quorum을 사용하는 이유
- Quorum(쿼럼)은 분산 환경에서 장애를 확실하게 판단하기 위해 설정한 최소 합의 인스턴스 수입니다.
- 잘못된 판단 방지
  - 일시적인 네트워크 장애 또는 오탐지(false-positive)로 인한 잘못된 장애 판단을 최소화하기 위함입니다.
- 정확한 합의
  - 다수의 Sentinel 노드가 동의할 때만 실제 장애로 인정하여 신뢰할 수 있는 장애 처리를 수행합니다.

### 센티널 인스턴스 배치 방법
- 최소 3개 이상의 센티널 권장
- 별도 서버 혹은 컨테이너 분리 운영 권장 (내부 네트워크 장애 고려), 독립성 강화
- 센티널 인스턴스 실행하기 (예시 코드)
```
apt-get update

apt-get install -y redis

# sentinel.conf 경로 `/etc/redis/`

redis-server sentinel.conf --sentinel

redis-cli -p 26379 info sentinel
```

### 센티널 운영 및 관리

#### 센티널 기본설정
- 레디스 설정 확인 redis.conf
```
# redis.conf 예시 (마스터와 레플리카 설정 예시)
bind 0.0.0.0
port 6379

# protected-mode 확인
protected-mode no

# 마스터 인증 비밀번호 (권장)
requirepass your_master_password

# 레플리카가 마스터와 통신 시 사용하는 비밀번호 (마스터가 requirepass를 사용할 경우)
masterauth your_master_password

# 데이터 지속성(권장 옵션)
appendonly yes


```
- 센티널 설정 확인 sentinel.conf
```
# sentinel.conf 예시

# mymaster : 모니터링할 Redis 마스터 노드 이름(임의 지정 가능)
# 127.0.0.1 6379 : 마스터 노드의 IP 및 포트
# 2 : 장애 판단을 위한 최소 동의 인스턴스 수 (Quorum 값)

sentinel monitor mymaster 127.0.0.1 6379 2

sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1
```

#### 페일오버 테스트 방법
- Redis master 강제 종료 및 Sentinel 자동 failover 확인
```
SENTINEL masters
SENTINEL slaves <master_name>
SENTINEL failover <master_name>
```
- 센티널 구성의 주요 포인트
  - 패스워드 인증 (`requirepass` & `masterauth`)
  - 복제본 우선순위 관리 (`slave-priority`)
  - 운영 중 센티널 구성 변경 및 노드 추가/제거 방법 (`sentinel reset`, `sentinel remove`)
- 센티널 초기화 및 자동 페일오버 과정
  - 구성 정보 초기화 시의 동작 확인
  - failover 자동화와 안정성


## 센티널 문제 상황 대응

### 스플릿 브레인
- 스플릿 브레인(Split Brain) 이란, 네트워크 장애 등으로 클러스터 또는 센티널 노드들이 서로를 인식하지 못하고, 각각 독립적으로 마스터라고 판단하는 상황을 말합니다.
- 네트워크가 쪼개져 서로를 못 보는 상황에서, 각각이 자기만 마스터라고 착각하는 위험한 상태
- 두 개 이상의 노드가 동시에 마스터 역할 수행
- 데이터가 동기화되지 않은 상태로 병렬 처리되어 데이터 손상, 충돌, 유실 발생 가능
- down-after-milliseconds 및 failover-timeout 설정 조정