## 센티널

### Sentinel 개요

Redis Sentinel은 Redis의 고가용성을 보장하기 위한 메커니즘으로, 다음과 같은 기능을 제공한다

- **모니터링**: 마스터 및 복제본 인스턴스의 상태를 실시간으로 확인
- **자동 장애 조치 (Failover)**: 마스터 장애 시 복제본 중 하나를 자동으로 마스터로 승격
- **구성 정보 안내**: 현재의 마스터 정보를 클라이언트에게 제공하여 endpoint 변경 없이도 재접속 가능

---

### 분산 시스템으로서의 Sentinel

- Sentinel 자체의 SPOF 방지를 위해 **최소 3대 이상** 운영 권장
- **quorum** 개념으로 오탐 방지 (다수결 투표와 유사)
- 예: 3대 센티널 중 2대가 마스터 문제 판단 → failover 진행

---

### Sentinel 인스턴스 배치 및 실행

- **배치 권장사항**: 서로 다른 물리적 위치에 3대 이상 구성
- Redis와 Sentinel을 각각의 서버에 설치

### 주요 명령어
- `SENTINEL master <master-name>`: 모니터링 중인 마스터 정보 확인
- `SENTINEL replicas <master-name>`: 복제본 정보 확인
- `SENTINEL sentinels <master-name>`: 마스터를 모니터링 중인 다른 센티널 정보
- `SENTINEL chquorum <master-name>`: quorum 충족 여부 확인
- `SENTINEL RESET <master-name>`: 센티널 상태 초기화

---

### Sentinel 운영 포인트

#### 패스워드 인증
- 모든 Redis 인스턴스에 동일한 `requirepass`, `masterauth` 설정

#### 복제본 우선순위
- `replica-priority` 값이 낮을수록 failover 시 우선 선택됨

#### Sentinel 장애 시 처리
- 장애 Redis도 지속적으로 모니터링하며 PING 요청
- 중단 원할 시 `SENTINEL RESET` 사용

---

### Sentinel의 자동 Failover 과정

#### 마스터 장애 감지
- `down-after-milliseconds` 동안 응답 없을 경우 sdown 판단 (기본 30초)

#### sdown → odown 전환
- 한 센티널이 마스터에 sdown 판단 → 다른 센티널과 통신하여 동의 수집
- quorum 이상 동의 시 odown 전환

#### epoch 증가
- Failover 발생 시 epoch 값 증가하여 버전 관리

#### 리더 센티널 선출
- Failover 실행할 리더 선정: quorum 이상 + 센티널 과반수 동의 필요

#### 마스터 승격 절차
1. `replica-priority` 가장 낮은 복제본
2. 마스터로부터 더 많은 데이터를 받은 복제본
3. runID가 사전 순으로 가장 작은 복제본

---

### 스플릿 브레인 이슈

- 네트워크 단절로 인해 복제본이 마스터로 잘못 승격 → **2개의 마스터 존재 가능**
- 복구 시 기존 마스터가 새 마스터의 복제본이 되며, **데이터 초기화** 발생 가능

---

### 궁금한 점들

### Q. Sentinel이 짝수 대일 때 quorum은?

- N대 센티널일 경우 일반적으로 **N/2** 정도 설정이 적절
- `sentinel.conf`에서 quorum 값 조정 가능

### Q. Sentinel은 몇 대가 가장 적절한가?

- 1대는 SPOF 가능성 → 권장하지 않음
- 3대가 표준 (비용/효율/안정성 기준)
- Redisgate에서는 최소 quorum 2, 센티널 수 3 이상을 권장

### Q. AWS ElastiCache에서는 Sentinel 설정 필요?

- **불필요**: Multi-AZ가 활성화되어 있으면 AWS에서 유사 기능을 자동 제공
- 참고: [AWS 문서](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/FaultTolerance.html#FaultTolerance.Redis)

### Q. 클라이언트가 failover로 변경된 마스터를 어떻게 인지?

- **HAProxy 사용**: Redis 앞단에 프록시 레이어로 설정
- **애플리케이션 내 Sentinel 클라이언트 코드 작성** 필요
    - Sentinel에 직접 연결하여 현재 마스터 주소 확인 후 연결

---

### 참고 자료

- [카카오의 Redis Sentinel 사용 사례](https://tech.kakao.com/2020/11/10/if-kakao-2020-commentary-01-kakao/)
- [HAProxy를 이용한 failover 처리 영상 (25:38~)](https://www.youtube.com/watch?v=bDDfVQkOcQM)
