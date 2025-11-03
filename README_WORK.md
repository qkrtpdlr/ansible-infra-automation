# Ansible 인프라 자동화 - 작업용 가이드

> 이 문서는 실제 작업 시 참고용입니다. 포트폴리오용은 README.md를 참조하세요.

---

## 🔧 환경 구성

### VM 정보
| 호스트명 | IP | 역할 | 메모리 |
|---------|-----|------|--------|
| LB01 | 192.168.56.11 | HAProxy + BIND | 1GB |
| WEB01 | 192.168.56.21 | Apache + WordPress | 1GB |
| WEB02 | 192.168.56.22 | Apache + Static | 1GB |
| Storage01 | 192.168.56.30 | NFS Server | 1GB |
| DB-central01 | 192.168.56.40 | MariaDB | 2GB |

### 네트워크
- **NAT Network**: 10.0.2.0/24 (인터넷 연결용)
- **Host-Only**: 192.168.56.0/24 (내부 통신용)

---

## 🚀 빠른 시작

### 1. SSH 키 설정 (LB01에서 실행)
```bash
# SSH 키 생성
ssh-keygen -t rsa -b 2048

# 각 서버에 키 복사
ssh-copy-id root@192.168.56.21  # WEB01
ssh-copy-id root@192.168.56.22  # WEB02
ssh-copy-id root@192.168.56.30  # Storage01
ssh-copy-id root@192.168.56.40  # DB-central01
```

### 2. 전체 인프라 구축 (권장)
```bash
cd /home/LB01/ansible-infra-automation
ansible-playbook all_setup.yml

# 실행 시간: 약 5분
# 순서: Storage → DB → Web → LB (자동)
```

### 3. 개별 서버 구축
```bash
# 반드시 이 순서대로!
ansible-playbook Storage01/storage_setup.yml     # 1. NFS 먼저
ansible-playbook DB-central01/db-central01_setup.yml  # 2. DB
ansible-playbook WEB01/web1_setup.yml            # 3. 웹서버
ansible-playbook WEB02/web2_setup.yml
ansible-playbook LB01/lb_setup.yml               # 4. 로드밸런서 마지막
```

---

## 🔍 검증 방법

### 1. 로드밸런서 확인
```bash
# HTTP 접속 테스트
curl http://192.168.56.11

# HAProxy 통계 확인
# 브라우저: http://192.168.56.11:8404/stats
# ID: admin / PW: admin123
```

### 2. 웹서버 확인
```bash
# WEB01
curl http://192.168.56.21        # WordPress (80)
curl http://192.168.56.21:8080   # Static (8080)

# WEB02
curl http://192.168.56.22        # Static (80)
curl http://192.168.56.22:8080   # Static (8080)
```

### 3. NFS 확인
```bash
# Storage01에서
exportfs -v

# WEB01/WEB02에서
df -h | grep nfs
ls -la /srv/web/shared
```

### 4. DB 확인
```bash
# DB-central01에서
mysql -e "SHOW DATABASES;"

# WEB01에서 원격 접속 테스트
mysql -h 192.168.56.40 -u wpuser -p'SecurePassword123!' -e "SELECT 1;"
```

### 5. DNS 확인
```bash
dig @192.168.56.11 example.com
nslookup www.example.com 192.168.56.11
```

---

## 🐛 자주 발생하는 문제

### 1. Apache 8080 포트 안 열림
```bash
# 원인: Listen 8080 누락
# 해결:
echo "Listen 8080" | sudo tee -a /etc/httpd/conf/httpd.conf
sudo systemctl restart httpd
```

### 2. NFS 마운트 실패 (WEB02)
```bash
# 원인: SELinux 정책
# 해결:
sudo setsebool -P httpd_use_nfs 1
sudo mount -a
```

### 3. MariaDB 원격 접속 거부
```bash
# 원인: bind-address 127.0.0.1
# 해결:
sudo vi /etc/my.cnf.d/server.cnf
# bind-address = 0.0.0.0 으로 변경
sudo systemctl restart mariadb

# Firewall 열기
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
```

### 4. HAProxy Backend DOWN
```bash
# 원인: Apache 중지 또는 방화벽
# 해결:
sudo systemctl start httpd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

---

## 📝 중요 파일 위치

### Ansible 설정
- **전역 설정**: `ansible.cfg`
- **Inventory**: `inventory.ini`
- **템플릿**: `template/*.j2`

### 각 서버 로그
```bash
# Apache
/var/log/httpd/error_log
/var/log/httpd/access_log

# MariaDB
/var/log/mariadb/mariadb.log

# HAProxy
journalctl -u haproxy -f

# NFS
journalctl -u nfs-server -f
```

---

## 🔄 재구축 방법

### 전체 재구축
```bash
# 1. 모든 VM 스냅샷으로 복원
# 2. SSH 키 재설정 (위 참조)
# 3. Playbook 재실행
ansible-playbook all_setup.yml
```

### 특정 서버만 재구축
```bash
# 예: WEB01만 재구축
ansible-playbook WEB01/web1_setup.yml
```

---

## ⚙️ 변수 수정

### IP 주소 변경
```bash
# inventory.ini 수정
vi inventory.ini

[webservers]
WEB01 ansible_host=192.168.56.21  # 이 부분 수정
```

### 비밀번호 변경
```bash
# inventory.ini 또는 각 폴더의 inventory.ini
db_password=새로운비밀번호
```

---

## 🧪 테스트 명령어

### Ansible 연결 테스트
```bash
ansible all -m ping
```

### Playbook 문법 검사
```bash
ansible-playbook all_setup.yml --syntax-check
```

### Dry-run (실제 실행 안함)
```bash
ansible-playbook all_setup.yml --check
```

### 특정 태그만 실행
```bash
# (태그가 있다면)
ansible-playbook all_setup.yml --tags "web"
```

---

## 📊 성능 체크

### Load Balancer 부하 테스트
```bash
# 20번 요청 (Round-robin 확인)
for i in {1..20}; do
  curl -s http://192.168.56.11 | grep -oP '(?<=서버: )[^<]+'
done
```

### DB 연결 확인
```bash
# WEB01에서
mysql -h 192.168.56.40 -u wpuser -p'SecurePassword123!' \
  -e "SELECT COUNT(*) FROM information_schema.tables;"
```

---

## 🔐 보안 설정

### SELinux 상태 확인
```bash
getenforce
getsebool -a | grep httpd
```

### Firewall 규칙 확인
```bash
sudo firewall-cmd --list-all
```

---

## 📦 백업

### 설정 파일 백업
```bash
# Ansible 프로젝트 전체 백업
tar -czf ansible-backup-$(date +%Y%m%d).tar.gz \
  ansible-infra-automation/
```

### DB 백업
```bash
# DB-central01에서
mysqldump -u root -p wordpress > wordpress_backup.sql
```

---

## 🆘 긴급 복구

### Apache 중지 시
```bash
sudo systemctl start httpd
sudo systemctl status httpd
```

### MariaDB 중지 시
```bash
sudo systemctl start mariadb
sudo systemctl status mariadb
```

### NFS 중지 시
```bash
sudo systemctl start nfs-server
exportfs -ra
```

---

## 📞 도움말

### Ansible 버전 확인
```bash
ansible --version
```

### 모든 서비스 상태 확인
```bash
ansible all -m shell -a "systemctl status httpd" 2>/dev/null | grep -E "(WEB|Active)"
ansible database -m shell -a "systemctl status mariadb"
ansible storage -m shell -a "systemctl status nfs-server"
ansible lb -m shell -a "systemctl status haproxy"
```

---

## 🎯 다음 작업 시 체크리스트

- [ ] VM 5대 모두 실행 중
- [ ] 네트워크 설정 확인 (NAT + Host-Only)
- [ ] SSH 키 인증 설정됨
- [ ] LB01에서 다른 서버 ping 가능
- [ ] Ansible 설치됨
- [ ] 프로젝트 파일 준비됨

---

**작업 날짜**: _____년 __월 __일  
**작업자**: _________________  
**메모**: 
