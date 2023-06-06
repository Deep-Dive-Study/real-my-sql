# 02. 설치와 설정
- 가능하다면 리눅스의 RPM, 운영체제별 인스톨러를 이용하여 설치하는 것을 권장

## 2.1 MySQL 서버 설치

### 2.1.1 버전과 에디션

- 다른 제약 사항이 없다면 가능한 최신 버전을 설치하는 것이 좋다
	- 15~20번 이상 패치된 버전을 선택하는게 안정적, 갓 출시된 메이저는 약간 위험

- MySQL 서버의 사용화 전략은 핵심 기능의 차이가 없는 `오픈 코어 모델`
	- 즉, 에디션 간에 특정 부가 기능만 차이남
	- 플러그인을 잘 활용하면 부족한 부분 보완 가능

- 엔터프라이즈 에디션에서만 지원하는 부가 기능
    - Thread Pool
    - Enterprise Audit
    - 등등..

### 2.1.2 MySQL 설치

- 자세한 설치 과정은 생략

#### 알아 두면 좋은 내용

- MySQL 서버가 설치된 기본 디렉토리는 **`/usr/local/mysql`** <br>

**밑의 하위 디렉터리는 중요하기 때문에 삭제하면 안된다.** <br>

- `bin` : MySQL 서버와 클라이언트 프로그램. 유틸리티를 위한 디렉터리 
- `data` : 로그 파일 및 데이터 파일들이 저장되는 디렉터리
- `include` : C/C++ 헤더 파일들이 저장된 디렉터리
- `lib` : 라이브러리 파일이 저장된 디렉터리
- `share` : 다양한 지원 파일들(에러메시지, 샘플 설정 파일)이 있는 디렉터리


## 2.2 MySQL 서버의 시작과 종료

### 2.2.1 설정 파일 및 데이터 파일 준비

- MySQL 서버가 설치되면 `/etc/my.cnf` 설정이 준비된다.
  - 해당 설정 파일에는 기본적인 설정만 존재
  - 초기 데이터 파일(시스템 테이블, 등)과 트랜잭션 로그 파일이 준비가 필요함

```shell
# 필요한 초기 데이터 파일과 로그 파일 생성
# 비밀번호가 없는 관리자 계정(root) 유저 생성
linux> mysqld --defaults-file=/etc/my.cnf --initialize-insecure

# 임시 비밀번호를 생성, 에러 로그 파일(/var/log/mysqld.log)에서 확인 가능
linux> mysqld --defaults-file=/etc/my.cnf --initialize
```


### 2.2.2 시작과 종료

- 유닉스 계열에서 설치시 자동으로 service 등록 (윈도우는 선택사항으로 등록 가능)
- systemctl 유틸리티를 통해 실행 및 종료 가능

```shell
# 시작|상태|종료
linux> systemctl start|status|stop mysqld
```

- mysqld_safe 스크립트를 이용해서 MySQL 서버를 시작 및 종료 가능
    - mysqld_safe는 오류가 발생할 때 서버를 다시 시작하고 몇가지 안전 기능을 추가 

- 원격으로 종료하려면 `SHUTDOWN` 명령 실행(권한 필요)

- **`클린 셧다운`** : 서버 종료될 때 모든 트랜잭션 커밋에 대해 파일에 반영하고 종료하는 옵션
	- mysql 기동할 때 트랜잭션 복구 과정을 불필요하게 함

```shell
mysql> SET GLOBAL innodb_fast_shutdown=0;
```


### 2.2.3 서버 연결 테스트

- 기본 클라이언트 프로그램(mysql)를 통해 접속

#### 직접 접속

- host가 localhost인 경우 `IPC 통신`(UDS)
- 127.0.0.1(루프백)로 하면 `TCP/IP 통신`
- host를 명시하지 않을 경우 localhost
	- 설정 파일에서 소켓 파일 위치를 읽어서 사용
	- 삭제하면 재시작하지 않으면 만들 수 없기 때문에 삭제하지 않도록 주의

- 리눅스에서 클라이언트 프로그램 사용시 프로그램의 경로를 PATH 환경 변수에 등록

```shell
linux> mysql -uroot -p --host=localhost --socket=/tmp/mysql.sock
linux> mysql -uroot -p --host=127.0.0.1 --port=3306
linux> mysql -uroot -p
```

#### 원격 접속

- 원격으로 접속하고자 하면 TCP/IP 통신 사용
- 원격에서 서버를 직접 로그인하지 않고 접속 가능 여부만 확인할 경우 telnet 사용

```shell
linux> telnet IP 3306
linux> nc IP 3306
```

## 2.3 MySQL 서버 업그레이드

- 서버의 업그레이드 방법에는 `In-Place Upgrade` 와 `Logical Upgrade` 두가지의 방법이 있다.

- **`In-Place`** : 데이터 파일은 그래도 두고 서버의 버전만 변경
	- 여러 제약 사항이 있지만 적은 시간 소요

- **`Logical`** : 데이터를 SQL 문장이나 텍스트 파일로 덤프한 후, 업그레이드된 새로운 서버에 적재
	- 제약은 적지만 많은 시간 소요

### 2.3.1 In-Place Upgrade

- **`마이너 버전`** :  보통 여러 버전을 건너뛰어서 업데이트 가능
	- 8.0.16 -> 8.0.21

- **`메이저 버전`** : 반드시 직전 버전에서만 가능
	- 5.6 -> 5.7 -> 8.0
	- 데이터 파일의 변경이 존재

#### 주의 사항
* 메이저 버전 업그레이드가 특정 마이너 버전에서만 가능한 경우가 있다. 따라서, GA 버전 이상을 권장

### 2.3.2 MySQL 8.0 업그레이드 시 고려 사항

1. **사용자 인증 방식 변경** : Caching SHA-2 Authentication 인증 방식이 기본 인증 방식으로 변경

2. **MySQL 8.0과의 호환성 체크** : 손상된 FRM파일이나 호환되지 않는 데이터 파일이 있는지 mysqlcheck 유틸 활용 권장

3. **외래키 이름의 길이** : 외래키의 이름이 64글자로 제한

4. **인덱스 힌트** : 인덱스 힌트가 성능 저하의 원인이 될 수 있다. 성능테스트 해볼 것을 권장

5. **Group By에 사용된 정렬 옵션** :  Group By에 정렬 옵션 존재시 제거 혹은 다른 방식으로 변경할 것

6. **파티션을 위한 공용 테이블 스페이스** : 파티션의 각 테이블스페이스를 공용 테이블스페이스에 저장할 수 없음, ALTER TABLE REORGANIZE 명령을 통해 변경할 것


#### 위에서 언급한 내용들을 확인해 볼 수 있는 명령어

```shell
# mysqlcheck 유틸리티 실행 방법
linux> mysqlcheck -u root -p --all database --check-upgrade
```

```sql
-- 외래키 이름의 길이 체크
SELECT TABLE_SCHEMA, TABLE_NAME
FROM information_schema.TABLES 
WHERE TABLE NAME IN 
	(SELECT LEFT(SUBSTR(ID, INSTR(ID, '/') + 1),
                 INSTR(SUBSTR(ID, INSTR(ID, '/') + 1), '_ibfk_') - 1)
	 FROM information_schema.INNODB_SYS_FOREIGN 
     WHERE LENGTH(SUBSTR(ID, INSTR(ID, '/') + 1)) > 64);

-- 공용 테이블스페이스에 저장된 파티션이 있는지 체크
SELECT DISTINCT NAME, SPACE, SPACE_TYPE
FROM information_schema.INNODB_SYS_TABLES
WHERE NAME LIKE '%#P#%' AND SPACE_TYPE NOT LIKE '%Single%'
```


### 2.3.3 MySQL 8.0 업그레이드

- MySQL 8.0 부터는 '시스템 테이블 정보' '데이터 딕셔너리 정보' 의 포맷이 변경 

- **`데이터 딕셔너리 업그레이드`** :  5.7 버전까지는 FRM 확장자 파일로 별도로 보관했지만, 8.0 부터 트랜잭션이 지원되는 InnoDB 테이블로 저장된다. 8.0.16 이전 버전으로 업그레이드를 할때는 mysql_upgrade 유틸리티를 따로 실행해주어야 한다.

- **`서버 업그레이드`** : 시스템 데이터베이스(perfomance_schema, information_schema, mysql db)의 테이블 구조를 버전에 맞게 변경

#### 업그레이드 절차

1. MySQL 셧다운
2. MySQL 이전 버전(5.7) 삭제
3. MySQL 8.0 설치
4. MySQL 8.0 실행 (실행시 업그레이드 절차 수행)

## 2.4 서버 설정

- 설정 파일로 유닉스 계열은 `my.cnf`(윈도우는 my.ini) 단 하나의 설정 파일을 사용
	- 경로는 지정된 후보를 순차적으로 탐색하며 처음 발견된 파일을 사용

```shell
# my.cnf 경로를 확인할 수 있음 (클라이언트에서 실행)
linux> mysql --help
```

#### 후보 경로 (예제)
1. `/etc/my.cnf`
2. `/etc/mysql/my.cnf`
3. `/usr/etc/my.cnf`
4. `~/ .my.cnf`

- mysql 서버를 중복으로 실행할 경우 경로가 충돌할 수 있음
	- 별도 디렉터리에 설정 파일을 준비


### 2.4.1 설정 파일의 구성

- 하나의 `my.cnf` 파일에 여러 개의 설정 그룹을 담아 사용.
	- 일반적으로 실행 프로그램 이름을 그룹명으로 활용
	- 각 그룹은 같은 파일을 공유하지만 서로 무관하게 적용 


### 2.4.2 MySQL 시스템 변수의 특징

- MySQL 서버는 기동시에 설정 파일의 내용을 읽어 메모리나 작동 방식을 초기화하고 사용자를 제어하기 위한 시스템 변수를 별도로 저장한다.

- 변수는 `글로벌 변수`와 `세션 변수`로 나뉘어진다.

```shell
# 글로벌 변수 확인
mysql> SHOW GLOBAL VARIABLES;
```

#### 시스템 변수의 5가지 속성

1. `Cmd-Line` : 명령행 인자로 설정될 수 있는지 여부
2. `Option file` : 설정 파일로 제어할 수 있는지 여부
3. `System Var` : 시스템 변수인지 여부 
4. `Var Scope` : 시스템 변수의 적용 범위
5. `Dynamic` : 동적 변수 인지 여부

### 2.4.3 글로벌 변수와 세션 변수

- 시스템 변수는 적용 범위에 따라 글로벌과 세션으로 나뉜다.
- 세션과 글로벌 둘 다 될 수 있다. (Both)

**`글로벌 변수`** : 서버 인스턴스에서 전체적으로 영향을 미치는 시스템 변수. 일반적으로 서버 자체 설정 ex) innodb_buffer_pool_size

**`세션 변수`** : 개별 커넥션 단위로 다른 값으로 변경할 수 있는 변수 ex) autocommit

- 둘다 사용되는 경우는 글로벌 변수가 세션 변수의 기본 값으로 활용된다.

```shell
# 검색을 틍해 변수 확인
mysql> SHOW GLOBAL VARIABLES LIKE '%max%'
...

# 수정
mysql> SET GLOBAL max_connections=500;
```

### 2.4.4 정적 변수와 동적 변수

- 기동중인 상태에서 변경이 가능한지에 따라 동적 변수와 정적 변수로도 나눌 수 있다.
- 설정 파일을 변경하는 경우(정적)와 서버의 메모리에 있는 변수를 변경하는 경우(동적)
- 동적 변수는 변경하더라도 영구 적용하기 위해서는 설정 파일에는 따로 변경해야한다.
- 변수의 범위가 "Both"인 경우 변경하더라도 이미 존재하는 커넥션에는 적용되지 않는다.

### 2.4.5 SET PERSIST

- 동적 변수의 경우 변경 후 현재 실행중인 서버에는 적용되지만 재시작시에는 원래의 값으로 되돌아 간다. 
- 이러한 문제를 보완하기 위해 `SET PERSIST` 명령이 도입 되었다.

- 시스템 변수 변경시 즉시 적용함과 동시에 별도의 설정 파일에도 변경된 내용을 추가 (단, 세션 변수에는 적용되 않음)
- 즉시 반영하지 않고, 설정 파일의 내용만 변경하고 싶다면 `SET PERSIST ONLY` 사용
- 일반 설정 파일(`my.cnf`)가 아닌 별도의 설정 파일(`mysqld-auto.cnf`)에 변경 내용을 추가로 기록
- 변수를 변경하면 언제 누구에 의해 변경되었는지 정보도 함께 기록됨
- 추가된 시스템 변수의 내용을 삭제하려고 하면 직접 건드리기보다 `RESET` 명령을 통해 사용하는 것이 안전

```shell
# 특정 시스템 변수 삭제
mysql> RESET PERSIST max_connections;ß
mysql> RESET PERSIST IF EXISTS max_connections;

# mysqld-auth.cnf 파일의 모든 시스템 변수 삭제
mysql> RESET PERSIST;
```

### 2.4.6 my.cnf  파일 

- MySQL 서버를 제대로 사용하려면 시스템 변수에 대한 이해가 상당히 필요함
- 실행 중인 서버의 하드웨어 특성과 서비스의 특성에 따라 성능에 영향을 줄 수 있다.
- 자세한 내용은 앞으로 추가될 장에서 확인할 수 있다.
--- 

**질문 내용**

1. 어떤 상황에 Logical Upgrade를 사용하는 것이 좋을까?
2. MySQL의 시스템 변수의 특징과 분류를 설명해주세요
3. SET PERSIST 동작에 대해 설명해주세요

**새롭게 알게된 내용**

- Localhost 와 127.0.0.1 접속 방법의 차이

**궁금해서 찾아본 내용**

- 왜 mysql 버전은 5에서 8로 넘어갈까?
	- 오라클에 인수되기전 Sun사의 6.0alpha 버전이 있었는데 폐기되었고, MySQL Cluster 제품이 7.0 버전으로 사용되고 있었으므로 혼동을 막기 위해 MySQL 8.0으로 출시하였다고 한다.
	- https://opensource.com/article/17/2/mysql-8-coming

- mysql은 어떤 언어로 개발되었을까?
	- C, C++로 개발됨
	- https://ko.wikipedia.org/wiki/MySQL
