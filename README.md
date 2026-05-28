# MariaDB 10.6 Single Instance 설치 가이드

Oracle Linux 7.7 환경에서 MariaDB 10.6을 설치하는 가이드입니다.

---

## 환경 구성

| 항목 | 내용 |
|------|------|
| OS | Oracle Linux 7.7 |
| DB 버전 | MariaDB 10.6 (LTS) |
| SELinux | Disabled |
| firewalld | Disabled |
| 가상화 | VirtualBox 7.0.18 |

---

## 설치 순서

| 순서 | 내용 |
|------|------|
| 1 | [MariaDB 10.6 설치](./mariadb-install.md) |

---

## 주요 특징

- MariaDB 공식 Repo 등록 후 최신 버전 설치
- OS 내장 구버전 MariaDB 제거 후 신규 설치
- mysql_secure_installation 보안 초기화
- 각 설치 단계별 이유 설명 포함
- Oracle vs MariaDB 주요 차이점 정리
- 트러블슈팅 포함
