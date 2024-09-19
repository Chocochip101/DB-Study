MySQL에서는 날짜만 저장하거나 시간만 따로 저장할 수도 있으며, 날짜와 시간을 합쳐서 하나의  럼에 저장할 수 있게 여러 가지 타입을 지원한다. 다음 표는 MySQL에서 지원하는 날짜나 시간에 관련 된 데이터 타입으로, DATE와 DATETIME 타입이 많이 사용된다. 

MySQL 5.6 버전부터 TIME 타입과 DATETIME, TIMESTAMP 타입은 밀리초 단위의 데이터를 저장할 수 있게 됐다. 그래서 칼럼의 저장 공간 크기는 밀리초 단위를 몇 자리까지 저장하느냐에 따라 달라진다.

|데이터 타입|MySQL 5.6.4 이전|MySQL 5.6.4부터|
|---|---|---|
|YEAR|1바이트|3바이트|
|DATE|3바이트|3바이트|
|TIME|3바이트|3바이트 + (밀리초 단위 저장 공간)|
|DATETIME|8바이트|5바이트 + (밀리초 단위 저장 공간)|
|TIMESTAMP|4바이트|4바이트 + (밀리초 단위 저장 공간)|


다음과 같이 밀리초 단위는 2자리당 1바이트씩 공간이 더 필요하다. 그래서 MySQL 8.0에서는 마이크 로초까지 저장 가능한 DATETIME(6) 타입은 8바이트(5바이트+3바이트)를 사용한다. 


- 없음: 0바이트
- 1, 2 자릿수: 1바이트
- 3, 4 자릿수: 2바이트 
- 5, 6 자릿수: 3바이트  

밀리초 단위로 데이터를 저장하기 위해서는 다음과 같이 DATETIME이나 TIME, TIMESTAMP 타입 뒤에 괄 호와 함께 숫자를 표기하면 된다. NOW() 함수를 이용해 현재 시간을 가져올 때도 NOW(6) 또는 NOW(3)과 같이 가져올 밀리초의 자릿수를 명시해야 한다. 그렇지 않고 NOW()로 현재 시간을 가져오면 자동으로 NOW(0) 으로 실행되어 밀리초 단위는 0으로 반환된다.

MySQL의 날짜 타입은 칼럼 자체에 타임존 정보가 저장되지 않으므로 DATETIME이나 DATE 타입은 현재 DBMS 커넥션의 타임존과 관계없이 클라이언트로부터 입력된 값을 그대로 저장하고 조회할 때도 변환 없이 그대로 출력한다. 하지만 TIMESTAMP는 항상 UTC 타임존으로 저장되므로 타임존이 달라져도 값이 자동으로 보정된다. 다음 예제는 한국에 있는 사용자가 DATETIME 타입과 TIMESTAMP 타입에 저장한 날짜 값을 미국의 로스앤젤레스의 사용자가 조회하는 과정을 보여준다. 여기서는 각 사용자의 위치(타임존) 를 설정하기 위해 "SET time_zone=, 명령을 사용한다.

```sql
mysql> CREATE TABLE tb_timezone (fd_datetime DATETIME, fd_timestamp TIMESTAMP); 
--// 현재 세션의 타임존을 한국(Asia/Seoul)으로 변경 
mysql> SET time_zone= Asia/Seoul';
mysql> INSERT INTO tb_timezone VALUES (NOW(), NOW()); 
-- // 저장된 시간 정보를 확인
```
위 예제에서 TIMESTAMP 칼럼의 값은 현재 클라이언트(커넥션)의 타임존에 맞게 변환됐지만 DATETIME에 저장된 날짜와 시간 정보는 커넥션의 타임존이 한국에서 미국의 로스앤젤레스로 변경돼도 전혀 차이가 없이 똑같은 값이 조회된다. 이는 DATETIME 칼럼은 타임존에 대해 아무런 타임존 변환 처리가 수행되지 않음을 의미한다. 자바 응용 프로그램의 타임존을 한국 서울로 한 경우와 미국 로스앤젤레스로 변경했을 때 MySQL 서 버의 데이터가 어떻게 변환되는지 간단히 확인해보자. 한국 서울로 JVM의 타임존이 설정된 경우 

```java
// JVM 타임존을 서울로 변경 
System.setProperty ("user. timezone", "Asia/Seoul"); Connection conn = DriverManager.getConnection("jdbc:mysql://1:27.0.0.1:33067server1imezone=Asia/Seoul" "id", "pass"); 
Statement stmt = conn.createStatement(); 
ResultSet res = stmt.executeQuery("select fd_datetime, fd_timestamp from test.tb_timezone"); 
if(res.next())( 
    System.out.println("fd_datetime : + res.getTimestamp("fd_datetime")); 
    System.out.print.ln("fd_timestamp - + res.getTimestamp("fd_timestamp"); 
} 
// 결과 화면 출력 fd_datetime : 2020-09-10 09:25:23.0
fd_timestamp: 2020-09-10 09:25:23.0 
미국 로스앤젤레스로 JVM의 타임존이 설정된 경우 

// JVM 타임존을 로스앤젤레스로 변경 
System.setProperty("user. timezone", "America/Los_Angeles"); 
Connection conn = DriverManager.getConnection("jdbc:mysql://127.0.0.1:3306?serverTimezone=Asia/Seoul", "id", "pass"); 
Statement stmt = conn.createStatement(); 
ResultSet res = stmt.executeQuery ("select fd_datetime, fd_timestamp from test.tb_timezone"); 
if(res.next()){ 
    System.out.println("fd_datetime - + res.getTimestamp("fd_datetime"));
    System.out.print.ln("fd_timestamp : · + res.getTimestamp("fd_timestamp");
}

결과 화면 출력 fd_datetime : 2020-09-09 17:25:23.0 fd_timestamp : 2020-09-09 17:25:23.0 

```
MySQL 서버의 칼럼 타입이 TIMESTAMP이든 DATETIME이든 관계없이, JDBC 드라이버는 날짜 및 시 간 정보를 MySQL 타임존에서 JVM의 타임존으로 변환해서 출력하는 것을 확인할 수 있다. 사실 자 바의 ResultSet 클래스에서 MySQL 서버의 DATETIME 칼럼의 값을 온전히 가져올 수 있는 함수가 getTimestamp()뿐이기 때문에 DATETIME이나 TIMESTAMP 타입 칼럼 모두 타임존 변환이 된 것이기도 하다. 그런데 DATETIME 타입 칼럼의 값을 다른 방식으로 가져온다면 타임존 변환이 되지 않을 수도 있다. 단순 히 조회뿐만 아니라 응용 프로그램에서 데이터베이스로 날짜 및 시간 데이터를 저장할 때도 동일한 규 칙이 적용된다. 요즘은 Hibernate 나 MyBatis 등과 같은 ORM(Object-Relational Mapping)을 사용하기 때문에 ORM 코드 내부적으로 값을 자동으로 페치해서 응용 프로그램 코드로 반환한다. 이렇게 ORM을 사용 하는 경우에는 DATETIME 타입의 칼럼값을 어떤 JDBC API를 이용해서 폐치하는지, 그리고 타임존 변환 이 기대하는 대로 자동 변환하는지를 실제 응용 프로그램으로 테스트해 볼 것을 권장한다. 최대한 응용 프로그램에서 시간 정보를 강제로 타임존 변환을 하거나 MySQL 서버의 SQL 문장으로 CONVERT_TZ() 함수를 이용해 타임존 변환을 하지 않도록 하자. 타임존 관련 설정은 한 번 문제가 되기 시작하면 해결 하기가 매우 어려운 문제가 될 수도 있다. 참고 위의 두 예제 코드에서 System.setProperty()를 이용해 JVM의 타임존만 변경했다. 두 예제 코드 모두 JDBC 연결 문자열에서 serverTimezone은 "Asia/Seoul' "을 설정하고 있는데, 이는 자바 응용 프로그램이 아니라 MySQL 서버의 타임존을 지정하는 것이다. 즉, JDBC 드라이버에게 MySQL 서버의 타임존이 "Asia/Seoul"로 설정돼 있다는 것을 알려주는 역할이다. 위의 두 예제 코드의 JDBC 연결 문자열에서 "serverTimezone" 파라미터는 굳이 설정하지 않아도 JDBC 드라이버 가 자동으로 MySQL 서버의 타임존을 인식하게 된다. MySQL 서버의 타임존을 자동으로 인식하지 못하는 경우 사용 자는 JDBC 연결 문자열에 "serverTimezone" 파라미터를 설정해서 MySQL 서버의 타임존을 JDBC 드라이버에게 알려줄 수 있다. 여기 예제에서는 명시적으로 보여주기 위해 "serverTimezone" 파라미터를 설정해둔 것이다. 

이미 데이터를 가지고 있는 MySQL 서버의 타임존(system_ time_zone 변수)을 변경해야 한다면 타임존 설정뿐만 아니라 테이블의 DATETIME 타입의 칼럼이 가지고 있는 값도 CONVERT_TZ() 같은 함수를 이용해 변환해야 한다. 하지만 TIMESTAMP 타입의 값은 MySQL 서버의 타임존에 의존적이지 않고 항상 UTC로 저장되므로 MySQL 서버의 타임존을 변경한다고 해서 별도로 변환 작업이 필요하지는 않다. MySQL 서버의 기본 타임존을 확인하거나 변경하는 방법은 다음과 같다.

```sql
mysql> SET time_zone= . America/Los_Angeles'; mysq1> SHOW VARIABLES LIKE '동time_zone%'; Variable_name Value system_ time... zone KST time_zone America/Los_Angeles 
```
system_time_zone 시스템 변수는 MySQL 서버의 타임존을 의미하며, 일반적으로 이 값은 운영체제의 타임존을 그대로 상속받는다. 시스템 타임존은 MySQL을 기동하는 운영체제 계정의 환경 변수(일반 적으로 운영체제 계정의 타임존 환경 변수의 이름은 "TZ"다)를 변경하거나 mysqld_safe를 시작할 때 "--timezone" 옵션을 이용해 변경할 수 있다. time_zone 시스템 변수는 MySQL 서버로 연결하는 모든 클라이언트 커넥션의 기본 타임존을 의미한다. 위 예제에서는 time_zone 변수가 America/Los_Angeles' 이기 때문에 이 서버에 접속하는 클라이언트 커넥션은 America/Los_Angeles' 타임존을 초깃값으로 가 지게 된다. 물론 커넥션의 타임존은 응용 프로그램 코드에 의해 다른 값으로 언제든지 변경될 수 있다. time_zone 시스템 변수에 아무것도 설정하지 않으면 time_zone 시스템 변수는 "SYSTEM" 으로 자동 설정되 는데, 이는 time_zone 시스템 변수가 system_time_zone 시스템 변수의 값을 그대로 사용한다는 의미다. system_time_zone과 time_zone 시스템 변수는 MySQL 서버를 시작할 때 --timezone과 --default-time_ zone 명령행 옵션으로 변경할 수 있다. 두 시스템 변수의 이름을 보면 system_time_zone이 더 중요하고 큰 영향을 미칠 것처럼 보이지만 실제 MySQL 서버에 접속된 커넥션에서 시간 관련 처리(NOW() 함수나 TIMESTAMP 칼럼 초기화 등)를 할 때는 time_zone 시스템 변수의 영향만 받는다. time_zone 시스템 변수의 값이 SYSTEM으로 설정돼 있다면 system_time_zone의 영향을 받게 된다. 

## 자동 업데이트 
MySQL 5.6 이전 버전까지는 TIMESTAMP 타입의 칼럼은 레코드의 다른 칼럼 데이터가 변경될 때마다 시 간이 자동 업데이트되고, DATETIME은 그렇지 않은 차이를 가지고 있었다. 하지만 MySQL 5.6 버전부터 는 TIMESTAMP와 DATETIME 칼럼 모두 INSERT와 UPDATE 문장이 실행될 때마다 해당 시점으로 자동 업데이트 되게 하려면 테이블을 생성할 때 칼럼 정의 뒤에 다음 옵션을 정의해야 한다. 다음 예제에서 TIMESTAMP 와 DATETIME 타입 각각에 대해 비교해보기 위해 데이터 타입별로 created_at과 updated_at 칼럼을 중복 으로 추가했다. mysq1> CREATE TABLE tb_autoupdate ( id BIGINT NOT NULL AUTO_INCREMENT, title VARCHAR(20), created_at_ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP, updated_at_ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP, created_at_dt DATETIME DEFAULT CURRENT_TIMESTAMP updated_at_dt DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP, PRIMARY KEY (id) ); "DEFAULT CURRENT_TIMESTAMP" 옵션은 레코드가 INSERT될 때의 시점을 자동으로 업데이트하며, "ON UPDATE CURRENT_TIMESTAMP" 옵션은 해당 레코드가 UPDATE될 때의 시점을 자동으로 업데이트하게 해준다.


UPDATE 시점으로 변경됨 위의 테스트 결과를 보면 TIMESTAMP와 DATETIME 타입의 칼럼 모두 동일하게 INSERT 시점으로 초기화되 고, UPDATE 문장으로 테이블의 title 칼럼의 값만 변경했는데 "ON UPDATE CURRENT_TIMESTAMP" 옵션을 가 진 칼럼 2개에 대해서는 타입과 관계없이 자동으로 UPDATE 시점으로 변경된 것을 확인할 수 있다. MySQL 서버를 처음으로 접하는 사용자는 "당연한 거 아닌가?"라고 생각할 수도 있다. 하지만 예 전 버전의 MySQL 서버에서는 TIMESTAMP만 이러한 자동 업데이트가 가능했으며, 테이블의 첫 번째 TIMESTAMP 타입만 자동 업데이트가 되는 등의 복잡한 규칙이 있었는데, 이 같은 규칙들이 모두 사라진 것이다. MySQL 5.6 버전부터는 DATETIME 타입과 TIMESTAMP 타입 사이에 커넥션의 time_zone 시스템 변 수의 타임존으로 저장할지, UTC로 저장할지의 차이만 남고 모든 것이 같아졌다.