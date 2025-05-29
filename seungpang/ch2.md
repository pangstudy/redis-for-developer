# 레디스 시작하기


### 설치
#### macOS 설치

```bash
brew install redis
brew services start redis
redis-cli
```

* `ping` → `pong` 이 나오면 성공

#### Docker 설치

```bash
docker run -d --name=redis -p 6379:6379 redis:7.4.3
```

### 서버 환경 설정 체크리스트

#### maxclients / openfiles 설정

* 기본값: 10000, 내부용 파일 디스크립터 32 포함 필요 → 최소 10032 이상 필요

```bash
ulimit -a | grep open
```

* 설정 예시 (`/etc/security/limits.conf`):

```conf
*        hard    nofile 100000
*        soft    nofile 100000
```

#### THP (Transparent Huge Pages) 비활성화

* 성능 저하 방지 목적

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

* `/etc/rc.local`에 자동 비활성화 추가:

```bash
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
  echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi
chmod +x /etc/rc.d/rc.local
```

#### vm.overcommit_memory 설정

* fork + COW로 인해 순간 메모리 초과 방지

```bash
echo 'vm.overcommit_memory=1' >> /etc/sysctl.conf
sysctl vm.overcommit_memory=1
```

#### somaxconn / syn\_backlog 설정

```bash
# /etc/sysctl.conf
net.core.somaxconn = 1024
net.ipv4.tcp_max_syn_backlog = 1024

# 적용
sysctl net.core.somaxconn=1024
sysctl net.ipv4.tcp_max_syn_backlog=1024
```

#### redis.conf 주요 항목 정리

```conf
port 6379
bind 127.0.0.1 -::1
protected-mode yes
requirepass (기본 없음)
daemonize no
dir ./
```

* 외부 연결 시 bind 설정 변경 필요
* daemonize = yes 로 설정 시 백그라운드 실행 가능

해당 부분은 설정에 관련된 부분이여서 간략하게 요약만 했습니다.
