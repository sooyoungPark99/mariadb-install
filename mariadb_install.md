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

<img width="1100" height="253" alt="image" src="https://github.com/user-attachments/assets/524cff78-cc2e-43d9-b5ec-a5edf666b896" />

---

## 2. 기존 MySQL/MariaDB 제거 (설치된 경우/root에서)

> 새 버전 설치 전에 기존 패키지를 제거해야 한다.
> 제거하지 않으면 패키지 충돌로 설치가 실패한다.

```bash
# MySQL 설치 여부 확인
rpm -qa | grep -i mysql

# MySQL이 있는 경우 서비스 중지 후 제거
systemctl stop mysqld

yum remove mysql-community-server mysql-community-client \
  mysql-community-common mysql-community-libs \
  mysql-community-client-plugins \
  mysql-community-icu-data-files \
  mysql80-community-release -y
```
<img width="1100" height="253" alt="스크린샷 2026-05-28 154026" src="https://github.com/user-attachments/assets/7f2a1297-171d-4f72-8304-5cb5461a4877" />

```bash
# MariaDB 설치 여부 확인
rpm -qa | grep -i mariadb

# MariaDB가 있는 경우 제거
yum remove mariadb mariadb-server mariadb-libs -y

# 제거 완료 확인 (결과 없으면 정상)
rpm -qa | grep -i mysql
rpm -qa | grep -i mariadb
```
<img width="1097" height="76" alt="image" src="https://github.com/user-attachments/assets/7e74c5d6-c7d1-402e-a6b1-d3e37e1d387c" />

설정 파일 잔여 여부 확인 및 백업:

```bash
ls -la /etc/my.cnf /etc/my.cnf.d/ 2>/dev/null
[ -f /etc/my.cnf ] && cp /etc/my.cnf /etc/my.cnf.mariadb.bak || echo "/etc/my.cnf 없음"
```

`&&`: 앞 명령 성공 시 뒤 명령 실행

`||`: 앞 명령 실패 시 뒤 명령 실행

<img width="1090" height="78" alt="image" src="https://github.com/user-attachments/assets/ac48a275-5c21-477b-b3e7-9c206db6663f" />

---

## 3. MariaDB 공식 Repo 등록

> Oracle Linux 기본 yum repo에는 구버전(5.x)만 있다.
> 최신 버전(10.6)을 설치하려면 MariaDB 공식 repo를 별도로 등록해야 한다.

```bash
vi /etc/yum.repos.d/mariadb.repo
```

아래 내용을 입력한다:

```
[mariadb]
name = MariaDB
baseurl = https://mirror.mariadb.org/yum/10.6/rhel7-amd64
gpgkey = https://downloads.mariadb.com/MariaDB/RPM-GPG-KEY-MariaDB
gpgcheck = 1
enabled = 1
```
> `baseurl`의 `mariadb-10.6` 부분이 설치할 버전을 지정한다. 10.6은 LTS(Long Term Support) 버전으로 장기 지원이 보장된다.

repo 등록 확인:

```bash
yum repolist | grep mariadb

-- > 32개 패키지가 확인됐다.

```
<img width="547" height="47" alt="image" src="https://github.com/user-attachments/assets/49cef62e-d8b1-460d-b0cb-3c3d16f54483" />

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
<img width="1101" height="183" alt="image" src="https://github.com/user-attachments/assets/3a3b402b-c3f5-49eb-a21a-50371177583a" />

---

## 5. 서비스 시작 및 자동 시작 등록

> ⚠️ MySQL을 사용했던 서버라면 /var/lib/mysql 디렉토리를 정리해야 한다.

```bash
# MySQL 데이터 디렉토리 백업
mv /var/lib/mysql /var/lib/mysql_bak

# MariaDB 데이터 디렉토리 생성 및 초기화
mkdir -p /var/lib/mysql
chown mysql:mysql /var/lib/mysql
chmod 755 /var/lib/mysql
mysql_install_db --user=mysql --datadir=/var/lib/mysql
```

- `systemctl start`: 현재 즉시 시작
- `systemctl enable`: 서버 재부팅 시 자동으로 시작되도록 등록

둘 다 설정해야 재부팅 후에도 MariaDB가 유지된다.

```bash
systemctl start mariadb
systemctl enable mariadb
systemctl status mariadb
```
<img width="700" height="253" alt="image" src="https://github.com/user-attachments/assets/d86de56e-9a3e-48ba-ab7d-be7f248ab88e" />

`Active: active (running)` 이면 정상이다.

---

## 6. 보안 초기화 (mysql_secure_installation)

> 최초 설치 시 root 비밀번호가 없고 익명 유저, 원격 root 접속 등 보안에 취약한 상태다.
> mysql_secure_installation으로 기본 보안 설정을 한 번에 처리한다.

```bash
[root@ora-19c ~]# mariadb-secure-installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user. If you've just installed MariaDB, and
haven't set the root password yet, you should just press enter here.

Enter current password for root (enter for none): 그냥 엔터
OK, successfully used password, moving on...

Setting the root password or using the unix_socket ensures that nobody
can log into the MariaDB root user without the proper authorisation.

You already have your root account protected, so you can safely answer 'n'.

Switch to unix_socket authentication [Y/n] n
 ... skipping.

You already have your root account protected, so you can safely answer 'n'.

Change the root password? [Y/n] Y
New password:
Re-enter new password:
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] Y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] Y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] Y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!

```

| 질문 | 권장 답변 | 이유 |
|------|---------|------|
| Enter current password for root | 그냥 Enter | 초기 비밀번호 없음 |
| Switch to unix_socket authentication? | n | 비밀번호 방식 유지 |
| Change the root password? | Y | root 비밀번호 설정 |
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
<img width="699" height="422" alt="image" src="https://github.com/user-attachments/assets/eb1f36e8-0943-486e-87ee-97e70d146d67" />

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
<img width="1107" height="311" alt="image" src="https://github.com/user-attachments/assets/689ebc14-c596-4bc5-bb03-2e78c9e341f6" />

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
<img width="1088" height="258" alt="image" src="https://github.com/user-attachments/assets/01d20b91-8ff5-45d7-b130-439dd8458595" />

#### scott@'%' 유저를 삭제하고 싶다면

```sql
DROP USER 'scott'@'%';
FLUSH PRIVILEGES;
```
> 위 상황같이 **DROP USER, GRANT, REVOKE** 같은 공식 권한 명령어를 쓸 때는 FLUSH PRIVILEGES가 불필요하지만, DBA들은 안전하게 습관적으로 붙이는 경우가 많다.

```sql
-- 이렇게 직접 테이블 수정하면 FLUSH PRIVILEGES 필요
DELETE FROM mysql.user WHERE user='scott';
FLUSH PRIVILEGES;  -- ← 이때는 반드시 필요
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
| InnoDB: MySQL-8.0 tablespace 오류 | MySQL 데이터 파일 잔재 | /var/lib/mysql 백업 후 삭제, mysql_install_db 재실행 |
| Can't lock aria control file 오류 | mariadbd 프로세스 잔재 | pkill -9 mariadbd 후 재시도 |



#### 참고 자료

- [MariaDB 공식 설치 문서](https://mariadb.com/kb/en/yum/)
- [MariaDB vs MySQL 차이점](https://mariadb.com/kb/en/mariadb-vs-mysql-compatibility/)
