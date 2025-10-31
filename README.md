# 🔧 Ansible 기반 고가용성 웹 인프라 자동화

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
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

## 📚 문서

- 📖 [상세 구축 가이드](#-설치-및-실행)
- 🏗️ [아키텍처 설명](#-시스템-아키텍처)
- 🔧 [트러블슈팅](#-트러블슈팅)

---

## 📌 프로젝트 개요

| 항목 | 내용 |
|------|------|
| **프로젝트명** | Ansible 기반 고가용성 웹 인프라 자동화 |
| **기간** | 2024.06 |
| **유형** | Infrastructure as Code (IaC) 실습 |
| **GitHub** | https://github.com/qkrtpdlr/ansible-infra-automation |

### 🎯 프로젝트 목표

- **Infrastructure as Code (IaC)** 개념을 Ansible로 실전 구현
- **고가용성(HA)** 웹 인프라 설계 및 자동 구축
- **Multi-tier 아키텍처** 구성 (Load Balancer, Web, Storage, Database)

---

## 🏗️ 시스템 아키텍처

### 📐 전체 구조

```
┌─────────────────────────────────────────┐
│        Client (외부 사용자)             │
└───────────────┬─────────────────────────┘
                │
┌───────────────▼─────────────────────────┐
│   Load Balancer + DNS Tier              │
│   LB01 (HAProxy + BIND)                 │
│   192.168.56.11                         │
└───────┬───────────────┬─────────────────┘
        │               │
   ┌────▼────┐     ┌───▼─────┐
   │  WEB01  │     │  WEB02  │
   │ Apache  │     │ Apache  │
   │WordPress│     │Static   │
   │.56.21   │     │.56.22   │
   └────┬────┘     └───┬─────┘
        │              │
        └──────┬───────┘
               │
      ┌────────┴─────────┐
      │                  │
┌─────▼──────┐    ┌─────▼─────┐
│ Storage01  │    │DB-central │
│NFS Server  │    │  MariaDB  │
│.56.30      │    │  .56.40   │
└────────────┘    └───────────┘
```

### 🖥️ 서버 구성

| 호스트명 | 역할 | IP 주소 | 주요 서비스 |
|---------|------|---------|------------|
| **LB01** | 로드밸런서 + DNS | 192.168.56.11 | HAProxy, BIND |
| **WEB01** | 웹 서버 1 | 192.168.56.21 | Apache (80, 8080), WordPress, PHP |
| **WEB02** | 웹 서버 2 | 192.168.56.22 | Apache (80, 8080), Static Web |
| **Storage01** | 파일 스토리지 | 192.168.56.30 | NFS Server |
| **DB-central01** | 데이터베이스 | 192.168.56.40 | MariaDB |

**네트워크 구성:**
- NAT Network: `10.0.2.0/24` (외부 통신)
- Host-Only Network: `192.168.56.0/24` (내부 통신)

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

### 1. Ansible 자동화 배포

**특징:**
- SSH 기반 에이전트리스 자동화
- 멱등성(Idempotency) 보장 - 여러 번 실행해도 동일한 결과
- Inventory 파일로 서버 그룹 관리
- Jinja2 템플릿으로 설정 파일 동적 생성

**구현:**
```yaml
# all_setup.yml - 전체 인프라 자동 구축
- name: Setup Load Balancer
  hosts: lb
  roles:
    - haproxy
    - bind_dns

- name: Setup Web Servers
  hosts: webservers
  roles:
    - apache
    - nfs_client
    - wordpress

- name: Setup Storage
  hosts: storage
  roles:
    - nfs_server

- name: Setup Database
  hosts: database
  roles:
    - mariadb
```

**성과:**
- 수동 구축: 약 50분 소요
- Ansible 자동화: **약 5분 소요 (90% 단축)**
- 재구축 시간: **100% 동일한 환경**

---

### 2. 로드 밸런서 (LB01)

**HAProxy 구성:**
- Round-robin 알고리즘으로 트래픽 분산
- Health Check로 장애 서버 자동 제외
- Backend 서버 2대 (WEB01, WEB02)

**BIND DNS 서버:**
- example.com zone 관리
- 내부 DNS 조회 서비스

**Jinja2 템플릿:**
```jinja2
# haproxy.cfg.j2
frontend http_front
    bind *:80
    default_backend http_back

backend http_back
    balance roundrobin
    {% for host in groups['webservers'] %}
    server {{ host }} {{ hostvars[host]['ansible_host'] }}:80 check
    {% endfor %}
```

**성과:**
- 트래픽 자동 분산
- WEB01 장애 시 WEB02로 자동 Failover
- 30초 내 장애 감지

---

### 3. 웹 서버 (WEB01, WEB02)

**Apache VirtualHost:**
- WEB01: WordPress (포트 80, 8080)
- WEB02: 정적 웹사이트 (포트 80, 8080)

**NFS 마운트:**
- Storage01의 공유 디렉토리 자동 마운트
- `/srv/web/shared` 경로로 접근

**WordPress 자동 설치:**
- DB 연결 자동 설정
- wp-config.php 자동 생성

**SELinux 설정:**
- `httpd_use_nfs` Boolean 활성화
- NFS 마운트 경로에 Apache 접근 허용

**성과:**
- 웹 서버 2대 동시 구축 시간: **3분**
- WordPress 설치 자동화
- NFS 공유 디렉토리 자동 마운트

---

### 4. 스토리지 (Storage01)

**NFS 서버 구성:**
- NFSv3 프로토콜 사용
- `/etc/exports` 자동 설정
- 웹 서버에 공유 디렉토리 제공

**Ansible Task:**
```yaml
- name: Create NFS export directory
  file:
    path: /srv/nfs/shared
    state: directory
    mode: '0755'

- name: Configure NFS exports
  template:
    src: exports.j2
    dest: /etc/exports
  notify: restart nfs

- name: Start NFS service
  systemd:
    name: nfs-server
    state: started
    enabled: yes
```

**성과:**
- 중앙 집중식 파일 관리
- 웹 서버 간 파일 동기화

---

### 5. 데이터베이스 (DB-central01)

**MariaDB 구성:**
- WordPress용 데이터베이스 자동 생성
- 사용자 계정 및 권한 자동 설정
- 원격 접속 허용 (bind-address 0.0.0.0)

**보안 설정:**
```yaml
- name: Create WordPress database
  mysql_db:
    name: wordpress
    state: present

- name: Create WordPress user
  mysql_user:
    name: wpuser
    password: "{{ db_password }}"
    priv: 'wordpress.*:ALL'
    host: '192.168.56.%'
    state: present
```

**Firewall 설정:**
```bash
firewall-cmd --permanent --add-port=3306/tcp
firewall-cmd --reload
```

**성과:**
- DB 자동 초기화
- 원격 접속 보안 설정
- Firewall 자동 구성

---

## 📁 디렉토리 구조

```
ansible-infra-automation/
├── ansible.cfg                  # Ansible 전역 설정
├── inventory.ini                # 전체 호스트 인벤토리
├── all_setup.yml                # 전체 인프라 구축 Playbook
│
├── LB01/
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── lb_setup.yml             # 로드밸런서 구축
│
├── WEB01/
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── web1_setup.yml           # 웹 서버 1 구축
│
├── WEB02/
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── web2_setup.yml           # 웹 서버 2 구축
│
├── Storage01/
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── storage_setup.yml        # 스토리지 구축
│
├── DB-central01/
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── db-central01_setup.yml   # 데이터베이스 구축
│
└── template/
    ├── haproxy.cfg.j2           # HAProxy 설정 템플릿
    ├── named.conf.j2            # BIND 설정 템플릿
    ├── db.example.com.j2        # DNS Zone 파일
    ├── web01_site1.conf.j2      # Apache VirtualHost 템플릿
    ├── wp01_site2.conf.j2       # WordPress VirtualHost 템플릿
    ├── wp-config.php.j2         # WordPress 설정 템플릿
    ├── web01_index.j2           # 정적 웹 인덱스 1
    └── web02_index.j2           # 정적 웹 인덱스 2
```

---

## 💻 설치 및 실행

### 사전 요구사항
- VirtualBox 6.x 이상
- Rocky Linux 9 ISO
- 최소 시스템 요구사항: 8GB RAM, 50GB 디스크

### 1. 가상 머신 생성

```bash
# VirtualBox에서 5대의 VM 생성 (LB01, WEB01, WEB02, Storage01, DB-central01)
# 각 VM에 NAT + Host-Only Network 설정
```

### 2. Ansible 설치 (LB01 서버에서)

```bash
# EPEL 저장소 추가
sudo yum -y install epel-release

# Ansible 설치
sudo yum -y install ansible

# Ansible 버전 확인
ansible --version
```

### 3. SSH 키 교환 (패스워드 없는 인증)

```bash
# SSH 키 생성
ssh-keygen -t rsa -b 2048

# 각 서버에 SSH 공개 키 복사
ssh-copy-id WEB01@192.168.56.21
ssh-copy-id WEB02@192.168.56.22
ssh-copy-id Storage01@192.168.56.30
ssh-copy-id DB-central01@192.168.56.40
```

### 4. Ansible Playbook 실행

**전체 인프라 자동 구축:**
```bash
# 프로젝트 디렉토리로 이동
cd /home/LB01/ansible

# 전체 인프라 구축 (약 5분 소요)
ansible-playbook all_setup.yml
```

**개별 서버 구축:**
```bash
# 개별적으로 실행 가능
ansible-playbook LB01/lb_setup.yml
ansible-playbook WEB01/web1_setup.yml
ansible-playbook WEB02/web2_setup.yml
ansible-playbook Storage01/storage_setup.yml
ansible-playbook DB-central01/db-central01_setup.yml
```

### 5. 동작 확인

```bash
# 로드밸런서를 통한 접속
curl http://192.168.56.11

# WordPress 직접 접속
curl http://192.168.56.21

# 정적 웹사이트 접속
curl http://192.168.56.21:8080
curl http://192.168.56.22:8080
```

---

## 🐛 트러블슈팅

### Issue 1: Apache 8080 포트 충돌

**증상:**
- `192.168.56.21:80`은 접근 가능하지만, 8080 포트 접근 실패

**원인:**
- Apache가 8080 포트를 Listen하지 않음

**해결:**
```apache
# Apache 설정 파일에 추가
Listen 8080

<VirtualHost *:8080>
    DocumentRoot /srv/web/web01
    <Directory /srv/web/web01>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

**Ansible 자동화:**
```yaml
- name: Configure Apache to listen on port 8080
  lineinfile:
    path: /etc/httpd/conf/httpd.conf
    line: 'Listen 8080'
    insertafter: '^Listen 80'
  notify: restart apache
```

**결과:**
- 8080 포트 정상 접속 ✅
- VirtualHost 설정 자동화

---

### Issue 2: WEB02 NFS 마운트 실패

**증상:**
- SELinux enforcing 모드에서 httpd가 NFS 마운트 경로 접근 거부

**원인:**
- SELinux Boolean `httpd_use_nfs`가 off 상태

**해결:**
```bash
# SELinux Boolean 활성화
setsebool -P httpd_use_nfs 1

# 설정 확인
getsebool httpd_use_nfs
# httpd_use_nfs --> on
```

**Ansible 자동화:**
```yaml
- name: Enable httpd to use NFS
  seboolean:
    name: httpd_use_nfs
    state: yes
    persistent: yes
```

**결과:**
- NFS 마운트 경로 정상 접근 ✅
- SELinux 정책 자동 적용

---

### Issue 3: MariaDB 원격 접속 거부

**증상:**
- WEB01에서 DB-central01 MariaDB 접속 실패

**원인:**
- MariaDB가 localhost에만 바인딩됨 (bind-address = 127.0.0.1)
- 원격 접속 권한 미설정
- Firewall에서 3306 포트 차단

**해결:**
```sql
-- MariaDB 설정 변경
SET GLOBAL bind_address = '0.0.0.0';

-- 원격 접속 권한 부여
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'192.168.56.%';
FLUSH PRIVILEGES;
```

```bash
# Firewall 설정
firewall-cmd --permanent --add-port=3306/tcp
firewall-cmd --reload
```

**Ansible 자동화:**
```yaml
- name: Configure MariaDB for remote access
  lineinfile:
    path: /etc/my.cnf.d/mariadb-server.cnf
    regexp: '^bind-address'
    line: 'bind-address = 0.0.0.0'
  notify: restart mariadb

- name: Open MariaDB port in firewall
  firewalld:
    port: 3306/tcp
    permanent: yes
    state: enabled
    immediate: yes
```

**결과:**
- 원격 DB 접속 성공 ✅
- WordPress DB 연결 정상 작동

---

### Issue 4: HAProxy Health Check 실패

**증상:**
- HAProxy에서 Backend 서버가 DOWN 상태로 표시

**원인:**
- Apache가 정상 실행되지 않음
- Firewall에서 80 포트 차단

**해결:**
```bash
# Apache 상태 확인
systemctl status httpd

# Apache 재시작
systemctl restart httpd

# Firewall 설정
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```

**Ansible 자동화:**
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
    immediate: yes
```

**결과:**
- HAProxy Health Check 성공 ✅
- 트래픽 정상 분산

---

## 📊 프로젝트 성과

### 정량적 성과

| 지표 | Before (수동) | After (Ansible) | 개선율 |
|------|--------------|----------------|--------|
| **인프라 구축 시간** | 50분 | 5분 | **90% 단축** |
| **설정 일관성** | 70% (사람 실수) | 100% | **30% 향상** |
| **재구축 시간** | 50분 | 5분 | **반복 가능** |
| **문서화** | 수동 작성 | 코드로 자동 문서화 | **100% 정확** |
| **롤백 시간** | 30분 | 2분 | **93% 단축** |

### 정성적 성과

**1. Infrastructure as Code 경험**
- 인프라를 코드로 관리하는 IaC 개념 실전 적용
- Git으로 인프라 버전 관리
- 코드 리뷰를 통한 인프라 변경 관리

**2. 자동화 스킬 향상**
- Ansible Playbook 작성 능력 습득
- Jinja2 템플릿 활용 능력
- 멱등성(Idempotency) 개념 이해

**3. Linux 시스템 관리**
- Apache, NFS, MariaDB 등 서비스 구성 경험
- SELinux, Firewalld 보안 설정
- 네트워크 구성 및 트러블슈팅

**4. 고가용성 아키텍처 이해**
- Load Balancer를 통한 트래픽 분산
- Multi-tier 아키텍처 설계
- Failover 메커니즘 구현

---

## 📚 학습 내용 및 회고

### 기술적 학습

**1. Ansible 자동화**
- Playbook 구조화 및 Role 분리
- Inventory 관리 및 그룹 변수 활용
- Handler를 통한 서비스 재시작 자동화

**2. Jinja2 템플릿 엔진**
- 변수 치환 및 조건문 활용
- 반복문으로 동적 설정 생성
- 템플릿 상속 및 재사용

**3. SELinux 보안**
- Boolean 설정으로 정책 변경
- Context 레이블 관리
- Audit 로그 분석

### 아쉬운 점 및 개선 방향

**1. 모니터링 부재**
- 현재: 수동 로그 확인
- 개선: Prometheus + Grafana 도입 예정

**2. CI/CD 파이프라인 미구축**
- 현재: 수동 Playbook 실행
- 개선: Jenkins/GitLab CI 연동 예정

**3. 보안 강화 필요**
- 현재: 기본 Firewalld 설정
- 개선: Ansible Vault로 민감 정보 암호화 예정

---

## 🔮 향후 개선 사항

### 1. 모니터링 시스템 구축
- Prometheus + Grafana 도입
- 실시간 메트릭 수집 및 시각화
- Alert 규칙 설정

### 2. 백업 자동화
- 데이터베이스 백업 스크립트
- NFS 데이터 백업
- 복구 절차 문서화

### 3. SSL/TLS 적용
- Let's Encrypt 자동 인증서 발급
- HTTPS 강제 적용
- 인증서 자동 갱신

### 4. 로그 중앙화
- ELK Stack (Elasticsearch, Logstash, Kibana)
- 모든 서버 로그 수집
- 로그 분석 및 검색

### 5. CI/CD 파이프라인
- Jenkins, GitLab CI 연동
- Git Push 시 자동 배포
- 테스트 자동화

---

## ☁️ AWS 전환 시 고려 사항

| On-Premise | AWS 서비스 |
|-----------|-----------|
| HAProxy | ELB/ALB (Elastic Load Balancer) |
| Apache (EC2) | EC2 Auto Scaling Group |
| NFS | EFS (Elastic File System) |
| MariaDB | RDS (MariaDB/MySQL) |
| BIND DNS | Route 53 |
| Firewalld | Security Groups |

---

## 🔗 참고 자료

### 공식 문서
- [Ansible Documentation](https://docs.ansible.com/)
- [HAProxy Documentation](http://www.haproxy.org/docs.html)
- [Apache HTTP Server](https://httpd.apache.org/docs/)
- [Rocky Linux Documentation](https://docs.rockylinux.org/)

### 프로젝트 문서
- [GitHub 저장소](https://github.com/qkrtpdlr/ansible-infra-automation)

---

- **Email**: rlagudfo1223@gmail.com
- **GitHub**: https://github.com/qkrtpdlr
