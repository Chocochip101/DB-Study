# 쿼리 작성 및 최적화

## 스키마 조작(DDL)

DBMS 서버의 모든 오브젝트를 생성하거나 변경하는 쿼리를 DDL(Data Definition Language)이라고 한다.  
스토어드 프로시저나 함수, DB, 테이블 등을 생성하거나 변경하는 대부분의 명령이 DDL에 해당한다.  
MySQL 서버가 업그레이드되면서 많은 DDL이 온라인 모드로 처리될 수 있게 개선됐지만, 여전히 스키마를 변경하는 작업 중에는
상당히 오랜 시간이 걸리고 MySQL 서버에 많은 부하를 발생시킨다.

### 온라인 DDL

5.5 이전 버전까지는 MySQL 서버에서 테이블의 구조를 변경하는 동안 다른 커넥션에서 DML을 실행할 수가 없었다.  
이 같은 문제점을 해결하기 위해 Percona에서 개발한 pt-online-schema-change라는 도구를 사용했다.  
하지만 8.0 버전으로 업그레이드되면서 대부분의 스키마 변경 작업은 MySQL 서버에 내장된 온라인 DDL 기능으로 처리가 가능해졌다.
  
온라인 DDL은 스키마를 변경하는 작업 도중에도 다른 커넥션에서 DML을 실행할 수 있게 해준다.  
테이블의 구조를 변경하거나 인덱스 추가와 같은 대부분의 작업에 대해서도 작동한다.
  
온라인 DDL은 `ALGORITHM`과 `LOCK` 옵션을 이용해 어떤 모드로 스키마 변경을 실행할지 결정할 수 있다.  
MySQL 서버에서는 `old_alter_table` 시스템 변수를 통해 ALTER TABLE 명령이 온라인 DDL로 작동할 지 아니면 예전 방식으로 처리할 지를 결정할 수 있다.
ALTER TABLE 명령을 실행하면 MySQL 서버는 다음과 같은 순서로 스키마 변경에 적합한 알고리즘을 찾는다.

- `ALGORITHM=INSTANT`로 스키마 변경이 가능한지 확인하고 가능하면 선택
    - 테이블의 데이터는 전혀 변경하지 않고, 메타데이터만 변경하고 작업을 완료. 테이블이 가진 레코드 건수와 무관하게 작업 시간이 매우 짧다.
    - 스키마 변경 도중 테이블의 읽고 쓰기는 대기하게 되지만 스키마 변경 시간이 매우 짧아서 다른 커넥션에 크게 영향을 미치지 않는다.
- `ALGORITHM=INPLACE`로 스키마 변경이 가능한지 확인하고 가능하면 선택
    - 임시 테이블로 데이터를 복사하지 않고 스키마 변경을 실행. 하지만 내부적으로 테이블의 리빌드를 실행할 수도 있다. 테이블의 모든 레코드를 리빌드해야 하기 때문에 테이블의 크기에 따라 많은 시간이 소요될 수 있다.
    - 스키마 변경 중에도 테이블의 읽기 쓰기 모두 가능하다. 그런데 최초 시작 시점과 마지막 종료 시점에는 테이블의 읽고 쓰기가 불가능한데 이 시간은 매우 짧아서 다른 커넥션에 대한 영향도는 높지 않다.
- `COPY`
    - 변경된 스키마를 적용한 임시 테이블을 생성하고, 테이블의 레코드를 모두 임시 테이블로 복사한 후 최종적으로 임시 테이블을 RENAME해서 스키마 변경을 완료
    - 이 방법은 테이블 읽기만 가능하고 DML은 실행할 수 없다.

```SQL
mysql> ALTER TABLE salaries CHANGE to_date end_date DATE NOT NULL,
ALGORITHM=INPLACE, LOCK=NONE;
```
 
온라인 DDL은 알고리즘과 함께 잠금 수준도 명시할 수 있는데 명시되지 않으면 MySQL 서버가 적절하게 알고리즘과 잠금 수준을 선택한다.
그런데 알고리즘이 INSTANT일 경우 매우 짧은 시간 동안만의 메타데이터 잠금만 필요로 하기 때문에 LOCK 옵션은 명시할 수 없다.
INPLACE나 COPY일 경우 LOCK은 다음 3가지 중 하나를 명시할 수 있다.

- NONE: 아무런 잠금을 걸지 않음
- SHARED: 읽기 잠금을 걸고 스키마 변경을 실행. 스키마 변경 중 읽기는 가능하지만 쓰기는 불가능
- EXCLUSIVE: 쓰기 잠금을 걸고 스키마 변경을 실행. 읽기 쓰기 모두 불가능

INPLACE 알고리즘을 사용하더라도 내부적으로는 테이블의 리빌드가 필요할 수도 있다. 대표적으로 테이블의 PK를 추가하는 작업은
데이터 파일에서 레코드의 저장 위치가 바뀌어야 하기 때문에 테이블 리빌드가 필요한 반면, 단순히 칼럼의 이름만 변경하는 경우 INPLACE 
알고리즘을 사용하긴 하지만 실제 테이블 레코드 리빌드 작업이 필요하지 않다. 

- 데이터 재구성(테이블 리빌드)이 필요한 경우: 잠금을 필요로 하지 않기 때문에 읽고 쓰기는 가능하지만 여전히 테이블의 레코드 건수에 따라 상당히 많은 시간이 소요될 수도 있다.
- 데이터 재구성(테이블 리빌드)이 필요치 않은 경우: INPLACE 알고리즘을 사용하지만 INSTANT 알고리즘과 비슷하게 매우 빨리 작업이 완료될 수 있다.

### 온라인 DDL 진행 상황 모니터링

온라인 DDL을 포함한 모든 ALTER TABLE 명령은 MySQL 서버의 performance_schema를 통해 진행 상황을 모니터링할 수 있다.

```sql
-- // performance_schema 시스템 변수 활성화
mysql> SET GLOBAL performance_schema=ON;

-- // "stage/innodb/alter%" instrument 활성화
mysql> UPDATE performance_schema.setup_instruments
          SET ENABLED = 'YES', TIMED = 'YES'
        WHERE NAME LIKE 'stage/innodb/alter%';

-- // "%stages%" consumer 활성화
mysql> UPDATE performance_schema.setup_consumers
          SET ENABLED = 'YES'
        WHERE NAME LIKE '%stages%';
```

스키마 변경 작업 진행 상황은 performance_schema.events_stages_current 테이블을 통해 확인할 수 있는데, 실행 중인 스키마 변경 종류에 따라
기록되는 내용이 조금씩 달라진다.

## 데이터베이스 변경

MySQL에서 하나의 인스턴스는 1개 이상의 데이터베이스를 가질 수 있다. 다른 RDBMS에서는 스키마와 데이터베이스를 구분해서 관리하지만
MySQL 서버에서는 스키마와 데이터베이스는 같은 개념이다. 그래서 MySQL 서버에서는 굳이 스키마를 명시적으로 사용하지는 않는다.
MySQL 서버의 데이터베이스는 디스크의 물리적인 저장소를 구분하기도 하지만 여러 데이터베이스의 테이블을 묶어서 조인 쿼리를 사용할 수도 있기 때문에
단순히 논리적인 개념이기도 하다.

### 데이터베이스 생성
```SQL
mysql> CREATE DATABASE [IF NOT EXISTS] employees;
```

### 데이터베이스 목록
```SQL
mysql> SHOW DATABASES;
mysql> SHOW DATABASES LIKE '%emp%';
```

### 데이터베이스 선택
```SQL
mysql> USE employees;
```

### 데이터베이스 속성 변경
```SQL
mysql> ALTER DATABASE employees CHARACTER SET=euckr;
```

### 데이터베이스 삭제
```SQL
mysql> DROP DATABASE [IF EXISTS] employees;
```

## 테이블 스페이스 변경

MySQL 서버에는 전통적으로 테이블별로 전용의 테이블스페이스를 사용했었다. InnoDB 스토리지 엔진의 시스템 테이블 스페이스만 제너럴 테이블스페이스를
사용했는데, 제너럴 테이블스페이스는 여러 테이블의 데이터를 한꺼번에 저장하는 테이블스페이스를 의미한다.
  
MySQL 8.0 버전이 되면서 MySQL 서버에서도 사용자 테이블을 제너럴 테이블스페이스로 저장하는 기능이 추가되고 테이블스페이스를 관리하는 DDL 명령들이 추가됐다.

## 테이블 변경

테이블은 사용자의 데이터를 가지는 주체로서, MySQL 서버의 많은 옵션과 인덱스 등의 기능이 테이블에 종속되어 사용된다.

### 테이블 생성

```SQL
CREATE [TEMPORARY] TABLE [IF NOT EXISTS] tb_test (
...
) ENGINE=INNODB;
```

### 테이블 구조 조회

```SQL
mysql> SHOW CREATE TABLE employees \G
```

SHOW CREATE TABLE 명령은 칼럼의 목록과 인덱스, 외래키 정보를 동시에 보여주기 때문에 SQL을 튜닝하거나 테이블의 구조를 확인할 때
주로 이 명령을 사용한다.

```SQL
mysql> DESC employees;
```

DESC 명령은 DESCRIPBE 약어 형태의 명령으로 테이블의 칼럼 정보를 보기 편한 표 형태로 표시해준다.  
하지만 인덱스 칼럼의 순서나 외래키, 테이블 자체의 속성을 보여주지는 않으므로 테이블의 전체 구조를 한 번에 확인하기는 어렵다

### 테이블 구조 변경

```SQL
mysql> ALTER TABLE employees
         CONVERT TO CHARACTER SET UTF8MB4 COLLATE UTF8MB4_GENERAL_CI,
         ALGORITHM=INPLACE, LOCK=NONE;
```

```SQL
mysql> ALTER TABLE employees ENGINE=InnoDB,
       ALGORITHM=INPLACE, LOCK=NONE;
```
위 쿼리는 테이블 스토리지 엔진을 변경하는 명령이다. 위 명령은 내부적인 테이블의 저장소를 변경하는 것이라서 항상 테이블의 모든 레코드를 복사하는 작업이 필요하다.
ALTER TABLE 문장에 명시된 엔진이 기존과 동일하더라도 테이블의 데이터를 복사하는 작업은 실행되기 때문에 주의해야 한다.
  
그런데 이 명령은 실제 테이블 스토리지 엔진을 변경하는 목적으로도 사용하지만 `테이블 데이터를 리빌드`하는 목적으로도 사용한다.
테이블 리빌드 작업은 주로 레코드의 삭제가 자주 발생하는 테이블에서 프래그멘테이션을 제거해 디스크 사용 공간을 줄이는 역할을 한다.

> 테이블이 사용하는 디스크 공간의 프래그멘테이션을 최소화하고, 테이블 구조를 최적화하기 위한 OPTIMIZE TABLE이라는 명령이 있다.
> OPTIMIZE TABLE 명령을 사용하면 스토리지 엔진은 내부적으로 위 쿼리와 동일한 작업을 수행한다.
> 결국 InnoDB 테이블에서 테이블 최적화란 테이블의 레코드를 한 건씩 새로운 테이블에 복사함으로써 테이블 레코드의 배치를 컴팩트하게 만들어주는 것이다.

### 테이블 명 변경

```SQL
mysql> RENAME TABLE table1 TO table2;
mysql> RENAME TABLE batch TO batch_old, batch_new TO batch;
```

### 테이블 상태 조회

```SQL
mysql> SHOW TABLE STATUS LIKE 'employees' \G
```

### 테이블 삭제

```SQL
mysql> DROP TABLE [IF EXISTS] table1;
```

용량이 매우 큰 테이블을 삭제하는 작업은 상당히 부하가 큰 작업에 속한다. 데이터 파일을 삭제하는 작업이 매우 많고
디스크 파일 조각들이 너무 분산되어 저장돼 있다면 많은 디스크 읽기 쓰기 작업이 필요하다. MySQL 서버의 디스크 읽기 쓰기 부하가 높아지면
다른 커넥션의 쿼리 처리 성능이 떨어질 수 있다.
  
또 한 가지는 어댑티브 해시 인덱스다. 어댑티브 해시 인덱스가 활성화돼 있는 경우 테이블이 삭제되면 어댑티브 해시 인덱스 정보도 모두 삭제해야 한다.
어댑티브 해시 인덱스가 삭제될 테이블에 대한 정보를 많이 가지고 있다면 어댑티브 해시 인덱스 삭제 작업으로 인해 MySQL 서버 부하가 높아지고 간접적으로
다른 쿼리 처리에 영향을 미칠 수도 있다.

## 칼럼 변경

### 칼럼 추가

MySQL 8.0 버전으로 업그레이드되면서 테이블의 칼럼 추가 작업은 대부분 INPLACE 알고리즘을 사용하는 온라인 DDL로 처리가 가능하다.
그뿐만 아니라 칼럼을 테이블의 제일 마지막 칼럼으로 추가하는 경우에는 INSTANT 알고리즘으로 즉시 추가된다.

```SQL
mysql> ALTER TABLE employees ADD COLUMN emp_telno VARCHAR(20), ALGORITHM=INSTANT;
mysql> ALTER TABLE employees ADD COLUMN emp_telno VARCHAR(20), AFTER emp_no, ALGORITHM=INPLACE, LOCK=NONE;
```

스키마 변경 작업의 종류별로 어떤 알고리즘이 가능한지를 기억하기는 쉽지 않다. 그래서 항상 ALTER TABLE 명령에는 ALGORITHM과
LOCK 절을 추가해서 원하는 성능과 잠금 레벨로 스키마 변경이 가능한지 차례대로 다음과 같이 확인해가면서 스키마 변경을 해보는 것이 좋다.

### 칼럼 삭제

칼럼 삭제는 항상 테이블 리빌드가 필요하기 때문에 INPLCAE로만 가능하다.

```SQL
mysql> ALTER TABLE employees DROP COLUMN emp_telno< ALGORITHM=INPLACE, LOCK=NONE;
```

### 칼럼 이름 및 칼럼 타입 변경

```SQL
-- // 칼럼 이름 변경
mysql> ALTER TABLE salaries CHANGE to_date end_date DATE NOT NULL, ALGORITHM=INPLACE, LOCK=NONE;

-- // INT 칼럼을 VARCHAR 타입으로 변경
mysql> ALTER TABLE salaries MODIFY salary VARCHAR(20), ALGORITHM=COPY, LOCK=SHARED;

-- // VARCHAR 타입 길이 확장
mysql> ALTER TABLE employees MODIFY last_name VARCHAR(30) NOT NULL, ALGORITHM=INPLACE, LOCK=NONE;

-- // VARCHAR 타입 길이 축소
mysql> ALTER TABLE employees MODIFY last_name varchar(10) NOT NULL, ALGORITHM=COPY, LOCK=SHARED;
```

칼럼 타입을 변경하는 경우 가장 빈번한 경우가 VARCHAR나 VARBINARY 타입의 길이를 확장하는 것이다.  
VARCHAR나 VARBINARY 타입의 경우 칼럼의 최대 허용 사이즈는 메타 데이터에 저장되지만, 실제 칼럼이 가지는 값의 길이는 데이터 레코드의 칼럼 헤더에 저장된다.
  
값의 길이를 위해서 사용하는 공간의 크기는 VARCHAR 칼럼이 최대 가질 수 있는 바이트 수만큼 필요하다. 즉 칼럼 길이 저장용 공간은 최대 가질 수 있는 바이트 수가
255 이하인 경우 1바이트만 사용하는데 256바이트 이상인 경우 2바이트를 사용한다. 그래서 동일한 값이라고 하더라도 VARCHAR(10) 칼럼에 저장될 때보다
VARCHAR(1000) 칼럼에 저장될때는 1바이트를 더 사용한다.
  
UTF8MB4 문자 셋을 사용하는 경우 VARCHAR(10) 에서 VARCHAR(64)로 변경하는 경우 테이블 리빌드가 필요하다. UTF8MB4 문자 셋에서 VARCHAR(64)는 최대 256 바이트를 사용하는데
칼럼값의 길이를 1바이트에서 2바이트로 변경해야 하므로 테이블의 레코드 전체를 다시 리빌드해야 한다.


## 인덱스 변경

### 인덱스 추가

```SQL
mysql> ALTER TABLE employees ADD PRIMARY KEY (emp_no), ALGORITHM=INPLACE, LOCK=NONE;
mysql> ALTER TABLE employees ADD INDEX ix_lastname (last_name), ALGORITHM=INPLCAE, LOCK=NONE;
```

### 인덱스 조회

```SQL
mysql> SHOW INDEX FROM employees;
```
또는 테이블 생성 구문을 통해 확인
```SQL
mysql> SHOW CREATE TABLE employees;
```

### 인덱스 이름 변경

```SQL
mysql> ALTER TABLE salaries RENAME INDEX ix_salary TO ix_salary2, ALGORITHM=INPLACE, LOCK=NONE;
```

인덱스 이름을 변경하는 작업은 INPLACE 알고리즘을 사용하지만 실제 테이블 리빌드를 하지는 않는다. 그래서 응용 프로그램에서
힌트로 해당 인덱스의 이름을 사용 중이라고 하더라도 빠른 시간에 인덱스를 교체할 수 있다.

### 인덱스 가시성 변경

MySQL 에서 인덱스를 삭제하는 작업은 ALTER TABLE DROP INDEX 명령으로 즉시 완료되지만 한 번 삭제된 인덱스를 새로 생성하는 것은 매우 많은 시간이 걸릴 수 있다.
  
8.0 버전부터 인덱스의 가시성을 제어할 수 있는 기능이 도입됐다. 인덱스의 가시성이란 MySQL 서버가 쿼리를 실행할 때 해당 인덱스를 사용할 수 있게 할지 말지를 결정하는 것이다.

```SQL
mysql> ALTER TABLE employees ALTER INDEX ix_firstname INVISIBLE;
```

인덱스가 INVISIBLE 상태로 변경되면 MySQL 옵티마이저는 INVISIBLE 상태의 인덱스는 없는 것으로 간주하고 실행 계획을 수립한다.
INVISIBLE 상태의 인덱스를 다시 사용할 수 있게 하려면 VISIBLE 옵션을 명시하면 된다.

```SQL
mysql> ALTER TABLE employees ALTER INDEX ix_firstname VISIBLE;
```

8.0 버전부터 가시성 변경으로 먼저 해당 인덱스를 보이지 않게 변경해서 상황을 모니터링한 후 안전하게 인덱스를 삭제할 수 있게 됐다.

```SQL
mysql ALTER TABLE employees ADD INDEX ix_firstname_lastname (first_name, last_name) INVISIBLE;
```

그뿐만 아니라 최초 인덱스를 생성할 때도 가시성을 설정할 수 있다.

## Reference 

**위 글은 책 RealMySQL 8.0 2권을 구입하여 읽고 정리한 내용입니다.**
- [도서 홈페이지 https://wikibook.co.kr/realmysql802/](https://wikibook.co.kr/realmysql802/)
