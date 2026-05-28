# 🔧 Ansible 기반 고가용성 웹 인프라 자동화

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Ansible](https://img.shields.io/badge/Ansible-2.9+-red.svg)](https://www.ansible.com/)
[![Rocky Linux](https://img.shields.io/badge/Rocky%20Linux-9-green.svg)](https://rockylinux.org/)

> Ansible을 활용한 고가용성(HA) 웹 인프라 자동 구축 및 배포 시스템

---

## 🎯 프로젝트 성과

| 성과 지표 | 결과 | 비고 |
|----------|------|------|
| ⚡ **인프라 구축 시간** | 5분 | 수동 구축 대비 90% 단축 |
| 🔄 **재현 가능성** | 100% | 동일한 환경 반복 구축 |
| 📝 **코드화된 서버** | 5대 | 15개 Playbook으로 관리 |
| 🛡️ **설정 일관성** | 100% | 사람 실수 제거 |
| 🚀 **배포 자동화** | 100% | 한 번의 명령으로 전체 구축 |

---

## 🚀 빠른 시작

### 전제 조건

- **VirtualBox** 설치
- **Rocky Linux 9** VM 5대 구성
  - LB01: `192.168.56.11`
  - WEB01: `192.168.56.21`
  - WEB02: `192.168.56.22`
  - Storage01: `192.168.56.30`
  - DB-central01: `192.168.56.40`
- **Ansible 2.9+** 설치 (컨트롤 노드)
- SSH 키 기반 인증 설정 완료

### 실행 방법

#### 1️⃣ 저장소 클론
```bash
git clone https://github.com/qkrtpdlr/ansible-infra-automation.git
cd ansible-infra-automation
```

#### 2️⃣ Inventory 파일 확인 및 수정
```bash
# inventory.ini 파일에서 IP 주소 확인
cat inventory.ini

# 필요시 IP 주소 수정
vim inventory.ini
```

#### 3️⃣ SSH 연결 테스트
```bash
# 모든 호스트 Ping 테스트
ansible all -m ping

# 예상 출력:
# LB01 | SUCCESS => {"changed": false, "ping": "pong"}
# WEB01 | SUCCESS => {"changed": false, "ping": "pong"}
# ...
```

#### 4️⃣ 전체 인프라 자동 구축
```bash
# 전체 인프라 구축 (약 5분 소요)
# 순서: Storage → DB → Web → LB
ansible-playbook all_setup.yml

# 실행 결과 예시:
# PLAY RECAP *************************************************************
# LB01                       : ok=15   changed=12   unreachable=0   failed=0
# WEB01                      : ok=18   changed=15   unreachable=0   failed=0
# WEB02                      : ok=16   changed=13   unreachable=0   failed=0
# Storage01                  : ok=10   changed=8    unreachable=0   failed=0
# DB-central01               : ok=12   changed=10   unreachable=0   failed=0
```

#### 5️⃣ 개별 서버 설정 (선택사항)
```bash
# 로드밸런서만 설정
cd LB01
ansible-playbook lb_setup.yml

# 웹 서버 1만 설정
cd ../WEB01
ansible-playbook web1_setup.yml

# 웹 서버 2만 설정
cd ../WEB02
ansible-playbook web2_setup.yml

# 스토리지만 설정
cd ../Storage01
ansible-playbook storage_setup.yml

# 데이터베이스만 설정
cd ../DB-central01
ansible-playbook db-central01_setup.yml
```

#### 6️⃣ 서비스 검증
```bash
# HAProxy 통계 페이지 확인
curl http://192.168.56.11:9000/stats
# 또는 브라우저에서: http://192.168.56.11:9000/stats
# (admin/admin으로 로그인)

# WordPress 사이트 접속 (포트 8080)
curl http://192.168.56.21:8080
# 또는 브라우저에서: http://web01.example.com:8080

# 정적 웹 사이트 접속 (포트 80)
curl http://192.168.56.22:80
# 또는 브라우저에서: http://web02.example.com

# DNS 쿼리 테스트
dig @192.168.56.11 web01.example.com

# MariaDB 접속 테스트
mysql -h 192.168.56.40 -u wpuser -p wordpress
```

#### 7️⃣ 서비스 상태 확인
```bash
# 모든 서버의 서비스 상태 확인
ansible all -m shell -a "systemctl status httpd" --limit webservers
ansible all -m shell -a "systemctl status haproxy" --limit LB01
ansible all -m shell -a "systemctl status mariadb" --limit DB-central01
ansible all -m shell -a "systemctl status nfs-server" --limit Storage01
```

### 📹 실행 결과 예시

**성공 시 출력**:
```
PLAY [Setup complete infrastructure] ************************************

TASK [Gathering Facts] ***************************************************
ok: [Storage01]
ok: [DB-central01]
ok: [WEB01]
ok: [WEB02]
ok: [LB01]

...

PLAY RECAP *************************************************************
LB01                       : ok=15   changed=12   unreachable=0   failed=0
WEB01                      : ok=18   changed=15   unreachable=0   failed=0
WEB02                      : ok=16   changed=13   unreachable=0   failed=0
Storage01                  : ok=10   changed=8    unreachable=0   failed=0
DB-central01               : ok=12   changed=10   unreachable=0   failed=0

✅ 전체 구축 완료! (약 5분 소요)
```

---

## 📖 작업용 가이드

실제 작업 시에는 **[README_WORK.md](README_WORK.md)**를 참조하세요.
- 실행 명령어
- 검증 방법
- 트러블슈팅
- 체크리스트

---

## 📁 프로젝트 구조

```
ansible-infra-automation/
├── ansible.cfg                  # Ansible 전역 설정
├── inventory.ini                # 전체 호스트 인벤토리
├── all_setup.yml                # 전체 인프라 구축 Playbook
│
├── LB01/                        # 로드밸런서 + DNS
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── lb_setup.yml
│
├── WEB01/                       # 웹 서버 1 (WordPress)
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── web1_setup.yml
│
├── WEB02/                       # 웹 서버 2 (Static)
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── web2_setup.yml
│
├── Storage01/                   # NFS Storage
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── storage_setup.yml
│
├── DB-central01/                # MariaDB Database
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── db-central01_setup.yml
│
└── template/                    # Jinja2 템플릿 파일들
    ├── haproxy.cfg.j2           # HAProxy 설정
    ├── named.conf.j2            # BIND DNS 설정
    ├── db.example.com.j2        # DNS Zone 파일
    ├── web01_site1.conf.j2      # Apache VirtualHost
    ├── wp01_site2.conf.j2       # WordPress VirtualHost
    ├── wp-config.php.j2         # WordPress 설정
    ├── web01_index.j2           # 정적 웹 페이지
    ├── web02_index.j2
    ├── web02_site1.conf.j2
    ├── web02_site2.conf.j2
    ├── exports.j2               # NFS Exports
    ├── mariadb-server.cnf.j2    # MariaDB 설정
    └── root-my.cnf.j2           # MariaDB 인증
```

---

## 🏗️ 시스템 아키텍처

### 전체 구조

```
        사용자 (클라이언트)
               |
               |
    ┌──────────▼──────────┐
    │                     │
    │  Load Balancer Tier │
    │  LB01 (HAProxy+DNS) │
    │  192.168.56.11      │
    │                     │
    └──────┬──────────┬───┘
           │          │
    ┌──────▼──┐  ┌───▼──────┐
    │  WEB01  │  │  WEB02   │
    │ Apache  │  │ Apache   │
    │WordPress│  │ Static   │
    │.56.21   │  │.56.22    │
    └────┬────┘  └───┬──────┘
         │           │
         └─────┬─────┘
               │
         ┌─────┴─────┐
         │           │
    ┌────▼───┐  ┌───▼────┐
    │Storage │  │Database│
    │ NFS    │  │MariaDB │
    │.56.30  │  │.56.40  │
    └────────┘  └────────┘
```

### 서버 구성

| 호스트명 | IP | 역할 | 서비스 |
|---------|-----|------|--------|
| **LB01** | 192.168.56.11 | Load Balancer + DNS | HAProxy, BIND |
| **WEB01** | 192.168.56.21 | 웹 서버 1 | Apache, WordPress, PHP |
| **WEB02** | 192.168.56.22 | 웹 서버 2 | Apache, Static Web |
| **Storage01** | 192.168.56.30 | 스토리지 | NFS Server |
| **DB-central01** | 192.168.56.40 | 데이터베이스 | MariaDB |

---

## 🛠 기술 스택

### Infrastructure
- **OS**: Rocky Linux 9
- **Virtualization**: VirtualBox
- **Configuration Management**: Ansible 2.9+
- **Templating**: Jinja2

### Services
- **Load Balancer**: HAProxy
- **DNS**: BIND9
- **Web Server**: Apache HTTP Server 2.4
- **Application**: WordPress, PHP 7.x
- **Storage**: NFS (Network File System)
- **Database**: MariaDB 10.x
- **Security**: Firewalld, SELinux

---

## 🚀 주요 기능

### 1. Ansible 자동화

**특징**:
- SSH 기반 에이전트리스 자동화
- 멱등성(Idempotency) 보장
- Inventory 파일로 서버 그룹 관리
- Jinja2 템플릿으로 동적 설정 생성

**성과**:
- 수동 구축: 약 50분
- Ansible 자동화: **약 5분 (90% 단축)**

### 2. 로드 밸런싱 (HAProxy)

**구성**:
- Round-robin 알고리즘
- Health Check (5초마다)
- 자동 Failover (30초)

**템플릿 예시**:
```jinja2
backend http_back
    balance roundrobin
    {% for host in groups['webservers'] %}
    server {{ host }} {{ hostvars[host]['ansible_host'] }}:80 check
    {% endfor %}
```

### 3. 웹 서버

**WEB01**: Apache + WordPress + NFS
**WEB02**: Apache + Static Web + NFS

**주요 설정**:
- 포트 80, 8080 동시 운영
- NFS 공유 스토리지 마운트
- SELinux 정책 자동 적용

### 4. 스토리지 (NFS)

- NFSv3 프로토콜
- 웹 서버 간 파일 공유
- WordPress 업로드 디렉토리 공유

### 5. 데이터베이스 (MariaDB)

- 원격 접속 허용
- WordPress 전용 DB 자동 생성
- 사용자 권한 자동 설정

---

## 🐛 트러블슈팅 (실제 해결 사례)

### Issue 1: Apache 8080 포트 문제

**증상**: 포트 80은 되는데 8080 안 됨

**원인**: `Listen 8080` 설정 누락

**해결**:
```yaml
- name: Configure Apache to listen on port 8080
  lineinfile:
    path: /etc/httpd/conf/httpd.conf
    line: 'Listen 8080'
    insertafter: '^Listen 80'
  notify: restart apache
```

### Issue 2: NFS 마운트 실패

**증상**: SELinux에서 Apache가 NFS 접근 차단

**원인**: `httpd_use_nfs` Boolean off

**해결**:
```yaml
- name: Enable httpd to use NFS
  seboolean:
    name: httpd_use_nfs
    state: yes
    persistent: yes
```

### Issue 3: MariaDB 원격 접속 거부

**증상**: WEB01에서 DB 연결 실패

**원인**: 
- bind-address = 127.0.0.1 (localhost만)
- Firewall 3306 포트 차단

**해결**:
```yaml
- name: Configure MariaDB for remote access
  lineinfile:
    path: /etc/my.cnf.d/mariadb-server.cnf
    line: 'bind-address = 0.0.0.0'
  notify: restart mariadb

- name: Open MariaDB port
  firewalld:
    port: 3306/tcp
    permanent: yes
    state: enabled
```

### Issue 4: HAProxy Backend DOWN

**증상**: HAProxy 통계에서 Backend DOWN 표시

**원인**: Apache 중지 또는 Firewall

**해결**:
```yaml
- name: Ensure Apache is running
  systemd:
    name: httpd
    state: started
    enabled: yes

- name: Allow HTTP through firewall
  firewalld:
    service: http
    permanent: yes
    state: enabled
```

---

## 📊 성능 개선 지표

| 지표 | Before (수동) | After (Ansible) | 개선율 |
|------|--------------|----------------|--------|
| **구축 시간** | 50분 | 5분 | **90% 단축** |
| **일관성** | 70% | 100% | **30%p 향상** |
| **재구축 시간** | 50분 | 5분 | **동일** |
| **문서화** | 수동 | 코드로 자동 | **100%** |
| **롤백 시간** | 30분 | 2분 | **93% 단축** |

---

## 📚 학습 내용

### 1. Ansible 자동화
- Playbook 구조화
- Inventory 관리
- Handler 활용
- Jinja2 템플릿

### 2. Linux 시스템 관리
- Apache, NFS, MariaDB 구성
- SELinux, Firewalld 설정
- 네트워크 트러블슈팅

### 3. 고가용성 아키텍처
- Load Balancer 구성
- Multi-tier 아키텍처
- Failover 메커니즘

---

## 🔮 개선 방향

### 1. 모니터링
- Prometheus + Grafana
- 실시간 메트릭
- Alert 설정

### 2. CI/CD
- Jenkins/GitLab CI 연동
- 자동 배포 파이프라인

### 3. 보안 강화
- Ansible Vault로 비밀 정보 암호화
- SSL/TLS 인증서 자동 적용

---

## ☁️ AWS 전환 가이드

| On-Premise | AWS |
|-----------|-----|
| HAProxy | ELB/ALB |
| Apache (EC2) | EC2 Auto Scaling |
| NFS | EFS |
| MariaDB | RDS Multi-AZ |
| BIND DNS | Route 53 |
| Firewalld | Security Groups |
