# 2장

## 1. MySQL 설치

### 기본 파일 경로

`MySQL/MySQL Server 8.0/Data/my.ini`

### 디렉터리 구조

- bin: MySQL 서버와 클라이언트 프로그램 그리고 유틸리티를 위한 디렉터리
- include: C/C++ 헤더 파일들이 저장된 디렉터리
- lib: 라이브러리 파일들이 저장된 디렉터리
- share: 다양한 지원 파일들이 저장돼 있으며, 에러 메시지나 샘플 설정파일(my.ini)이 있는 디렉터리

## 2. MySQL 실행

### 종료시 변경 기록을 모두 반영하는 방법(클린 셧다운)

```bash
mysql> SET GLOBAL innodb_fast_shutdown = 0;
linux> systemctl stop mysqld.service

## 원격 종료
mysql> SET GLOBAL innodb_fast_shutdown = 0;
mysql> SHUTDOWN;

```

클린 셧다운을 적용하면 MySQL서버를 가동할 때 별도의 **트랜잭션 복구과정을 진행하지 않아서 더 빠르게 시작**할 수 있다.

## 3. MySQL 업그레이드

서버 업그레이드 방식은 두가지 방법으로 나뉜다.

1. In Place: 데이터 파일을 그대로 두고 업그레이드
2. Logical: 데이터를 덤프한 후 새로 업그레이드된 서버에 덤프된 데이터를 적재하는 방법

In Place 업그레이드는 여러가지 제약이 있지만, 속도면에서 우월하다.

### In Place 업그레이드 제약사항

- 마이너 패치는 제약이 없다. ex)8.0.1 → 8.0.15
- 메이저 패치는 데이터 파일 변경이 필요해서 직전 버전에서만 가능하다. ex)8.0.1 → 8.1.1
- 메이저 업그레이드를 지원하지 않는 마이너 버전도 있다. ex)5.7.8

### 업그레이드 고려사항

5.7 → 8.0 업그레이드에서 많은 부분이 변경되어 사용할 수 없거나 달라진 기능이 있다.
따라서 8.0 버전으로 업그레이드 전에 변경된 내용을 검토해보는 것을 추천한다.

- 사용자 인증 방식: 8.0버전부턴 Caching SHA-2 방식을 기본으로 사용한다.
- 호환성: 5.7 버전에서 손상된 FRM파일, 호환되지 않는 데이터 타입, 함수를 mysqlcheck 유틸리티를 통해 확인하자.
- 외래키 이름 길이: 8.0부터 외래키 길이는 64글자로 제한된다.
- 인덱스 힌트: 인덱스 힌트가 새롭게 추가되어 8.0에선 성능 저하를 유발할 수 있으니 성능테스트를 해보자.
- Group By 정렬 옵션: Group By절에 ASC, DESC 또는 field ASC등을 사용하고 있다면 제거 또는 변경하자.
- 파티션을 위한 공용 테이블스페이스: 8.0부터 각 테이블스페이스를 공용 테이블스페이스에 저장할 수 없다. 공용 테이블스페이스에 저장되어 있다면 ALTER TABLE REORGANIZE 명령을 통해 변경하자.

* FRM파일: 테이블 구조가 저장된 파일

* Index Hint: http://minsql.com/mysql8/B-2-D-optimizerHint/

* Group By: 8.0 버전부터 Group By절에서 정렬을 할 수 없다.

### 업그레이드 내용

- 데이터 딕셔너리: 
이전 버전은 FRM파일로 별도로 보관되었는데, InnoDB시스템 테이블(트랜잭션 지원)로 저장한다.
버전 호환성을 위해 서버 버전의 정보도 함께 보관된다.
- 서버: 시스템 DB(perfomance_schema, information_schema, mysql db)구조가 변경되었다.

* 8.0.16 이전 버전 업그레이드를 할때 mysql_upgrade 유틸리티를 실행해 딕셔터리와 서버 업그레이드를 진행하자. 8.0.16 부턴 자동 진행

## 4. 서버 설정

MySQL은 보통 하나의 설정 파일을 사용한다. (리눅스-my.cnf, 윈도우-my.ini)

MySQL서버는 시작될때 설정파일을 참조하는데, 참조하는 순서는 `--verbose --help`명령어로 볼 수 있다.

### 설정 파일 탐색 순서

1. /etc/my.cnf
2. etc/mysql/my.cnf 
3. /usr/etc/my.cnf (컴파일 내장 경로)
4. ~/.my.cnf

### 설정 파일 구성

설정파일은 다음과 같이 설정 그룹과 내용으로 구성된다. 각 그룹은 파일을 공유하지만 서로 무관하게 적용된다.

```bash
[설정 그룹]
설정 내용

[mysql]
default-character-set =utf8mb4
socket =/usr/local/mysql/tmp/mysql.sock port =3304
```

### 시스템 변수

서버를 기동하면서 설정파일의 내용을 읽어 메모리나 작동방식을 초기화하고, 접속된 사용자를 제어하기 위해 저장한 값을 시스템 변수라고 한다. 글로벌변수와 세션변수로 나뉘고 `SHOW VARIABLES` 명령을 통해 확인할 수 있다.

![image](https://github.com/Deep-Dive-Study/real-my-sql/assets/85796588/ee100130-33db-414a-a34b-a7ed11b135c0)
명령어를 입력해보면 굉장히 많은데, 변수에 대한 정보는 mysql 페이지에 정리되어 있다.

**변수에 대한 설명**

https://dev.mysql.com/doc/refman/8.0/en/server-system-variable-reference.html

- Cmd-Line: 서버의 명령행 인자로 설정되는지 여부
- Option file: 설정 파일인 my.cnf로 제어가 가능한지 여부
- System Var: 시스템 변수 여부
- Var Scope: 시스템 변수의 적용 범위(Global, Both, Session)
- Dynamic: 시스템 변수의 동적 유무

**Var Scope**

- Global: 
MySQL서버에 전체적으로 영향을 미치는 변수.
주로 서버 자체에 관련된 설정을 가진다. ex)버퍼 풀, 캐시 크기
- Session:
MySQL서버에 접속할 때 부여하는 옵션으로 클라이언트마다 다른값으로 변경할 수 있다. ex)autocommit
- Both:
Session 변수중 my.cnf에 명시해 초기화할 수 있는 변수.
명시된 변수는 클라이언트의 커넥션이 생성될 때 기본값으로 사용된다.

**Dynamic**

일반적으로 글로벌 변수는 서버 가동중에 변경할 수 없는 

변수를 변경하는 방법은 두가지로 나뉜다.

1. my.cnf 파일을 수정하는 방법
이 경우 MySQL 서버를 재시작 하기 전까지 반영되지 않는다.
2. 서버 메모리에 있는 변수를 변경하는 방법 (SET GLOBAL @@@@@ = ####;
이 경우 현재 가동중인 MySQL 인스턴스에만 유효하다. (디스크에도 반영하려면 SET PERSIST 명령을 통해 가능)

**SET PERSIST**

현재 실행중인 인스턴스와, my.cnf파일의 내용을 모두 반영할 때 사용된다. 이때 JSON포 형식으로 mysqld-auto.cnf파일이 생성되고, 기록이 관리된다.(버전, 내용, 시간, 유저, 호스트등)

`RESET PERSIST` 명령을 통해 mysqld-auto.cnf 파일내용을 삭제 수 있다.
