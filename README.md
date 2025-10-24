# 🚀 Ansible 기반 온프레미스 인프라 자동화 프로젝트

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Ansible](https://img.shields.io/badge/Ansible-2.9+-red.svg)](https://www.ansible.com/)
[![Rocky Linux](https://img.shields.io/badge/Rocky%20Linux-9-green.svg)](https://rockylinux.org/)

> Ansible을 활용한 고가용성(HA) 웹 서비스 인프라 완전 자동화 구축 프로젝트

## 📋 목차

- [프로젝트 개요](#-프로젝트-개요)
- [시스템 아키텍처](#-시스템-아키텍처)
- [기술 스택](#-기술-스택)
- [주요 기능](#-주요-기능)
- [설치 및 실행](#-설치-및-실행)
- [디렉토리 구조](#-디렉토리-구조)
- [트러블슈팅](#-트러블슈팅)
- [프로젝트 성과](#-프로젝트-성과)
- [향후 개선 사항](#-향후-개선-사항)

## 🎯 프로젝트 개요

| 항목 | 내용 |
|------|------|
| **프로젝트명** | Ansible 기반 HA 웹 서비스 인프라 자동 구축 |
| **프로젝트 기간** | 2024.06 |
| **프로젝트 유형** | Infrastructure as Code (IaC) |
| **주요 목표** | 온프레미스 인프라의 완전 자동화 및 고가용성 구현 |

### 핵심 목표

- ✅ **Infrastructure as Code** 구현을 통한 인프라 자동화
- ✅ **일관되고 반복 가능한** 인프라 배포 프로세스 확립
- ✅ **고가용성(HA)** 웹 서비스 아키텍처 구성
- ✅ **Multi-tier 아키텍처** 실무 경험

## 🏗️ 시스템 아키텍처

```
┌─────────────────────────────────────────────────────┐
│                   클라이언트                          │
└──────────────────┬──────────────────────────────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │   LB01 (HAProxy)     │
        │   + BIND DNS Server  │
        │   192.168.56.11      │
        └─────────┬────────────┘
                  │
         ┌────────┴─────────┐
         ▼                  ▼
  ┌─────────────┐    ┌─────────────┐
  │   WEB01     │    │   WEB02     │
  │   Apache    │    │   Apache    │
  │  WordPress  │    │  Static Web │
  │ 192.168.56.21│   │ 192.168.56.22│
  └──────┬──────┘    └──────┬──────┘
         │                  │
         └────────┬─────────┘
                  │
         ┌────────┴─────────┐
         ▼                  ▼
  ┌─────────────┐    ┌─────────────┐
  │ Storage01   │    │DB-central01 │
  │  NFS Server │    │   MariaDB   │
  │192.168.56.30│    │192.168.56.40│
  └─────────────┘    └─────────────┘
```

### 서버 구성

| 서버명 | 역할 | IP 주소 | 주요 서비스 |
|--------|------|---------|------------|
| **LB01** | 로드밸런서 + DNS | 192.168.56.11 | HAProxy, BIND |
| **WEB01** | 웹서버 1 | 192.168.56.21 | Apache (80, 8080), WordPress, PHP |
| **WEB02** | 웹서버 2 | 192.168.56.22 | Apache (80, 8080), Static Web |
| **Storage01** | 스토리지 | 192.168.56.30 | NFS Server |
| **DB-central01** | 데이터베이스 | 192.168.56.40 | MariaDB |

**네트워크 구성**:
- NAT Network: `10.0.2.0/24` (외부 인터넷 연결)
- Host-Only Network: `192.168.56.0/24` (내부 통신)

## 🛠️ 기술 스택

### Infrastructure
- **OS**: Rocky Linux 9
- **Virtualization**: VirtualBox
- **Configuration Management**: Ansible
- **Templating**: Jinja2

### Services
- **Load Balancer**: HAProxy
- **DNS**: BIND9
- **Web Server**: Apache HTTP Server 2.4
- **Application**: WordPress, PHP 7.x
- **Storage**: NFS (Network File System)
- **Database**: MariaDB 10.x
- **Security**: Firewalld, SELinux

## 💻 주요 기능

### 1. Ansible 자동화

- **SSH 키 기반 자동 접속** 구성
- **Inventory 관리**를 통한 호스트 그룹화
- **공통 패키지 배포** 자동화
- **Jinja2 템플릿**을 활용한 동적 설정 파일 생성

### 2. 로드밸런서 (LB01)

- **HAProxy** Round-robin 로드밸런싱
- **BIND DNS** 서버 구성 (example.com zone)
- 웹 서버 health check 및 자동 failover

### 3. 웹 서버 (WEB01, WEB02)

- **Apache VirtualHost** 다중 사이트 구성
- **NFS 마운트**를 통한 콘텐츠 공유
- **WordPress** 자동 설치 및 설정
- **SELinux** Boolean 설정으로 보안 정책 적용

### 4. 스토리지 서버 (Storage01)

- **NFS 공유** 디렉토리 3개 생성 및 관리
- **추가 디스크** 마운트 및 자동 설정
- `/etc/exports` 파일 자동 생성

### 5. 데이터베이스 서버 (DB-central01)

- **MariaDB** 설치 및 보안 설정
- **WordPress 데이터베이스** 및 사용자 자동 생성
- **원격 접속** 허용 설정 (bind-address 0.0.0.0)

## 🚀 설치 및 실행

### 사전 요구사항

- VirtualBox 6.x 이상
- Rocky Linux 9 ISO 이미지
- 최소 8GB RAM, 50GB 디스크 공간

### 1. 가상머신 생성

```bash
# 5대의 가상머신 생성 (LB01, WEB01, WEB02, Storage01, DB-central01)
# NAT + Host-Only Network 구성
```

### 2. Ansible 설치 (LB01에서 실행)

```bash
# EPEL 저장소 설치
sudo yum -y install epel-release

# Ansible 설치
sudo yum -y install ansible

# Ansible 버전 확인
ansible --version
```

### 3. SSH 키 생성 및 배포

```bash
# SSH 키 생성
ssh-keygen -t rsa -b 2048

# 각 서버에 SSH 키 복사
ssh-copy-id WEB01@192.168.56.21
ssh-copy-id WEB02@192.168.56.22
ssh-copy-id Storage01@192.168.56.30
ssh-copy-id DB-central01@192.168.56.40
```

### 4. Ansible Playbook 실행

```bash
# 프로젝트 디렉토리로 이동
cd /home/LB01/ansible

# 공통 패키지 설치
ansible-playbook all_setup.yml

# 각 서버별 설정 실행
ansible-playbook LB01/lb_setup.yml
ansible-playbook WEB01/web1_setup.yml
ansible-playbook WEB02/web2_setup.yml
ansible-playbook Storage01/storage_setup.yml
ansible-playbook DB-central01/db-central01_setup.yml
```

### 5. 서비스 접속 확인

```bash
# 로드밸런서를 통한 접속
curl http://192.168.56.11

# WordPress 접속
curl http://192.168.56.21

# 웹서버 직접 접속
curl http://192.168.56.21:8080
curl http://192.168.56.22:8080
```

## 📁 디렉토리 구조

```
ansible/
├── ansible.cfg                  # Ansible 기본 설정
├── inventory.ini                # 호스트 인벤토리
├── all_setup.yml                # 공통 패키지 배포
│
├── LB01/
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── lb_setup.yml             # 로드밸런서 설정
│
├── WEB01/
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── web1_setup.yml           # 웹서버1 설정
│
├── WEB02/
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── web2_setup.yml           # 웹서버2 설정
│
├── Storage01/
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── storage_setup.yml        # 스토리지 설정
│
├── DB-central01/
│   ├── ansible.cfg
│   ├── inventory.ini
│   └── db-central01_setup.yml   # 데이터베이스 설정
│
└── template/
    ├── haproxy.cfg.j2           # HAProxy 설정 템플릿
    ├── named.conf.j2            # BIND 설정 템플릿
    ├── db.example.com.j2        # DNS Zone 파일 템플릿
    ├── web01_site1.conf.j2      # Apache VirtualHost 템플릿
    ├── wp01_site2.conf.j2       # WordPress VirtualHost 템플릿
    ├── wp-config.php.j2         # WordPress 설정 템플릿
    ├── web01_index.j2           # 웹페이지 템플릿
    └── web02_index.j2           # 웹페이지 템플릿
```

## 🔧 트러블슈팅

### Issue 1: Apache 8080 포트 접속 불가

**문제**: 192.168.56.21:80은 정상 접속되지만, 8080 포트는 연결 거부

**해결**:
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

### Issue 2: WEB02에서 NFS 마운트 콘텐츠 미노출

**문제**: SELinux enforcing 상태에서 httpd가 NFS 접근 불가

**해결**:
```bash
# SELinux Boolean 설정
setsebool -P httpd_use_nfs 1

# 설정 확인
getsebool httpd_use_nfs
```

### Issue 3: MariaDB 원격 접속 불가

**문제**: 웹 서버에서 데이터베이스 연결 실패

**해결**:
```sql
-- MariaDB 설정
SET GLOBAL bind_address = '0.0.0.0';

-- 사용자 권한 설정
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'192.168.56.%';
FLUSH PRIVILEGES;
```

```bash
# 방화벽 설정
firewall-cmd --permanent --add-port=3306/tcp
firewall-cmd --reload
```

## 📊 프로젝트 성과

### 정량적 성과
- ✅ **5대의 서버** 완전 자동화 구축
- ✅ **15개 이상의 Ansible Playbook** 작성
- ✅ **8개의 Jinja2 템플릿** 파일 구현
- ✅ **3개 이상의 실무 장애** 해결 경험

### 정성적 성과
- **Infrastructure as Code** 실전 경험
- **고가용성 아키텍처** 이해 및 구현
- **Linux 시스템 관리** 역량 강화
- **실무 트러블슈팅** 능력 배양

## 💡 향후 개선 사항

### 추가 구현 권장사항

1. **모니터링 구축**
   - Prometheus + Grafana
   - 서버 리소스 및 서비스 가용성 실시간 모니터링

2. **백업 전략**
   - 정기적인 데이터베이스 백업 스크립트
   - NFS 스토리지 백업 정책

3. **보안 강화**
   - Let's Encrypt SSL/TLS 인증서 적용
   - HTTPS 프로토콜 전환

4. **로그 중앙화**
   - ELK Stack (Elasticsearch, Logstash, Kibana)
   - 모든 서버 로그 중앙 집중 관리

5. **CI/CD 파이프라인**
   - Jenkins, GitLab CI 구축
   - 코드 변경 시 자동 배포

### 클라우드 전환 고려사항

| 온프레미스 | AWS 클라우드 |
|-----------|-------------|
| HAProxy | ELB/ALB |
| Apache (EC2) | EC2 Auto Scaling Group |
| NFS | EFS (Elastic File System) |
| MariaDB | RDS (MariaDB/MySQL) |
| BIND DNS | Route 53 |
| Firewalld | Security Groups |

## 📚 참고 문서

- [Ansible 공식 문서](https://docs.ansible.com/)
- [HAProxy 설정 가이드](http://www.haproxy.org/docs.html)
- [Apache HTTP Server 문서](https://httpd.apache.org/docs/)
- [Rocky Linux 공식 문서](https://docs.rockylinux.org/)

## 📝 라이선스

이 프로젝트는 MIT 라이선스 하에 배포됩니다.

## 👤 작성자

**박세익**  
- 프로젝트 기간: 2024.06
- 역할: 인프라 설계, Ansible 자동화, 전체 구축 및 트러블슈팅

---

⭐ 이 프로젝트가 도움이 되셨다면 Star를 눌러주세요!
