# 🔧 Ansible 기반 고가용성 웹 인프라 자동화

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Ansible](https://img.shields.io/badge/Ansible-2.9+-red.svg)](https://www.ansible.com/)
[![Rocky Linux](https://img.shields.io/badge/Rocky%20Linux-9-green.svg)](https://rockylinux.org/)

> Ansible을 활용한 고가용성(HA) 웹 인프라 자동 구축 및 배포 시스템

---

이 프로젝트는 실제 작업했던 Ansible 인프라 자동화 프로젝트입니다.

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
    ├── haproxy.cfg.j2
    ├── named.conf.j2
    ├── db.example.com.j2
    ├── web01_site1.conf.j2
    ├── wp01_site2.conf.j2
    ├── wp-config.php.j2
    ├── web01_index.j2
    ├── web02_index.j2
    ├── web02_site1.conf.j2
    ├── web02_site2.conf.j2
    ├── exports.j2
    ├── mariadb-server.cnf.j2
    └── root-my.cnf.j2
```

## 🚀 빠른 시작

### 전체 인프라 구축
```bash
ansible-playbook all_setup.yml
```

### 개별 서버 구축
```bash
# 로드밸런서
ansible-playbook LB01/lb_setup.yml

# 웹 서버
ansible-playbook WEB01/web1_setup.yml
ansible-playbook WEB02/web2_setup.yml

# 스토리지
ansible-playbook Storage01/storage_setup.yml

# 데이터베이스
ansible-playbook DB-central01/db-central01_setup.yml
```

## 📖 상세 문서

자세한 내용은 원본 GitHub README를 참조하세요:
https://github.com/qkrtpdlr/ansible-infra-automation

## 📧 Contact

- Email: rlagudfo1223@gmail.com
- GitHub: https://github.com/qkrtpdlr
