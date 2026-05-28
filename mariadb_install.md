# MariaDB 10.6 Single Instance 설치 가이드


## 환경 구성

| 항목 | 내용 |
|------|------|
| OS | Oracle Linux 7.7 |
| DB | MariaDB 10.6 (LTS) |
| SELinux | Disabled |
| firewalld | Disabled |
| 작업 계정 | root |

---

## MariaDB란?

MySQL을 개발한 개발자가 Oracle의 MySQL 인수 이후 오픈소스로 만든 RDBMS다.
MySQL과 구문이 거의 동일하며 Oracle Linux에는 기본적으로 MariaDB가 내장되어 있다.

---

## 1. 사전 확인

### 기존 MariaDB / MySQL 설치 여부 확인

> 이미 설치된 패키지가 있으면 충돌이 발생할 수 있어서 사전에 확인한다.

```bash
rpm -qa | grep -i mariadb
rpm -qa | grep -i mysql
```

### 3306 포트 사용 여부 확인

> MariaDB 기본 포트는 3306이다. 이미 사용 중이면 설치 후 서비스가 시작되지 않는다.

```bash
ss -tlnp | grep 3306
```

---

## 2. 기존 MariaDB 제거 (설치된 경우)

> OS 기본 내장된 구버전 MariaDB가 있으면 새 버전 설치 전에 제거해야 한다.
> 제거하지 않으면 패키지 충돌로 설치가 실패한다.

```bash
yum remove mariadb mariadb-server mariadb-libs -y
```

설정 파일 잔여 여부 확인 및 백업:

```bash
ls -la /etc/my.cnf /etc/my.cnf.d/ 2>/dev/null

[ -f /etc/my.cnf ] && cp /etc/my.cnf /etc/my.cnf.mariadb.bak || echo "/etc/my.cnf 없음"
```

> `&&`: 앞 명령 성공 시 뒤 명령 실행
> `||`: 앞 명령 실패 시 뒤 명령 실행

---

## 3. MariaDB 공식 Repo 등록

> Oracle Linux 기본 yum repo에는 구버전(5.x)만 있다.
> 최신 버전(10.x)을 설치하려면 MariaDB 공식 repo를 별도로 등록해야 한다.

```bash
vi /etc/yum.repos.d/mariadb.repo
```

아래 내용을 입력한다:

```
[mariadb]
name = MariaDB
baseurl = https://downloads.mariadb.com/MariaDB/mariadb-10.6/yum/rhel7-amd64
gpgkey = https://downloads.mariadb.com/MariaDB/RPM-GPG-KEY-MariaDB
gpgcheck = 1
enabled = 1
```

> `baseurl`의 `mariadb-10.6` 부분이 설치할 버전을 지정한다. 10.6은 LTS(Long Term Support) 버전으로 장기 지원이 보장된다.

repo 등록 확인:

```bash
yum repolist | grep mariadb
```

---

## 4. MariaDB 설치

> yum으로 설치하면 서버, 클라이언트, 공통 라이브러리가 한 번에 설치된다.

```bash
yum install MariaDB-server MariaDB-client -y
```

설치 완료 후 버전 확인:

```bash
rpm -qa | grep MariaDB
mysqld --version
```

---

## 5. 서비스 시작 및 자동 시작 등록

> `systemctl start`: 현재 즉시 시작
> `systemctl enable`: 서버 재부팅 시 자동으로 시작되도록 등록
> 둘 다 설정해야 재부팅 후에도 MariaDB가 유지된다.

```bash
systemctl start mariadb
systemctl enable mariadb
systemctl status mariadb
```

`Active: active (running)` 이면 정상이다.

---

## 6. 보안 초기화 (mysql_secure_installation)

> 최초 설치 시 root 비밀번호가 없고 익명 유저, 원격 root 접속 등 보안에 취약한 상태다.
> mysql_secure_installation으로 기본 보안 설정을 한 번에 처리한다.

```bash
mysql_secure_installation
```

| 질문 | 권장 답변 | 이유 |
|------|---------|------|
| Enter current password for root | 그냥 Enter | 초기 비밀번호 없음 |
| Set root password? | Y | root 비밀번호 설정 |
| New password | oracle | 실습 환경 편의상 간단하게 |
| Remove anonymous users? | Y | 익명 접속 차단 |
| Disallow root login remotely? | Y | root 원격 접속 차단 (실습이면 N 가능) |
| Remove test database? | Y | 불필요한 test DB 제거 |
| Reload privilege tables? | Y | 변경 사항 즉시 적용 |

---

## 7. root 접속 확인

```bash
mysql -u root -p
```

```sql
select version();
show databases;
exit;
```

---

## 8. 데이터베이스 및 유저 생성

> MariaDB는 Oracle의 Schema 개념이 Database로 구분된다.
> 유저는 '계정명'@'접속허용IP' 형식으로 접속 위치를 함께 지정한다.

```bash
mysql -u root -p
```

```sql
-- 데이터베이스 생성
CREATE DATABASE orcl CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 유저 생성 (로컬 접속용)
CREATE USER scott@'localhost' IDENTIFIED BY 'tiger';

-- 유저 생성 (원격 접속용 - 필요 시)
CREATE USER scott@'%' IDENTIFIED BY 'tiger';

-- 권한 부여
GRANT ALL PRIVILEGES ON orcl.* TO scott@'localhost';
GRANT ALL PRIVILEGES ON orcl.* TO scott@'%';

-- 권한 적용
FLUSH PRIVILEGES;

SHOW DATABASES;
EXIT;
```

> `FLUSH PRIVILEGES`: 권한 테이블을 메모리에 다시 로드한다. 이 명령 없이는 변경 사항이 즉시 반영되지 않을 수 있다.

---

## 9. scott 유저 접속 및 테이블 생성

```bash
mysql -u scott -p orcl
```

```sql
-- emp, dept 테이블 생성
CREATE TABLE dept (
  deptno INT(2),
  dname  VARCHAR(20),
  loc    VARCHAR(20)
);

CREATE TABLE emp (
  empno    INT(4) NOT NULL,
  ename    VARCHAR(10),
  job      VARCHAR(9),
  mgr      INT(4),
  hiredate DATE,
  sal      INT(7),
  comm     INT(7),
  deptno   INT(4)
);

-- 데이터 입력
INSERT INTO dept VALUES (10,'ACCOUNTING','NEW YORK');
INSERT INTO dept VALUES (20,'RESEARCH','DALLAS');
INSERT INTO dept VALUES (30,'SALES','CHICAGO');
INSERT INTO dept VALUES (40,'OPERATIONS','BOSTON');

INSERT INTO emp VALUES (7839,'KING','PRESIDENT',NULL,'1981-11-17',5000,NULL,10);
INSERT INTO emp VALUES (7698,'BLAKE','MANAGER',7839,'1981-05-01',2850,NULL,30);
INSERT INTO emp VALUES (7782,'CLARK','MANAGER',7839,'1981-05-09',2450,NULL,10);
INSERT INTO emp VALUES (7566,'JONES','MANAGER',7839,'1981-04-01',2975,NULL,20);
INSERT INTO emp VALUES (7654,'MARTIN','SALESMAN',7698,'1981-09-10',1250,1400,30);
INSERT INTO emp VALUES (7499,'ALLEN','SALESMAN',7698,'1981-02-11',1600,300,30);
INSERT INTO emp VALUES (7844,'TURNER','SALESMAN',7698,'1981-08-21',1500,0,30);
INSERT INTO emp VALUES (7900,'JAMES','CLERK',7698,'1981-12-11',950,NULL,30);
INSERT INTO emp VALUES (7521,'WARD','SALESMAN',7698,'1981-02-23',1250,500,30);
INSERT INTO emp VALUES (7902,'FORD','ANALYST',7566,'1981-12-11',3000,NULL,20);
INSERT INTO emp VALUES (7369,'SMITH','CLERK',7902,'1980-12-09',800,NULL,20);
INSERT INTO emp VALUES (7788,'SCOTT','ANALYST',7566,'1982-12-22',3000,NULL,20);
INSERT INTO emp VALUES (7876,'ADAMS','CLERK',7788,'1983-01-15',1100,NULL,20);
INSERT INTO emp VALUES (7934,'MILLER','CLERK',7782,'1982-01-11',1300,NULL,10);

COMMIT;

-- 확인
SELECT COUNT(*) FROM emp;
SELECT COUNT(*) FROM dept;

EXIT;
```

---

## 10. 최종 확인

```bash
systemctl status mariadb
ss -tlnp | grep 3306
mysql -u root -p -e "SELECT user, host FROM mysql.user;"
```

---

## 11. Oracle vs MariaDB 주요 차이점

| 항목 | Oracle | MariaDB |
|------|--------|---------|
| Schema 개념 | User = Schema | Database = Schema |
| 접속 방식 | sqlplus user/pass@sid | mysql -u user -p db |
| 자동 커밋 | 기본 OFF | 기본 ON (autocommit=1) |
| 날짜 형식 | DATE (날짜+시간) | DATE (날짜만) / DATETIME (날짜+시간) |
| 시퀀스 | CREATE SEQUENCE | AUTO_INCREMENT |
| ROWNUM | ROWNUM | LIMIT |
| NVL | NVL(a,b) | IFNULL(a,b) |
| 문자열 결합 | `||` | CONCAT() |

> MariaDB는 기본적으로 autocommit=1이라 COMMIT 없이도 DML이 즉시 반영된다.

---

## 12. 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| 서비스 시작 실패 | 포트 3306 이미 사용 중 | `ss -tlnp | grep 3306` 으로 확인 후 해당 프로세스 종료 |
| Access denied (root) | 비밀번호 오류 | 6번 단계 재수행 |
| 패키지 충돌 | 구버전 MariaDB 잔재 | 2번 단계 제거 후 재설치 |
| repo 접근 불가 | 인터넷 연결 문제 | `ping 8.8.8.8` 으로 확인 |

---

## 참고 자료

- [MariaDB 공식 설치 문서](https://mariadb.com/kb/en/yum/)
- [MariaDB vs MySQL 차이점](https://mariadb.com/kb/en/mariadb-vs-mysql-compatibility/)
