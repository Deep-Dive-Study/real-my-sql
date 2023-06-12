# 3장

### 3.1 사용자 식별

MySQL은 다른 RDB와 다르게 사용자@IP로 확인한다. 이때 IP까지 맞아야 접속할 수 있다.

```bash
'user_id'@'127.0.0.1'
'user_id'@'%' (전체 ip를 뜻함)
```

중첩된 계정을 생성할때 주의하자. 위같은 상황에선 더 좁은 범위부터 먼저 선택한다.

### 3.2 사용자 계정 관리

**시스템 계정**

MySQL 서버 내부의 백그라운드 스레드와 무관하며, 일반 계정처럼 사용자를 위한 계정이다.

일반 계정과 다른 점은 관리자(DBA)를 위한 계정으로 일반 계정을 **관리** 할 수 있다.

여기서 관리란?

- 계정 생성/삭제, 권한 부여/제거
- 다른 세션 또는 쿼리 종료
- 스토어드 프로그램 생성시 DEFINER를 타 사용자로 설정

* 스토어드 프로그램은 데이터베이스 내에서 실행되는 절차적인 로직을 담고 있는 프로그램이다. 
스토어드 프로그램을 생성할 때, 해당 프로그램을 생성한 사용자에 대한 정보를 정의할 수 있는데, 이 정보에는 DEFINER라는 속성이 있다.

* DEFINER는 스토어드 프로그램을 실행할 때 사용될 사용자를 지정하는 역할을 한다. DEFINER가 되면 스토어드 프로그램을 실행할 수 있는 권한이 생긴다.

**일반 계정**

일반 사용자가 사용할 수 있도록 생성된 계정으로 시스템 계정과는 다르게 권한이 적다.

**내장 계정**

MySQL에는 내장된 계정들이 있는데, 앞으로 설명할 3가지는 **삭제하지 않도록 주의**하는 것이 좋다.

`account_lock = Y` 로 설정되어 있어 악의적으로 사용될 걱정은 안해도 된다.

- ‘mysql.sys’@’localhost’ : 기본 내장된 sys 스키마의 객체들의 DEFINER 계정
- ‘mysql.session’@’localhost’ : MySQL 플러그인이 서버로 접근할 때 사용되는 계정
- ‘mysql.infoschema’@’localhost’ : information_schema에 정의된 뷰의 DEFINER 계정

**계정 생성**

지금까지는 GRANT 명령을 통해 권한부여와 동시에 계정 생성이 가능했지만, 8.0부터는 안된다.

계정 생성은 CREATE USER 명령으로만 가능한데, 

```sql
CREATE USER 'user'@’%‘
IDENTIFIED WITH 'mysql_native_password‘ BY 'password' REQUIRENONE
PASSWORD EXPIRE INTERVAL 30 DAY
ACCOUNT UNLOCK
PASSWORD HISTORY DEFAULT
PASSWORD REUSE INTERVAL DEFAULT
PASSWORD REQUIRE CURRENT DEFAULT;
```

<details>
<summary>여러 옵션이 있다.</summary>
<div markdown="1">

- IDENTIFIED WITH
사용자 인증방식과 비밀번호 설정한다. MySQL의 인증 플러그인은 대표적으로 4가지가 있다.
보안성을 생각하면 Caching SHA-2 Pluggable Authenticalion을, 호환성을 생각하면 Native Pluggable Authentication라는 옵션을 생각해봐도 좋다.

  - Native Pluggable Authentication
  MySQL 5.7 버전까지 기본으로 사용되던 방식으로, 단순히 비밀번호에 대한 해 시(SHA-1 알고리즘) 값을 저장해두고, **클라이언트가 보낸 값과 해시값이 일치하는지 비교**하는 인증 방식이다.
  입력이 **동일 해시값**을 출력한다.
  - Caching SHA-2 Pluggable Authenticalion
  MySOL 5.6 버전에 도입되고 MySOL 8.0 버전에서는 조금 더 보완된 인증 방식으로 SHA-2(256비트) 알고리즘을 사용한다. Native Authentication과 의 가장 큰 차이는 사용되는 암호화 알고리즘의 차이이며, 저장된 해시값의 보안에 더 중점을 둔 알고리즘이다. 
  내부적으로 Salt 키를 활용하며, **5000번 이상의 해시 계산**을 수행해서 결과를 만들어 내기 때문에 **동일한 키 값에 대해서도 결과가 달라**진다. 이처럼 해시값을 계산하는 방식은 상당히 시간 소모적이 어서 성능이 매우 떨어지는데, 이를 보완하기 위해 **결과값을 캐시**해서 사용한다. 
  이 인증 방식을 사용하려면 SSL/TLS 또는 RSA 키페어 를 반드시 사용해야 하는데, 이를 위해 클라이언트에서 접속할 때 SSL 옵션을 활성화해야 한다.
  - PAM Pluggable Authentication
  유닉스나 리눅스 패스워드 또는 LDAP(Lightweight Directory Access Protocol) 같은 외부 인증을 사용할 수 있게 해주는 인증 방식으로, MySQL 엔터프라이즈 에디션에서만 사용 가능하다.
  - LDAP Pluggable Authentication
  LDAP을 이용한 외부 인증을 사용할 수 있게 해주는 인증 방식으로, MySQL 엔터프라이즈 에디션에서만 사용 가능하다.
- REQUIRE
MySQL에 접속할 때 **SSL/TLS 채널 사용 여부**. default는 N이다.
따로 설정을 하지 않아도 Caching SHA-2 Pluggable Authenticalion 인증을 사용하면 암호화된 채널만으로 접속해야 한다.
- PASSWORD EXPIRE
**비밀번호의 유효기간**을 설정하는 옵션. default는 default_password_lifetime 변수.
설정 가능한 옵션은 다음과 같다.
  - PASSWORD EXPIRE 계정 생성과 동시에 만료
  - PASSWORD EXPIRE NEVER 만료시간 없음
  - PASSWORD EXPIRE DEFAULT default값 사용.
  - PASSWORD EXPIRE INTERVAL n DAY 오늘부터 n일까지

- PASSWORD HISTORY
한 번 사용한 비밀번호는 **재사용하지 못하도록** 하는 옵션. 저장된 비밀번호는 재사용하지 못한다.(테이블에 저장한다.)
설정 가능한 옵션은 다음과 같다.
  - PASSWORD HISTORY DEFAULT password_history 변수에 저장된 개수만큼 이력 저장.
  - PASSWORD HISTORY n 비밀번호의 이력을 n개까지만 관리한다.

- PASSWORD REUSE INTERVAL
한 번 사용한 비밀번호의 **재사용기간을 설정**하는 옵션. default는 password_reuse_interval변수.
설정 가능한 옵션은 다음과 같다.
  - PASSWORD REUSE INTERVAL DEFAULT default값 사용
  - PASSWORD REUSE INTERVAL n DAY n일 이후에 재사용 가능

- PASSWORD REQUIRE
만료된 비밀번호를 새로 설정할 때 **전 비밀번호를 필요**한지 정하는 옵션 default는 password_require_current 변수.
설정 가능한 옵션은 다음과 같다.
  - PASSWORD REQUIRE CURRENT 전 비밀번호 입력 o
  - PASSWORD REQUIRE OPTIONAL 전 비밀번호 입력 x
  - PASSWORD REQUIRE DEFAULT default값 사용

- ACCOUNT LOCK / UNLOCK
계정 생성 또는 수정시 계정을 잠글지 정하는 옵션.
설정 가능한 옵션은 다음과 같다.
  - ACCOUNT LOCK 잠금
  - ACCOUNT UNLOCK 안잠금


</div>
</details>

### 3.3 비밀번호 관리

비밀번호 유효기간, 재사용 금지, 금칙어, 유효성 체크를 지원한다. 이와 같은 기능을 사용하기 위해선 validate_password 컴포넌트를 설치해야한다.

 

```sql
INSTALL COMPOMENT 'file://component_validate_password';
```

**비밀번호 정책**

- LOW 길이 검증
- MEDIUM 길이 검증, 숫자, 대소문자, 특수문자 배합
- STRONG 길이 검증, 숫자, 대소문자, 특수문자 배합, 금칙어 확인

* 금칙어
보통은 qwer, 1234 같은 간단한 단어가 있지만, validate_password.dictionary_file에 금칙어를 등록할 수 있다.

* validate_password
5.7버전까진 플러그인이었지만, 8.0부터 컴포넌트로 제공된다.
플러그인은 외부 모듈이기 떄문에 서버와 독립적이며, 컴포넌트는 서버 내부에서 작동하는 기능이다. 플러그인은 컴포넌트에 비해 호환성, 안정성, 기능이 떨어진다.

**다중(2개) 비밀번호**
서버를 중지하지 않고 비밀번호를 변경하기 위해서 비밀번호를 동시에 2개 갖도록 할 수 있다.

Primary와 Secondary가 있는데, 새로 설정하게 되면 이전 비밀번호는 Secondary가 되고, Secondary는 사라지게 된다.

사용하는 방법은 비밀번호 변경 구문에 RETAIN CURRENT PASSWORD 옵션을 추가하는 것.

```sql
ALTER USER 'root'@'localhost' IDENTIFY BY 'new_pw' RETAIN CURRENT PASSWORD;
```

### 3.4 권한

MySQL 5.7까지는 글로벌 권한과 객체 권한으로 구분되었다.

여기서 글로벌 권한이란 DB, 테이블 이외에 적용되는 권한을 뜻하며, DB나 테이블에 적용되는 권한은 객체 권한이다.
객체 권한은 GRANT 명령어 뒤에 반드시 객체를 명시해야 하고, 글로벌 권한은 명시하지 않아야 한다.

MySQL 8.0부터는 동적 권한이 추가되었다. 동적 권한은 서버가 시작되며 동적으로 생성하는 권한이다. ex)컴포넌트, 플러그인이 설치될때.

5.7까지는 SUPER라는 권한이 있었지만, 8.0부터는 여러 동적 권한으로 분산시켰다.

권한을 부여하는 명령어는 다음과 같다. 권한은 여러개 명시할 수 있다.

```sql
GRANT 권한 ON 객체 TO 'user'@'localhost';
```

글로벌 권한은 객체를 *.*만 지정할 수 있다.

DB 권한은 *.*과 DB를 지정할 수 있다.

테이블 권한은 *.*과 DB, 테이블 까지 지정할 수 있다.

UPDATE(특정컬럼)을 통해 특정 컬럼에만 권한을 부여할 수 있다. → 이렇게 해도 **전체적인 성능에 영향** 끼침

### 3.5 역할

8.0버전부터 **권한을 묶은 역할**이라는 개념이 생겼다. MySQL 내부적으로는 권한과 같다.

역할에 여러 권한을 부여할 수 있는데, 이는 직접적으로 사용할 수 없고, 계정에 부여해 사용된다.

```sql
CREATE ROLE 권한이름; ## 역할 생성
GRANT SELECT ON 테이블 TO 권한이름; ## 권한 부여
CREATE USER user@'localhost'; ## 계정 생성
GRANT 권한이름 TO user@'localhost'; ## 역할 부여
```

이와같이 역할을 생성하고 권한을 부여하고, 계정에 부여할 수 있다.

하지만 이대로는 권한을 사용할 수 없다. SET ROLE 명령어로 **활성화**를 하지 않았기 때문이다.
**로그아웃 후 로그인해도 역할은 비활성화** 되니 주의하자. (activate_all_roles_on_login=ON;)을 통해 수정 가능

역할을 도입함으로써 여러 사용자가 가진 권한을 **병합해서 제어**가 가능해졌다.

**ROLE과 USER의 IP가 다른 경우는 어떻게 될까?**

```sql
CREATE ROLE role_emp_local_read@localhost;
CREATE USER reader®'127.0.0.1' IDENTIFIED BY 'qwerty';
GRANT SELECT ON employees.* TO role_emp_local_read@'localhost';
GRANT role_emp_local_read@'localhost' TO reader®'127.0.0.1';
```

역할과 계정을 생성하면 사용자 계정은 DB객체들에 대해 SELECT 권한이 부여되기 때문에 다음과 같은 상황에선 역할의 IP는 아무런 영향이 없다.
만약 역할을 부여하지 않고, 로그인용으로 사용하게 된다면 역할의 IP가 중요해진다.

**왜 굳이 CREATE ROLE과 CREATE USER 명령을 구분해서 지원할까?** 
데이터베이스 관리의 직무를 분리할 수 있게 해서 **보안을 강화**하는 용도로 사용될 수 있게 하기 위해서다. CREATE USER 명령에 대해서는 권한이 없지만 CREATE ROLE 명령만 실행 가능한 사용자는 ROLE을 생성할 수 있다. 
이렇게 생성된 역할은 계정과 동일한 객체 를 생성하지만 실제 이 역할은 account_locked 칼럼의 값이 'y'로 설정돼 있어서 로그인 용도로 사용할 수가 없게 된다.

### ROLE의 장단점

**장점**

- 보안 관리 용이 - 사용자 그룹에 대한 권한을 한 곳에서 관리할 수 있도록 도와준다.
- 권한 관리 간편화 - 권한을 부여하고 취소하는 작업이 간소화된다.
- 중복 제거 - 사용자 그룹에 대한 권한을 관리하면 중복된 권한 설정을 피할 수 있다.

**단점**

- 복잡성 - 계정과 역할간의 관계를 설정하고 유지하기 위해 추가적인 설정 및 관리 작업이 필요하다. 데이터베이스 구조가 더 복잡해질 수 있고, 초기 설정 및 유지 관리에 노력이 필요할 수 있다.
- 제한된 유연성 - 그룹 단위로 권한을 관리할 수 있지만, 특정 권한 조정이 필요한 경우에는 유연성이 제한된다.
- 호환성 문제 - 8.0 버전 이전과 호환성 문제가 발생한다.
